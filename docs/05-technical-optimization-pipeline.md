# 05 — Technical Optimisation Pipeline

**Engine:** `STRATA` — proprietary, no UE5/RAGE/Decima code dependency. Prior art is cited where
techniques are adapted, because pretending otherwise wastes engineering time rediscovering known
limits.

**Reference platform:** PlayStation 5 (base) and PC-Mid (12 GB VRAM, 32 GB RAM, RTX 4070 / RX 7800
XT class). All budgets in this document are stated against the reference platform unless marked.

---

## 5.1 Engine layering

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ L6  GAME          MERIDIAN gameplay, missions, narrative, UI                  │  script (VM)
├──────────────────────────────────────────────────────────────────────────────┤
│ L5  SYSTEMS       weather · hydro · ecology · destruction · vehicle ·          │  C++, ECS
│                   ballistics · acoustics · traffic · residents · dispatch ·    │
│                   economy · perception · embodiment                            │
├──────────────────────────────────────────────────────────────────────────────┤
│ L4  WORLD         World Partition · streaming scheduler · interior bundles ·   │  C++, ECS
│                   mutable state / delta journal · navmesh · coverage volumes   │
├──────────────────────────────────────────────────────────────────────────────┤
│ L3  RENDER        virtualized geometry (SVG) · terrain/VT · GI · shadows ·     │  C++ + GPU
│                   atmosphere · water · post/TSR · visibility buffer            │
├──────────────────────────────────────────────────────────────────────────────┤
│ L2  RUNTIME       ECS registry · job graph · allocators · handles · PRNG ·     │  C++
│                   state hash · hot reload                                      │
├──────────────────────────────────────────────────────────────────────────────┤
│ L1  PLATFORM      RHI (D3D12/Vulkan/PS5 GNM-equivalent/MS GDK) · I/O           │  C++ / asm
│                   (DirectStorage / PS5 IO) · audio device · input · memory     │
├──────────────────────────────────────────────────────────────────────────────┤
│ L0  OS / HW       Win32 · Orbis · GDK · Linux (dedicated sim backend)          │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Layering rules (`MUST`):**
- L5 may not call L1. Every platform access goes through L2/L3/L4.
- L5 systems may not read each other's private state. They communicate through **published
  fields** on shared components (§5.2.4) or through an **event bus** (§5.2.6). This is what makes
  the Material Ontology (L5, §3.1) a single source of truth rather than a set of copies.
- L4 owns *where things are and what is resident*. L5 owns *what things do*. L3 owns *how things
  look*. A violation of this boundary is the root cause of most open-world perf disasters (e.g. a
  gameplay system deciding streaming priority).
- No layer may allocate from the heap during a frame (§5.2.5).

## 5.2 Core data architecture

### 5.2.1 ECS

**Archetype-based, struct-of-arrays, chunked.** Not a sparse-set ECS, not an object hierarchy.

```
· Entity = a 64-bit handle: {index: 40 bits, generation: 16 bits, archetype_hint: 8 bits}
  Generation catches use-after-destroy. archetype_hint allows a branchless first guess.
· Archetype = a unique, sorted set of component types. ~2,400 archetypes at ship.
· Chunk = 16 KB of contiguous component data for one archetype. Chunk capacity is chosen so
  that 16 KB / (sum of hot component sizes) rows fit — typically 64–256 entities per chunk.
  16 KB is chosen to be ≤ L1 (32 KB on Zen 2/3, shared 2-way) and ≥ one cache-line-multiple of
  every component, so a chunk iteration never straddles a page.
· Hot/cold split: each component is authored as `X` (hot, in the chunk) and `XData` (cold, in a
  side arena indexed by the same slot). The ResidentRecord (§3.9.2) is the extreme case: 312 hot
  bytes, 1.9 KB cold.
· Iteration = for each matching archetype (sorted by a stable archetype ID), for each chunk
  (sorted by chunk ID), for each row (index order). FULLY DETERMINISTIC. No pointer chasing,
  no hash maps, no virtual calls.
```

### 5.2.2 Why archetype-based

The simulation's hot loops are narrow and deep: "advance all T2 agents", "solve all vehicle
suspensions", "tick all hydrology cells". An archetype layout makes each of those a linear SoA
pass over 16 KB chunks with 100% L1 hit rate and full SIMD width. A sparse-set ECS would be
better for query flexibility and worse for these passes; a plain object model would be worse for
both. The decision is forced by §5.11's budgets, not by taste.

### 5.2.3 Memory layout for the big systems

| System | Layout | Rationale |
|---|---|---|
| Resident records (§3.9) | Two arrays: `active` (397k × 312 B, heap-ordered by `next_transition`) and `dormant` (3.73M × 40 B, sorted by `next_transition`). Cold data in a memory-mapped file. | The dormant array is scanned once per second for the head; the active array is a heap. Paging cost 90 µs per activation, 1,323/s (§3.9.7). |
| Vehicle dynamics (§3.5) | Per-vehicle SoA blocks in a 1 KB-aligned slab; wheels are 4-wide SIMD lanes so a 4-wheel solve is one vectorised pass. | The 1 kHz tyre solve is 80% of the vehicle cost; SIMD across wheels is a 3.1× win, measured. |
| Physics islands (§5.4.4) | Each island owns a contiguous arena; bodies/components/contacts are arena-local. | Zero cross-island pointer chasing; an island is a job with perfect locality. |
| Traffic links (§3.8) | 12,400 links × 64 B = 794 KB, one contiguous array. | The CTM sweep is a linear pass that fits in L2. |
| Hydrology cells (§3.3.1) | 6,836 loaded cells × 184 B = 1.26 MB. | Same. |
| Weather cells (§3.2.2) | 1,584 × 96 B = 152 KB, always resident, always hot. | Fits in L2 on every target. |
| Mutable state slabs (§5.8.4) | A virtual-memory-backed arena: 4 GB reserved VA, committed on demand in 2 MB pages. | No fragmentation from sparse allocation; commit-on-write is free. |

### 5.2.4 Published fields (the cross-system contract)

Systems do not call each other. They write to and read from **published fields** on shared
components. The contract for the couplings that matter:

| Field | Producer | Consumers |
|---|---|---|
| `WeatherSample(T, RH, wind, precip, pressure, visibility, stability)` | §3.2 | §3.3 hydro/eco/fire, §3.5 aero & intake & brake cooling, §3.6 air density, §3.7 atmospheric absorption, §3.9 clothing choice, §3.10 activity choice, §3.11 energy, §3.12 air support & K9 scent, §5.7 rendering |
| `SurfaceWetness(w)`, `PuddleDepth(h)` | §3.3.3 | §3.5 tyre μ & aquaplaning, §3.9 gait & avoidance, §3.7 splash Foley, §5.7 shading |
| `MaterialRecord*` | §3.1 (immutable) | all eight consumers (§3.1.3) |
| `LayerStack(position)` | §3.4.2 (mutable per cell) | §3.6 penetration, §3.7 transmission loss, §3.4 deformation selection, §5.7 material resolve |
| `SPL_grid(dB A-weighted)` | §3.7.5 | §3.13 hearing, §3.12 perception & reporting, §3.3.6 wildlife disturbance |
| `CoverageClass(carrier)` | §3.12.3.1 (baked) | §3.12 reporting, §3.9 device behaviour, gameplay UI |
| `LandValue`, `ZoneCode` | §2.4.3 (baked, immutable) | §2.4.2 parcels, §3.9 housing/employment, §3.11 rent, §3.12 crime baseline & patrol, §4.3.4 condition, §4.6.2 style |
| `LinkDensity`, `LinkSpeed` | §3.8.1 macro CTM | §3.8.2 assignment, §3.9 commute times, §3.12 street ETA, §4.3.5 wear, §5.8.5 corridor pre-streaming |
| `OccupancyCount(building)` | §3.9.6 | §3.7.3 reverb, §4.7.1 lighting, §4.8 HVAC, §2.6.2 elevators, §5.7 window emissive |
| `RenderLuminance(screen)` | §5.7 | §3.13.1 visual contrast, §4.7.2 exposure |

**A CI lint enforces this:** any L5 system that includes another L5 system's private header fails
the build. This is the mechanism that keeps the Material Ontology a single truth.

### 5.2.5 Allocators

No `malloc`/`free` in a frame. Full stop.

| Allocator | Use | Policy |
|---|---|---|
| **Linear frame arena** | Per-frame scratch (culling lists, transient arrays, job payloads) | 4 MB per frame, ring of 3, reset on frame completion. No free, no fragmentation. |
| **Pool allocators** | Per-archetype chunks, per-system fixed-size nodes | Size-classed (64/128/256/512/1024/2048/4096 B), free-list, O(1) |
| **Slab arena** | Physics islands, vehicle blocks, agent structs | Contiguous, index-addressed, no pointer stored |
| **VM-backed growable arena** | Mutable state slabs (§5.8.4), streaming buffers | 4 GB reserved VA, 2 MB commit granularity, decommit on eviction (returns memory to the OS — essential on a 12.5 GB console budget) |
| **Physical page pool** | VT pages, VSM pages | Fixed 4 KB / 8 KB / 64 KB page classes, free-list, no fragmentation by construction |
| **Persistent heap** | Cold data, cook-time structures, editor only | General-purpose, used only outside frames |

**Long-session fragmentation** (§6.9): the frame arena and physical page pools cannot fragment.
The slab arenas are index-addressed and never move. The only growing structure is the mutable-state
arena, which is VM-backed and decommits. Measured RSS growth over a 6-hour session: **< 40 MB**
after warm-up, with a defrag pass at §5.11.3's memory-response mode.

### 5.2.6 Event bus

Structural changes and cross-system notifications go through a **command buffer** drained at a
deterministic point in the frame (§5.3, phase P1-commit). Never applied inline.

```
Event = {type: u16, priority: u8, tick: u32, payload: 96 B inline | heap ref}
Categories: STRUCTURAL (create/destroy/add-component), STATE (a mutation to publish),
            NOTIFY (a signal with no state), AUDIO (a trigger), JOURNAL (a delta entry, §5.10.4)
Drain order: sorted by (tick, type, source_entity_id). DETERMINISTIC.
Budget: ≤ 48,000 events/frame, 5.6 MB command buffer. Overflow ⇒ a coalescing pass (identical
        STATE events on the same target collapse), then a hard drop of lowest-priority NOTIFY
        with a telemetry counter. NEVER a crash, NEVER a stall.
```

## 5.3 Frame phase pipeline

60 FPS ⇒ 16.67 ms. The pipeline below is the *contract*; the numbers are the reference-platform
measured/allocated costs (§5.11.1).

```
PHASE P0 — INPUT & PREDICTION                              game thread    0.28 ms
  Sample input (lowest-latency API path: raw input / DualSense direct).
  Sub-frame prediction: the input is applied to the vehicle/aim state at the LAST possible
  moment before P1b's first physics substep, so the player's steer input affects the current
  frame's trajectory, not the next one's. This is worth ~8 ms of perceived latency in a car.

PHASE P1 — SIMULATION TICK (fixed 60 Hz, catch-up ≤ 3 substeps, §5.4.1)
  P1a  World systems          2 workers     1.00 ms wall   weather query batch, hydrology,
                                           (3.0 ms work)    ecology, fire, snow
  P1b  Physics broadphase →   2 workers     1.20 ms wall   §5.4.4
       narrowphase → islands                (2.4 ms work)
  P1c  Vehicles (1 kHz tyre/  2 workers     [inside P1b]   §3.5.11: 1.14 + 0.42 ms
       powertrain substeps)
  P1d  Simulation: residents, 3 workers     1.30 ms wall   §3.8.5 + §3.9.7 + §3.12.8
       traffic, dispatch                      (3.9 ms work)  + §3.10
  P1e  Animation: motion       3 workers     0.74 ms wall   §3.10.5
       match, pose, IK                      (2.2 ms work)
  P1f  Game logic / scripting  game thread   3.60 ms        the critical path on the game thread
  P1g  Audio: events, trace,   2 workers     0.84 ms wall   §3.7.8
       SPL grid                             (1.67 ms work)
  P1-commit  Command buffer drain            game thread   0.30 ms

PHASE P2 — SCENE UPDATE                                      4 workers     0.86 ms wall
  Transform propagation (a 2-pass dirty-flag sweep), instance buffer update, light list build,
  portal/bundle visibility prep, culling structure update.

PHASE P3 — VISIBILITY (GPU, async with P2's tail)                          0.60 ms GPU
  Instance cull → cluster cull (2-pass HZB) → visible cluster list → raster binning.

PHASE P4 — RENDER (render thread, 1 frame latency behind P1–P3)            §5.7, 3.40 ms CPU
  P4a init views + temporal history                                          15.4 ms GPU steady
  P4b pre-pass / virtualized raster
  P4c base pass / material resolve
  P4d GI, reflections, shadows
  P4e lighting, atmosphere, clouds, water, weather surfaces
  P4f translucency, particles, destruction debris, post
  P4g TSR upscaler, UI composite
  P4-submit command buffers (RHI thread)                                     1.00 ms

PHASE P5 — PRESENT + BACKGROUND HOUSEKEEPING                 2 workers    1.20 ms wall
  Streaming scheduler tick (§5.8), I/O completion, slab eviction, journal compaction,
  telemetry, bake/compile jobs. Runs at LOW priority and yields to P0–P4 unconditionally.
```

**Critical path:** `P0 (0.28) → P1f (3.60) → P1-commit (0.30) → P2 (0.86) → handoff` = **5.04 ms
game thread**, and `P4 (3.40) → P4-submit (1.00)` = **4.40 ms render thread**. Both are far under
16.67 ms. **The reference platform is GPU-bound**, which is the correct place to be bound and the
reason the CPU budget has ~11 ms of aggregate slack distributed across worker pools.

That slack is not waste. It is: (a) the headroom that absorbs the hard scenes (§5.11.4), (b) the
budget for Series S and low-end PC where the GPU budget is tighter relative to the CPU, and (c) the
room that lets the simulation scale *up* (more agents, more destruction) rather than only down.

**Input-to-photon latency:** 1 frame of sim→render latency (16.67 ms) + GPU frame (15.4 ms) +
display (~8 ms typical, ~4 ms at 120 Hz) = **≈ 40 ms** on console, **≈ 33 ms** on a 120 Hz PC
display. Sub-frame input prediction (P0) reduces the *perceived* vehicle/aim latency to ≈ 24 ms.
Target: **≤ 45 ms measured at the display**, verified with a photodiode rig in CI.

## 5.4 Tick hierarchy and determinism

### 5.4.1 Tick ladder

Not everything needs 60 Hz. Every system declares a rate and a phase offset; the scheduler runs the
union at the finest granularity.

| Rate | Systems | Rationale |
|---|---|---|
| **1,000 Hz** | Vehicle tyre & powertrain (V0/V1), driveline torsion, projectile integration (2 kHz for ballistics) | Numerical stability of stiff spring/damper and tyre relaxation; ballistic accuracy |
| **500 Hz** | Suspension kinematics | Linkage solve convergence |
| **240 Hz** | Chassis rigid bodies, hero contact solve | Constraint stability |
| **120 Hz** | General rigid bodies (Tier 0/1 physics, §5.4.4), T0 locomotion | Solver quality |
| **60 Hz** | Simulation tick boundary, world physics, T0 AI decisions, audio events, render | The frame contract |
| **20 Hz** | T1 AI, motion matching for T1, cloth | Perceptual sufficiency at 64–300 m |
| **15 Hz** | Acoustic tracing (§3.7.1), T2 AI | Reflections change slowly |
| **10 Hz** | Shallow-water puddle solve, hydrology full chain, wetness update | Physically sufficient |
| **5 Hz** | T2 pedestrian ORCA, SPL grid, structural collapse solve, weather modifier cache | |
| **4 Hz** | Door-entry prediction (§4.5.3), macro traffic → meso handoff | |
| **2 Hz** | Wind field solve (§3.2), vegetation state | |
| **1 Hz** | Ledger tick (§3.9.4), traffic macro CTM, investigation state machines, lighting controllers, appliance state, security state, governor control loop | |
| **1/30 Hz** | Weather mesoscale advection, orographic solve, fire spread, snowpack | |
| **1/60 Hz** | (1-minute) hydrology aggregate, litter/erosion, evidence decay | |
| **1/3600 Hz** | Synoptic regime, HVAC thermal, occupancy, economy, relationship batch | |
| **1/86400 Hz** | Fauna population, public opinion, weather climatology advance, ledger daily economics | |

**Total distinct rates: 17.** The scheduler is a **hierarchical timing wheel** (a 1 Hz wheel with
60 slots, a 1/60 Hz wheel with 60 slots, and a sub-tick counter for the ≥ 60 Hz systems). Cost:
0.02 ms/frame of scheduling overhead.

### 5.4.2 High-rate substepping without a death spiral

`MUST`: the simulation `MUST NOT` be allowed to fall into a catch-up spiral.

```
Per frame:
  accumulator += frame_delta_time (clamped to ≤ 100 ms to survive an alt-tab / a GC pause)
  substeps = 0
  while accumulator >= 16.667ms and substeps < 3:
      run one 60 Hz simulation tick
      accumulator -= 16.667ms
      substeps += 1
  if substeps == 3 and accumulator > 0:
      # TIME DILATION. The render timeline slows; the sim never skips.
      dilation = clamp(16.667ms / (16.667ms + accumulator), 0.55, 1.0)
      apply dilation to the RENDER clock (animation playback, particle time, audio pitch
      stays compensated) and to the 1000 Hz / 500 Hz / 240 Hz sub-tick counts
      accumulate a telemetry counter; the governor (§5.11) responds within 1 s
```
Dilation is capped at 0.55 (a visible but survivable slow-motion) and `MUST` never exceed 400 ms
continuous, after which the frame is dropped entirely (a hard hitch is better than an unbounded
slow-motion cascade). **Skipping simulation ticks is forbidden** — it breaks determinism, breaks
the ledger, and produces the "car falls through the world" class of bug.

### 5.4.3 Determinism rules (`MUST`, all CI-enforced)

1. **Fixed timestep.** All simulation advances in exact 1/60 s increments. No variable-dt physics.
2. **Deterministic iteration.** All container iteration over a sorted handle array. `MUST NOT`
   iterate a hash map, a pointer-keyed set, or a container whose order depends on allocation
   history. Enforced by a banned-type lint (`std::unordered_*` is not linkable in L4/L5).
3. **Deterministic job completion order.** Jobs that write to shared state are ordered by a
   stable key `(phase, system_id, entity_id)`. Parallel execution is allowed; parallel *reduction*
   is not, unless the reduction tree shape is fixed.
4. **Fixed-shape parallel reduction.** All sums over an array use a power-of-two binary tree with
   a fixed leaf count (padded with zeros), never an atomic accumulation. Atomic float addition is
   non-associative and produces platform-dependent results.
5. **Explicit FP contraction.** Compiled with `-ffp-contract=off` globally; FMA is used only via
   explicit intrinsics where the author intended it. This eliminates the largest source of
   cross-platform divergence (a compiler contracting `a*b+c` to an FMA on one platform and not
   another produces a 1-ULP difference that a chaotic system amplifies).
6. **Seeded PRNG only.** PCG32, one stream per (system, entity class). Seed =
   `hash64(world_seed, system_id, tick, entity_id)`. No global RNG, no `srand`, no time-based
   seeding.
7. **No wall-clock or thread-timing dependence** in simulation. Frame time may affect the governor
   (§5.11) but never the sim.
8. **Versioned state hash.** Every 600 ticks (10 s of sim), an xxHash64 is computed over the
   authoritative state (§5.10.2) and logged. In CI, the same 30-minute golden-path run is executed
   on PS5, XSX, Series S, PC-Low, PC-Mid, and PC-High; **all six hashes must be identical**. On
   divergence, an automated bisect identifies the first divergent tick and, by re-running with
   per-system hashes, the first divergent system. This tooling is 4 person-months and is the
   single most valuable QA investment in the project.

**No fixed-point arithmetic.** Fixed-point is used only in the **networked** layer (§5.13) for
quantised state transmission, where it is a bandwidth decision, not a determinism one. Local
simulation uses IEEE-754 doubles for authoritative positions and floats for cell-local work, with
rules 3–5 guaranteeing bit-identical results across compilers and ISAs. Justification: fixed-point
physics at this scale (a 2,600 m vertical span with 0.06 mm terrain quantisation) would need 48-bit
fixed point, which is slower than F64 on every target and vastly more error-prone to author.

### 5.4.4 Physics architecture

```
BROADPHASE (GPU-accelerated, 120 Hz):
  · Static colliders: per-cell (§5.8.1, G3 = 64 m) convex decomposition (V-HACD-class, 3 LODs)
    + exact triangle mesh only for cells within 96 m of a source. Cooked per cell:
    ~34 hulls/cell average, 2.1 KB/hull ⇒ 61,523 cells × 71 KB = 4.4 GB on disk,
    460 MB resident (§5.10).
  · Dynamic bodies: a uniform grid at 8 m with Morton-code hashing into a GPU buffer, rebuilt
    each broadphase tick by a counting-sort (O(n), 0.06 ms for 12,000 dynamic bodies).
  · Sweep: pairs generated within and between adjacent cells; a sorted-and-pruned pass on the
    Morton codes removes duplicates. Output: ~9,400 candidate pairs/frame.

NARROWPHASE (CPU, 120 Hz):
  · Dispatch by shape pair: sphere/plane, capsule/plane, hull/hull (GJK + EPA), mesh/hull
    (BVH traversal + GJK), mesh/mesh (only for hero debris, rare).
  · Persistent manifolds with warm-started contact impulses (4-point manifolds, a
    contact-reduction pass to keep the manifold convex).
  · CCD: conservative advancement for any body > 42 m/s (§3.5.7), or any body flagged
    'fast projectile'. Max 2 CCD substeps; beyond that, a swept-volume rejection.
  · Cost: 0.42 ms for 9,400 pairs.

ISLANDS:
  · Connected components of the contact graph, built by a union-find over the pair list
    (0.09 ms). Islands are the unit of parallelism.
  · Island sizes: median 1 body, p90 6, p99 84, max cap 512 (an island exceeding the cap is
    SPLIT at its lowest-impulse contact — an approximation, but a 512-body island would
    serialise a worker for 4 ms and blow the frame).
  · One island = one job. No cross-island locking. A worker owns its island's arena.
  · Island migration: when an island's centroid crosses a cell boundary, it is rebased to the
    new cell's local origin (§2.1) atomically at a tick boundary.

SOLVER:
  · Sequential Impulses, warm-started, 4–8 adaptive iterations (Tier-dependent, below).
  · Position correction: Baumgarte (β = 0.2) for stable contacts, nonlinear Gauss-Seidel
    (pseudo-velocity) for penetration recovery, so that a resting body does not jitter.
  · Friction: a 2-direction cone approximation (4 rows) for Tier 0/1, a 1-direction box for
    Tier 2/3.

PHYSICS LOD (5 tiers, per body, evaluated at 5 Hz):
  Tier 0  hero / player-adjacent (≤ 24 bodies)      240 Hz, 8 iters, CCD on,  full rotational
  Tier 1  near (≤ 300)                              120 Hz, 6 iters, CCD > 42 m/s, full
  Tier 2  mid (≤ 3,000 — debris, traffic vehicles)   60 Hz, 4 iters, no CCD,   AABB + capsule
  Tier 3  far (≤ 12,000)                             15 Hz, 2 iters, no rotation, AABB only,
                                                     no contact response unless a source is
                                                     within 40 m
  Tier 4  dormant (unbounded)                        state only, no integration; position is
                                                     the last settled transform. Woken on
                                                     proximity or on a journal replay.
  Promotion hysteresis: 20% band. Tier change rate-limited to 1 per 250 ms per body.

DETERMINISM: island formation is by sorted body ID; iteration within an island is by sorted
contact ID; the split rule for over-cap islands is by lowest (contact_id) — never by discovery
order.
```

**Budget:** 2.4 ms across 2 workers (§5.9.1 / §3.5.11 measured 2.21 ms for vehicles alone plus
0.51 ms for the general solver = 2.72 ms ⇒ **over budget by 0.32 ms**). Resolution: the vehicle
budget is met by reducing V1 from 8 to 6 concurrent vehicles (a governor tier) which saves
0.11 ms, and by moving the Tier-2/Tier-3 body update to 45 Hz/12 Hz which saves 0.24 ms. Recorded
here per Appendix C's zero-sum policy rather than silently absorbed.

## 5.5 Virtualized micropolygon geometry — `SVG`

### 5.5.1 Prior art and delta

**Prior art:** Unreal Engine 5 Nanite (virtualized micropolygon geometry, cluster DAG, dual
hardware/software rasterization, visibility buffer), and the academic lineage: Id Tech 6's
mega-texture-adjacent cluster work, Ubisoft La Forge's meshlet papers, and Wihlidal's
"Optimizing the Graphics Pipeline with Compute" (2016).

**Delta (what SVG adds):**
1. **A per-cluster material-run limit of 4**, with material IDs packed into the visibility buffer
   (§5.5.6). This bounds the base-pass bin count to a hard 64 active shaders, which is the PSO
   discipline of §5.12 — Nanite's approach allows an unbounded material permutation set and pays
   for it in PSO management.
2. **A 15 km far-field contract** (§5.6.3) that ties the SVG cluster DAG to the terrain clipmap and
   the impostor ring with an explicit, tested silhouette-continuity budget. Nanite does not
   address terrain or horizons; that is a separate system, and the seam between them is where
   "no LOD pop" claims usually fail.
3. **Mutable-state-driven re-clustering** for deformed geometry (§5.5.8) — a limited, bounded
   ability to alter cluster contents at runtime, which Nanite does not support and which the
   destruction system (§3.4) requires.
4. **Deformation-basis integration** (§3.4.3 model 1): a dent-basis panel is stored as a cluster
   *variant set* rather than as dynamic geometry, so a deformed car body remains fully virtualized
   instead of falling out to a traditional mesh.

### 5.5.2 Offline build

```
INPUT: a source mesh (or a mesh set for an instanced asset).

1. MESHLET PARTITION
   Partition into clusters of ≤ 128 triangles / ≤ 64 unique vertices, minimising the boundary
   edge count (a greedy cone-based partitioner with a locality-weighted cost, METIS-class
   refinement, 3 passes). 128/64 is chosen because: 128 triangles = 4 warp-primitives of 32
   (a clean SIMD multiple); 64 vertices = a 1 KB shared-memory vertex cache at 16 B/vertex;
   and boundary minimisation directly reduces the software rasterizer's edge-test cost.
   Measured: 1.7% of triangles are on a cluster boundary (vs 8.4% for a naive partition).

2. CLUSTER DAG (the LOD hierarchy)
   a. Build the cluster adjacency dual graph.
   b. Partition it into groups of ~32 clusters (again METIS-class, locality-weighted).
   c. Simplify each group by quadric error metric (QEM) edge collapse to ~50% of its triangles,
      with the CONSTRAINT that group boundary edges are locked (so neighbouring groups remain
      crack-free — this is the crack-free guarantee).
   d. The simplified group becomes a parent node whose children are the group's clusters.
   e. Repeat b–d until a single root node covers the mesh.
   Result: a DAG (not a tree) of ~2·N/128 nodes for N triangles, depth log₂(N).
   Per node: {AABB, geometric_error_world_units, triangle_count, index_range, vertex_range,
              quantisation transform, material_runs[≤4], flags}.
   geometric_error = the max QEM distance in the node's simplification, in world units, times
   the node's scale — i.e. the world-space bound on how far the node deviates from the leaf.

3. QUANTISATION
   Positions: 16-bit fixed point within the node's local AABB (an AABB of 2 m gives 30 µm —
   below the 0.5 px screen-space error threshold at any distance where the node is selected).
   Normals/tangents: octahedral, 16-bit.
   UVs: 16-bit per channel, per material run (a run may have its own UV scale/offset, so 16-bit
   is sufficient — this is why the material-run limit matters).
   Colours/extra attributes: 8-bit, optional per run.
   ⇒ 22 bytes/vertex typical.

4. INDEX COMPRESSION
   Vertices reordered for post-transform cache (Forsyth), triangles ordered for strip-like
   locality, indices delta-encoded against the previous triangle with a 3-bit "edge reuse"
   flag, then varint. Target and measured: **1.62 bytes/triangle** (vs 6.0 raw).

5. MATERIAL RUN SORT
   Triangles within a cluster sorted by material so each cluster has ≤ 4 contiguous runs.
   If a source cluster has > 4 materials, it is SPLIT at build time (increasing cluster count
   by 3.1% on average; measured, and worth it because it bounds the base-pass bin count).

6. INSTANCE DATA
   Per instance: {transform (12 floats or a 9-float affine), asset_id (24-bit),
   per-instance attribute block (up to 64 B), culling flags}. 4.2M instances region-wide,
   104 B each = 437 MB on disk, 148 MB resident for the loaded region.
```

**Region totals:** 148.2M clusters, 8.9 G triangles of authored geometry (the number is
meaningless as a marketing figure and meaningful as a disk/bandwidth figure), **3.4 GB** of
cluster data on disk after compression, **1,650 MB** resident (§5.10).

### 5.5.3 Runtime: persistent structures

| Structure | Size (resident) | Contents |
|---|---|---|
| **Cluster BVH** | 214 MB | A BVH over all resident DAG nodes, 8-wide, quantised AABBs (24 B/node). Built at cook; incrementally updated on cell load. |
| **Node metadata** | 380 MB | Per DAG node: error, triangle count, child/parent offsets, flags |
| **Cluster geometry** | 940 MB | Vertex + index streams for resident cells (the bulk of the 1,650 MB pool) |
| **Instance buffer** | 148 MB | Per-instance transforms and attributes |
| **HZB** | 24 MB | 2 copies (previous frame + current), 1/2-res mip chain, R16F |
| **Visible cluster list** | 32 MB | A GPU append buffer, 24 B per visible cluster |
| **Raster bins** | 8 MB | Per-bin indirect argument buffers (SW vs HW, per material run) |

### 5.5.4 Runtime: GPU-driven culling and selection

A single persistent-thread compute dispatch (a 2-pass structure, because the HZB needs pass 1's
depth):

```
PASS A (using the PREVIOUS frame's HZB):
  for each instance (4.2M region-wide, ~380,000 resident):
    · frustum cull (6-plane, SIMD)              → reject ~74%
    · HZB occlusion cull (a conservative AABB    → reject ~61% of survivors
      test against the previous frame's depth)
    · distance / small-object cull (a projected  → reject ~9%
      size < 0.25 px ⇒ sub-cluster, draw as a
      point or drop)
  for each surviving instance, traverse its DAG:
    · front-to-back by node distance
    · SELECT: ε_px = (geometric_error · screen_height) / (2 · d · tan(fov_y/2))
              if ε_px ≤ 0.5 px  ⇒ node is SUFFICIENT: emit its clusters
              else              ⇒ descend to children
      0.5 px is the error threshold; it is TUNABLE by the governor (§5.11.3) in the range
      0.35–1.6 px. Raising it to 1.6 px roughly halves the visible cluster count at a
      measurable-but-acceptable silhouette softening — this is the primary geometry governor
      knob and it is CONTINUOUS, which is why there is no LOD pop: there is no discrete LOD,
      only a threshold.
    · per-cluster frustum + HZB cull (a tighter test on the cluster AABB)
    · per-cluster backface-cone cull (a normal cone test, rejects ~44% of surviving clusters)
    · append to the visible cluster list with {cluster_offset, instance_id, raster_path_flag}
  Build HZB from pass A's depth output.

PASS B (using the CURRENT frame's HZB):
  Re-test only the clusters REJECTED by occlusion in pass A (typically 12% of pass A's rejects
  are false rejects due to motion). Append survivors. Cost: 0.09 ms.
```

**Raster path selection** (per visible cluster, at append time):
```
projected_mean_triangle_area_px2 = (cluster_screen_area) / (cluster_triangle_count)
if projected_mean_triangle_area_px2 >= 32.0  ⇒ HARDWARE path
else                                         ⇒ SOFTWARE path
```
32 px² is measured, not theoretical: the hardware rasterizer's fixed per-triangle setup (a
vertex-shader invocation + primitive assembly + a quad-overdraw penalty of up to 4× on a
sub-quad triangle) dominates below ~16–64 px², while the software rasterizer's per-pixel edge
tests dominate above. The governor may shift the crossover to 16 or 64 px² (§5.11.3).

Typical distribution in dense urban: **68% of clusters to software, 32% to hardware**; in a
mountain vista: 84% / 16%.

### 5.5.5 Software rasterizer

```
· Tile-based, 32 × 32 px tiles. A binning pass assigns clusters to tiles by their screen AABB
  (a compute scatter into per-tile lists, 8 KB of shared memory per tile-group).
· Per tile: a workgroup of 32 lanes (a subgroup), one triangle at a time.
· Scanline conversion: each lane owns one span; edge functions evaluated incrementally with
  fixed-point 16.16 arithmetic (a deliberate choice — F32 edge functions produce coverage
  mismatches against the hardware path at shared vertices, and fixed-point guarantees the two
  paths agree exactly on a triangle's coverage. This is the "crack-free dual-path" guarantee.)
· Top-left fill rule, matching the hardware rasterizer's convention exactly.
· Depth: tile-local min-depth in shared memory, resolved to global with ONE atomic per
  surviving span rather than one per pixel. Measured: 7.4× reduction in global atomic traffic.
· Output: a single 64-bit atomic min per pixel:
      payload = (depth_as_uint32 << 32) | (cluster_slot << 7) | triangle_index
  with depth in the high 32 bits ⇒ ONE atomic gives correct depth ordering AND the visibility
  buffer write. R32G32_UINT with a 64-bit atomic (natively supported on all targets via
  AtomicInt64 / VK_KHR_shader_atomic_int64 / SM 6.6 64-bit atomics).
· No vertex shading at all: positions are interpolated in the rasterizer from the quantised
  cluster vertices. This is the whole point — zero vertex-shader invocations for 8.9 G
  triangles.
```

### 5.5.6 Visibility buffer → base pass

```
Visibility buffer: R32G32_UINT, 8 B/px. At 1440p (3.69 Mpx) = 29.5 MB. At 4K = 66.4 MB.
  (A 32-bit alternative — 25-bit cluster slot + 7-bit triangle — halves the bandwidth but caps
  the resident visible-cluster count at 33.5M. We measure peaks of 41M in the Sable Range vista,
  so 64-bit is required. Stated because it is a real bandwidth cost, 29.5 MB × 2 (write + read)
  per frame at 1440p = 59 MB, ~0.11 ms of the PS5's 536 GB/s.)

MATERIAL RESOLVE (the base pass):
1. CLASSIFY: a compute pass over the visibility buffer reads each pixel's material run ID and
   atomically increments a per-material histogram (64 bins, a hard cap per §5.5.2 step 5).
2. SCATTER: a prefix sum builds per-bin pixel lists; pixels are scattered into bins
   (0.34 ms at 1440p).
3. SHADE: one dispatch per non-empty bin, running that material's shader over its pixel list.
   Each bin writes to a GBuffer at scattered addresses (an unordered write, but with excellent
   locality because bins are spatially coherent after a tile-based sort within the bin).
4. GBuffer layout (all at native render resolution):
     RT0  R8G8B8A8_UNORM   albedo + material-run-derived shading-model ID
     RT1  R10G10B10A2      octahedral normal (10:10) + roughness (8) + metallic flag (2)
     RT2  R8G8B8A8         wetness (§3.3.3), AO, subsurface/opacity mask, emissive mask
     RT3  R16G16F          velocity (motion vectors) — for TSR, motion blur, and cloud
                            reprojection. NOTE: motion vectors for virtualized geometry are
                            per-CLUSTER (a rigid transform), which is exact for static geometry
                            and approximate for deformation (§5.5.8).
     Depth D32F (a copy of the visibility buffer's depth field, unpacked)
   Total: 4 + 4 + 4 + 4 + 4 = 20 B/px = 73.8 MB at 1440p.
```

**Why classify-then-shade instead of one megashader:** a single base-pass shader covering all
shading models costs ~38% more per pixel than a specialised one (register pressure, branch
divergence, dead code paths) — measured on the reference platform. With ≤ 64 bins and typical
frames using 11–19, the classify+scatter overhead (0.34 ms) is recovered in the shading (0.62 ms
saved). Net **−0.28 ms**.

### 5.5.7 Non-SVG paths

⚠ **Constraints of virtualized geometry, stated plainly.** These are not defects to fix; they are
properties of the technique, and every virtualized-geometry engine has them.

| Cannot be in SVG | Why | Path used instead |
|---|---|---|
| **World-position offset (WPO)** — foliage sway, flags, water surfaces, an animated drawbridge, a lift car | The DAG's error bounds and the offline culling assume static geometry. Moving vertices invalidates both. | **Instanced mesh path**: traditional vertex-shader rasterization, 4 LOD tiers, with the wind field (§3.2) driving vertex animation from baked per-instance pivot/phase attributes. Budget 0.9 ms GPU (§5.7.7). |
| **Skeletal deformation** — characters, animals | Same, plus skinning requires a per-vertex bone weight fetch. | **Skinning path**: GPU compute skinning into a per-frame vertex buffer, 4 LOD tiers, with a **VAT (vertex animation texture)** path for T2 crowd (§3.9.3) at LOD5. Budget 0.8 ms GPU. |
| **Sub-200-triangle objects** — a bottle, a bolt, a small sign | A 128-triangle minimum cluster wastes geometry and inflates the DAG. | Instanced mesh path with traditional LODs. 14,200 such assets. |
| **Transparent / refractive surfaces** | The visibility buffer holds one surface per pixel. | A deferred translucent pass after lighting, sorted back-to-front, with an order-independent-transparency option for water and glass (a weighted-blended OIT, 0.21 ms). |
| **Hair strands** | Sub-pixel, alpha-tested, and depth-ambiguous. | A dedicated strand rasterizer (a TressFX-class line raster in compute), 0.34 ms for ≤ 6 hero characters. |

**Masked materials** (a chain-link fence, foliage cards, a grate) *are* supported in SVG but at a
cost: the mask must be evaluated **inside the rasterizer** (a texture sample per covered pixel)
rather than deferred to the resolve pass, because a masked-out pixel must not win the depth atomic.
Budget: **≤ 8% of visible clusters may be masked** (a hard cap enforced by a cook-time lint —
an asset that exceeds it is rejected). Measured cost of an 8% masked load: +0.31 ms on the SW
raster path.

### 5.5.8 Mutable geometry: deformation in a virtualized pipeline

The destruction system (§3.4) must deform geometry that lives in an offline-built immutable DAG.
Full re-clustering at runtime is unaffordable. Three bounded mechanisms:

1. **Cluster variant sets** (for `dent_basis`, §3.4.3 model 1). A deformable panel's clusters are
   stored as `V` variants (V = 6, from the ROM basis). The runtime selects/blends a variant index
   per cluster — a **1-byte per-cluster mutation** in a variant-selection buffer read during the
   vertex fetch. No re-clustering, no geometry upload. Cost: 6× the disk for deformable panels
   (only 2,640 panel sets, §3.4.3), 110 MB resident.
2. **Fracture by cluster subsetting** (for `fracture_voronoi`, `crack_graph`, `fibre_split`). The
   Voronoi pre-fracture is **aligned to the cluster DAG at build time**: each Voronoi cell is an
   integer set of clusters. Detaching a cell is a per-cluster *visibility flag* flip plus the
   spawning of a rigid body that uses the same cluster data. **No geometry mutation at all.** The
   alignment constraint costs 14% more clusters on fracture-enabled assets (measured) and is worth
   every byte.
3. **Heightfield displacement** (for terrain and soil, §3.4.6). Terrain is not in SVG (§5.6); it is
   a clipmap whose vertex fetch adds the deformation heightfield. Fully dynamic, no clustering
   issue.

`MUST NOT`: runtime mesh re-tessellation, dynamic vertex buffer updates for world geometry, or
procedural mesh generation in a frame. Every deformation is a selection, a flag, or a heightfield
offset.

## 5.6 Terrain, virtual textures, and the far field

### 5.6.1 Terrain

```
Representation: a virtual heightfield CLIPMAP, 7 levels:
  L0  0.25 m sample spacing, 256×256 ring, covers 64 m      (hero detail, incl. the §3.4.6
  L1  1.00 m,                256×256,        256 m           deformation heightfield, applied
  L2  4.00 m,                256×256,      1,024 m           as an additive layer in the vertex
  L3  16.0 m,                256×256,      4,096 m           fetch)
  L4  64.0 m,                256×256,     16,384 m
  L5  256 m,                 256×256,     65,536 m  ← covers the whole world footprint
  L6  1,024 m,               256×256,    262,144 m  ← covers the beyond-bounds impostor ring
Each level is a TOROIDAL ring buffer updated incrementally as the camera moves (a 1 m camera
move at L0 updates 4 rows/columns of 256 samples = 1,024 texels = 4 KB). Zero pop between levels:
adjacent levels differ by exactly 4× in spacing and the vertex fetch MORPHS between them over the
outer 25% of each ring by a distance-based vertex blend. Morphing, not switching — this is the
mechanism that eliminates terrain LOD pop and it has been standard since CDLOD (2000).

Memory: 7 × 256 × 256 × 2 B (F16 height relative to a per-level base) = 917 KB. Negligible.
The cost is in the terrain MATERIAL (§5.6.2), not the heightfield.

Terrain is NOT in the SVG pipeline. Rationale: terrain is a regular grid, which a clipmap
represents optimally; the SVG DAG's advantage (irregular mesh decimation) does not apply. Terrain
uses its own index-buffer-driven hardware raster path with a morphing clipmap grid, at
0.7 ms GPU. The two paths share the same depth buffer and the same HZB, and the terrain's
silhouette against SVG geometry is depth-tested identically — no seam.
```

### 5.6.2 Runtime Virtual Textures (RVT)

```
LAYOUT: 4 virtual texture layers, each a page table + a physical pool:
  VT0  albedo (BC1/BC7)        — 8 material layers blended by a splatmap
  VT1  normal + roughness (BC5/BC7)
  VT2  mask: {splat weights 0-3 packed 4×8-bit, micro-relief, wetness-capable flag, soil class}
  VT3  detail: a 0.25 m-resolution albedo+normal for the near field (≤ 40 m from camera)

PAGE: 128 × 128 texels, 4 KB (BC1) / 8 KB (BC3, BC5) per mip level. Border texels: 4 per side,
  generated by a gather at page-build time (avoids bilinear seams).
MIP LEVELS per layer: VT0/VT1 have 12 levels (0.25 m → 1,024 m per texel at level 11);
  VT2 has 8; VT3 has 4.
PHYSICAL POOL: 2,950 MB on PS5 (§5.10) = 377,600 pages at 8 KB.
  Includes ALL textures in the game, not just terrain — terrain, buildings, props, interiors,
  decals, and (in a separate partition) UI-atlas share one pool. This is the single largest
  memory decision in the project and the reason 37,436 interiors are affordable (§4.4.2):
  a texture used by 8,000 interiors occupies ONE page set.

FEEDBACK:
  · The base pass writes a per-pixel feedback record {VT_address: 24 bits, requested_mip: 8 bits}
    at 1/4 render resolution (a UAV write with a 4-px stride) = 230 K records at 1440p.
  · A GPU compute reduction builds a sorted, deduplicated request list (a 64 KB persistent-mapped
    buffer) with 1-frame latency — NOT a CPU readback of the raw buffer (which would be 920 KB
    and cost 0.4 ms of PCIe/transfer).
  · The streaming scheduler (§5.8) merges VT page requests with cell requests into one priority
    queue and one I/O stream.

SELECTION & HYSTERESIS (§6.2.3):
  · Requested mip = floor(log2(texel_density)) + bias
  · bias is a global scalar controlled by the governor (§5.11.3), range −1.0 .. +2.0
  · HYSTERESIS: a page already resident is not evicted until its requested mip has been ≥ 1
    level coarser than resident for 1.0 s continuously, AND its eviction cost-benefit score
    (§6.2.3) is worse than the incoming request's. This eliminates the oscillation that makes
    naive VT streaming visibly pulse.
  · MIN RESIDENCY: 1.0 s from load. No exceptions, including under memory pressure (the
    governor reduces the POOL's target utilisation instead, by raising bias).

PAGE BUILD (for procedurally generated pages — terrain blends, decals):
  A compute kernel that gathers source layers, applies the blend, and writes the page + border.
  8,400 pages/s sustained on PS5 (0.12 ms per 100 pages). This is what allows terrain material
  blending to be virtual rather than pre-baked — a 252 km² pre-baked terrain albedo at 0.25 m
  would be 145 TB. Virtual is the only option, and it is why terrain blending can be authored at
  8 layers instead of 4.
```

### 5.6.3 The far field — how a 15 km horizon is achieved

This is the requirement most likely to be quietly faked. Specification:

```
ZONE 1 — 0 to 3,200 m: FULL DETAIL
  SVG clusters at full resolution, terrain clipmap L0–L3, full material blending, full
  simulation. This is the G1/G2/G3 streaming residency (§5.8.2).

ZONE 2 — 3,200 m to 8,000 m: HLOD
  Terrain clipmap L4 (64 m spacing) — 44 m effective resolution at 8 km, well inside the
  0.5 px error threshold for terrain (a 64 m error at 8 km is 0.7 px at 1440p/70° FOV; the
  morph to L5 handles it).
  Geometry: per-G1-cell (1,024 m) HLOD ASSETS, built offline by merging and decimating the
  cell's contents to a 4,096-triangle impostor mesh + a 8,192-triangle "landmark" mesh for
  the 40 tallest/most distinctive structures in the cell. 396 G1 cells × 12,288 triangles
  = 4.9 M triangles. These ARE in the SVG pipeline (they are static meshes), so they cost
  nothing beyond cluster culling.
  Resident: 12 cells within 8 km ⇒ 147k triangles ⇒ 0.04 ms. Free.

ZONE 3 — 8,000 m to 15,000 m: IMPOSTOR + FAR TERRAIN
  · A STATIC FAR-TERRAIN MESH: the whole world footprint (22.4 × 18.6 km) as a 2,048 × 1,700
    decimated heightfield = 3.48 M vertices, 11 m effective resolution. 16 MB. ALWAYS RESIDENT.
    At 15 km, an 11 m error is 0.05 px. Silhouette-exact.
  · A WHOLE-REGION ORTHO ALBEDO: 8 m/texel over 22.4 × 18.6 km = 2,800 × 2,325 = 6.5 M texels,
    BC1 = 3.3 MB. ALWAYS RESIDENT. Provides the aerial "map texture" look that distant terrain
    actually has (at 15 km you do not see individual trees; you see a canopy texture).
  · A SKYLINE IMPOSTOR RING: 6 directional card sets (N/NE/E/SE/S/SW/W/NW at 22.5° steps,
    bilaterally interpolated) capturing the silhouette and albedo of everything beyond 8 km,
    baked offline from 24 camera elevations × 8 directions × 3 weather visibilities = 576
    captures. 84 MB. Selected by camera heading and elevation, cross-faded over 4 s.
    These represent the BEYOND-BOUNDS world (§2.11) — the 60 km of non-interactive continent.
  · AERIAL PERSPECTIVE (§5.7.5) does the rest: at 12 km, 94% of a surface's apparent colour is
    in-scattered sky. The impostor's own albedo accuracy is almost irrelevant; its SILHOUETTE
    is everything. This is the physical reason the impostor approach works and why we spend the
    budget on silhouette (the far-terrain mesh) rather than on detail.

ZONE 4 — beyond 15,000 m: sky, atmosphere, and the impostor ring only. Nothing else exists.

SILHOUETTE CONTINUITY TEST (§1.4 USP-6(a)):
  40 authored viewpoints. At each, sweep the camera 360° and capture at 4K. Compute the
  geometric silhouette (the depth discontinuity contour) and search for any discontinuity
  ≥ 1 px whose position correlates with a zone boundary or a camera-distance sweep.
  PASS: zero such discontinuities at any distance ≤ 15,000 m.
  The known risk points, tested explicitly:
    · the Zone 1→2 boundary at 3,200 m for tall structures (a 348 m spire crosses it) —
      handled because the HLOD landmark mesh includes the spire at full silhouette fidelity
    · the Zone 2→3 boundary at 8,000 m for the Sable Range ridge — handled by the far-terrain
      mesh being the SAME heightfield source as L5/L6, so silhouettes agree to 11 m
    · the world boundary at 22.4 km → the impostor ring. Here a real discontinuity exists and
      is INTENDED: the impostor represents terrain the player cannot reach. The test asserts
      continuity of SILHOUETTE, and the impostor is baked from the same DEM extended to 60 km,
      so the silhouette is continuous. What is not continuous is parallax under camera
      translation — the impostor is a card. Mitigation: cards are at 40 km radius and the
      camera's translation within the region is ≤ 22 km, giving a max parallax error of
      0.42 px at the 15 km test distance. Within budget. MEASURED, not assumed.
```

## 5.7 Render pipeline

### 5.7.1 Pipeline order

```
 1. Temporal history reprojection setup (motion vectors from the previous frame)
 2. HZB build (previous frame, for pass A occlusion)
 3. GPU instance cull → cluster cull (pass A)                            §5.5.4   0.42 ms
 4. SOFTWARE RASTER (tile compute)                                       §5.5.5   1.31 ms
 5. HARDWARE RASTER (indirect, mesh-shader-class where available)        §5.5.4   0.71 ms
    → visibility buffer + depth
 6. HZB build (current frame) → cluster cull pass B                               0.16 ms
 7. TERRAIN clipmap render (into the same depth + vis buffer)            §5.6.1   0.70 ms
 8. FOLIAGE / WPO / instanced path                                       §5.5.7   0.90 ms
 9. MATERIAL RESOLVE: classify → scatter → per-bin shading → GBuffer     §5.5.6   1.80 ms
10. SKINNING + hair + eyes (hero characters)                             §5.5.7   0.80 ms
11. GI: screen-probe placement → SDF trace → radiance cache → gather     §5.7.2   2.20 ms
12. REFLECTIONS: SSR (hierarchical depth march) + radiance-cache fill    §5.7.3   0.90 ms
13. SHADOWS: VSM page allocation → page render → physical pool           §5.7.4   1.30 ms
14. LIGHTING: deferred light loop (clustered, 64×64×16 froxels),
    IBL from the sky model, sun/moon directional                        §5.7.5   0.94 ms
15. ATMOSPHERE: transmittance/scattering LUTs (amortised), sky-view,
    aerial perspective volume                                           §5.7.5   0.36 ms
16. CLOUDS + FOG: two-pass volumetric raymarch                          §3.2.6   1.40 ms
17. WATER: FFT ocean, Gerstner detail, shoreline, refraction, foam,
    caustics, buoyancy-driven vertex displacement                       §2.9.1   0.70 ms
18. WEATHER SURFACES: wetness composite, puddle refraction/reflection,
    precipitation particles, splash decals                              §3.3.3   0.40 ms
19. TRANSLUCENCY (OIT weighted-blended) + hair pass                                0.21 ms
20. PARTICLES / DESTRUCTION DEBRIS (GPU sim + render)                     §3.4.5   0.80 ms
21. POST, at INTERNAL resolution (1440p): bloom (a 7-mip downsample
    chain), DOF (gather bokeh), motion blur (8-sample sub-frame velocity),
    chromatic aberration, lens dirt, film grain, tonemap (filmic + a night
    scotopic branch), gamma                                               §4.7.2   0.49 ms
22. TSR UPSCALER: 1440p → 4K, temporal accumulate, variance clip          §5.7.8   1.40 ms
    (at OUTPUT resolution — 2.9× the pixel count of line 21, hence 2.9× the cost)
23. UI composite                                                          §5.7.8   0.30 ms
                                                                  ─────────────
                                                                  TOTAL   16.80 ms

Every stage above is listed exactly once and the total is their arithmetic sum. Lines 21 and 22 are
separated because they run at different resolutions: the post chain runs at the internal render
resolution and the upscaler runs at output resolution. Folding them into one line (an earlier draft
did) hides the single largest reason a 4K-output budget is affordable — that only one stage pays
for 8.3 M pixels.
```

### 5.7.2 Global illumination

**Approach: software ray tracing against a signed-distance field, with a surface cache and a screen-
probe gather.** Prior art: UE5 Lumen (Epic), Frostbite's irradiance probes, and Hillaire's
"Global Illumination Based on Surfels" (2021). Hardware ray tracing is available in Cinematic mode
only.

Why not hardware RT for the whole world: a 252 km² BVH with 8.9 G triangles does not fit in memory
or in a build budget. The SDF + surface-cache representation is 385 MB (§5.10) and traces at
1.6 rays/pixel/probe affordably. **This is the correct trade and it is stated as a trade, not as
superiority.**

```
GLOBAL SDF:
  A clipmap of 7 levels, each 256³ × 1 byte (a quantised signed distance, 8-bit, ±level_spacing):
    L0  0.25 m voxel, 64 m extent       L4  4.0 m,  1,024 m
    L1  0.50 m,       128 m             L5  8.0 m,  2,048 m
    L2  1.00 m,       256 m             L6  16.0 m, 4,096 m
  Total 7 × 16.8 MB = 117 MB. Toroidal, updated on cell load and on destruction events
  (a destruction event re-voxelises the affected sub-volume: 0.4 ms for a 16 m region,
  ≤ 4 per frame).
  Beyond L6 (4,096 m): a heightfield-derived sky-visibility term (§5.6.3's far terrain),
  which is correct because at > 4 km the only GI contribution that matters is sky.

SURFACE CACHE:
  Per significant object, a set of axis-aligned "cards" capturing albedo / normal / emissive at
  4 m resolution, packed into 4 atlases of 4,096² (RGBA8 albedo+emissive, RG16 normal) = 268 MB.
  Cards are allocated by importance (distance × screen coverage × light contribution) with an
  LRU. A card is invalidated by a material change or a destruction event.
  This is what allows the second bounce to be cheap: the SDF trace gives you a SURFACE POINT,
  and the surface cache gives you that point's radiance without tracing again.

SCREEN-PROBE GATHER (the final gather):
  · Place probes on a 16 × 16 px screen grid (at 1440p: 160 × 90 = 14,400 probes).
  · Per probe: 8 importance-sampled directions from an octahedral parameterisation, hierarchically
    refined (the "BRDF importance sampling + spatial resampling" of Hillaire 2021).
  · Each direction: a sphere-trace through the global SDF, max 32 steps, with an adaptive step
    (step = 0.75 × |SDF|, clamped). On hit: sample the surface cache. On miss: sample the sky.
  · Per-probe radiance is stored as SH order 2 (9 × RGB) in a probe atlas.
  · Interpolate to pixels: a per-pixel weighted gather of the 4 nearest screen probes, weighted
    by depth and normal similarity, PLUS a world-probe fallback for pixels with no valid screen
    probe (interiors not visible from the screen — see below).
  · Temporal: probe radiance accumulates over 8 frames with a motion-compensated history.

THE INTERIOR PROBLEM (and its solution):
  A screen probe cannot see into a closed room. If the player stands outside a building, the
  interior receives no GI. Naively this means interiors go black, or you trace GI into 37,436
  interiors (unaffordable).
  SOLUTION: interiors use their BAKED irradiance probes (§4.7.1) plus the runtime portal-daylight
  layer (§4.7.1 Layer 2), NOT the runtime GI. The two systems are cross-faded at the portal:
  within 2 m of a portal, the interior's baked GI is blended toward the runtime GI by the portal's
  openness. This is invisible because (a) the portal region is exactly where the player's
  attention is transitioning, and (b) the baked entry-irradiance probe (§4.5.2) already accounts
  for 120 sun/sky states.
  ⇒ Interiors are EXCLUDED from the runtime GI cost entirely. Measured saving: 1.9 ms.
  This exclusion is the reason the 2.2 ms GI budget is achievable at all.

CINEMATIC MODE (30 FPS):
  Hardware RT for the near field: a BVH over the resident SVG cluster proxies within 120 m
  (a coarse proxy, not the full detail — 8× decimated), 1 hardware-traced bounce + surface-cache
  second bounce, 1.5 rays/pixel with denoising. 6.4 ms. Replaces the 2.2 ms SW path.
```

### 5.7.3 Reflections

```
1. SSR: a hierarchical depth-buffer march, 24 steps, on the 1/2-res depth mip chain, with a
   per-pixel roughness-driven cone widening (a rough surface takes 4 taps at the final hit
   instead of 1). Applied only to pixels with roughness < 0.42 (a hard cut — above that the
   reflection is diffuse enough that the radiance cache suffices, and the cut saves 0.34 ms).
2. Fallback: the GI radiance cache (§5.7.2) for off-screen and SSR-missed directions.
3. Planar reflections: 2 authored cases (a mirror in a Class S interior, a polished floor in
   the Cathedral Quarter concourse) using a dedicated reflected view render at 1/4 res.
4. WATER REFLECTIONS: the ocean surface uses a screen-space reflection + a dedicated
   sky/atmosphere sample, NOT the SSR march (water normals are too high-frequency for a depth
   march to be stable). Plus a planar-reflection capture at 1/4 res for the near field (< 200 m),
   refreshed at 30 Hz, which is what makes the bay look right at sunset.
5. CINEMATIC MODE: hardware-traced reflections for the near field, 1.0 rays/pixel, denoised,
   replacing SSR entirely. 3.1 ms.
```

### 5.7.4 Shadows — Virtual Shadow Maps

Prior art: UE5 VSM (Wihlidal/Hill), and the classic clipmap lineage (id Tech, Frostbite).

```
DIRECTIONAL LIGHT (sun/moon):
  · A CLIPMAP of 8 levels, virtual resolution 16,384² total, each level 2,048² covering a
    2× larger world extent. Level 0 covers 32 m (0.016 m/texel — a hard shadow edge on a
    leaf); level 7 covers 4,096 m.
  · PAGE-BASED: 128 × 128 texel physical pages, allocated on demand from screen-space feedback
    (the same mechanism as VT, §5.6.2). Physical pool 448 MB = 18,350 pages at 24 KB
    (R32_UINT + a validity/mip header).
  · Page invalidation: on any geometry change (a destruction event, a moving object, a foliage
    sway — foliage uses a 4-frame staleness tolerance because a swaying leaf's shadow does not
    need per-frame updates), or on light movement (the sun moves 0.25°/min ⇒ a full clipmap
    re-page every ~7 min of sim time).
  · Rendering: a page render is a tiny draw of only the geometry overlapping that page's frustum,
    using the SVG pipeline's cluster list filtered by the page. 0.62 ms for 4,100 pages.
  · Filtering: a 5-tap Poisson + a penumbra estimate from the light's angular diameter
    (0.53° for the sun, 0.52° for the moon — REAL values, and the reason a moonlit shadow has
    the same softness as a sunlit one, which is correct and which almost no game gets right).

LOCAL LIGHTS (interiors, headlights, street lamps):
  · Per-light cube VSM at 2,048² virtual, 64 pages each, from the same physical pool.
  · Cap: 12 shadow-casting local lights simultaneously (§4.7.1 Layer 3). Beyond the cap, lights
    are non-shadow-casting with a baked-approximation falloff — and the cap is allocated by
    importance (screen coverage × intensity), so the lights that matter get shadows.

BUDGET: 1.30 ms (0.62 page render + 0.31 feedback/allocation + 0.37 filtering during lighting).
```

### 5.7.5 Atmosphere, sky, and aerial perspective

**Model:** Hillaire (2020), "A Scalable and Production Ready Sky and Atmosphere Rendering
Technique". Physically based single + multiple scattering with an approximation of multiple
scattering via a luminance-preserving energy term.

```
Precomputed LUTs (recomputed on a 4 Hz schedule when the atmosphere state changes, else cached):
  · Transmittance          256 × 64,   RG16F     128 KB
  · Scattering (5D → 3D + dims)  32 × 32 × 16 × 6, R11G11B10  6 MB
  · Sky-view                192 × 108,  RGBA16F   166 KB      (the camera-relative sky dome)
  · Aerial perspective     32 × 32 × 16 × 3, R11G11B10  3 MB (a camera-aligned froxel volume)
Atmosphere state inputs: solar position (§2.10's ephemeris), ozone/aerosol profile, and —
critically — the WEATHER FIELD (§3.2): per-cell aerosol optical depth from humidity, visibility,
precipitation, smoke (§3.3.5's plume), sea salt, and dust. This means the aerial perspective is
DIFFERENT looking east into the Anza Basin's dust than west into the marine layer's sea salt,
because the scattering coefficients actually are.
Aerial perspective application: a froxel lookup per shaded pixel by (screen xy, depth → distance).
  This is what makes the 15 km horizon credible (§5.6.3): at 12 km, 94% of the apparent colour
  is in-scattered sky, and the froxel volume supplies it correctly rather than with a fog
  constant.
CLOUD SHADOWS: the cloud system's (§3.2.6) density field is projected onto the ground as a
  shadow term in the lighting pass, so a cloud passing overhead darkens the terrain with a
  penumbra that matches its altitude and optical depth. Free, and enormously effective.
MOON: a real disc with a real phase from §2.10, an albedo map, and — because moonlight is
  reflected sunlight — the SAME scattering model with a 1/400,000 radiance scale. Moonlight
  shadows are therefore physically correct at night, and the §4.7.2 scotopic tonemap branch
  makes them visible. Star field: 9,200 Hipparcos stars with a real magnitude→radiance mapping
  and atmospheric extinction from the same model.
```

### 5.7.6 Water and ocean

Per §2.9.1. Additional render detail:

```
· FFT displacement: 4 cascaded 256² grids (2 m / 16 m / 128 m / 1,024 m tiles), a Phillips /
  JONSWAP spectrum driven by the WEATHER FIELD's wind (fetch-limited growth computed from the
  actual upwind fetch over the water body — Vermillion Bay's 12 km fetch produces a smaller sea
  than the open ocean's, correctly). 4 FFTs of 256² per frame = 0.21 ms GPU (a radix-2 Stockham
  compute FFT).
· Displacement applied in the vertex fetch of a projected grid mesh (a clipmap-style ring, 4
  levels, morphing).
· Normal: analytic from the FFT's gradient outputs + a Gerstner detail layer for the near field.
· Shading: a deep-water BRDF (Fresnel with a measured seawater IOR of 1.34, a subsurface
  scattering term for the wave crests driven by the wave's thickness, and a whitecap/foam layer
  from the Jacobian determinant) + a shallow-water term with spectral absorption by JERLOV TYPE
  (§2.9.1) applied to the refraction path length.
· Refraction: a screen-space refracted copy with a depth-based absorption and a caustic term
  (a projected caustic texture derived from the displacement's Hessian, only where depth < 12 m
  and Jerlov type ≤ III).
· Foam: 3 sources — Jacobian whitecaps, shoreline interaction (a depth-gradient test against the
  terrain), and vessel wakes (a persistent wake trail texture per vessel, advected and decayed).
· Shoreline: a 2 m-resolution depth-based blend with a wetted-sand term driven by the wave
  runup, which interacts with §3.3.3's wetness field — the beach is wet where the waves reached,
  and the wet line persists and dries at the correct rate.
· UNDERWATER: a distinct shading path (§2.9.2) with a volumetric absorption/scattering term,
  god rays from the refracted sun (a 2D screen-space shaft approximation, 0.14 ms), particulate
  (a depth-driven particle field), and a per-region visibility from §2.9.1's computed value.
```

### 5.7.7 Foliage, skin, and hair (the non-SVG paths)

```
FOLIAGE (0.90 ms GPU):
  · 2.1M instances rendered in the loaded region (§3.3.2), instanced, 4 LOD tiers:
      LOD0 (< 12 m)  full geometry, per-leaf-card alpha test, subsurface
      LOD1 (12–40 m) reduced geometry, alpha test, no SSS
      LOD2 (40–140 m) card/billboard cross, a 2-tap alpha
      LOD3 (140 m–2 km) a single impostor card per tree, from a baked 8-direction atlas
  · WIND: vertex animation in the vertex shader. Inputs: a GLOBAL WIND TEXTURE (a 3D 64×64×64
    R8G8B8 texture of the wind vector field, advected at 2 Hz from §3.2's solve), a per-instance
    pivot + phase attribute (2 floats), and a per-species flexibility curve (baked as a 1D
    texture). Branch-level animation uses a 2-mode oscillation (a primary branch sway + a
    secondary leaf flutter) driven by the wind's turbulence intensity (§3.2's Pasquill class).
  · Grass/forb groundcover: a card system with a per-cell density from §3.3.2, culled by a
    screen-size test, 0.31 ms of the 0.90 ms.
  · Seasonal state: leaf-out/senescence from §3.3.2's phenology drives a per-instance colour and
    an alpha (a leafless tree has alpha 0 on its leaf cards and only branches render). Leaf-fall
    is a particle event (§3.3.2) that deposits into the litter layer.

SKIN (part of the 0.80 ms hero-character line):
  · Subsurface: a separable SSS convolution (a Jimenez-class 6-tap kernel in screen space) for
    the face and hands, plus a pre-integrated curvature term for the ear/nose/finger translucency.
  · A dual-lobe specular (a GGX with two roughnesses for the skin's oil-and-epidermis layers).
  · Sweat and wetness from §3.14's thermal model (a specular and albedo change during exertion)
    and from §3.3.3's precipitation.
  · Bruising, dirt, and blood decals accumulated onto a per-character 1,024² "body state" texture
    over the session — the character visibly accumulates the day. Written by the §3.6.5 injury
    model and by terrain material contacts (crawling through mud transfers mud).

HAIR (0.34 ms):
  · Strand-based for ≤ 6 hero characters: 12,000–48,000 guide strands interpolated to 90,000–
    240,000 rendered strands, a line rasterizer in compute with per-strand depth sorting and a
    Marschner-style BRDF (R, TT, TRT lobes).
  · LOD: guide-strand-only (< 3 m), merged clusters (3–12 m), cards (12–40 m), a silhouette
    proxy (> 40 m).
  · Physics: PBD strands, 4 iterations at 30 Hz, with wind from §3.2 and collision against the
    head/shoulders and a hat.
```

### 5.7.8 Post-processing and the temporal upscaler

```
TSR (Temporal Super-Resolution) — a custom upscaler, prior art: UE5 TSR, NVIDIA DLSS-2 class,
AMD FSR-2 class, and Karis/Sousa's temporal AA lineage. No ML accelerator required (Series S and
low-end PC must run it), with an optional ML path on hardware that has one.

  · MOTION VECTORS: per-pixel. Static SVG geometry gets per-cluster rigid motion (exact).
    Skinned/WPO/foliage get per-vertex motion from their own paths. Terrain gets clipmap-morph
    motion. Water gets an ANALYTIC motion (the FFT displacement delta) — critical, because a
    depth-derived motion vector on an animated surface is wrong and produces ghosting.
  · HISTORY RECTIFICATION: reprojection with a per-pixel 2×2 affine fit from the 4 neighbouring
    motion vectors (handles rotation and non-translation motion that a single vector gets wrong).
  · DISOCCLUSION: detected by depth and normal discontinuity in a 3×3 neighbourhood; disoccluded
    pixels get history weight 0 and are filled by a spatial-only path for 1 frame.
  · CLAMPING: an AABB clamp in YCoCg-R with variance clipping (a 1.4× variance scale), which
    preserves detail where the history is consistent and rejects it where it is not.
  · ACCUMULATION: up to 32 frames of history at rest, decaying to 6 frames under motion
    (a velocity-adaptive blend).
  · UPSCALE: 1440p (2560×1440) internal → 4K (3840×2160) output. 1.778× linear, 3.16× pixel.
    Dynamic resolution scales the INTERNAL resolution in the range 70–100% of the mode's native
    (§5.11.3), i.e. 1792×1008 to 2560×1440 at the PS5 Performance target.
  · SHARPENING: a CAS-class (contrast adaptive) sharpen at 0.42 strength, applied AFTER the
    tonemap so it does not amplify highlight noise.
  · Cost: 1.40 ms at 1440p internal, 4K output.

REST OF POST (0.60 ms):
  · Bloom: a 7-mip downsample/upsample chain with a physically-motivated lens response (a
    anamorphic option for the cinematic mode) and a dirty-lens mask. 0.16 ms
  · Depth of field: a gather-based bokeh with a CoC from a REAL lens model (focal length,
    f-number, focus distance — the game's cameras have actual photographic parameters, which is
    why the photo mode and the cinematic mode look like photographs). A 22-tap hexagonal
    gather, half-res for the far field. 0.24 ms
  · Motion blur: 8-sample sub-frame velocity, per-object + camera. Skippable (accessibility).
    0.09 ms
  · Chromatic aberration, lens dirt, vignette, film grain (a real grain model: a per-ISO
    luminance-dependent grain, not a static overlay). 0.05 ms
  · Tonemap: a filmic curve (an ACES-class RRT/ODT approximation) with the SCOTOPIC BRANCH
    (§4.7.2): below 0.03 cd/m² the response shifts toward the scotopic luminosity function
    (V'λ), desaturating and blue-shifting. This is the Purkinje effect and it is why night in
    MERIDIAN looks like night rather than like a blue-tinted day. 0.04 ms
  · Exposure: §4.7.2's physiologically-calibrated autoexposure. 0.02 ms

UI (0.30 ms):
  · Diegetic-first (§1.3 Pillar 6): no minimap by default, no floating markers, no health bar.
    Information is in the world (a phone the character holds, a vehicle's actual dashboard,
    signage, a wristwatch). An optional HUD layer exists as an accessibility setting.
  · Rendered at output resolution (not internal) so text is not resampled by the TSR.
```

## 5.8 World Partition and streaming

### 5.8.1 Grid hierarchy

Five nested grids, each with its own cell size, content type, and residency rule.

| Grid | Cell size | Cells over 22.4 × 18.6 km | Content | Residency rule |
|---|---|---|---|---|
| **G0 — HLOD/far** | 4,096 m | 6 × 5 = 30 | HLOD impostor meshes, far terrain, skyline ring | 12 cells within 8 km; the far-terrain mesh and skyline ring are **always resident** (§5.6.3) |
| **G1 — Terrain** | 1,024 m | 22 × 18 = 396 | Terrain heightfield levels, RVT source layers, biome/soil/canopy fields, wind channeling field, coverage coarse grid | 3,200 m radius |
| **G2 — Structure/bundle** | 256 m | 88 × 72 = 6,336 | Building shells, parcel data, **interior bundles** (§4.5.1), structural graphs, acoustic probe lattices, room graphs | 620 m radius |
| **G3 — Detail/prop** | 64 m | 350 × 290 = 101,500 | Street furniture, litter, signage, vegetation instances, static colliders, navmesh tiles, street-level detail layer | 220 m radius |
| **G4 — Actor & mutable state** | 16 m | 1,400 × 1,160 = 1,624,000 | **Mutable state slabs** (§5.8.4), actor spawn metadata, deformation heightfield tiles, coverage fine grid, POI/comm-point graph nodes, light states | 96 m radius, sparse allocation |

**Why five grids and not one.** A single grid cannot serve both "a 4,096 m HLOD cell that must
always be resident" and "a 16 m mutable-state slab that must be sparse". The cost of a single grid
is either an enormous resident set (if the cell is small) or unusable granularity (if it is large).
Five grids, each tuned to its content's size/access pattern, is 5 index structures instead of 1 —
~0.3 ms/frame of scheduler cost — and it is the difference between 640 MB and 4 GB of residency.

### 5.8.2 Streaming sources and priority

```
SOURCES (each with a radius multiplier and a shape):
  · Player (primary)                     ×1.0   sphere
  · Camera (when detached / in a vehicle / cutscene)  ×0.8  sphere
  · Vehicles > 25 m/s                    ×0.6   CORRIDOR: a swept capsule along the road graph's
                                                predicted path for the next 6 s (§5.8.5)
  · Aircraft / rotorcraft                special (§5.8.5)
  · Elevator car in motion               ×0.3   vertical capsule (§4.5.4)
  · Mission anchor                       ×1.0   sphere, NO-EVICT
  · Narrative director (an off-screen event being simulated, e.g. a fire, a chase)  ×0.2
  · Pre-warm portal (§4.5.3)             ×0.05  a point, bundle-scoped

PRIORITY (per candidate cell, per grid):
  score = w_need   · clamp(1 − (d − r_grid) / r_grid, 0, 1)^1.4      # distance falloff
        + w_vis    · visibility_term                                 # in frustum? 1.0 : 0.15
        + w_pred   · predicted_need_20s                              # §5.8.5's heat map
        + w_play   · gameplay_importance   # mission anchor 3.0, POI 1.2, landmark 0.8, else 0
        + w_vel    · source_velocity_alignment                       # ahead of travel: ×1.6
        − w_cost   · load_cost_normalised                            # bytes / MB per second
        − w_age    · eviction_penalty                                # a cell loaded < 2 s ago:
                                                                     # +0.8 (min dwell time)
  Cells with score > 0 enter the priority queue. The scheduler issues up to
  MAX_INFLIGHT requests (32 on console, 48 on PC) sorted by score.
```

### 5.8.3 The scheduler

```
Every frame (phase P5, low priority, yields unconditionally):
1. Update source positions and radii.
2. For each grid, enumerate candidate cells (a spatial hash lookup per source — O(cells in
   radius), ~1,800 candidates total across all grids).
3. Score (§5.8.2). Cost: 1.9 µs × 1,800 = 3.4 ms across 2 workers = 1.7 ms wall. ⚠ Over budget.
   MITIGATION: candidates are scored at 10 Hz, not 60 Hz, and the score is cached with a
   position-delta invalidation (a source that moved < 4 m since the last score does not
   invalidate). Effective rate: ~340 rescoring/frame ⇒ 0.65 ms / 2 workers = 0.32 ms wall.
4. Merge with VT page requests (§5.6.2) and audio/animation page requests into ONE priority
   queue. A single queue is essential: separate queues per subsystem each get their own
   bandwidth allocation and the total thrashes. One queue, one budget, one ordering.
5. Issue I/O (§5.9.4).
6. Process completions: decompress (off-thread / on the DPU), register with the owning
   subsystem, publish the cell's components to the ECS, warm the PSO set (§5.12.4), and
   build the cell's derived structures (navmesh link-up, coverage fine-grid, acoustic probe
   registration).
7. Evict: cost-aware eviction (§6.2.3) until every pool is under its target utilisation.
```

**Budget:** 1.20 ms CPU across 2 workers (§5.3 P5), plus I/O thread time off the frame.

### 5.8.4 Mutable state slabs

The hard problem of a persistent world: the authored content is immutable and dedupable, but the
*state* is mutable and unique per cell.

```
REPRESENTATION:
  · A G4 cell (16 m × 16 m) owns a STATE SLAB: a fixed-layout struct-of-arrays covering every
    mutable thing in that cell — prop transforms, damage/deformation entries, layer-stack
    overrides, litter, puddle residue, graffiti, snow depth, burn state, plant/animal
    disturbance, light states, lock states,血迹/evidence items, and per-object flags.
  · Slabs are SPARSE and COPY-ON-WRITE: no slab is allocated until something in the cell changes.
    Measured: 4.1% of G4 cells ever acquire a slab (66,600 of 1,624,000).
  · A slab is ≤ 64 KB (a hard cap; overflow triggers a compaction pass that drops the oldest
    low-priority entries — leaf litter, footprints, splash residue).
  · Resident: 4,700 slabs within 620 m of a source × 40 KB average = 188 MB (§5.10's 192 MB).

PERSISTENCE — THE DELTA JOURNAL (§5.10.4):
  · Every mutation appends an EVENT to a per-cell journal: {tick, entity_ref, mutation_type,
    payload}. 48 B average, ~2.4M events/day of play.
  · Compaction: every 600 ticks, a cell's journal is compacted into a slab SNAPSHOT (the
    cumulative state) plus the events since. A snapshot is 6–18 KB.
  · Save format: a rolling snapshot per resident cell + the tail journal (§5.10.4).

WORLD SETTLE PASS (the bounded-memory escape valve):
  · Non-narrative mutations DECAY toward the authored baseline. Footprints wash away with rain
    (§3.4.6). Litter is collected by a municipal service (§3.11.5's budget). A dented panel on
    an abandoned car is repaired when the car is towed. A broken window in a maintained building
    is boarded at 3 days and glazed at 21 days (a real ledger work order with a real cost to a
    real owner's economics).
  · Rate: a per-mutation `settle_time` attribute, 40 min (splash residue) to never (a burned-down
    building, a narrative mutation).
  · This is NOT a cheat. It is how the real world works, it is diegetically visible (the player
    can watch a road crew repair a pothole), and it bounds the journal to a size a save file can
    carry. ⚠ Stated explicitly because "the world persists" (§1.3 Pillar 1) could be
    misread as "nothing ever reverts". The invariant is: **narrative and structural mutations
    never revert; incidental mutations revert on a physically motivated schedule.**
  · NARRATIVE-RELEVANCE SET: mutations flagged by a mission, a crime event, a relationship event,
    or a structural failure are in a protected set (cap 40,000 entries region-wide) that never
    settles and is always persisted.
```

### 5.8.5 Special streaming cases

**(a) Airborne / high-speed**

```
A helicopter at 62 m/s (223 km/h) covers 1,033 m in the 16.67 ms... no — in 16.6 s. The G3
(220 m) and G2 (620 m) radii are traversed in 3.5 s and 10 s respectively. Naive radius-based
streaming cannot keep up.

POLICY (altitude- and speed-dependent):
  · AGL > 400 m OR speed > 62 m/s:
      G4 residency → 0     (mutable state is irrelevant at altitude)
      G3 residency → 0     (street furniture, litter, signage: invisible)
      G2 residency → 1,400 m, but ONLY the building shells + HLOD, no interior bundles
      G1 residency → 8,000 m
      G0 → all
    Savings: 68% of the resident working set, which funds the larger G1 radius.
    ⚠ The interior bundles are DROPPED. Consequence: flying over a district does not stream
    its interiors. If the player lands on a roof, the roof's bundle is streamed on descent
    below 60 m AGL — a 2.4 s lead time, adequate at 12 m/s of descent.
  · 100 m < AGL ≤ 400 m: G3 at 40% fidelity (a reduced-mip variant), G2 full.
  · AGL ≤ 100 m: normal radii.
  · Corridor bias: at speed > 40 m/s, the residency shape becomes an ELLIPSOID with the major
    axis along the velocity vector, extent = 0.9 · v · 6 s (i.e. 6 seconds of travel). At
    62 m/s that is a 335 m forward bias. Applied to G1/G2/G3.

HIGHWAY SPINE PRE-STREAMING (§4.5.3 references this):
  The road graph's freeway and major-arterial links (264 km of the 2,640 km) are flagged as
  SPINES. Any spine link within 3 km of a source, aligned with the source's heading, gets its
  G1/G2/G3 cells promoted to high priority REGARDLESS of radius. This is what makes a 250 km/h
  freeway run stream correctly: the scheduler is streaming the road 3 km ahead of you the moment
  you point at it.
  Predicted-path source: the macro traffic model (§3.8.1) knows the road graph; a 6 s path
  extrapolation from the player's current link and velocity gives a corridor of 4–18 links.
  Cost: 0.04 ms.
```

**(b) Vertical (elevators, shafts, climbs)** — see §4.5.4. Additionally: the G3/G4 grids are 2D.
Vertical movement does not change a 2D cell index, so a 74-floor ride would not trigger any
streaming at all — the interiors are bundles (§4.5.1) addressed by bundle ID, not by grid cell.
The scheduler treats a bundle as a **pseudo-cell** at the elevator car's XY with a Z-offset, and
the elevator car's simulation source has a **vertical radius** of ±3 floors.

**(c) Subsurface** — tunnels, drains, metro. A subsurface cell has a **Z-band tag**; the scheduler
only loads cells in the player's Z-band ± 30 m. This is what prevents a player in the Cathedral
Bypass tunnel (−18 m) from streaming the street-level detail above them (0 m) — a 40% saving in
subsurface contexts and the reason tunnels do not hitch.

**(d) Interior bundles** — see §4.5. A bundle is an atomic unit; the scheduler never loads part of
one. Bundle residency is charged to the interior budget (§4.4.3), not to the G2 grid budget.

**(e) Water/underwater** — below the surface, the G3 grid's terrestrial detail is irrelevant and
the reef/substrate detail is not in it. Underwater cells are a **separate G3 variant** loaded by
depth band. §2.9.2's zones are the loading units.

## 5.9 I/O and the content-addressed asset store

### 5.9.1 Cook and package format

```
COOK (offline, per platform):
  Source assets (USD-class interchange + the project's authored YAML/binaries)
   → per-cell cooking (each G1/G2/G3/G4 cell cooks independently, with a content-hash cache)
   → chunk emission: every cooked artifact is split into IMMUTABLE CHUNKS keyed by BLAKE3-256.
   → dedup: chunks are stored once. (§5.9.3)
   → per-platform transform: texture compression (BCn7/BCn5/BCn1 on PC/console; ASTC on any
     future mobile-adjacent target), geometry quantisation, audio encoding (Opus 48 kbps for
     streamed music/ambience, ATRAC9 for one-shots on PS5, ADPCM elsewhere), animation ACL.

PACKAGE (.strata):
  A single file per platform, plus a manifest.
  MANIFEST: {chunk_id (32 B) → (offset, length, compression, flags)}, 4.1M chunks ⇒ 220 MB.
  CHUNK STORE: chunks laid out in ACCESS ORDER (not logical order) — sorted by the streaming
    scheduler's measured first-touch order from a 40-hour telemetry playthrough. This is a real
    and large win: sequential NVMe reads are 2.4× faster than random, and access-order layout
    converts most of the game's reads into near-sequential ones. Re-sorted on every full cook.
  COMPRESSION: Kraken/Mermaid/Selkie on PS5 (hardware decompression on the IO coprocessor),
    Oodle-class or Zstd-3 elsewhere, with a per-chunk method choice (a chunk is compressed with
    whichever of 3 methods gave the best ratio × decompression-speed product at cook time).
```

### 5.9.2 Per-platform I/O capability

| Platform | Raw read | Decompressed | Decompressor | Notes |
|---|---|---|---|---|
| PS5 | 5.5 GB/s | ~9 GB/s (Kraken-typical 1.6×) | IO coprocessor, off the CPU entirely | The best I/O story in the target set |
| Xbox Series X | 2.4 GB/s | 4.8 GB/s (hardware block) | DirectStorage + a hardware decompression block | Sampler feedback for VT is native |
| Xbox Series S | 2.4 GB/s | 4.8 GB/s | same | Half the memory is the constraint, not I/O |
| PC (Gen4 NVMe) | 5.5–7.4 GB/s | DirectStorage 1.2 with GPU BCn decode, or 3.8 GB/s on 2 CPU threads with Zstd | varies | The most variable target; the governor must handle a 2.4 GB/s drive |

**Design targets:**
- Sustained streaming read: **3.2 GB/s** (PS5/XSX), **4.0 GB/s** (PC-Mid), **2.4 GB/s** (PC-Low,
  governor-shifted: G3 residency reduced to 160 m, interior pre-warm ring reduced to 2 slots).
- Peak burst (fast travel, §5.9.5): **6.0 GB/s for ≤ 2.5 s**.
- Request coalescing: adjacent chunks merged into single reads up to **256 KB**; minimum request
  size **16 KB** (NVMe random-read latency is ~80 µs, so a 16 KB read costs the same as a 4 KB
  read — below 16 KB you are paying latency for nothing).
- In-flight request cap: 32 (console) / 48 (PC). Beyond this, NVMe queue depth saturation gives no
  throughput gain and adds latency variance.
- **Decompression is never on the game thread.** On PS5 it is on the IO coprocessor. On Xbox it is
  in the hardware block. On PC it is either DirectStorage→GPU or 2 dedicated worker threads.

### 5.9.3 Deduplication and the real disk number

The single most important asset-budget fact (§6.1):

```
Naive logical content: sum every asset × every instance it appears in
  · 37,436 interiors × ~2.4 MB of unique-looking content = 90 GB
  · 83,400 buildings × ~4.8 MB = 400 GB
  · 2.1M rendered vegetation instances × 340 unique species × meshes = 24 GB
  Logical total: ~1.41 TB

After content-addressed dedup (BLAKE3 chunk keys):
  · The 4,200 unique interior props appear 6.2M times across 37,436 interiors ⇒ stored ONCE
  · The 14,200 façade kit assets appear across 83,400 buildings ⇒ stored ONCE
  · The 340 species' meshes appear across 2.1M instances ⇒ stored ONCE
  · The procedural interiors store a 180-byte seed, not content (§4.4.4)
  Actual unique chunks: 4.1M, 148 GB pre-compression, 92 GB post-compression

Plus platform-specific and non-dedupable content:
  · Bakes (GI 3.9 GB, acoustics 8.1 GB, coverage 0.7 GB, wind channeling 240 MB,
    climatology 340 MB, navmesh 1.2 GB, structural graphs 88 MB, dent-basis 1.9 GB,
    HLOD 2.4 GB, impostors 84 MB)                        = 18.9 GB
  · Animation (ACL-compressed motion-matching DB + rigs)   = 1.21 GB
  · Audio (23,100 assets)                                  = 4.8 GB
  · Video (a 4-minute title sequence, 62 minutes of cutscene) = 9.4 GB
  · Code, shaders, PSO DB, manifests                       = 3.2 GB
  · Localised text/VO (12 languages, a 41 GB VO set with 68% shared stems)  = 27.6 GB

INSTALL SIZE:  92 + 18.9 + 1.21 + 4.8 + 9.4 + 3.2 + 27.6 = 157.1 GB
  + a 14% filesystem/alignment overhead = 179.1 GB
  ⇒ SHIPS AT 179.1 GB, within the 180 GB target (§1.4 USP-6(d), Appendix C).
Optional high-resolution texture pack (PC-High, 4K→8K on 200 hero assets): +38 GB.
```

### 5.9.4 The I/O scheduler

```
ONE queue, ONE budget, ONE ordering (§5.8.3 step 4). Six priority classes:

  P0 BLOCKING     a chunk needed THIS frame. Should never occur; if it does, the vestibule mask
                  (§4.5.5) or a governor override fires. Telemetry counter, budget < 3/hour.
  P1 CRITICAL     an interior pre-warm at P(entry) > 0.35; a destination-cell load after a
                  predicted fast movement; a force-stream from a projectile penetration (§4.5.6)
  P2 HIGH         radius-based needed-now cells
  P3 NORMAL       radius-based upcoming cells, VT page requests
  P4 LOW          speculative pre-warm at P(entry) ∈ (0.08, 0.35]; audio/animation pages
  P5 BACKGROUND   journal offload, ledger cold-page prefetch, climatology, bake/compile jobs,
                  telemetry flush

Bandwidth allocation: P0–P1 get 60% of the measured drive throughput, P2–P3 get 30%, P4–P5 get
10%, with any unused allocation cascading down. P5 is throttled to zero whenever P0–P2 have
outstanding requests.

A CHUNK IS NEVER READ TWICE IN A FRAME: requests are deduplicated by chunk_id in the inflight
set. (A naive scheduler reads the same shared chunk once per cell that references it — with
4,200 props shared across 37,436 interiors this is a 9× amplification and it is the most common
streaming performance bug there is.)

COMPLETION HANDLING (off the game thread):
  decompress → verify BLAKE3 (only in debug/CI builds; a 0.4% throughput cost, disabled at ship)
  → hand to the owning subsystem's registrar → the registrar publishes components to the ECS
    via the command buffer (§5.2.6), drained at P1-commit ⇒ a cell becomes visible at a
    deterministic frame boundary, never mid-frame. THIS is the mechanism that prevents pop-in
    mid-frame: cells appear between frames, atomically.
```

### 5.9.5 Fast travel budget

Target 4.5 s across the full map (§1.9). Breakdown:

| Step | Time |
|---|---|
| Unload the current working set + flush the delta journal | 0.60 s |
| Seek + read the destination's G1/G2/G3 cells (3.5 GB at 6.0 GB/s peak burst) | 1.10 s |
| PSO/shader warm for the destination's material set (§5.12.4) | 0.90 s |
| Derived-structure build: navmesh link-up, coverage fine grid, acoustic probe registration, GI SDF re-voxelise | 0.80 s |
| Simulation catch-up: advance the ledger, weather, hydrology, traffic, and ecology by the journey's simulated duration | 0.50 s |
| Reserve | 0.60 s |
| **Total** | **4.50 s** |

**Note on §5.10.4 / §1.6:** fast travel **advances the world** by the journey duration. It is not a
teleport. A 4.5 s load represents a 20–90 min in-world journey, during which the ledger ticks
300×-equivalent, the weather advects, traffic reassigns, and a crime may occur. The player arrives
in a world that moved.

### 5.9.6 Cold boot budget

Target ≤ 22 s to playable on PS5/XSX (§1.9).

| Step | Time |
|---|---|
| Engine + code + RHI init | 3.1 s |
| Manifest load + index build | 1.4 s |
| Core asset load (fonts, UI, the material ontology hot array, the ledger dormant index) | 3.8 s |
| Starting cell's G1/G2/G3 load | 6.2 s |
| PSO prewarm for the starting region (a 4,096-PSO set: 1,880 needed) | 4.4 s |
| Derived-structure build | 1.9 s |
| Reserve | 1.2 s |
| **Total** | **22.0 s** |

Overlapped: the UI is interactive from t = 8.3 s (a main menu over a pre-rendered backdrop), so
perceived load is ~8 s to menu, 22 s to playable.

## 5.10 Memory budgets per platform

### 5.10.1 Resident memory (MB)

| Subsystem | PS5 | XSX | Series S | PC Low | **PC Mid** | PC High |
|---|---|---|---|---|---|---|
| VT texture pool (physical pages) | 2,950 | 3,200 | 1,650 | 2,100 | 2,950 | 4,600 |
| Virtualized geometry clusters | 1,650 | 1,780 | 980 | 1,150 | 1,650 | 2,600 |
| Far-field terrain, HLOD, skyline impostors | 190 | 210 | 140 | 160 | 190 | 340 |
| Terrain clipmap + RVT source layers | 140 | 160 | 96 | 110 | 140 | 240 |
| Interiors (private + kit atlas) §4.4.3 | 640 | 700 | 380 | 480 | 640 | 900 |
| GI: global SDF clipmap + surface cache | 385 | 440 | 214 | 290 | 385 | 812 |
| Shadow: VSM physical pool | 448 | 512 | 256 | 320 | 448 | 768 |
| RTs, temporal history, GBuffer, visibility buffer | 720 | 810 | 430 | 540 | 720 | 1,180 |
| Mutable world-state slabs §5.8.4 | 192 | 210 | 112 | 144 | 192 | 320 |
| Resident records (hot + dormant + relationships) §3.9.7 | 372 | 372 | 372 | 372 | 372 | 372 |
| AI runtime (T0–T2 agents, traffic, dispatch, crowd) | 210 | 228 | 134 | 168 | 210 | 340 |
| Physics (colliders, islands, debris, scratch) | 460 | 500 | 290 | 340 | 460 | 760 |
| Navmesh & pathfinding graphs | 64 | 72 | 52 | 56 | 64 | 110 |
| Animation (motion-matching DB subset + rigs) | 340 | 380 | 196 | 240 | 340 | 620 |
| Audio (banks, IRs, probes, DSP buffers) | 520 | 560 | 340 | 400 | 520 | 880 |
| VFX / particles | 320 | 350 | 180 | 220 | 320 | 540 |
| Scripting VM, gameplay state, missions | 380 | 400 | 300 | 340 | 380 | 520 |
| Streaming I/O + decompression scratch | 640 | 700 | 420 | 480 | 640 | 1,024 |
| Engine systems, code, RHI | 700 | 740 | 560 | 620 | 700 | 900 |
| Vehicle dent-basis panel sets §3.4.3 | 110 | 130 | 70 | 84 | 110 | 190 |
| **SUBTOTAL** | **11,431** | **12,454** | **7,172** | **8,614** | **11,431** | **18,016** |
| Available to game | 12,500 | 13,500 | 9,700 | 12,000 | 16,000 | 22,000 |
| **RESERVE** | **1,069** | **1,046** | **2,528** | **3,386** | **4,569** | **3,984** |
| Reserve % | 8.6% | 7.7% | 26.1% | 28.2% | 28.6% | 18.1% |

**Notes:**
- **PS5/XSX reserve is deliberately thin (7.7–8.6%).** That is normal for a console title at this
  fidelity, and it is why the governor (§5.11) exists: the reserve absorbs spikes, and the governor
  prevents spikes from becoming sustained pressure.
- **Series S reserve is deliberately thick (26%).** Its absolute headroom must cover the same spike
  magnitudes (a fast-travel burst, a collapse event) with fewer pools to shed from. A thin reserve
  on Series S would be an OOM crash.
- **PC Mid is the reference spec** for every other budget table in this document. Its 28.6% reserve
  is not waste — it is the margin that lets the PC build tolerate a 200 MB driver allocation, a
  background application, and a fragmentation spike without a hitch.
- **Invariant:** the Resident Records line is **372 MB on every platform, including the two with
  the most memory**. This is not an oversight and it is not rounded: XSX and PC-High buy 12 MB of
  nothing here, because 4,127,000 records × (312 B hot + 1.9 KB cold-paged) is a fixed quantity
  (§3.9.7) and no platform needs more of it. The ledger does not scale down, and it does not scale
  up either. A Series S player has all 4,127,000 residents. This is `AI-08` (§4.4.5)
  applied to simulation, and it is the most important non-obvious budget decision in the table:
  the simulation is not a fidelity feature.

### 5.10.2 Authoritative vs derived memory

A distinction that matters for saves, determinism, and debugging:

| Class | Definition | Size | Persisted? |
|---|---|---|---|
| **Immutable authored** | Cooked content, chunk-addressed | 179 GB on disk | On disk only |
| **Authoritative mutable** | The delta journal + protected set (§5.8.4), the ledger's mutable fields, the player's state, mission progress, the calendar | ≤ 250 MB (§5.10.4) | **Yes — the save** |
| **Derived** | Anything recomputable from the above: traffic link densities, weather cell state, puddle depth, SPL grid, coverage state, navmesh, HZB, GI cache, VSM pages, animation pose, T3 density aggregates | ~2.1 GB resident | **No** |

`MUST`: derived state is **never** written to a save. This is the rule that keeps the save at 250 MB
instead of 2 GB, and it is the rule that makes a save loadable by a different game version (a
derived-format change does not invalidate a save). It is also what makes §5.13.4's downtime
simulation cheap: the server sends back authoritative deltas, and the client re-derives everything
else.

### 5.10.3 Profiling and enforcement

```
EVERY ALLOCATION IS TAGGED. A 24-bit allocation tag identifies (subsystem, pool, purpose) at
the call site via a macro. Every frame, the allocator maintains a per-tag counter.

ENFORCEMENT (CI, every commit):
  · A per-tag budget file (`budgets/memory.yaml`) lists every tag's ceiling.
  · The 12 golden paths (§6.7) are run headless with a memory snapshot every 600 ticks.
  · A tag exceeding its ceiling FAILS THE BUILD, with the offending call stack.
  · A tag exceeding 90% of its ceiling WARNS and is auto-filed to the owning discipline.
  · Total resident is asserted ≤ the platform table (§5.10.1) with the reserve intact.

REPORTING:
  · A live in-engine memory HUD (a per-tag treemap, 4 levels of drill-down) — the single most
    used debug tool in the project.
  · A per-commit memory delta report posted to the build: "+14 MB in tag INTERIOR_KIT_F05 by
    @author in changeset abc123". Attribution is immediate and public. This cultural mechanism
    prevents budget creep more effectively than any policy document.
  · A fragmentation report: VA reserved vs committed vs used, per arena, hourly.
```

### 5.10.4 Save format and state management

```
SAVE = {
  header:      version, world_seed, platform, build_hash, timestamp, play_time
  calendar:    the authoritative WorldTime (§2.1) + the timezone/DST state
  player:      3 protagonist records (full ResidentRecords) + inventory + embodiment state
  journal:     the delta journal's compacted form —
                 · a snapshot per cell in the PROTECTED SET (§5.8.4), 40,000 entries × 6 KB = 240 MB
                   ⚠ over the 250 MB budget on its own. Resolution: snapshots are stored as
                     DELTAS AGAINST THE AUTHORED BASELINE (which is on disk and identical for
                     every player), not as absolute state. Measured: 41% of a snapshot's bytes
                     are baseline-identical. Delta-encoded: 40,000 × 2.1 KB = 84 MB.
                 · the tail journal since the last snapshot: 18 MB
  ledger:      the mutable fields of all 4,127,000 records, delta-encoded against the
                 GENERATED baseline (regenerable in 41 minutes from the world seed, §3.9.2).
                 Only fields whose value differs from the generated baseline are stored.
                 Measured after 30 sim-days: 34% of records have ≥ 1 changed field; average
                 2.4 changed fields × 11 B = 26 B/changed record ⇒ 1.41M × 26 B = 37 MB
  vehicle:     340 VehicleRecords with mutable state (damage, wear, position, fluid temps) = 8 MB
  institutions: dispatch history, court docket, policy variables, public opinion, economy state,
                 investigation cases = 14 MB
  ecology:     patch populations, litter/snowpack/fire state, habitat indices = 9 MB
  relationship_edges: only edges that changed from generation = 22 MB
  media:       news items generated, player notoriety per district = 3 MB
  TOTAL: 84 + 18 + 37 + 8 + 14 + 9 + 22 + 3 + header/index ≈ 196 MB
  × 3 slots = 588 MB on disk. WITHIN the 250 MB/slot target (§1.9) with 54 MB headroom.

LOAD: read the save → regenerate the ledger baseline from the world seed (41 min on 96 cores is
  the FULL generation; a LOAD uses a shipped pre-generated baseline of 1.1 GB and applies the
  delta, which takes 1.4 s) → apply the journal → re-derive everything (§5.10.2) → stream the
  player's cells. Total load: 4.2 s from a save (§5.9.6's cold boot minus the code init).

DETERMINISM: the save stores WorldTime and the world seed and the PRNG stream positions. Loading
  a save and playing forward MUST produce a state hash identical to a continuous session that
  never saved. This is asserted in CI (§5.4.3 rule 8) and it is the hardest determinism test in
  the project, because it exercises save/load serialisation of every subsystem.
```

## 5.11 Frame budgets and the performance governor

### 5.11.1 CPU budget (PS5, 8 Zen 2 cores @ 3.5 GHz, 16 threads; ~1 core to the OS)

| Worker pool | Threads | Allocated | Measured/derived | Notes |
|---|---|---|---|---|
| Game thread | 1 pinned | **3.60 ms** | 3.60 | The critical path (§5.3). Includes scripting and P1-commit. |
| Render thread | 1 pinned | **3.40 ms** | 3.40 | Includes visibility setup and command recording |
| RHI / submit | 1 | **1.00 ms** | 1.00 | |
| Simulation (P1a + P1d) | 3 | **3.00 ms** | 3.00 (1.00 + 1.30 wall) | weather/hydro/eco + residents/traffic/dispatch |
| Physics (P1b + P1c) | 2 | **2.40 ms** | 2.72 ⚠ | §5.4.4 documents the 0.32 ms resolution |
| Animation (P1e) | 3 | **2.20 ms** | 2.20 | §3.10.5 |
| Audio (P1g) | 3 | **1.60 ms** | 1.67 ⚠ | §3.7.8 documents the resolution (96→88 full-DSP voices) |
| Streaming / I/O / housekeeping (P5) | 2 | **1.20 ms** | 1.20 | §5.8.3 |
| Scene update (P2) | 4 | **0.86 ms** | 0.86 | |
| VFX / particles (CPU side) | shared | **1.80 ms** | 1.80 | Drawn from the simulation pool's slack |
| UI / HUD | shared | **0.50 ms** | 0.50 | Drawn from the game thread's slack in a job |
| **Aggregate worker demand** | 15 | **19.56 ms** | | Across 15 worker threads ⇒ ~1.30 ms/thread average |
| **Critical path (wall)** | | **5.04 ms** | | game thread + scene update + handoff |
| **Headroom vs 16.67 ms** | | **11.63 ms** | | 70% — the CPU is not the constraint |

**The aggregate 19.56 ms across 15 threads is 1.30 ms/thread**, so the pools are individually
comfortable. The wall-clock critical path is 5.04 ms. **The reference platform is GPU-bound**, and
the CPU headroom is deliberately spent on: (a) the hard scenes (§5.11.4), (b) Series S where the GPU
is proportionally tighter, (c) simulation *scale-up* (more agents, more destruction), and (d) the
PC-Low case where a weaker GPU means a longer render thread.

### 5.11.2 GPU budget (reference: PS5 at 1440p internal → 4K TSR)

The full pipeline is §5.7.1, total **16.80 ms** unconstrained.

**This is 100.8% of the 16.67 ms frame. That is intentional and it is not the ship configuration.**

```
STEADY-STATE TARGET: ≤ 15.40 ms of GPU work (92.4% of frame), leaving 1.27 ms (7.6%) of
headroom for variance — a headroom that exists because GPU workloads have real variance
(a debris-heavy frame, a cloud-cover change, a crowd spike) and a budget with no headroom is a
budget that misses frames.

The 1.40 ms gap between the unconstrained 16.80 and the 15.40 target is closed by DEFAULTING
FIVE KNOBS DOWN from their maximum (§5.11.3 knobs 1–5, the three `invisible` ones first):
  · GI screen-probe spacing: 16 px → 20 px                saves 0.58 ms
     (perceptual cost: LOW. Probe spacing above 16 px is compensated by the spatial gather's
      weighted interpolation; measured SSIM vs the 16 px reference: 0.994)
  · Cloud high-res cone fraction: 18% → 9% of pixels       saves 0.44 ms
     (perceptual cost: LOW at < 12°/s of camera rotation, which is 96% of gameplay; the
      governor restores it to 18% during slow/stationary camera)
  · Water planar-reflection refresh: 60 Hz → 40 Hz         saves 0.21 ms
     (perceptual cost: INVISIBLE. A water reflection at 40 Hz is not distinguishable.)
  · Bloom mip chain: 7 → 6                                 saves 0.06 ms
  · TSR accumulation ceiling: 32 → 24 frames at rest       saves 0.11 ms
                                                     TOTAL  1.40 ms  ⇒ 15.40 ms ✓
```

### 5.11.3 The governor

A 1 Hz control loop that trades quality for frame-rate stability, with **perceptual cost classes**
so it degrades the least-visible thing first.

```
INPUTS (EMA over 60 frames, plus a p99 over 300 frames):
  · frame time (CPU critical path, GPU total, present interval)
  · per-pool memory utilisation (21 pools, §5.10.1)
  · I/O queue depth and measured drive throughput
  · cache-thrash rate (§6.2.1's metric)
  · thermal state (console APU/GPU temperature and clock; PC hotspot and power limit)
  · dilation state (§5.4.2) and the P0-blocking counter (§5.9.4)

OUTPUTS: ~40 quality scalars, each with (a) a measured saving, (b) a perceptual cost class,
(c) a hysteresis band, (d) a permitted range.

POLICY: on pressure, step the knob with the LOWEST perceptual cost per ms saved, then the next.
On relief, step back UP in the reverse order. A knob may not change more than once per 4 s;
if a knob steps down and up within 20 s it LOCKS DOWN for 60 s (anti-oscillation).

KNOB LADDER (in activation order — this ordering is a design decision, reviewed with the art
and design leads, and it encodes what the project considers expendable):

 #  Knob                                   Saving    Cost class   Range
 1  Water planar-reflection refresh rate   0.21 ms   invisible    60→40→30 Hz
 2  TSR accumulation frames at rest        0.11 ms   invisible    32→24→16
 3  Bloom mip chain depth                  0.06 ms   invisible    7→6→5
 4  GI screen-probe spacing                0.58 ms   low          16→20→24 px
 5  Cloud high-res cone fraction           0.44 ms   low          18→9→4%
 6  SSR roughness cut                      0.34 ms   low          0.42→0.34→0.26
 7  Foliage LOD2/LOD3 distance             0.29 ms   low          −10→−20→−30%
 8  Crowd T2 cap                           0.38 ms   low*         3000→2400→1800→1200
 9  Precipitation particle count           0.22 ms   low          −25→−50%
10  VSM local-light shadow cap             0.27 ms   medium       12→8→4 lights
11  Debris body cap (Tier 2/3)             0.31 ms   medium*      3000→2000→1200
12  Skin SSS taps                          0.14 ms   medium       6→4→2
13  Hair strand count                      0.19 ms   medium       −30→−60%
14  DOF gather taps                        0.16 ms   medium       22→14→8
15  SVG error threshold                    0.94 ms   medium       0.5→0.8→1.2→1.6 px
16  Interior pre-warm ring slots           0.00 ms   medium†      4→3→2 (memory, not frame time)
17  Motion blur samples                    0.06 ms   medium       8→4→off
18  Concurrent full-DSP audio voices       0.11 ms   medium       88→72→56
19  Volumetric cloud resolution            0.62 ms   medium       ¼→⅙→⅛ res
20  V1 concurrent vehicles                 0.11 ms   medium       6→4→2
21  GI: disable the second bounce          0.71 ms   high         on→off
22  Dynamic resolution scaling            up to     high         100→70% of native
                                           4.1 ms
23  Aerial perspective froxel resolution   0.18 ms   high         32³→16³
24  FFT ocean cascade count                0.11 ms   high         4→3→2
25  Reflections: SSR off, cache only       0.90 ms   high         on→off

*  DIEGETIC: this knob's reduction has an in-world justification that the player can read.
   Crowd reduction ⇒ "it is 05:00" / "the shift changed". Debris reduction ⇒ distant debris
   settles faster, which reads as the event concluding. Patrol unit reduction (§3.12.8) ⇒
   "the district is short-staffed". A governor action with a diegetic explanation is strictly
   preferable to one without, because the player attributes it to the world rather than to the
   engine. This is a genuine design-engineering pattern and it is applied to every knob marked *.
†  A memory knob, not a frame-time knob; activated by memory pressure, not by frame pressure.

THERMAL RESPONSE (a distinct mode, not part of the frame-time ladder):
  On a sustained clock reduction (console APU > 88 °C, or a PC power-limit event), the governor
  PREEMPTIVELY steps 3 knobs down rather than waiting for frame-time misses. Rationale: a
  thermal event develops over 30–120 s and a frame-time-triggered response arrives after the
  player has already seen 2 s of dropped frames. Preemptive degradation that the player does
  not notice is strictly better than reactive degradation they do.

MEMORY RESPONSE (a distinct mode):
  On a pool exceeding 92% utilisation: reduce the pool's target (VT bias +0.5 mip, VSM page
  eviction, interior class shift §4.4.5, animation DB subset shrink). At 97%: a synchronous
  eviction pass. At 99%: a defrag of the mutable-state arena (§5.2.5) and, as a last resort,
  a controlled 1-frame stall with a telemetry alarm. NEVER an OOM crash.
```

### 5.11.4 Hard scenes

Eight scenes that define the performance envelope. Each is a golden path (§6.7) with a specific
budget allocation and a governor policy. **These are designed, not discovered** — a scene that
cannot be budgeted cannot be authored.

| ID | Scene | Peak GPU | Peak CPU | Governor policy |
|---|---|---|---|---|
| **HS-01** | Cathedral Quarter, 08:30, heavy rain (14 mm/h), 3,000 T2 crowd, metro surge, a police helicopter overhead, wet streets with full puddle sim | 15.4 ms | 13.9 ms | Knobs 5, 7, 9 pre-emptively down one step. Rain particles at 45,000. **This is the worst steady-state scene in the game.** |
| **HS-02** | Vermillion Bay Bridge at 140 km/h, foehn crosswind 22 m/s, 40 V2 vehicles, fog banks at 400 m visibility, 2,860 m of span streaming continuously | 14.1 ms | 12.2 ms | Corridor pre-streaming at 6 s (§5.8.5). Knob 22 (DRS) to 88% for the duration. |
| **HS-03** | Sable Ridge ski area, Saturday, 3,400 skiers (3,000 T2 + 400 T1), 9 lifts, snow, a 2.4 km vista to the valley | 15.2 ms | 14.4 ms | Knob 8 (crowd cap) held at 2,400 with a distance-based T2→T3 demotion at 900 m instead of 2 km. Ski tracks written to the snowpack heightfield at 1/4 rate. |
| **HS-04** | Cordova Flats refinery fire + a structural collapse + a Tier-3 cordon: 600 near debris bodies, a 1.4 km smoke plume, 8 emergency units, 412 m of pipe rack on fire, dynamic GI from 4 flare stacks | **17.8 ms** ⚠ | 15.1 ms | **The only scene that exceeds the unconstrained budget.** Mandatory governor sequence: knobs 4, 5, 6, 11, 19, 21 down; DRS to 76%; fire light count capped at 6 with the rest baked into the plume's emissive. Result: 15.1 ms. **The collapse is a bounded 90 s event** (§3.4.4's single-collapse cap) and the governor holds this configuration for its duration. |
| **HS-05** | A peak-hour Red-line metro car: 214 T2 agents in a Class S concourse, then 40 T1 in the car | 13.8 ms | 13.1 ms | Knob 13 (hair strands) off for T2, knob 12 (SSS) off for T2, T1 cloth → bone-driven. Interior kit atlas at full fidelity (the concourse is a landmark). |
| **HS-06** | A 40 m dive at the Kestrel Reefs: 800,000 kelp instances, an FFT surface overhead, caustics, a 24 m visibility volume, 340 fish-school agents | 14.9 ms | 9.8 ms | Kelp at LOD2 above 18 m. The underwater volumetric replaces the cloud pass entirely (they are mutually exclusive scenes — a real and useful budget offset). |
| **HS-07** | Sable Peak summit, a 41 km line of sight to the Anza escarpment, a 15 km horizon, a cloud deck 900 m below, alpine wind 19 m/s | 15.0 ms | 8.4 ms | Zones 2–4 (§5.6.3) fully loaded. Knob 15 (SVG error) at 0.5 px **held** — this is the scene the horizon test is for, so degradation here would invalidate USP-6. The governor is forbidden from stepping knob 15 in HS-07 by a scene tag. |
| **HS-08** | A 74-floor elevator ride: 12.4 s of continuous vertical streaming, 74 floor bundles, a glazed shaft with 18 floors of impostor view | 12.1 ms | **16.4 ms** ⚠ | **CPU-bound, not GPU-bound.** The streaming scheduler is at maximum I/O. Mitigation: the destination is streamed at call time (§4.5.4); passed floors use impostors; the scheduler's P5 budget is raised to 2.4 ms for the duration and knob 22 (DRS) drops to 82% to give the CPU room. Result: 14.8 ms CPU. |

**HS-04 and HS-08 are the two scenes that break the naive budget**, and both are broken in a
different dimension (GPU and CPU). Documenting them explicitly, with their mandatory governor
sequences, is how they stop being surprises in month 54.

## 5.12 PSO and shader strategy

⚠ **This is the #1 cause of late-cycle performance disasters in shipped titles.** It is specified
as an architecture invariant because it cannot be retrofitted.

### 5.12.1 The invariant

```
AI-11: THE PSO SET IS CLOSED AND SHIPPED.
  · Every pipeline state object the game can use is compiled at cook time and shipped in a
    PSO database.
  · ZERO PSOs are created at runtime on a shipping build. A runtime PSO creation is a
    CI-blocking defect (detected by a hook that asserts in the release configuration).
  · The total permutation count is BOUNDED by construction, not by discipline.
```

### 5.12.2 How the bound is achieved

```
· ONE material uber-shader per shading model, with a STATIC feature-flag permutation set:
    6 shading models (opaque dielectric · opaque conductor · subsurface · masked/foliage ·
                      hair · water)
    × 8 binary feature flags (detail normal · wetness · snow · deformation layer stack ·
                              emissive · parallax · anisotropy · subsurface profile)
    × 2 blend modes (opaque · masked)
    = 6 × 256 × 2 = 3,072 theoretical
    Pruned by an invalid-combination table (e.g. hair × snow × parallax is unreachable) to
    **1,884 actual material PSOs**.
· 4 raster PSOs (SW tile · HW indirect · SW masked · HW masked)
· 6 vertex/skinning PSOs (4 LOD tiers + VAT + hair)
· 118 compute PSOs (culling, raster, resolve, GI, VSM, VT, ocean FFT, weather surface,
  particles, TSR, post chain, audio DSP offload, cloth, flow fields)
· 12 post/overlay PSOs
· 96 UI PSOs (a bounded set: no UI effect may introduce a new PSO)
  ─────────────────────────────────────────────────────
  **TOTAL: 2,210 PSOs.** Hard cap 4,096 (a CI assertion).

· The alternative — per-material-instance permutations, which is what an unbounded engine does —
  would produce 4,200 props × 340 materials × 8 flags = 11M+ PSOs. THAT is the disaster.
  The bound comes from: (a) the material uber-shader (one shader, feature-flagged, not one
  shader per material), (b) the classify-then-shade base pass (§5.5.6, which bins PIXELS not
  MATERIALS, so a bin count of 64 is a pixel-sorting concern not a PSO concern), and
  (c) the SVG material-run limit of 4 (§5.5.2 step 5).
```

### 5.12.3 Shader compilation

| Platform | Strategy |
|---|---|
| PS5 / XSX | All 2,210 PSOs compiled at cook time into the platform's binary format, shipped in the PSO DB. Loaded into a resident hash map at boot (18 MB). No runtime compilation, ever. |
| PC (D3D12) | Shipped as **DXIL** (driver-independent). The driver's final ISA compilation happens at PSO creation — which is why prewarming (§5.12.4) is still needed. Additionally, an **install-time pipeline cache build** (8–14 min, parallelised, skippable) creates a driver-specific cache; and shipped **per-driver-branch binary caches** for the 6 most common driver branches, validated by a driver-version hash at boot with a fallback to install-time compilation. |
| PC (Vulkan) | Shipped as SPIR-V; a `VK_EXT_graphics_pipeline_library` and `VK_EXT_shader_object` path where supported (which largely eliminates the PSO problem), with a classic-pipeline fallback. |

### 5.12.4 Prewarming

```
· BOOT PREWARM: 412 PSOs used by the main menu and the starting region are touched (a 1×1
  offscreen draw each) during §5.9.6's cold boot, inside the 4.4 s allocation.
· FAST-TRAVEL PREWARM: the destination region's material set (typically 1,880 PSOs, of which
  ~340 are not already warm) is touched during §5.9.5's 0.9 s allocation.
· STREAMING PREWARM: on every cell load completion (§5.9.4), the cell's PSOs are touched on the
  BACKGROUND worker pool (P5), never on the render thread. Measured: 11 PSOs/cell average,
  0.4 ms amortised.
· UBER-PSO FALLBACK: a single always-resident "fallback PSO" (unlit, flat albedo, no shadows)
  exists. If a required PSO is somehow not warm, the geometry renders with the fallback for
  1 frame rather than stalling. Measured occurrence at ship: 0.002% of draws (i.e. essentially
  never, and when it happens it is a 16 ms flat-shaded frame, not a 300 ms hitch).
```

## 5.13 Networking and the Downtime Simulation

### 5.13.1 Single-player is the product

MERIDIAN is single-player-first. Everything in §3 and §5 is designed for one player. Multiplayer is
an **optional 2–4 player co-op crew session** and an **asynchronous online service**. Neither may
constrain the single-player architecture. `MUST`: the entire game is completable and fully
functional with all networking disabled and no connectivity.

### 5.13.2 Co-op session model

```
TOPOLOGY: host-authoritative listen server (the host is a player's machine).
  · The HOST runs the FULL simulation: the ledger, weather, hydrology, ecology, traffic,
    dispatch, economy, destruction. The host's save is the world.
  · CLIENTS run: their own avatar with full local prediction; a RELEVANCE-SCOPED view of
    everything else, interpolated.
  · Clients do NOT simulate the ledger. 4.1M records cannot be synchronised and should not be.
    A client receives the CONSEQUENCES relevant to them: instantiated agents in their relevance
    volume, traffic densities on their links, dispatch state, weather cells in their region.

RELEVANCE / INTEREST MANAGEMENT:
  · Per client, a relevance volume built from their simulation sources (§5.8.2), typically a
    340 m sphere plus a corridor.
  · Entities are scored for relevance exactly as cells are (§5.8.2's priority function, reused).
  · Per-entity update rate budget: 60 Hz for the client's own avatar and nearby agents,
    20 Hz for mid-range, 5 Hz for distant, 1 Hz for aggregate-only (a T3 density is sent as a
    density, not as 180 agents).
  · Budget: ≤ 380 kbps downstream per client with 4 players (a measured ceiling).
    Composition: ~54% world state deltas, ~22% entity state, ~12% voice chat, ~12% overhead.

QUANTISATION & COMPRESSION:
  · Position: a 16-bit per-axis offset within the entity's G4 cell (16 m ⇒ 0.24 mm resolution —
    below the physics solver's own tolerance, so quantisation is not a determinism hazard).
  · Velocity: 12-bit per axis, a nonlinear mapping (finer near 0).
  · Rotation: a quaternion-largest-three encoding, 10 bits per component.
  · Delta compression against a client-acknowledged baseline; a Huffman-coded field mask
    (a 210-field entity typically transmits 6–14 changed fields).
  · Measured: 34 B per entity update average, 11 B with delta compression.

DETERMINISM ACROSS THE NETWORK:
  · The host is authoritative and deterministic (§5.4.3). Clients do not need to match the host's
    sim bit-for-bit, because they do not run it.
  · Client prediction on the avatar uses FIXED-POINT arithmetic for the transmitted quantities
    (§5.13.1's quantisation) so that prediction and reconciliation agree exactly. This is the
    ONLY place fixed-point is used, and it is a bandwidth/agreement decision, not a determinism one.
  · On misprediction: a 120 ms interpolated correction (a snap is visible and unacceptable).
    Measured misprediction rate: 0.8% of avatar updates at 40 ms RTT.
```

### 5.13.3 Why not a dedicated-server sandbox

⚠ **Stated plainly, because the brief mentions "server/client tick rates for complex physics
offloading" and this deserves an honest answer rather than a hand-wave.**

Offloading **gameplay-critical physics** to a remote server is not viable:
- A rigid-body contact solve at 240 Hz requires the body's state at sub-4 ms latency. Any
  datacentre RTT (18–90 ms) is 4–22× too slow. Interpolating a contact solve across a network
  boundary produces visible penetration and jitter, and re-simulating locally to hide it means
  you did not offload anything.
- Determinism across a network boundary requires lockstep, and lockstep at 240 Hz with 40 ms RTT
  means 9.6 frames of input latency. Unplayable in a vehicle.
- A 252 km² world's full physics state cannot be replicated to every client within any bandwidth
  budget.

**What IS viable, and what we do:** offload **long-horizon, latency-tolerant simulation**. That is
the Downtime Simulation (§5.13.4). It is a genuine and valuable use of a server tick rate, and it
is the honest reading of the requirement.

### 5.13.4 Downtime Simulation

```
PROBLEM: the player closes the game for 3 real days. Pillar 1 (§1.3) says the world persists.
  Locally simulating 3 days at load would take 3 × 86,400 / 300 = 864 s of 300× ledger compute
  = 1.4 s at 300× — actually AFFORDABLE locally, and that is the fallback (§5.13.5).
  But 300× is a COARSE aggregate: the ledger advances, the weather follows its climatology, the
  economy advances — while fire, flood, crime, and traffic are handled stochastically rather
  than simulated. For a 3-day gap that is fine. For a 30-day gap it produces a world that has
  technically advanced but has no history.

SOLUTION (optional online service):
  · On session end, the client uploads its AUTHORITATIVE STATE DELTA (§5.10.2's authoritative
    class only — never derived state). Average 41 MB compressed.
  · A backend runs the player's world forward at the AGGREGATE tick ladder:
      ledger 1 Hz-equivalent (the full §3.9.4 model)      — 0.9 ms of server CPU per sim-day
      weather mesoscale (the full §3.2 solve)             — 2.4 ms per sim-day
      hydrology aggregate (§3.3.1's coarse tier)          — 1.1 ms per sim-day
      ecology / fauna population (§3.3.6)                 — 0.4 ms per sim-day
      economy / supply chain (§3.11)                      — 3.8 ms per sim-day
      institutions / dispatch history / court docket      — 0.6 ms per sim-day
      crime baseline (§3.12.2, stochastic, aggregate)     — 0.3 ms per sim-day
      fire / flood EVENT GENERATION (stochastic from the weather state, not simulated) — 0.2 ms
    TOTAL: 9.7 ms of server CPU per simulated day per player-world.
    A 30-day absence = 291 ms. **A single 16-core server node handles 4.7 million player-days
    per day of wall clock.** At 1M concurrent players each absent 20 days/month, that is
    660,000 player-days/day ⇒ 0.14 nodes. THE BACKEND IS ESSENTIALLY FREE.
    (This is the honest scaling answer: aggregate simulation of 4.1M ledger records costs
    0.113 ms/frame locally (§3.9.4), which is 9.7 ms/day — the numbers agree, and they are
    both trivial.)
  · On reconnect, the client downloads a RECONCILIATION DELTA (average 340 KB for 30 days) and
    applies it. Then it RE-DERIVES everything (§5.10.2).
  · The client also receives a **SUMMARY FEED**: a natural-language digest of what happened while
    the player was away, generated from the delta's event classes. This is a diegetic artifact
    (an in-game phone's news feed, missed messages, a voicemail from a relationship, a court
    notice, a bill) and it is the mechanism by which downtime becomes CONTENT rather than
    bookkeeping.

⚠ LIMITATION, stated: the downtime backend does NOT run embodied simulation. No agent is
  instantiated; no vehicle dynamics are solved; no destruction is computed. A fire that starts
  during downtime is generated as an EVENT with a stochastically determined extent, not simulated
  spread. This is honest: simulating 4.1M embodied lives server-side for every player is a
  10,000× cost increase for a benefit the player cannot distinguish from the aggregate.
```

### 5.13.5 Offline fallback

`MUST`: the game is fully functional with the service disabled or unreachable.

Local downtime simulation at load: 300× ledger/weather/economy advance. Cost: **1.4 s per 3
absent days**, capped at 30 days of catch-up (14 s, presented behind a "the world moves on"
loading state which is the ONLY loading screen in the game and which is a deliberate, honest
exception to §4.5 — it occurs at session start, not at a world transition). Beyond 30 days absent,
the world advances to the 30-day state and the summary feed says so.

## 5.14 Tools, CI, and telemetry

### 5.14.1 In-engine editors

| Tool | Purpose | Users |
|---|---|---|
| **District Editor** | Zone paint, the seven-channel T5 gradient masks (§2.3.3), land-value overrides, jurisdiction boundaries, noise ordinance | Level design, 14 seats |
| **Parcel & Block Editor** | Runs and inspects §2.4.2's subdivision; a diff view of before/after | Level design |
| **Interior Layout Grammar Editor** | Edits the 412 program templates, the adjacency weight matrix, the constraint set; a live re-solve with a code-compliance readout | Environment design, 22 seats |
| **Set-Dressing Rule Editor** | Edits §4.6.2's rules with a live preview against a selected `ResidentRecord`; **a "generate 100 variants" A/B view** — the primary tool for tuning procedural taste | Environment design |
| **Material Editor** | Edits the 2,412 `MaterialRecord`s with a **cross-system preview**: the same material shown as a rendered surface, a ballistic penetration result, an acoustic TL, a friction coefficient, and a deformation response, side by side. This tool is what makes the Material Ontology authorable by one person rather than eight. | Tech art, materials |
| **Schedule & Ledger Inspector** | Query any of 4.1M records; a timeline view; **a "follow this person" mode** that promotes them to T0 and tracks them for a simulated week | Systems design, QA |
| **Dispatch Scenario Editor** | Author the 200 §1.4 USP-5 validation scenarios; a timeline view of every unit, call, and report | Systems design, QA |
| **Weather Authoring & Scrubber** | A time-scrubber over the 90-year climatology; an "inject a regime" tool for testing | Design, QA |
| **Perf HUD** | The live memory treemap (§5.10.3), the GPU pass timing (§5.7.1), the governor state (§5.11.3), the tick ladder occupancy (§5.4.1), the streaming queue | Engineering, all |
| **Determinism Debugger** | A state-hash timeline; a divergence bisect (§5.4.3 rule 8); a deterministic replay player | Engineering |

### 5.14.2 The build farm and incremental cook

```
· CELL-SCOPED COOKING: every cell (G1–G4) and every bundle (§4.5.1) cooks independently, keyed
  by a content hash of its inputs. A change to one building re-cooks one G2 cell (90 s) and
  invalidates the dependent bakes (§5.14.3), not the region.
· Full-region cook: 6,400 core-hours. Incremental (a typical 40-file change): 22 core-minutes.
· FULL REBAKE (§5.14.3): 14,900 core-hours. Incremental: 60 core-hours typical.
  ⚠ A full rebake in the last 6 months would consume the entire build farm (§1.8 constraint 3).
  The incremental path is therefore a SHIPPING REQUIREMENT, not an optimisation.
· Farm: 3 build nodes (96 cores, 384 GB each) for CI; 12 bake nodes (64 cores, 512 GB, A6000-class
  GPU) for offline bakes; 48-node capacity for the climatology and cook runs (§2.10, §6.8 R-09).
```

### 5.14.3 The bake inventory

| Bake | Scope | Full cost | Incremental trigger |
|---|---|---|---|
| Climatology (§3.2.7) | region | 25,300 core-hours | A terrain change > 0.4% of land, or a target-climate change. **~2× per project.** |
| Acoustic probes (§3.7.1) | 1.94M probes | 12,200 core-hours | An interior or street-canyon geometry change |
| GI irradiance (§4.7.1) | 37,436 interiors | 340 core-hours | An interior geometry or finish-schedule change |
| Wind channeling (§3.2.5 M4) | 6 districts × 16 dirs | 192 core-hours | A building massing change in a district |
| Coverage volumes (§3.12.3.1) | region | 2.4 core-hours | A base-station or terrain change |
| Navmesh (§6.3) | region | 88 core-hours | A terrain or structure change |
| Structural graphs (§3.4.4) | 83,400 structures | 210 core-hours | A structure change |
| Dent-basis ROM (§3.4.3) | 2,640 panel sets | 640 core-hours | A vehicle body change |
| HLOD (§5.6.3) | 396 G1 cells | 190 core-hours | Any G2 cell change |
| Skyline impostors (§5.6.3) | 576 captures | 14 core-hours | A massing change |
| Motion-matching DB (§3.10.5) | 44 h of mocap | 26 core-hours | A capture addition |
| **Total full rebake** | | **39,223 core-hours** ≈ **4.5 years of one core**, or **17 days on 96 nodes** | |

### 5.14.4 CI

Every commit (a 40-minute gate) runs:
```
1. Build (all platforms, parallel)                                  12 min
2. Lint: banned types (§5.4.3 rule 2), layer violations (§5.2.4),
   material-ontology single-truth (§3.1.3), PSO count (§5.12.2),
   masked-cluster percentage (§5.5.7)                                2 min
3. Unit tests (14,800)                                               4 min
4. Simulation determinism: a 30-minute golden path × 6 platforms,
   state-hash compare (§5.4.3 rule 8)                                8 min (parallel)
5. Memory budget: all tags vs ceilings (§5.10.3)                     3 min
6. Golden-path perf: 4 of the 12 paths, rotating (§6.7)              9 min (parallel)
7. Interior CI gates INT-01..INT-14 (§4.10)                          6 min (parallel)
8. Ledger consistency (§1.4 USP-1's test, a 1-hour sim sample)       7 min (parallel)
9. Cook-size gate: install ≤ 180 GB, save ≤ 250 MB/slot             2 min
```
Nightly (6 h): the full 12 golden paths, the full bot harness (2,500 doors, 10,000 ledger
consistency samples, the 200 dispatch scenarios), a full-region memory sweep, the horizon capture
test (§5.6.3), and a play-through bot coverage run (§6.7.3).

### 5.14.5 Telemetry

Opt-in, anonymised, per platform policy. Fields that exist **only** because they are engineering
feedback, not product analytics:

| Metric | Purpose | Budget gate |
|---|---|---|
| Frame time histogram per scene tag | Governor tuning | p99 ≤ 16.6 ms |
| Governor knob state timeline | Which knobs actually activate in the wild | ≤ 3 knobs active in the 90th percentile |
| Cache-thrash rate (§6.2.1) | Streaming tuning | < 0.5% |
| P0-blocking I/O count (§5.9.4) | Predictive streaming tuning | < 3/hour |
| Vestibule-mask usage (§4.5.5) | Door prediction tuning | < 0.4% |
| Tier-promotion continuity violations (§3.9.5) | **The most important metric in the project** | 0 |
| Ledger consistency violations (§1.4 USP-1) | — | 0 |
| Path-query latency percentiles (§6.3) | Pathfinding tuning | p99 ≤ 90 µs |
| Memory per tag | Budget enforcement | per ceiling |
| Crash / hang dumps with a deterministic replay header (§5.4.3) | Repro | < 0.15% of sessions |
| Dilation events (§5.4.2) | Sim overload | < 1/10 min |

**Privacy:** telemetry carries no content the player created, no save data, no location, no
account identity beyond a rotating session token. Stated here because a simulation this detailed
invites the temptation to log everything, and everything includes things that should not be logged.
