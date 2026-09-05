# 01 — Vision Statement & Unique Selling Propositions

---

## 1.1 Vision statement

> **MERIDIAN is a world that does not know the player is in it.**
>
> Four million people live in New Vermillion. They have homes, shifts, debts, medical conditions,
> commutes, and opinions about the council election. Rain falls on the windward face of the Sable
> Range because air is forced uphill and cools to its dew point — not because a designer toggled a
> weather state. A gunshot in a canyon carries 480 metres; the same gunshot in the Vermillion City
> financial district is swallowed by concrete at 90 metres, and the only witness is a night porter
> who is asleep, has no mobile, and is 340 metres from the nearest landline. Steel crumples at a
> rate set by its yield strength. Puddles form where the ground is impermeable and drain where it
> is not, and a car aquaplanes on them at a speed you can compute from its tyre pressures.
>
> The player is not the protagonist of this world. The player is a **perturbation** of it — and the
> world's response to that perturbation is the game.

Everything downstream of this sentence is engineering in service of one design outcome: the
player must never be able to identify the seam where simulation ends and theatre begins.

## 1.2 The design problem we are actually solving

The open-world genre's central unsolved problem is not scale. GTA V, RDR2, and Cyberpunk all
shipped large maps. The unsolved problem is **continuity of causation at scale**:

- In every shipped open world, NPCs exist to populate a frame. They arrive when observed and cease
  when unobserved. The player learns this within four hours and the world stops being a world.
- In every shipped open world, weather is a global state machine with five states. It rains
  everywhere or nowhere.
- In every shipped open world, materials are visual categories. "Wood" looks like wood. It does not
  have a modulus of elasticity that the ballistics system, the acoustics system, the vehicle system,
  and the destruction system all consult.
- In every shipped open world, interiors are separate levels behind a load.

MERIDIAN's bet: these four are the **same** problem, and it is solved by the **same** architecture —
a single source of truth per entity, simulated continuously across a fidelity continuum, with one
shared physical property database consumed by every subsystem.

## 1.3 Design pillars

Pillars are decision filters. When two features conflict, the higher pillar wins. These are ranked.

### Pillar 1 — **The world persists whether or not it is rendered.**
No entity's existence, position, or state may be a function of camera visibility. Rendering is a
*view* over simulation state; it never *is* the state. Concretely: if the player leaves a district
for three in-game days and returns, the shop that was robbed is still boarded up, the resident who
was hospitalised is still hospitalised, and the pothole the player drove through has been repaired
by a road crew whose work order the simulation generated.

### Pillar 2 — **One material, one truth.**
Every physical surface has exactly one `MaterialRecord` (§3.1). Rendering, physics friction,
collision restitution, ballistic penetration, acoustic absorption and transmission loss, thermal
conductivity, hydrological permeability, and fire behaviour all read the **same** record. No
subsystem may own a private material table. This is the single most important architectural
decision in the project — see §1.4 USP-3.

### Pillar 3 — **Causation must be legible.**
Hyper-realism is worthless if the player cannot read it. Every systemic outcome must be traceable
by the player to a cause they can observe and learn. If a police response is slow, the player must
be able to discover *why* (dead zone, shift change, jurisdictional handoff, understaffing) and
exploit it. Simulation that the player experiences as randomness is worse than scripted randomness,
because it teaches nothing.

### Pillar 4 — **Fidelity is a continuum, never a cliff.**
No system may have a discrete boundary the player can detect by crossing it. LOD, AI tier, traffic
model, weather resolution, audio ray count, and physics iteration count all degrade **continuously
and hysteresically** with distance and importance. §5.11's governor and §3.9's Resident Continuum
are the two implementations of this pillar.

### Pillar 5 — **60 FPS is a design constraint, not a post-process.**
No feature may be approved without a frame-time cost attached. Every budget in Appendix C is
enforced in CI on every commit. Designers author against budgets, not around them.

### Pillar 6 — **The player is a person, not a camera.**
Full embodiment: visible body, weight-bearing locomotion, momentum, reach limits, grip states,
breathing tied to exertion and altitude, thermal comfort tied to weather and clothing insulation
(clo value), caloric and hydration state over multi-day play. No floating camera. No teleport
movement. Interaction is physical.

## 1.4 Unique Selling Propositions

Six USPs. Each is stated with a **falsifiable definition** — a measurable claim that QA can test and
marketing cannot inflate. This is deliberate: a USP that cannot be falsified is a slogan.

---

### USP-1 — **The Ledger: 4.1 million persistent residents**

Every one of 4,127,000 residents of New Vermillion has a `ResidentRecord` (§3.9.2, Appendix A.2):
identity, date of birth, household, address, employment, income, shift pattern, transit subscription,
vehicle ownership, relationships graph, health record, criminal history, possessions, and a memory
of every interaction with the player. These records are **allocated at world generation and never
destroyed**. An embodied NPC on screen is a *rendered view* of a record, not an entity in its own
right. Records are simulated continuously: at Tier 4 by an abstracted stochastic ledger tick
(1 Hz in-process, or on the Downtime Simulation backend between sessions, §5.13.4), and promoted
through five fidelity tiers as the player approaches (§3.9.3).

**Falsifiable definition:** Pick any named resident at random from the ledger. Travel to their
recorded home at their recorded off-shift time. They are there, or the ledger contains a coherent
entry explaining where they are (a shift at their recorded workplace, a hospital admission, a
transit leg in progress). Run this test 10,000 times across 30 simulated in-game days. **Pass
condition: zero unexplained absences, zero duplicate identities, zero records whose schedule places
them in two locations at once.**

**Delta vs shipped competition:** GTA V and RDR2 have no equivalent. RDR2's ambient NPCs are
schedule-driven but finite (~1,200 authored individuals) and the world does not persist a population.
Cyberpunk 2077's crowd is instanced decoration. No shipped title maintains a per-capita record for
a multi-million population and rehydrates individuals from it on demand.

**Cost:** 380 MB resident runtime memory (§3.9.7), 1.1 GB ledger storage on disk, one dedicated
simulation worker set, ~22 person-months of tooling.

---

### USP-2 — **A simulated climate, not a weather state machine**

MERIDIAN runs a real mesoscale weather model over the map (§3.2): a synoptic driver sets pressure
systems and frontal boundaries; a 512 m-resolution advected weather field carries temperature,
humidity, precipitation rate, wind vector, pressure, cloud base/ceiling, fog density, and visibility;
microclimate modifiers apply orographic lift and rain shadow from actual terrain, urban heat island
from actual building density, canyon wind channelling from a baked per-district flow field, sea-breeze
inversion from actual coast geometry, and katabatic drainage from actual slope. It can be raining in
Meridian Valley while the Anza Basin is clear and the Sable Range is above a cloud deck at 1,180 m.

Weather drives: tyre and shoe friction, puddle formation and depth, aquaplaning onset, intake air
density and therefore engine torque, brake cooling, fire behaviour, sound propagation, helicopter
dispatch legality, tide and sea state, foliage animation, clothing choice by residents, transit
ridership, crop yield, river discharge, and — via the hydrology model — whether the underpass at
Cordova Flats floods.

**Falsifiable definition:** Place 64 virtual weather stations on a 3 km grid. Run 90 simulated days.
Verify: (a) the annual precipitation total at the windward Sable Range station exceeds the leeward
Anza Basin station by ≥ 2.4×; (b) the urban core is ≥ 1.8 °C warmer at 03:00 than the surrounding
rural stations, on average, with the delta shrinking to < 0.5 °C on windy nights; (c) fog forms in
Meridian Valley on ≥ 60% of clear, calm, high-humidity autumn nights and does not form on windy
nights; (d) no two grid cells report identical state for more than 3% of the simulation.

**Delta:** RDR2 has the strongest shipped weather (regional, elevation-aware). MERIDIAN adds
advection, orography, urban heat island, a hydrological response, and per-cell state consumed by
friction, acoustics, and powertrain models.

---

### USP-3 — **The Material Ontology: one physical truth, many consumers**

A single database of ~2,400 `MaterialRecord` entries (§3.1, Appendix A.1). Each record carries
density, Young's modulus, Poisson ratio, yield and ultimate strength, fracture toughness, hardness,
static/kinetic/rolling friction coefficients (dry, wet, icy, muddy), restitution, permeability,
specific heat and thermal conductivity, six-band acoustic absorption coefficients, STC-rated
transmission loss, ballistic penetration resistance with obliquity curves, deformation model
selection, spall factor, fuel load for fire behaviour, and optical parameters.

This record is consumed by: the renderer (BRDF + subsurface), the physics solver (friction,
restitution, mass), the destruction system (which model to run, how much energy to absorb), the
ballistics system (penetration, ricochet, residual energy), the acoustics system (absorption per
bounce, transmission loss through walls, reverb time), the hydrology system (infiltration rate,
runoff), the fire system (fuel load, ignition temperature, spread rate), and the AI perception
system (can a witness see/hear through this?).

**Falsifiable definition:** Enumerate all subsystems that consume material properties. Assert:
**exactly one** source of truth, zero private tables, 100% of material references resolve to a
`MaterialRecord` ID. Then assert cross-system coherence: a wall whose material is `concrete_cmU_200`
must (a) stop a 9 mm FMJ at 200 mm, (b) attenuate speech by ≥ 45 dB STC, (c) have μ ≈ 0.65 dry /
0.42 wet under tyre, (d) spall rather than dent under vehicle impact, and (e) not burn. Test all five
from the same record ID with no per-system override.

**Why this is the strongest USP:** it is invisible to marketing and it is the reason everything else
works. Games that let each system own its own material table end up with drywall that stops bullets
but not sound, or asphalt that is slippery for cars but grippy for feet. Players notice incoherence
even when they cannot name it. This is also a **productivity** USP: one authored property set instead
of eight.

**Cost:** ~2,400 records × 9 fields measured or sourced from published engineering data; 6 person-
months of materials research; a validation harness per consumer subsystem.

---

### USP-4 — **37,436 enterable interiors with zero loading screens**

44.9% of the region's 83,400 structures are enterable (§4.2). Content is produced by a
**procedural-authorial hybrid** (§4.3): 180 bespoke hero interiors, ~1,400 composed signature
interiors from a bespoke kit, and ~35,840 procedurally generated stock interiors whose floorplans
come from a parametric solver driven by the building's exterior envelope, use class, construction
era, and socioeconomic tier — and whose **set dressing is seeded by the Resident Record of the
occupant**. A studio apartment in Riverton occupied by a 61-year-old retired transit worker with
$31k income and a recorded estrangement from one of two children is furnished differently from the
identical footprint occupied by two 24-year-old hospitality workers, because the decorator reads
their records.

Transitions use **interior bundles** (portal-paired exterior shell + interior volume, loaded as one
atomic streaming unit), 8–12 s of predictive pre-streaming from player velocity and heading, portal
culling so unoccupied interiors cost nothing, and a diegetic **vestibule mask** as the worst-case
fallback (§4.5).

**Falsifiable definition:** An automated bot walks through 2,500 randomly selected doors across all
biomes and all times of day, at speeds from 1.2 m/s to 40 m/s (including vehicle entry into garages).
**Pass condition: zero full-frame loading screens, zero frame time spikes > 33 ms, zero pop-in
visible in the 90th-percentile capture, zero interiors with unloaded geometry at the moment the
camera crosses the portal plane.** Additionally: sample 500 residential interiors; blind-test 20
players on whether the occupant is (from a list) a student, a retiree, a family with children, or a
shift worker. **Pass condition: ≥ 70% correct** — interiors must be readable as belonging to someone.

**Delta:** GTA V has ~200 enterable interiors and loads every one. Cyberpunk 2077 has ~150 major
interiors plus apartment blocks, all loaded. Assassin's Creed titles have high exterior density and
near-zero interiors. Nothing shipped approaches 37k seamless interiors.

**⚠ HONEST LIMITATION:** Seamless entry is achieved for the designed case. The fallback vestibule
mask (a 0.3–0.6 s door animation + exposure adaptation) is used when predictive streaming misses —
targeted at < 0.4% of transitions. We do not claim zero-cost transitions; we claim zero *visible
loading*.

---

### USP-5 — **Consequence: crime → evidence → report → dispatch → investigation → court**

A seven-stage pipeline (§3.12) where every stage has explicit, physically grounded failure modes:

1. **Perception** — witnesses have vision cones gated by actual scene lighting and weather
   visibility, and hearing gated by the ray-traced acoustic solution's computed SPL at their
   position (§3.7.5). Detection is probabilistic on distance, attention, sobriety, and personality.
2. **Report** — delay computed from witness reaction time, mobile ownership, distance to the nearest
   working phone-equivalent, **cellular coverage at their location** (baked RF propagation model
   with terrain knife-edge diffraction, building attenuation, and foliage loss — tunnels, subways,
   canyons, and deep forest are genuine dead zones), and dispatch queue depth.
3. **Dispatch** — a CAD simulator with real unit statuses, 10-hour tour shifts with meal breaks and
   two daily shift changes where response times measurably degrade, radio channel capacity as a
   real constraint, and **seven jurisdictions** with mutual-aid request latency across boundaries.
4. **Response** — five escalating tiers from single-unit to cordon-and-negotiation, including
   dynamic roadblocks that reroute civilian traffic through the macro traffic model, air support
   grounded by weather minima, and K9/drones/FLIR.
5. **Investigation** — crime scene processing: ballistic fingerprinting per weapon, CCTV coverage
   map, tyre impressions read from the soil deformation layer (§3.4.6), DNA/blood if the player was
   injured, witness canvass with memory-degraded descriptions.
6. **Arrest & booking** — the player can be arrested non-lethally, booked, arraigned, released on
   bail, and given a court date on the simulated calendar.
7. **Adjudication** — evidence quality determines outcome. Insufficient evidence ⇒ dropped charges
   but a persistent record.

**Falsifiable definition:** Run 200 authored incident scenarios through the harness. Assert the
model reproduces: (a) median report delay of 45–120 s in dense urban daytime with mobile coverage;
(b) report delay > 8 min or **never** in a mapped dead zone unless the witness travels to coverage;
(c) response ETA degrades ≥ 40% during the 07:00 shift change; (d) crossing from VC PD to County
Sheriff jurisdiction adds ≥ 90 s of mutual-aid latency; (e) a cordon established at Tier 3 actually
prevents egress on 100% of roads within the perimeter unless the player breaches a barricade
physically; (f) a weapon used in two crimes is linked by forensic fingerprint with probability
matching its recorded `traceability` attribute.

**Delta:** GTA V's wanted level is a scalar that decays. RDR2's witness system is the closest
shipped analogue and is genuinely good — but witnesses report to a scalar law-meter, there is no
coverage model, no dispatch simulation, no forensics, and no judicial consequence. MERIDIAN replaces
the scalar with a **simulated institution that has capacity limits, shift patterns, jurisdictional
friction, and paperwork.**

---

### USP-6 — **252 km² at locked 60 FPS with 15 km horizons and no LOD pop**

Proprietary virtualized micropolygon geometry (§5.5) with offline cluster DAG construction,
GPU-driven hierarchical culling, dual raster paths (software scanline for sub-pixel triangles,
hardware raster with mesh-shader-style indirect dispatch for larger), visibility-buffer material
resolve, and a clipmap virtual heightfield terrain with runtime virtual textures. Draw distance to a
**15 km horizon** with aerial-perspective scattering, FFT ocean, and far-clipmap terrain — no
silhouette discontinuity ≥ 1 px at any distance.

**Falsifiable definition:** (a) Capture the horizon from 40 authored viewpoints at 4K. Assert zero
geometric silhouette discontinuities ≥ 1 px attributable to LOD transitions, at any distance out to
15 km. (b) 12 golden-path perf captures (§6.7) on PS5 and PC-Mid: **p99 frame time ≤ 16.6 ms, zero
frames > 33 ms across a 45-minute run each.** (c) Streaming: measure cache-thrash rate
(§6.2 metric) — **< 0.5% of cell loads are re-loads of a cell evicted within the previous 10 s.**
(d) Install size ≤ 180 GB with 100% of content reachable (Appendix C).

---

## 1.5 Anti-USPs (what we are explicitly not)

Stating these prevents scope creep and gives marketing honest guardrails.

| We are not | Why | What we do instead |
|---|---|---|
| **A multiplayer sandbox** | Persistent world sim at this fidelity cannot be made authoritative for 30+ players within budget. Trying kills both the sim and the multiplayer. | Optional 2–4 player co-op crews in a host-authoritative session (§5.13.2), plus an asynchronous Downtime Simulation service (§5.13.4). |
| **A flight/driving/racing sim** | Full-fidelity aero and tyre models exist (§3.5) but tuning toward sim-racing purity conflicts with accessibility and with the 60 FPS budget during high-speed streaming. | Authentic-feeling dynamics with a 3-level assist ladder; the assist ladder changes *only* input smoothing and stability control, never the underlying model. |
| **A military/tactical shooter** | Ballistics are authentic (§3.6) but terminal effects are grounded in trauma physiology, not arcade hit markers, and we are not making a lethality fantasy. | Consequence-heavy combat where the dominant risk is legal and social, not just mortal. |
| **Photorealism for its own sake** | Uncanny visual fidelity without systemic fidelity reads as a tech demo. | Fidelity budget spent first on *behaviour*, then on appearance. Where they conflict, behaviour wins. |
| **Satire** | GTA's defining authorial voice is satirical exaggeration. Satire and systemic authenticity are opposed: satire requires the world to bend to the joke. | Straight-faced, humane, observational tone. The comedy comes from the world's indifference, not from caricature. This is a **significant market positioning risk** — see §6.6.4. |

## 1.6 Narrative frame (interface contract only)

The campaign is authored in a separate GDD volume. What is fixed *here*, because systems depend on it:

- **Three playable protagonists**, each with a distinct position in the region's socioeconomic
  structure (a municipal transit operator, a freelance recovery driver operating out of Halstead Bay,
  and a county sheriff's deputy in Anza Basin). Switching is **non-instant**: the outgoing character
  continues to be simulated at Tier 4 in the ledger, so returning to them after two in-game days
  finds their life has moved — missed shifts, unpaid rent, a relationship that progressed or decayed.
  This is the single largest narrative use of USP-1 and it must not be compromised by a "freeze
  off-screen protagonists" optimisation.
- **No time-skip mechanics** that bypass the ledger. Time advances continuously or via an explicit
  "rest" action that runs the ledger forward at 300× and reports what changed.
- **Missions are perturbations, not set-pieces.** Mission objectives are expressed as ledger and
  world-state mutations (steal *this* container, which is manifested to *that* store, which will
  then be *empty*), so mission outcomes propagate through the economy and ecology systems without
  bespoke code.
- **Fail states are systemic.** Missing a mission window means the world moved on: the target left,
  the shift ended, the rain washed the tracks away. There is no rewind-to-checkpoint in the diegesis
  (a menu-level checkpoint exists for accessibility, but it restores world state too — see §5.10.4).

## 1.7 Target audience & market position

| Segment | Size estimate | Reach strategy |
|---|---|---|
| **Primary:** open-world action-adventure players, 18–40, GTA V / RDR2 / Cyberpunk buyers | ~60M lifetime buyers of the top 3 comparable titles | Launch-window console parity, 60 FPS messaging, scale messaging (252 km², 4.1M residents) |
| **Secondary:** simulation enthusiasts (flight sim, sim racing, city builders, immersive-sim players) | ~15M | Deep-dive technical content: material ontology, weather model, dispatch sim. This audience generates the word-of-mouth that "realistic" claims depend on. |
| **Tertiary:** PC enthusiasts / hardware reviewers | ~8M | Benchmark-friendly, ultrawide and 120 Hz support, path-traced option, honest published budgets |

**Positioning statement:** *the open-world genre's answer to the flight simulator audience* — the
game that treats a city as an engineered system rather than a level.

**Comparable-title reference points** (for internal calibration only; public claims must be
verified by Legal before use — §6.6.5):

| Title | Playable land | Enterable interiors | Persistent population | Weather model |
|---|---|---|---|---|
| GTA V (2013) | ≈ 78 km² | ≈ 200 (all loaded) | none (ambient only) | global state, 5 states |
| RDR2 (2018) | ≈ 75 km² | ≈ 100 (loaded) | ~1,200 scheduled | regional, elevation-aware |
| Cyberpunk 2077 (2020) | ≈ 75 km² | ≈ 150 major (loaded) | none | global state, 4 states |
| MS Flight Simulator (2020) | global | n/a | n/a | **real global weather ingest** |
| **MERIDIAN** | **252 km²** | **37,436 seamless** | **4,127,000 ledger records** | **advected mesoscale field + microclimate modifiers** |

The MS Flight Simulator row is the honest benchmark for USP-2: they ingest real meteorology, we
simulate it. Simulating means we can make it respond to terrain we authored, which real ingest
cannot.

## 1.8 Content-volume math (the part that decides whether this ships)

Scale claims are worthless without throughput math. Full derivation in §4.3 and §2.2; the summary
that Production must plan against:

| Content class | Volume | Production method | Throughput assumption | Person-months |
|---|---|---|---|---|
| Landmass terrain | 252 km² | Procedural base from GIS-derived DEM + hydrology, then hand-sculpted | 6 km²/artist-month sculpted to ship quality | 42 |
| Drivable road network | 2,640 km | Procedural spline from parcel graph + hand-finished junctions | 55 km/artist-month | 48 |
| Pedestrian network | 4,120 km | Procedural from sidewalk parcels, auto-navmeshed | 210 km/artist-month | 20 |
| Transit network | 148 km metro, 42 bus routes, 3 ferry, 2 commuter rail | Bespoke | — | 26 |
| Exterior buildings | 83,400 structures | 68% modular kit assembly from parcel grammar, 22% procedural façade from envelope + style grammar, 10% (8,340) bespoke | 190 modular/mo, 60 procedural-configured/mo, 11 bespoke/mo per artist | 310 |
| Hero interiors | 180 | Fully bespoke | 1.4/month per designer (incl. bake, dress, QA) | 128 |
| Signature interiors | 1,420 | Bespoke kit + layout grammar, hand-tuned | 9/month per designer | 158 |
| Stock interiors | 35,836 | Procedural plan solve + record-seeded decoration + archetype anchor passes | 62/month per designer (mostly QC, not authoring) | 578 |
| Named residents (full personality) | 41,000 | Procedural generation + authored seeding for narrative-relevant ~2,400 | — | 46 |
| Background residents | 4,086,000 | Procedural generation from socioeconomic model | — | 12 (tooling) |
| Vehicles | 340 | Bespoke (each with full powertrain/chassis spec) | 5.5/month per designer | 62 |
| Weapons | 96 | Bespoke (each with full ballistic spec) | 7/month per designer | 14 |
| Mocap for motion-matching DB | 44 hours | Studio capture + cleanup | 2.1 h/month | 21 |
| **Total content person-months** | | | | **≈ 1,465** |

At a peak content team of 210 and a 34-month content window, capacity is 7,140 person-months — so
content is **not** the binding constraint. The binding constraints are:

1. **Tooling must be production-ready by month 18 of 66.** The 578 person-months of stock interiors
   and 310 of exterior buildings *assume working procedural pipelines*. If the layout grammar ships
   late, those numbers become 4,000+ person-months and the project does not ship. **This is
   project risk #1** (§6.8, R-01).
2. **The Material Ontology must be populated before any content is authored**, because every asset
   references it. Retrofitting material IDs onto 83,400 buildings is a 6-month cleanup. **Risk #2**
   (§6.8, R-02).
3. **Bake capacity.** 37,436 interiors require acoustic probe bakes and irradiance bakes. At 40 min
   per Class B/C interior on the build farm and 3 h per hero interior, the farm needs ~2,600 core-
   hours per full rebake. Full rebakes must therefore be **incremental and cell-scoped** (§5.14.3),
   never global. A global rebake in the last 6 months would consume the entire build farm.

## 1.9 Success criteria (measured at ship)

| Criterion | Target | Measurement |
|---|---|---|
| Frame rate stability | p99 ≤ 16.6 ms on all golden paths, all platforms | Perf CI (§6.7) |
| Install size | ≤ 180 GB | Cook report |
| Load time: cold boot to playable | ≤ 22 s (PS5/XSX), ≤ 30 s (PC-HDD-cache-cold) | Telemetry |
| Load time: fast-travel across full map | ≤ 4.5 s | Telemetry |
| Save size | ≤ 250 MB × 3 slots | Cook report |
| Cache thrash rate | < 0.5% | Telemetry (§6.2) |
| Ledger consistency violations | 0 | Ledger test harness (§6.7) |
| Interior transition hitches > 33 ms | < 0.4% of transitions | Bot harness |
| Crash rate at launch | < 0.15% of sessions | Telemetry |
| Softlock reports | < 40 in first 30 days, all with menu-level recovery | Support + §6.4 kill-switches |
| Player-verifiable systemic depth | ≥ 70% accuracy on the interior-occupant blind test (§1.4 USP-4) | Playtest |
