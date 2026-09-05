# MERIDIAN — Document Set Index

**Project codename:** `MERIDIAN`
**Engine:** `STRATA` (proprietary; no UE5/RAGE/Decima code dependency)
**Document class:** Game Design Document (GDD) + Technical Architecture Specification (TAS)
**Version:** 0.9 (architecture-complete pre-greenlight draft)
**Last revised:** 2026-09-05
**Owners:** Lead Systems Architect · Technical Director · Open-World Design Director

---

## 0.1 How to read this document set

This is a **specification**, not a pitch deck. Every claim about scale, cost, or capability is
attached to a number, a budget, an algorithm, or a named failure mode. Where a technique is
borrowed from known shipped technology, the prior art is named so the reader can judge the
delta honestly.

Three conventions are used throughout:

| Convention | Meaning |
|---|---|
| **`MUST` / `SHOULD` / `MAY`** | RFC-2119 style. `MUST` items are architecture invariants — violating one is a defect, not a style choice. |
| **Budget boxes** | Hard numeric ceilings enforced by CI. Exceeding a budget fails the build. |
| **⚠ HONEST LIMITATION** | A place where the requirement as literally stated is not achievable in real time on target hardware, and where we specify the approximation instead. These are deliberate and non-negotiable — they are how the project ships. |

## 0.2 Document map

| # | File | Contents | Primary audience |
|---|---|---|---|
| 00 | [`00-index.md`](./00-index.md) | Document map, reading conventions, platform targets summary, requirements traceability matrix | All |
| 01 | [`01-vision-and-usps.md`](./01-vision-and-usps.md) | Vision statement, design pillars, six USPs with falsifiable definitions, competitive delta analysis, audience, content-volume math, production shape | Design, Production, Publishing |
| 02 | [`02-world-geography.md`](./02-world-geography.md) | Region geography, area budget, biome map, district zoning, transition grammar, verticality, transit network, hydrology, underwater, simulated calendar/astronomy | Design, Environment, Level Streaming |
| 03 | [`03-systemic-engines.md`](./03-systemic-engines.md) | Material Ontology, weather & microclimate, hydrology/ecology/fire, destruction & deformation, vehicle dynamics, ballistics & terminal effects, ray-traced acoustics, traffic (macro/meso/micro), the Resident Continuum, schedules, crime→report→dispatch→investigation→court pipeline, economy & supply chain | Systems Design, Simulation Engineering, Audio, Physics |
| 04 | [`04-interiors-and-detailing.md`](./04-interiors-and-detailing.md) | Interior taxonomy and counts, layout grammar, procedural vs bespoke split, per-class memory budgets, zero-loading-screen technique, set dressing tied to Resident Records, bake strategy, portal culling | Environment Art, Tools, Streaming, Tech Art |
| 05 | [`05-technical-optimization-pipeline.md`](./05-technical-optimization-pipeline.md) | Engine layering, ECS & job graph, virtualized micropolygon geometry, terrain & virtual textures, renderer, World Partition streaming & I/O, per-platform memory budgets, performance governor, tick hierarchy & determinism, PSO/shader strategy, networking & downtime simulation, tools/CI | Engineering (all disciplines), Technical Director |
| 06 | [`06-edge-cases-and-risk-mitigation.md`](./06-edge-cases-and-risk-mitigation.md) | Asset bloat, cache thrashing, large-terrain pathfinding, sim instability, emergent softlocks, deterministic divergence, thermal throttle, platform/legal/ratings exposure, QA at 252 km², full risk register with owners and kill-criteria | Technical Director, Production, Legal |
| A | [`appendix-a-data-schemas.md`](./appendix-a-data-schemas.md) | Authoritative record schemas: `MaterialRecord`, `ResidentRecord`, `DistrictZone`, `InteriorCell`, `WeatherCell`, `CrimeEvent`, `DispatchOrder`, `GovernorRule` | Engineering, Tools |
| B | [`appendix-b-algorithms.md`](./appendix-b-algorithms.md) | Pseudocode for 13 load-bearing algorithms: the cell streaming scheduler with cost-aware eviction, cluster culling and raster-path assignment, resident tier promotion/demotion, crime-report delay with Erlang-C queueing, the tyre Magic Formula with thermal and brake fade, the puddle shallow-water solve, ballistic penetration resolve, the macro-traffic Cell Transmission Model, Rothermel fire spread, delta-journal compaction and the World Settle Pass, the interior layout-grammar solve, the cellular-coverage RF bake, and the cordon minimum-cut | Engineering |
| C | [`appendix-c-budgets.md`](./appendix-c-budgets.md) | Every numeric ceiling in one place: memory, frame time, asset size, disk throughput, entity counts, save size, install size | Technical Director, CI |

## 0.3 Platform targets (summary)

Full derivation in [`05-technical-optimization-pipeline.md` §5.10–§5.11](./05-technical-optimization-pipeline.md).

| Target | Mode | Resolution | Frame rate | Notes |
|---|---|---|---|---|
| PlayStation 5 (base) | **Performance** | 1440p native → 4K TSR upsample, DRS 70–100% | **60 FPS locked** | SW ray tracing GI, no HW RT |
| PlayStation 5 (base) | **Cinematic** | 1800p → 4K, DRS 80–100% | 30 FPS | + HW RT reflections/shadows on near field |
| PlayStation 5 Pro / Xbox next | Performance | 1800p → 4K | **60 FPS locked** | + HW RT near-field GI |
| Xbox Series X | Performance | 1440p → 4K | **60 FPS locked** | parity with PS5 |
| Xbox Series S | Performance | 900p → 1440p | 30 FPS | reduced interior density tier, T2 crowd cap halved, no volumetric cloud high-res cone |
| PC — Low (8 GB VRAM, 16 GB RAM, RTX 3060-class) | — | 1080p | 60 FPS | interior budget class shift: S→A, A→B, C→D |
| PC — Mid (12 GB VRAM, 32 GB RAM, RTX 4070 / RX 7800 XT-class) | — | 1440p | **60 FPS locked** | reference spec for all budget tables |
| PC — High (16 GB+ VRAM, 32 GB+ RAM) | Ultra | 4K | 60 FPS (120 unlocked optional) | + HW RT, path-traced near-field option |

**Hard requirement:** NVMe-class SSD (≥ 2.4 GB/s sustained sequential read) on **all** platforms,
including PC. There is no HDD SKU. Rationale in [§5.9.2](./05-technical-optimization-pipeline.md).

**Simulation targets are platform-invariant.** The Resident Continuum, weather field, traffic
assignment, dispatch simulation, and destruction rules `MUST` produce identical outcomes at
identical simulation tick counts on every platform. Only *presentation* fidelity scales. A
Series S player and a PC Ultra player get the same world; they get different pixels. This is an
architecture invariant, not a preference — it is what makes cross-platform co-op and shared QA
repro possible.

## 0.4 Requirements traceability

Mapping from the commissioning brief to the specification sections that satisfy it.

| Brief requirement | Satisfied by | Verification method |
|---|---|---|
| ≥ 250 km² landmass, ≥ 3× GTA V | §2.2 area budget (252.0 km² land; GTA V ≈ 78 km² ⇒ 3.23×) | Offline GIS measurement of cooked world bounds; CI asserts ≥ 250 |
| 7 biome classes incl. underwater | §2.3, §2.9 | Biome classifier coverage report; ≥ 99.5% of land assigned |
| 40–50% enterable structures | §4.2 (44.9% of 83,400 ⇒ 37,436 interiors) | Cook-time interior manifest count |
| No loading screens at interiors | §4.5 (portal-pair bundles + vestibule mask) | Automated bot traversal of 1,000 doors; 0 hitches > 33 ms |
| Volumetric clouds, localized rain, dynamic puddling, wet-friction, wind on foliage/cloth/projectiles | §3.2, §3.3.4, §3.5.6 | Golden-scene perf capture + friction coupling unit tests |
| Material-based deformation (metal/concrete/wood/soil) | §3.4 | Per-material deformation test suite (14 materials × 5 energy bands) |
| Ray-traced spatial audio, occlusion, dynamic reverb | §3.7 | Acoustic probe validation vs measured reference rooms; SPL→AI perception integration test |
| Routine-driven NPC life cycles, no random despawn | §3.9 Resident Continuum + §3.10 schedules | Ledger consistency test: 4.1M records, 30 simulated days, zero identity violations |
| Granular crime-report delay, cellular dead zones, multi-tier police tactics | §3.12 | Scenario harness: 200 authored incidents, report-delay distribution vs model |
| Real vehicle dynamics (powertrain, slip angles, brake fade, suspension geometry) | §3.5 | Lap-time and handling parity vs reference simulator; brake-fade thermal validation |
| Authentic ballistics (drop, falloff, penetration) | §3.6 | Ballistic table validation vs published G1/G7 retardation data, ±2% |
| Continuous world partition, aggressive streaming, memory budgeting, dynamic texture streaming, 60 FPS | §5.8, §5.9, §5.10 | Perf CI on 12 golden paths, every commit |
| Virtualized geometry, no LOD pops, 15 km horizon | §5.5, §5.6 | Horizon capture test: 0 silhouette discontinuities ≥ 1 px at 15 km |
| Distributed multithreading, server/client tick rates, deterministic state | §5.2, §5.4, §5.13 | State-hash divergence detector across platforms; downtime-sim reconciliation test |
| Asset bloat, cache thrashing, pathfinding over large terrain | §6.1, §6.2, §6.3 | Shipped-size gate, thrash-rate telemetry, path-query latency gate |

## 0.5 What this document deliberately does **not** contain

Being explicit about scope prevents the two failure modes that kill open-world projects: silent
scope creep, and design-by-optimism.

1. **No narrative script, dialogue, or mission-by-mission breakdown.** §1.6 states the narrative
   *frame* and how missions hook into systems. The 100+ hour campaign design is a separate GDD
   volume (`docs/narrative/`, not yet written) that consumes the system contracts defined here.
2. **No monetisation, no live-service roadmap.** Single-player-first with optional 2–4 player
   co-op crews (§5.13). Anything beyond that is out of scope and must not be assumed by systems.
3. **No art direction bible.** Palette, cinematography, character design language are separate.
   §2 and §4 constrain art only where it touches budget or systemic behaviour.
4. **No audio asset lists, no music licensing plan.** §3.7 specifies the *acoustic architecture*
   only.
5. **No platform certification checklist.** TRC/XR/LOT are tracked separately against §5.10 and §6.6.

## 0.6 Change control

Architecture invariants (`MUST` statements) may only be amended by a written ADR
(`docs/adr/NNNN-title.md`) signed by the Technical Director and the relevant discipline lead.
Budget numbers in Appendix C are enforced by CI and may only be raised by an ADR that names the
bytes being taken away from something else. **Budgets are zero-sum by policy.**
