# Appendix C — Budgets, Ceilings, and the Zero-Sum Ledger

Every number in this appendix is a **ceiling**, not an estimate. A ceiling has an owner, a
measurement method, and a documented consequence for exceeding it. Where a design decision in
§1–§6 forces a number here, the forcing section is cited.

**The zero-sum rule.** This project has no unallocated resources. Any new feature that consumes
memory, milliseconds, disk, or I/O bandwidth `MUST` name what it displaces, and the displacement
`MUST` be recorded in §C.15 before the feature is approved. "We'll optimise it later" is not a
displacement. The overage protocol is §C.16.

---

## C.1 Canonical constants (single source of truth)

These are the numbers the whole document set is consistent against. A change to any of them is a
**Tier-1 change** (§0.6) requiring re-derivation of every dependent table.

| Constant | Value | Derived from | Consumed by |
|---|---|---|---|
| Codename | **MERIDIAN** | — | all |
| Engine | **STRATA** | §5.1 | all |
| Region | **New Vermillion** | §2.1 | all |
| Metro | **Vermillion City** | §2.5.1 | all |
| Playable land area | **252.0 km²** | the §2.2 district table | USP-6(a), §6.3 |
| Inland water area | **78.0 km²** | §2.2 | §2.9, §3.3.1 |
| Bounding box | **22.4 × 18.6 km** (416.64 km², 60.48% fill) | §2.1 | §5.8.1, Appendix B.12 |
| Reference competitor area | GTA V: **78 km²** ⇒ **3.23×** | §1.4 USP-6(a) | §1.4 |
| Elevation range | **Z = −120 m … +2,480 m** | §2.6.1 | §3.2.4, §6.3.4 |
| Highest peak | Sable Range, **2,410 m** | §2.5.6 | §2.6.1 |
| Total structures | **83,400** | §2.2 | §4.3, §5.10.3 |
| Enterable structures | **37,436 (44.9%)** | §2.2 | §1.4 USP-3, §4.2 |
| Interior classes S/A/B/C/D | **180 / 1,420 / 13,400 / 19,200 / 3,236** | §4.2.1 | §4.4, §C.8 |
| Population (Resident Continuum) | **4,127,000** | §3.9.1 | USP-1, §C.9 |
| External ledger records | **9,400,000** | §2.11 | §6.4 #6 |
| Material Ontology records | **2,412** | §3.1.2 | §3.4, §3.5, §3.6, §3.7, B.5–B.7 |
| Layer-stack assemblies | **615 building** (312 wall / 94 roof / 61 floor / 148 glazing) **+ 74 vehicle panel = 689 total** | §3.4.2 | B.5, B.7, §4.3 |
| Districts / zones / parcels | **15 / 41,200 / 187,400** | §2.4, §2.5 | A.1, A.2 |
| Census tracts | **412** | §3.9.2 | §3.12.4 |
| Road-graph links / edges | **18,400 links / 412,000 OD edges** | §2.8, §3.11.2 | B.8, B.13 |
| Freeway + arterial spine | **264 km** | §2.8 | §5.8.5 |
| Subsurface network | **681 km** | §2.6.3 | §6.3.4 |
| Simulation epoch | **2031-01-01T00:00:00Z** | §2.1 | §5.4.3 |
| Coordinate policy | F64 world, +X east / +Y north / +Z up, **rebase at 4,000 m**, F32 cell-local render | §2.1 | §6.5 |
| Frame contract | **60 FPS ⇒ 16.67 ms** | §1.4 USP-6(b) | §C.3, §C.4 |
| Simulation tick | **60 Hz fixed** (never variable) | §5.4.2 | all |
| Install size | **≤ 180 GB**; actual **179.1 GB** | §5.10.3 | USP-6(d) |
| Save size | **≤ 250 MB/slot**; measured **196 MB** | §5.10.4 | §C.12 |
| Minimum sustained drive | **≥ 2.4 GB/s** (a hard PC requirement, no HDD path) | §5.9.2 | §6.6.5 |
| PSO set | **2,210 closed, zero runtime creation** | §5.12.4 | AI-11 |

---

## C.2 Memory budget

### C.2.1 Resident memory by platform (MB)

The authoritative table is §5.10.1; reproduced here as the budget record.

| Subsystem | PS5 | XSX | Series S | PC Low | **PC Mid** | PC High |
|---|---|---|---|---|---|---|
| VT texture pool (physical pages) | 2,950 | 3,200 | 1,650 | 2,100 | 2,950 | 4,600 |
| Virtualized geometry clusters | 1,650 | 1,780 | 980 | 1,150 | 1,650 | 2,600 |
| Far-field terrain, HLOD, impostors | 190 | 210 | 140 | 160 | 190 | 340 |
| Terrain clipmap + RVT source layers | 140 | 160 | 96 | 110 | 140 | 240 |
| Interiors (private + kit atlas) | 640 | 700 | 380 | 480 | 640 | 900 |
| GI: global SDF clipmap + surface cache | 385 | 440 | 214 | 290 | 385 | 812 |
| Shadow: VSM physical pool | 448 | 512 | 256 | 320 | 448 | 768 |
| RTs, temporal history, GBuffer, vis buffer | 720 | 810 | 430 | 540 | 720 | 1,180 |
| Mutable world-state slabs + journal | 192 | 210 | 112 | 144 | 192 | 320 |
| **Resident records (ledger)** | **372** | **372** | **372** | **372** | **372** | **372** |
| AI runtime | 210 | 228 | 134 | 168 | 210 | 340 |
| Physics | 460 | 500 | 290 | 340 | 460 | 760 |
| Navigation (navmesh + graphs) | 64 | 72 | 52 | 56 | 64 | 110 |
| Animation | 340 | 380 | 196 | 240 | 340 | 620 |
| Audio | 520 | 560 | 340 | 400 | 520 | 880 |
| VFX / particles | 320 | 350 | 180 | 220 | 320 | 540 |
| Scripting / gameplay state | 380 | 400 | 300 | 340 | 380 | 520 |
| Streaming I/O + decompression scratch | 640 | 700 | 420 | 480 | 640 | 1,024 |
| Engine systems, code, RHI | 700 | 740 | 560 | 620 | 700 | 900 |
| Vehicle dent-basis panel sets | 110 | 130 | 70 | 84 | 110 | 190 |
| **SUBTOTAL** | **11,431** | **12,454** | **7,172** | **8,614** | **11,431** | **18,016** |
| Available to game | 12,500 | 13,500 | 9,700 | 12,000 | 16,000 | 22,000 |
| **RESERVE** | **1,069** | **1,046** | **2,528** | **3,386** | **4,569** | **3,984** |
| Reserve % | 8.6% | 7.7% | 26.1% | 28.2% | 28.6% | 18.1% |

### C.2.2 The reserve is not free

The reserve has a **committed allocation policy**, not a "hope it's enough" policy:

| Claim on reserve | Peak MB (PS5) | Source |
|---|---|---|
| Fast-travel burst staging (3.5 GB read at 6.0 GB/s peak) | 420 | §5.9.5 |
| Structural collapse event (debris spawn + SDF re-voxelise + a new GI/acoustic set) | 260 | §3.4.4 |
| Mass-incident dispatch spike (400 orders, 526 units, 8 air assets, a cordon min-cut) | 84 | §3.12.5 |
| Interior class-shift transient (a Class S pre-warm overlapping a Class S active) | 180 | §4.4.3 |
| Allocator fragmentation allowance | 125 | §6.9 |
| **Committed** | **1,069** | **= the entire reserve** |

⚠ **HONEST LIMITATION.** The PS5 reserve is fully committed. This means a *second* simultaneous
spike (a collapse **and** a fast travel) is not covered by headroom — it is covered by the
governor's memory-response mode (§5.11.3), which sheds pools in the ladder order. That is a real
behaviour with a real cost (a visible quality step), and §6.9 quantifies the fragmentation
component. The design choice is to spend the reserve on spikes rather than on fidelity, because
an OOM is a crash and a quality step is a frame.

### C.2.3 Pool pressure thresholds

Every pool in §C.2.1 has three numbers, shipped as data (A.9's `GovernorRule`):

```
target            = the §C.2.1 line item                     (the budget)
release_threshold = 0.84 × target                            (evict down to here)
engage_threshold  = 0.92 × target                            (start evicting here)
hard_cap          = 1.00 × target                            (a load failure, not a slow-down)
```

The 8% hysteresis band between engage and release is what prevents eviction oscillation
(§6.2.2 #2). `eviction_yield` below 0.94 (§6.2.1) means the pool is over-committed and a
**governor memory response** fires: the pool's fidelity knob steps down before any other pool is
touched. Pools never steal from each other silently — a steal is an explicit, logged, and
telemetered event with a named donor.

### C.2.4 Authoritative vs derived

| Class | Definition | Persistent? | Resident |
|---|---|---|---|
| **Immutable authored** | Cooked content, chunk-addressed, BLAKE3-hashed | — | on disk, 179 GB |
| **Authoritative mutable** | The delta journal + slabs; the ledger; the player; save state | **Yes** | 192 + 372 MB |
| **Derived** | Anything recomputable: traffic densities, weather cell state, puddle depth, SPL grid, coverage state, navmesh, HZB, GI cache, VSM pages, animation pose, T3 density aggregates, flow fields | **No** | ~2.1 GB |

The derived class is 2.1 GB of the 11.4 GB PS5 subtotal (**18%**). It is the single largest reason
a save is 196 MB and not 2 GB, and the rule that makes a save loadable by a different game version
(§5.10.2).

---

## C.3 GPU frame budget (PS5, 1440p internal → 4K TSR)

Authoritative line items: §5.7.1. Summary:

```
UNCONSTRAINED PIPELINE TOTAL                     16.80 ms   (100.8% of frame)
STEADY-STATE SHIP TARGET                         15.40 ms   ( 92.4% of frame)
VARIANCE HEADROOM                                 1.27 ms   (  7.6%)
THE GAP, closed by five default-down knobs:      −1.40 ms
   1  Water planar-reflection refresh 60→40 Hz   −0.21 ms   invisible
   2  TSR accumulation ceiling 32→24 frames      −0.11 ms   invisible
   3  Bloom mip chain 7→6                        −0.06 ms   invisible
   4  GI screen-probe spacing 16→20 px           −0.58 ms   low   (SSIM 0.994 vs reference)
   5  Cloud high-res cone fraction 18→9%         −0.44 ms   low   (restored at <12°/s camera)
```

**Largest single consumers:** GI 2.20, material resolve 1.80, TSR 1.40, clouds 1.40, software
raster 1.31, VSM shadows 1.30. These six are 56% of the frame, and they are the six the governor
reaches first when a hard scene (§5.11.4) exceeds target.

**Platform GPU budgets (ms, steady-state):**

| Platform | Internal resolution | Upscaler | GPU budget | Headroom |
|---|---|---|---|---|
| PS5 | 1440p | TSR → 4K | **15.40** | 1.27 ms |
| PS5 Pro (if present) | 1728p | TSR → 4K | 15.40 | 1.27 ms |
| Xbox Series X | 1584p | TSR → 4K | 15.40 | 1.27 ms |
| Xbox Series S | 1080p | TSR → 1440p | **15.90** | 0.77 ms (knobs 1–9 default down two further steps) |
| PC Low (RX 6600 / RTX 3050) | 1080p | TSR → 1440p | 15.90 | 0.77 ms |
| PC Mid (RX 7800 XT / RTX 4070) | 1440p | TSR → 4K | 15.40 | 1.27 ms |
| PC High (RX 7900 XTX / RTX 4090) | 2160p native or 1440p + TSR | optional | 15.40 | 1.27 ms, spent on 4K native |

Series S and PC Low are **not** the same build with lower settings — they run the same code with
different `GovernorRule` defaults (§A.9) and a different internal resolution. The 0.77 ms headroom
is thinner because their GPU is proportionally the tighter resource, and §5.11.4's hard scenes are
gated per platform, not globally.

---

## C.4 CPU frame budget (PS5, 8 Zen 2 cores, ~1 core to the OS)

Authoritative: §5.11.1.

| Pool | Threads | Budget | Notes |
|---|---|---|---|
| Game thread | 1 pinned | **3.60 ms** | the critical path |
| Render thread | 1 pinned | **3.40 ms** | visibility setup + command recording |
| RHI / submit | 1 | **1.00 ms** | |
| Simulation (P1a + P1d) | 3 | **3.00 ms** | weather/hydro/ecology/fire + residents/traffic/dispatch |
| Physics (P1b + P1c) | 2 | **2.40 ms** | ⚠ measured 2.72; resolved in §5.4.4 |
| Animation (P1e) | 3 | **2.20 ms** | §3.10.5 |
| Audio (P1g) | 3 | **1.60 ms** | ⚠ measured 1.67; resolved in §3.7.8 |
| Streaming / I/O / housekeeping (P5) | 2 | **1.20 ms** | §5.8.3 |
| Scene update (P2) | 4 | **0.86 ms** | |
| VFX (CPU side) | shared | **1.80 ms** | from the simulation pool's slack |
| UI / HUD | shared | **0.50 ms** | from the game thread's slack, in a job |
| **Aggregate worker demand** | **15** | **19.56 ms** | ~1.30 ms/thread average |
| **Critical path (wall)** | | **5.04 ms** | P0 0.28 → P1f 3.60 → P1-commit 0.30 → P2 0.86 |
| **Render-thread path** | | **4.40 ms** | P4 3.40 → submit 1.00 |
| **Headroom vs 16.67 ms** | | **11.63 ms (70%)** | the CPU is not the constraint |

**The 11.63 ms of CPU headroom is pre-spent**, not banked:

| Claimant | ms | Why |
|---|---|---|
| Hard scenes (§5.11.4, HS-01…HS-08) | 3.10 | the worst-case frames are worse than the average by design |
| Series S, where the CPU:GPU ratio differs | 2.40 | a proportionally tighter GPU means a longer render thread relative to the sim |
| Simulation scale-up (more agents, more destruction) | 2.80 | the growth headroom for T0 40→56 and V1 8→12 |
| PC-Low, where a weaker GPU lengthens the render thread | 1.90 | |
| Genuinely unallocated | 1.43 | 8.6% of the frame — the number a new feature must fit inside or displace |

### C.4.1 Tick-rate ladder cost

17 distinct rates (§5.4.1), scheduled by a hierarchical timing wheel at **0.02 ms/frame** of
overhead. Per-tick amortised CPU cost of the low-rate systems:

| Rate | Amortised ms/frame | Systems |
|---|---|---|
| 1,000 / 500 / 240 / 120 Hz | 1.56 | vehicle tyre & powertrain, driveline torsion, suspension kinematics, chassis, projectile integration (2 kHz) |
| 60 Hz | 2.94 | the frame contract: world physics, T0 AI, audio events |
| 20 / 15 / 10 Hz | 0.62 | T1 AI, motion matching, cloth, acoustic tracing, T2 AI, puddle solve, hydrology |
| 5 / 4 / 2 Hz | 0.31 | T2 ORCA, SPL grid, collapse solve, door-entry prediction, wind field |
| 1 Hz | 0.19 | the ledger tick, macro CTM, investigation state machines, lighting/appliance/security controllers, **the governor control loop** |
| 1/30 – 1/86400 Hz | 0.14 | weather advection, orography, fire spread, snowpack, hydrology aggregate, evidence decay, HVAC, economy, fauna, public opinion |
| **Total** | **5.76 ms** | spread across worker pools; the wall-clock critical path stays 5.04 ms |

---

## C.5 Latency gates

Every one of these is measured in CI on a fixed rig, not estimated. A gate that fails blocks the
build (§5.14.4).

| Gate | Target | Measurement | Blocking? |
|---|---|---|---|
| Frame time, steady state | ≤ 16.67 ms p99 over 3,600 frames | an in-engine profiler on a golden path (§6.7.1) | yes |
| Frame time, hard scene | ≤ 16.67 ms p99 with knobs 1–22 active | §5.11.4's 8 scenes | yes |
| **Input-to-photon** | **≤ 45 ms at the display** | a photodiode rig; ≈ 40 ms console, ≈ 33 ms at 120 Hz | yes |
| Perceived vehicle/aim latency | ≤ 24 ms | the same rig, with P0 sub-frame prediction (§5.3) | no (advisory) |
| Fast travel, any two points in 252 km² | **≤ 4.5 s** | a scripted harness, 200 random pairs | yes |
| Cold boot to playable | **≤ 22 s** | a timed launch, SSD | yes |
| Save write | ≤ 250 ms | instrumented | yes |
| Save load | ≤ 3.4 s | instrumented | yes |
| Cell load, P0-blocking events | **< 3 per hour** | telemetry (§6.2.1) | yes |
| Cache thrash rate | **< 0.5%** | §6.2.1's definition, per golden path | yes |
| VT page oscillation | < 1.2% | §6.2.1 | yes |
| Mean cell dwell | > 22 s | §6.2.1 | yes |
| Speculative pre-warm hit rate | > 0.62 | §6.2.1 | no (advisory) |
| Eviction yield | > 0.94 | §6.2.1 | yes |
| **Pathfinding p99, L0/L1/L2** | **≤ 90 µs** | §6.3.6 | yes |
| Pathfinding p99.9, L3 cost field | ≤ 720 µs | §6.3.6 | yes |
| Flow-field build, amortised | ≤ 0.084 ms/frame | §6.3.6 | yes |
| Crime-report delay, no-coverage case | ≥ 4.5 min to a 911 dispatch | §3.12.3, GP-05 | yes (a gameplay gate) |
| 15 km horizon silhouette continuity | ≤ 0.42 px impostor parallax error | GP-07 (§6.7.1) | yes |
| LOD pop, perceptual | ≤ 0.5 px geometric error at the crossover | §5.5.4, GP-06 | yes |
| Determinism | a 600-tick replay hash match, 100% of runs | §5.4.3 rule 8, §6.5 | yes |
| Long-session stability | no leak > 12 MB/h over a 12-hour run | §6.9 | yes |

---

## C.6 Disk and install budget

Authoritative derivation: §5.10.3.

| Component | GB |
|---|---|
| World content — 4.1M unique chunks, 148 GB pre-compression, post BLAKE3 dedup + Oodle | **92.0** |
| Derived bakes: GI 3.9, acoustics 8.1, coverage 0.7, wind channelling 0.24, climatology 0.34, navmesh 1.2, structural graphs 0.088, dent basis 1.9, HLOD 2.4, impostors 0.084 | **18.9** |
| Animation (ACL-compressed motion-matching DB + rigs) | **1.21** |
| Audio (≈ 23,100 assets) | **4.8** |
| Video (a 4-minute title sequence + 62 minutes of cutscene) | **9.4** |
| Code, shaders, the PSO database, manifests | **3.2** |
| Localised text/VO (12 languages; a 41 GB VO set with 68% shared stems) | **27.6** |
| **Subtotal** | **157.1** |
| Filesystem / alignment overhead (14%) | 22.0 |
| **SHIPPED INSTALL** | **179.1** |
| **Ceiling** | **180.0** |
| **Remaining** | **0.9 GB (0.5%)** |

⚠ **The install budget has 0.9 GB of slack.** That is the tightest budget in the project and it is
tight on purpose: a loose install budget is how a project ends up at 240 GB. §6.1's resolution
policy governs every addition:

- A new asset enters only through **dedup** (a BLAKE3 hash collision with an existing chunk) or
  through **displacement** (naming what it replaces).
- Optional high-resolution texture pack (PC-High only, 4K→8K on 200 hero assets): **+38 GB**,
  a separate download, never in the base install.
- Cook-time lints (§6.1.2): `TEX-01/02`, `GEO-01/02`, `AUD-01`, `ANIM-02` — build-blocking.
  `TEX-01` alone (auto-downgrading the 3 highest mips of any asset never requested in a 40-hour
  telemetry playthrough) qualifies 14% of assets and saves **6.2 GB**.
- The **exposure-efficiency gate** (§6.1.3): `MB per player-minute-of-exposure`, reported per
  discipline monthly. A 2 MB asset seen for 12 s in 3 interiors scores 3.3 MB/min (poor); a 200 KB
  kit prop seen 8,000 times scores 0.002 (excellent). An asset with zero measured exposure in a
  64-bot coverage run (§6.7.3) is flagged for cut. This is how a 252 km² world avoids shipping
  40 GB nobody sees.

**Per-category sub-budgets within the 92 GB of world content:**

| Category | GB | Share |
|---|---|---|
| Building exteriors (83,400) | 27.4 | 29.8% |
| Interiors (37,436, procedural + kit) | 18.6 | 20.2% |
| Terrain, RVT source layers, splat | 11.2 | 12.2% |
| Vegetation (2.1M rendered instances, 340 species) | 9.8 | 10.7% |
| Props, set dressing, street furniture | 8.4 | 9.1% |
| Vehicles (340) + weapons + characters | 6.1 | 6.6% |
| VFX, fluids, sky, water | 4.2 | 4.6% |
| UI, fonts, icons | 1.4 | 1.5% |
| Manifests, indices, the cell-manifest table (118 MB) + world index (163 MB) | 0.9 | 1.0% |
| Slack | 4.0 | 4.3% |

This is an **asset-category** decomposition of the cooked world content. It is orthogonal to
§6.1.3's **discipline-allocation** decomposition, which is against the *shipped* 180 GB ceiling and
includes the bakes, code, and localisation that the table above excludes:

| Discipline allocation (§6.1.3, monthly, build-blocking at +40 MB per changeset without a named offsetting reduction) | GB |
|---|---|
| Environment | 74 |
| Character / animation | 22 |
| Audio | 5 |
| VFX | 8 |
| Video | 10 |
| Systems / bakes | 24 |
| Code / localisation | 37 |
| **Allocation total = the install ceiling** | **180.0** |
| **Actual** | **179.1** |

Allocations sum to the ceiling, not to the actual; that 0.9 GB difference is the project's entire
remaining disk margin (§C.6). The two decompositions do not sum to each other and are not supposed
to — a discipline owns bakes and code as well as assets.

---

## C.7 Streaming and I/O throughput budget

| Platform | Raw drive | Effective decompressed | Decompression path |
|---|---|---|---|
| PS5 | 5.5 GB/s | ~9 GB/s (Kraken-typical 1.6×) | the IO coprocessor, off the CPU entirely |
| Xbox Series X | 2.4 GB/s | 4.8 GB/s | DirectStorage + a hardware block |
| Xbox Series S | 2.4 GB/s | 4.8 GB/s | same |
| PC (Gen4 NVMe) | 5.5–7.4 GB/s | DirectStorage 1.2 with GPU BCn decode, or 3.8 GB/s on 2 CPU threads with Zstd | varies |

```
SUSTAINED STREAMING READ     3.2 GB/s (PS5, XSX)  |  4.0 GB/s (PC-Mid)  |  2.4 GB/s (PC-Low, Series S)
PEAK BURST (fast travel)     6.0 GB/s for ≤ 2.5 s
MAX INFLIGHT REQUESTS        32 (console)  |  48 (PC)
REQUEST COALESCING           floor 16 KB, ceiling 256 KB
MIN CHUNK READ SIZE          16 KB (a 4 KB read is a scheduler bug, not a small asset)
ONE QUEUE                    streaming + VT + audio + animation + journal offload share a single
                             priority-ordered queue (§6.2.2 #6). Competing per-subsystem bandwidth
                             allocations are FORBIDDEN — that is cause #6 of the ten thrash causes.
CHUNK CACHE                  64 MB LRU, chunk-level dedup in the inflight set (§6.2.2 #7).
                             Without it, 37,436 interiors referencing 4,200 shared props produce a
                             9× read amplification.
PRIORITY CLASSES             P0 blocking-the-player | P1 visible-this-frame | P2 visible-soon |
                             P3 VT feedback | P4 audio/animation | P5 journal offload
                             P5 is throttled to zero while any P0–P2 is outstanding (§5.9.4).
```

**Bandwidth accounting at steady state (PS5, 3.2 GB/s):**

| Consumer | MB/s | Share |
|---|---|---|
| G3/G4 cell streaming (geometry, textures, interiors) | 1,240 | 38.8% |
| VT page faults (8,400 pages/s sustained) | 780 | 24.4% |
| Audio banks + streamed ambience | 210 | 6.6% |
| Animation pages | 180 | 5.6% |
| Acoustic probe pages (a region load) | 340 | 10.6% |
| Journal offload (P5) | 90 | 2.8% |
| Derived-bake page-ins (navmesh, coverage, GI SDF) | 260 | 8.1% |
| Headroom | 100 | 3.1% |
| **Total** | **3,200** | **100%** |

The 3.1% headroom is the reason PC-Low and Series S run at a 2.4 GB/s budget with the same
consumer list scaled by 0.75 — and the reason **no HDD fallback is built** (§6.6.5): at 140 MB/s
this table does not fit at any fidelity, and supporting it would mean designing a second streaming
system.

---

## C.8 Interior memory budget

### C.8.1 Per-interior resident ceilings (§4.4.2)

| Class | Count | **Ceiling** | Composition highlights |
|---|---|---|---|
| **S — Hero** | 180 | **220 MB** | VT working set 96.0, GI bake 18.0, acoustic probes 6.2, convolution IRs 5.5, ≤ 220 dynamic lights 12.0, set dressing + mutable state 24.0, audio banks 12.2, geometry/collision/misc 46.1 |
| **A — Signature** | 1,420 | **95 MB** | VT 38.0, GI 8.0, probes 3.4, ≤ 60 lights 5.0, dressing 16.0, banks 6.0, geo/misc 18.6 |
| **B — Stock commercial/civic/industrial** | 13,400 | **34 MB** | VT 11.2, GI 2.4, probes 1.1, ≤ 24 lights 1.6, dressing 9.0, banks 2.3, geo/misc 6.4 |
| **C — Stock residential** | 19,200 | **16 MB** | VT 9.8, GI 0.62, probes 0.35, ≤ 6 lights 0.04, dressing 0.58, banks 1.6, geo/misc 3.01 |
| **D — Utility/back-of-house** | 3,236 | **6 MB** | VT 2.4, GI 0.14, probes 0.08, ≤ 2 lights 0.01, dressing 0.06, banks 0.72, geo/misc 2.59 |

**The marginal cost of the 19,200th apartment is 3.5 MB** (§4.4.2) — because the shared Interior
Kit Atlas and the streamed VT pool are already paid for. That ratio is what makes 37,436 interiors
affordable, and it is the single most important number in the interior strategy.

### C.8.2 Global interior budget: 640 MB (PS5 / PC-Mid)

| Claimant | MB |
|---|---|
| Shared Interior Kit Atlas (3 resident style families; 4,200 props, the 615 building assemblies) | **180** |
| Active interior, worst case Class S | **220** |
| Pre-warm ring, 4 slots × ≤ 40 MB private | **160** |
| Bundle shells (paired exterior shells, §4.5.1) | **40** |
| Eviction hysteresis reserve (recently-evicted bundles held 2.0 s against ping-pong) | **40** |
| **Total** | **640** |

The pre-warm ring is capped at **40 MB private per slot regardless of class** — a pre-warmed
Class S loads its geometry and GI but at a −1.5 mip texture bias. Without that cap the ring alone
could reach 880 MB.

### C.8.3 Platform variants

| Platform | Global interior budget | Class shift |
|---|---|---|
| PS5 / PC-Mid | 640 MB | none |
| XSX | 700 MB | none; +1 pre-warm slot |
| Series S | 380 MB | kit atlas 110 MB (2 families), 2 pre-warm slots × 24 MB, **every class shifts one down** (S→A, A→B, B→C, C→D; D stays D) |
| PC Low (8 GB VRAM) | 480 MB | class shift as Series S; VT pool 2,100 MB |
| PC High (16 GB+) | 900 MB | +2 pre-warm slots, full-mip active interior, 4 kit families resident |

**Interior count is unchanged on every platform. Only fidelity shifts.** A Series S player can
enter all 37,436 interiors (§1.4 USP-3 is not a fidelity feature).

### C.8.4 Authoring vs generation

| | Count | Disk |
|---|---|---|
| Bespoke / hand-tuned interiors (S + A) | 1,600 | 105.6 GB of the 18.6 GB interior line is *not* how this reads — the 18.6 GB is the **cooked** interior content; bespoke interiors dominate it |
| Procedural interiors (B + C + D) | 35,836 | **6.5 MB — 180 bytes each** (A.4: seed, archetype ID, program-template ID, occupant refs, override list) |

The 6.5 MB line is the architectural payoff (§4.4.4). Storing 35,836 authored interiors at even
120 KB each is 4.3 GB of disk, 4.3 GB of I/O, and 35,836 assets somebody has to review and rebake.
Generating them costs **18 minutes of cook time** and 6.5 MB of overrides.

---

## C.9 Simulation entity ceilings

Concurrent counts at steady state. Every one is a hard cap enforced by deterministic eviction or
by refusal, never by silent degradation.

### C.9.1 Residents (§3.9)

| Tier | Radius | **Ceiling** | Representation | Rate |
|---|---|---|---|---|
| T0 — Embodied | ≤ 64 m | **40** | full agent: BT + utility + GOAP-lite planner, full IK, cloth, facial | 60 Hz decision, 120 Hz locomotion |
| T1 — Near | 64–300 m | **260** | agent, no facial, motion matching at 20 Hz | 20 Hz |
| T2 — Crowd | 300 m–2 km | **3,000** | agent + ORCA-lite, no planner; VAT locomotion | 5 Hz |
| T3 — Aggregate | macro-region | **~180,000** | a density on a road/sidewalk link with an attribute histogram | 1 Hz |
| T4 — Ledger | everywhere | **~3.94M** | record only | 1 Hz batch, 1/86400 Hz economics |
| **Sum** | | **≈ 4,127,000** | | |

Governor knobs: T0 40→32→24, T1 260→200→150, T2 3,000→2,400→1,800→1,200 (knob 8, cost class
`low*` — the asterisk means a diegetic justification exists: fewer people on the street at 3 a.m.).

**Tier-change continuity invariants** (§3.9.5, CI-asserted by the promotion-continuity harness):
position error ≤ 0.6 m at T2→T1 and ≤ 0.1 m at T1→T0; activity preserved; plan cursor preserved;
no teleport in view of the player (§6.4 #1).

### C.9.2 Vehicles (§3.5.11)

| Tier | **Ceiling** | Rate | Model |
|---|---|---|---|
| V0 — Hero | **2** | 1,000 Hz tyre & powertrain, 240 Hz chassis | full §3.5.3–§3.5.8 |
| V1 — Near | **8** (→ 6 under the §5.4.4 physics resolution) | 500 Hz tyre, 120 Hz chassis | full model, 4-iteration contact |
| V2 — Traffic | **180** | 60 Hz | kinematic single-track (bicycle) + a lateral-load-transfer approximation + the same tyre friction model |
| V3 — Distant | **2,400** | 10 Hz | point mass along a road-graph edge, speed from the CTM |
| V4 — Abstract | **~9,000 in-region** | 1 Hz | a density in the macro traffic model |

Hysteresis: V3→V2 at 240 m, V2→V3 at 310 m; V2→V1 at 70 m, V1→V2 at 92 m.

### C.9.3 Physics (§5.4.4)

| Class | Ceiling | Rate | Solver |
|---|---|---|---|
| Tier 0 — hero | ≤ 64 bodies | 240 Hz | 8 iterations, CCD |
| Tier 1 — near | ≤ 900 | 120 Hz | 6 iterations, CCD on fast bodies (≤ 2 substeps) |
| Tier 2 — mid (debris, traffic) | ≤ 3,000 | 60 Hz | 4 iterations, no CCD, AABB + capsule |
| Tier 3 — far | ≤ 12,000 | 15 Hz | 2 iterations, no rotation, AABB only |
| Broadphase pairs | 9,400/tick | — | 0.42 ms |
| Counting-sort rebuild | 12,000 bodies | — | 0.06 ms |
| Live debris rigid bodies | 3,000 region-wide, **600 within 40 m of camera** | — | beyond: static rubble impostors (§3.4.5) |
| Concurrent structural collapses | **1** | — | a second queues (§3.4.4, §6.4 #11) |
| Structural graph nodes resident | 40 subgraphs of 14.2M total (6 MB of 88 MB) | — | §3.4.4 |

### C.9.4 Crime, dispatch, and investigation (§3.12)

| Entity | Ceiling |
|---|---|
| Crime events generated | ~1,840/sim-day (region-wide, from the ledger, not authored) |
| Active crime events | ≤ 4,000 |
| Concurrent dispatch orders | ≤ 400 |
| Dispatch units | 526 (412 patrol + 114 specialist) |
| PSAP centres / positions | 5 / **118** |
| Carrier cell sites | 412 (148 macro, 186 micro, 78 in-building DAS) |
| Police radio sites / repeaters | 34 / 9 |
| Communication points (non-mobile) | 7 call boxes, 34 roadside phones, 112 transit help points, 1,880 fire pull stations |
| Active evidence items | ≤ 16,800 (4,000 cases × avg 4.2) |
| Span of control per incident commander | 6 direct reports (beyond: a modelled coordination penalty) |
| Cordon hold time | 20 min – 8 h, then it stands down |

### C.9.5 Environment (§3.2, §3.3)

| Entity | Ceiling |
|---|---|
| Weather cells | 1,584 (a 400 m mesoscale grid) |
| Puddle simulation grids | 1 per simulation source, 256×256 at 0.5 m, radius 128 m, 10 Hz |
| Hydrology catchments / patches | 1,940 patches; fauna 312 species × 1,940 = 148k active pairs |
| Concurrent active fire cells | 4,000 (1/30 Hz, 64 m cells within 2 km of an ignition) |
| Wind field solve | 2 Hz over a 128 m lattice |

### C.9.6 Audio (§3.7)

| Entity | Ceiling |
|---|---|
| Concurrent voices | 512 (48 kHz) |
| Full-DSP-chain voices | **88** (design target 96; reduced in §3.7.8's documented resolution) |
| Reduced-chain voices (2-band EQ + gain + a send) | 424 |
| Acoustic probes | 1.94M region-wide (4.2 KB each = 8.1 GB disk, **240 MB resident per loaded region**) |
| Audio assets | ≈ 23,100 |
| Ray-traced reflection traces | 15 Hz |
| SPL grid | 5 Hz |

### C.9.7 Animation (§3.10.5)

| Entity | Ceiling |
|---|---|
| Motion-matching DB | 44 h at 30 fps, a 62-float feature vector, 124 B/frame ⇒ 5.9 GB raw |
| ACL-compressed on disk | **1.21 GB** |
| Resident subset | **340 MB** (chosen by the instantiated population's attribute distribution) |
| VP-tree visits per search | 900, then a graceful degradation to best-so-far |
| Searches per frame | 40 T0 at 60 Hz + 260 T1 at 20 Hz, across 4 threads ⇒ 0.175 ms/frame/thread |
| Hair strands | ≤ 6 hero characters, 12,000–48,000 guides → 90,000–360,000 interpolated |
| Cloth particles | 800–2,500 per T0 garment set |

---

## C.10 Navigation budget (§6.3)

**Resident: 64 MB** in the navigation pool + **30 MB** of crowd flow fields charged to the AI
runtime pool (210 MB).

| Component | MB |
|---|---|
| L0 contraction-hierarchy road graph + hub labels | 34.0 |
| L1 abstract region graph (HPA*) | 12.0 |
| L2 baked navmesh (within 256 m of sources, ~1.8 km²) | 11.0 |
| L3 runtime cost fields (4 m within 1.6 km, 2 m within 96 m) | 3.0 |
| Canopy navmesh | 1.4 |
| Dynamic carve scratch + dirty-tile queue | 1.6 |
| **Total** | **63.0** (+1.0 slack) |

⚠ **Documented overage and its resolution (§6.3.3).** Crowd flow fields at a naive 3 MB each ×
40 destination clusters = **120 MB**, which alone exhausts the 64 MB navmesh budget. Resolution:
fields are stored at 8 m within 96 m of the destination and 16 m beyond ⇒ **0.75 MB each ⇒ 30 MB**,
charged to the AI runtime pool. The coarsening is invisible because pedestrian steering already
includes ORCA local avoidance.

**Why a naive flat navmesh is impossible:** a 252 km² navmesh at 0.5 m triangles with a 4-byte
index is ~1.25 GB on disk and would need >600 MB resident — 10× the budget. The 4-level hierarchy
(L0 CH road / L1 HPA* abstract / L2 baked navmesh / L3 runtime cost field) is not an optimisation
on top of a navmesh; it is the only representation that fits.

---

## C.11 Cache budgets and thrash gates

```
GRID HIERARCHY (§5.8.1)
  G0  4,096 m   always resident        the far-field / horizon representation
  G1  1,024 m   3,200 m radius         terrain, HLOD, skyline impostors
  G2    256 m     620 m radius         building shells, road surfaces, vegetation fields
  G3     64 m     220 m radius         full exterior detail, foliage, props
  G4     16 m      96 m radius         mutable-state slabs, interiors, fine collision

  Total cells across the 5 grids: 1,734,172 ⇒ a 118 MB always-resident manifest table
  (0.9% of the PS5 budget; it is what makes the scheduler O(candidates) not O(cells))

CACHES
  Chunk cache (content-addressed, LRU)                 64 MB
  Cell score cache                                     per cell, 4-frame TTL (§5.8.3)
  VT page table + physical pool                        2,950 MB (§C.2.1)
  VT feedback buffer                                   1 frame, 59 MB at 1440p
  HZB                                                  2 levels, previous + current frame
  GI surface cache                                     385 MB (shared with the SDF clipmap line)
  VSM physical pool                                    448 MB
  Streaming staging + decompression scratch            640 MB
  Eviction hysteresis reserve (recently evicted)       40 MB (interiors) + per-pool bands
  Shared interior kit atlas                            180 MB
  Audio bank set cache                                 drawn from the 520 MB audio line
  Derived-bake cache (navmesh, coverage, GI SDF)       drawn from their own pools

EVICTOR
  GreedyDual-Size with Frequency (§6.2.3, Appendix B.1), ONE BATCH PASS per pool per frame.
  Constraint set: a minimum dwell of 2.0 s; a no-evict set (mission anchors, landmarks);
  no eviction of a cell containing a live Tier-0 physics island; no eviction of the bundle the
  player is inside; bundle-atomic eviction (§4.5.1) — never half an interior.
```

| Gate | Target | Consequence of breach |
|---|---|---|
| `thrash_rate` | **< 0.5%** | a golden-path failure; blocks the build |
| `reload_rate` | < 0.5% | as above |
| `p0_block_rate` | < 3/hour | as above |
| `vt_oscillation` | < 1.2% | the VT residency radius is too small for the traversal rate |
| `eviction_yield` | > 0.94 | the pool is over-committed ⇒ a governor memory response |
| `mean_dwell` | > 22 s | the radii are too small; §5.8.5's velocity scaling is mis-tuned |
| `speculative_hit` | > 0.62 | the predictor is wasting bandwidth; reduce speculation |

---

## C.12 Save / load budget

| Item | Budget | Measured |
|---|---|---|
| Save size per slot | ≤ 250 MB | **196 MB** |
| Slots | 3 rotated autosaves + 8 manual, each with a checksum | — |
| Save write | ≤ 250 ms | — |
| Load | ≤ 3.4 s | — |
| Ledger baseline (pre-generated, shipped) | 1.1 GB on disk | a load applies the journal to it |
| Ledger full generation (a new game) | 18 minutes of cook/compute | never at runtime |

**A save contains:** the authoritative mutable class only — the delta journal, the resident ledger
diff, the player state, the calendar/WorldTime, mission state, and a per-slot checksum. **It
contains none of the 2.1 GB of derived state** (§5.10.2). A load re-derives: traffic densities,
weather cell state, puddle depth, the SPL grid, coverage state, the navmesh, the HZB, the GI cache,
VSM pages, animation pose, T3 density aggregates.

**Recovery ladder on corruption (§6.4 #10):** (a) the previous autosave, (b) slot N−1, (c)
**partial recovery** — regenerate the ledger baseline from the world seed and apply whichever
journal entries validate, preserving the calendar and the player while losing recent world
mutations, (d) a new game. Tier (c) is tested nightly; it is the difference between "lost 40
hours" and "lost 20 minutes".

**Downtime reconciliation (§5.13.4, §6.10 C-13):** the World Settle Pass (Appendix B.10) runs as
part of a time advance, keyed on **authoritative WorldTime, never on real elapsed time** — otherwise
save → advance → load produces a different world than a continuous run. GP-11's hardest assertion.

---

## C.13 Cook, build, and CI budget

| Stage | Budget | Nodes |
|---|---|---|
| Full cook (all platforms) | ≤ 34 h | 12 bake nodes (64 cores, 512 GB, A6000-class) |
| Incremental cook (a content change) | ≤ 22 min | 3 build nodes (96 cores, 384 GB) |
| Interior generation (35,836 procedural floorplans) | 18 min | parallelised across bake nodes |
| Acoustic probe bake (1.94M probes) | 9.4 h | the single longest stage; incremental by cell |
| Coverage bake (RF propagation, §B.12) | 2.4 core-hours region-wide | — |
| GI bake (Class S + A interiors) | 6.1 h | incremental by bundle hash |
| Navmesh + CH + HPA* bake | 2.8 h | incremental by cell |
| PSO precache (2,210 pipelines) | 40 min | — |
| **Per-commit CI** | **≤ 38 min** to a green gate | — |
| Nightly full gate (12 golden paths + 15 harnesses) | ≤ 6 h | — |
| 64-bot coverage fleet run | ≤ 14 h | weekly |

Cook-time lints that **fail the build**: the install-size gate (≤ 180 GB), the save-size gate
(≤ 250 MB/slot), the per-interior memory gate (INT-01, ≤ the class ceiling), the procedural-interior
validation gates (INT-03, INT-05), the asset-bloat category gates (§6.1.3), and the determinism gate
(§6.5).

---

## C.14 Authoring-volume ceilings

These are **caps on how much content may exist**, not on how much is used. They are the defence
against §6.1's asset bloat at the authoring layer.

| Category | Ceiling | Rationale |
|---|---|---|
| Material Ontology records | **2,412** | 8 consumers each; a 2,413rd must displace one |
| Layer-stack assemblies | **615 building + 74 vehicle panel** | 312 wall / 94 roof / 61 floor / 148 glazing / 74 vehicle panel |
| Vehicle specifications | **340** | across 10 classes, 34 fictional marques (§6.6.1) |
| Weapon specifications | per §3.6.1 | — |
| Fauna species | **312** | × 1,940 patches |
| Interior archetypes | **412** | across 18 style families |
| Façade assets | **14,200** (68 MB geometry, 1.4 GB textures) | 40–190 modular pieces per family |
| Interior kit props | **4,200** | the shared atlas, 180 MB |
| Fallback floorplan templates | **54** | §B.11's solve-failure path: an 8.6% first-pass failure rate (a 91.4% yield), a **0.31% fallback rate at ship** after the 5-attempt re-solve budget (§4.3.1) |
| Audio assets | **≈ 23,100** | 4.8 GB cooked |
| Recorded dialogue lines | **14,800** | stratified by age × sex × region × emotion × context; **no per-resident VO** (§6.8 R-16) |
| Narrative-relevant residents with specific lines | **2,400** | of 4,127,000 |
| Adaptive music | 74 min, 9 stems, 340 transition points, 14 authored triggers | deliberately restrained |
| Collapse-capable structures | **8** | §3.4.4; R-03's kill criterion cuts this to 2 authored set-pieces |
| Structures with a "destroyed" GI/acoustic variant | **42** | 8 collapse-capable + 34 fire-exposed landmarks (§6.10 C-11) |
| Protected (unkillable-by-ambient-sim) residents | **2,400** | §6.4 #4 |
| Governor knobs | **26** | 25 in §5.11.3 + 1 in §6.3.5 |
| Kill-switches | **48** | §6.4 |
| Golden paths | **12** (GP-01…GP-12) | §6.7.1 |
| Verification harnesses | **15** | §6.7.2 |
| Coverage-fleet bots | **64** | §6.7.3 |
| Risk register entries | **24** (R-01…R-24) | §6.8 |
| Combinatorial edge cases | **14** (C-01…C-14) | §6.10 |

---

## C.15 The zero-sum ledger

Every row is a **documented trade**: if the left grows, the right shrinks, by the stated amount.
These are the relationships the project has agreed to in advance, so that a production decision in
month 34 does not have to invent one under pressure.

| If this grows… | …this shrinks | Exchange rate | Owner |
|---|---|---|---|
| VT pool | SVG geometry cluster cache | 1:1 MB, both draw from the same 4,600 MB envelope on PS5 | Rendering Lead |
| Interior class ceilings | the number of concurrent pre-warm slots | +10 MB active ⇒ −1 pre-warm slot (40 MB) | Interiors Lead |
| T2 crowd cap | GI probe spacing and cloud cone fraction | +600 T2 agents ≈ +0.38 ms ⇒ knobs 4 and 5 step down one | AI Lead / Rendering Lead |
| Structural collapse concurrency (1→2) | the debris body cap (3,000→1,800) | a second collapse costs 1.9 ms GPU + 260 MB reserve | Physics Lead |
| Live debris bodies | Tier-3 far body count | 600 near-camera is fixed; the far tier absorbs it | Physics Lead |
| Acoustic probe density | resident probe MB | +1 probe/100 m² ≈ +34 MB per loaded region | Audio Director |
| Navmesh bake resolution | L3 cost-field radius | 0.5 m → 0.35 m costs +180 MB; instead the L3 radius drops 1.6 km → 1.1 km | AI Lead |
| Install size | the derived-bake budget | 1:1 GB; the bakes are the only 18.9 GB with slack (acoustics 8.1 GB is the largest) | Technical Director |
| Localised VO languages | the shared-stem ratio | a 13th language at 68% sharing costs +2.3 GB | Audio Director |
| Resident record hot-set size | the number of platforms that fit the ledger | +20 B/record = +82 MB ⇒ the Series S ledger no longer fits at 372 MB | Technical Director |
| Save size | derived-state re-derivation on load | persisting puddle depth costs +40 MB/slot and +0.8 s load, for no benefit | Technical Director |
| Fast-travel time | the peak burst window | 4.5 s ⇒ 6.0 GB/s for 2.5 s; a 3.0 s target needs 7.4 GB/s, which Series S cannot do | Technical Director |
| Frame-time target (60→120 FPS mode) | the GPU budget per frame, 16.67→8.33 ms | knobs 1–22 all step to minimum, internal resolution drops to 1080p, GI becomes a 24 px probe grid, clouds become a single-pass march. **A 120 FPS mode is a different fidelity product, not a toggle.** | Rendering Lead |
| Cloud fidelity | GI probe spacing | both are ~0.5 ms and both are `low` perceptual cost; the governor orders GI first (knob 4 before 5) because clouds are more often in frame | Rendering Lead |
| Water planar reflections | nothing — this is the free knob | 0.21 ms for an invisible change; it is knob 1 for that reason | Rendering Lead |

---

## C.16 The overage protocol

When a measurement exceeds a ceiling in this appendix:

1. **Record it in the document, with the number.** Three overages are recorded this way and were
   resolved in design rather than discovered in production:
   - GPU pipeline 16.80 ms vs the 16.67 ms frame ⇒ §5.11.2, closed by five default-down knobs.
   - Physics 2.72 ms measured vs a 2.40 ms budget ⇒ §5.4.4, closed by V1 8→6 vehicles (−0.11 ms)
     and Tier-2/Tier-3 at 45/12 Hz (−0.24 ms).
   - Audio 1.67 ms measured vs a 1.60 ms budget (1.04% over) ⇒ §3.7.8, closed by 96→88 full-DSP
     voices.
   - Crowd flow fields 120 MB vs a 64 MB pool ⇒ §6.3.3, closed by an 8 m/16 m coarsening to 30 MB
     and a re-charge to the AI runtime pool.
2. **Name the displacement** (§C.15). If no row covers it, add a row and get it reviewed.
3. **State the kill criterion.** Every risk in §6.8 has one. R-03's is the clearest: *if HS-04
   cannot reach 16.6 ms by month 42 with knobs 1–22 active, cut structural collapse (§3.4.4) to 2
   authored set-pieces and reclaim 1.9 ms.*
4. **Never widen a ceiling without a Tier-1 change** (§0.6). A ceiling that moves to accommodate
   the thing that broke it is not a ceiling.

---

## C.17 Budget status summary

| Budget | Ceiling | Actual | Margin | Status |
|---|---|---|---|---|
| Playable land | ≥ 250 km² | 252.0 km² | +2.0 km² | ✅ tight but met |
| Enterable structures | 40–50% | 44.9% (37,436) | mid-band | ✅ |
| Install size | ≤ 180 GB | 179.1 GB | **0.9 GB (0.5%)** | ⚠ **the tightest budget in the project** |
| PS5 memory | 12,500 MB | 11,431 MB | 1,069 MB (8.6%) — **fully committed to spikes** (§C.2.2) | ⚠ |
| XSX memory | 13,500 MB | 12,454 MB | 1,046 MB (7.7%) | ⚠ |
| Series S memory | 9,700 MB | 7,172 MB | 2,528 MB (26.1%) | ✅ deliberately thick |
| PC-Mid memory | 16,000 MB | 11,431 MB | 4,569 MB (28.6%) | ✅ the reference spec |
| GPU steady state | ≤ 15.40 ms | 15.40 ms | 1.27 ms variance headroom | ✅ via five default-down knobs |
| CPU critical path | ≤ 16.67 ms | 5.04 ms | 11.63 ms (70%), **pre-spent** (§C.4) | ✅ GPU-bound by design |
| Interior global budget | 640 MB | 640 MB | 0 | ⚠ exactly at ceiling by construction |
| Navmesh pool | 64 MB | 63.0 MB | 1.0 MB | ✅ |
| Crowd flow fields | 30 MB (in the AI pool) | 30 MB | 0 | ⚠ resolved from a 120 MB overage |
| Resident ledger | 372 MB, identical on all platforms | 372 MB | 0 | ✅ non-negotiable |
| Save size | ≤ 250 MB | 196 MB | 54 MB (21.6%) | ✅ |
| Fast travel | ≤ 4.5 s | 4.5 s | 0 | ⚠ |
| Cold boot | ≤ 22 s | 22 s | 0 | ⚠ |
| Pathfinding p99 | ≤ 90 µs | 34/90/68 µs (L0/L1/L2) | met at p99 | ✅ |
| Cache thrash | < 0.5% | gated in CI | — | ✅ gated, not yet measured |
| Runtime PSO creation | 0 | 0 (2,210 closed) | — | ✅ AI-11 |
| Per-commit CI | ≤ 38 min | designed to 38 min | 0 | ⚠ |

**Nine budgets are at or within 1% of their ceiling.** That is the honest state of this design: a
252 km² world with 37,436 enterable interiors, 4,127,000 persistent residents, and a 60 FPS
console target does not have slack, and a document that claimed otherwise would be a marketing
document. The slack that does not exist in the budgets exists instead in the **governor** (26
knobs), the **kill-switches** (48), and the **kill criteria** (24 risks) — that is, in the
mechanisms for spending fidelity in a controlled order when a budget is breached. §C.15 is the
agreed order. §C.16 is the process for changing it.
