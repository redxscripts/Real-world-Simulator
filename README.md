# Real-world Simulator

**Project `MERIDIAN`** — a Game Design Document and Technical Architecture Specification for a
next-generation hyper-realistic open-world action-adventure, and for **`STRATA`**, the proprietary
engine it requires.

This repository is a **specification**, not a pitch deck. Every claim about scale, cost, or
capability is attached to a number, a budget, an algorithm, or a named failure mode. Where a
technique is borrowed from shipped technology (UE5's Nanite/Lumen/World Partition, RAGE's
scan-and-react traffic, Decima's volumetrics, Frostbite's irradiance probes, Hillaire's atmosphere
and surfel GI), the prior art is named so a reader can judge the delta honestly.

---

## The headline numbers

| | |
|---|---|
| Playable land | **252.0 km²** (GTA V ≈ 78 km² ⇒ **3.23×**), inside a 22.4 × 18.6 km bounding box |
| Inland water + marine | **78.0 km²**, diveable to −120 m |
| Elevation span | **−120 m to +2,480 m** (Sable Range, 2,410 m peak) |
| Structures | **83,400**, of which **37,436 (44.9%) are enterable** with no loading screen |
| Population | **4,127,000 persistent residents**, every one a named record with a schedule, a job, a household, relationships, and a criminal history — never randomly spawned, never randomly despawned |
| Material Ontology | **2,412 material records**, each consumed by 8 independent systems (deformation, friction, ballistics, acoustics, fire, hydrology, ecology, audio) |
| Frame contract | **60 FPS locked** on PS5, Xbox Series X, and PC-Mid — 15.40 ms of GPU work inside a 16.67 ms frame |
| Resident memory | **11,431 MB of 12,500 MB** on PS5, across 20 budgeted pools |
| Install size | **179.1 GB** against a 180 GB ceiling |
| Horizon | **15 km** with a measured silhouette-parallax error of ≤ 0.42 px |
| Runtime PSO creation | **zero** — a closed set of 2,210 pipelines, precached |
| Determinism | a 600-tick replay hash match on every platform, enforced in CI |

---

## The document set

Read in order. Each document is self-contained but cross-references the others by section number.

| # | Document | What it settles |
|---|---|---|
| 00 | [`docs/00-index.md`](docs/00-index.md) | Reading conventions (`MUST`/`SHOULD`/`MAY`, budget boxes, ⚠ honest limitations), platform targets, the requirements-traceability matrix, explicit out-of-scope list, change control |
| 01 | [`docs/01-vision-and-usps.md`](docs/01-vision-and-usps.md) | **Vision & USPs.** Six design pillars; six USPs each stated as a *falsifiable* definition with a test, a competitor delta, and a cost; content-volume math; production shape |
| 02 | [`docs/02-world-geography.md`](docs/02-world-geography.md) | **World geography & map layout.** Coordinate and origin policy; the 15-district area budget that sums to 252.0 km²; 13 biome classes and their transition grammar; zoning, parcels, and the seven gradient fields; road/rail/transit/utility networks; the 681 km subsurface; underwater terrain; the simulated calendar and astronomy |
| 03 | [`docs/03-systemic-engines.md`](docs/03-systemic-engines.md) | **Systemic engines.** The Material Ontology; weather, microclimate, hydrology, ecology, and fire; seven deformation models (metal crumple, concrete spall, wood fracture, soil displacement, glass, masonry, tissue); vehicle dynamics V0–V4 (Pacejka tyres, driveline torsion, brake thermal, aquaplaning); ballistics and terminal effects; ray-traced spatial audio; macro/meso/micro traffic; the Resident Continuum; schedules; the crime → report → dispatch → investigation → court pipeline; the economy |
| 04 | [`docs/04-interiors-and-detailing.md`](docs/04-interiors-and-detailing.md) | **Interior & detailing strategy.** The procedural-vs-bespoke split (1,600 hand-tuned, 35,836 generated from a 180-byte seed); five interior classes with per-class memory ceilings from 220 MB down to 6 MB; the 640 MB global interior budget; the interior layout grammar and its constraint solve; the zero-loading-screen technique; fourteen CI gates |
| 05 | [`docs/05-technical-optimization-pipeline.md`](docs/05-technical-optimization-pipeline.md) | **Technical optimization pipeline.** Engine layering and the archetype ECS; the P0–P5 frame pipeline; a 17-rate tick ladder; determinism rules; physics islands and five physics-LOD tiers; SVG virtualized micropolygon geometry; terrain clipmaps, RVT, and the four-zone far field; the full render pipeline with per-stage millisecond costs; World Partition grids G0–G4; mutable-state slabs and the delta journal; the streaming and I/O scheduler; per-platform memory budgets; the 26-knob performance governor; eight hard scenes |
| 06 | [`docs/06-edge-cases-and-risk-mitigation.md`](docs/06-edge-cases-and-risk-mitigation.md) | **Edge cases & risk mitigation.** Asset bloat and the exposure-efficiency gate; cache thrashing (a measurable `thrash_rate`, ten named causes, GreedyDual-Size-Frequency eviction); pathfinding over 252 km² (a four-level hierarchy, crowd flow fields, vertical/canopy/subsurface pathing, dynamic re-carving); twelve recovery ladders and 48 kill-switches; determinism and numerics; legal, ratings, accessibility, and platform certification; verification (12 golden paths, 15 harnesses, a 64-bot coverage fleet); a 24-entry risk register with kill criteria; fourteen combinatorial edge cases |
| A | [`docs/appendix-a-data-schemas.md`](docs/appendix-a-data-schemas.md) | Authoritative record layouts with byte counts: `DistrictZone`, `ParcelRecord`, `StructureRecord`, `InteriorBundle`, `LayerStack`, `VehicleRecord`/`VehicleState`, `DispatchUnit`/`DispatchOrder`, `EvidenceItem`, `GovernorRule`, `DeltaJournalEntry`, `StateSlab`, `StreamingCellManifest`, and the full storage summary |
| B | [`docs/appendix-b-algorithms.md`](docs/appendix-b-algorithms.md) | Pseudocode for the 13 load-bearing algorithms, with determinism-relevant detail (ordering, seeding, accumulator shape) made explicit |
| C | [`docs/appendix-c-budgets.md`](docs/appendix-c-budgets.md) | Every numeric ceiling in one place, plus the **zero-sum ledger** — 16 pre-agreed trades stating exactly what shrinks when something grows — and the overage protocol |

**≈ 9,960 lines / 686 KB of specification across ten documents.**

---

## How this was written

The six sections of the commissioning brief map onto the document set as follows:

| Brief section | Document |
|---|---|
| 1. Vision & USPs | `01` |
| 2. World geography & map layout | `02` |
| 3. Systemic engines (weather / ecology / AI / vehicle / ballistics) | `03` |
| 4. Interior & detailing strategy (procedural vs bespoke, memory budget per interior) | `04` |
| 5. Technical optimization pipeline (streaming, LOD/mesh clustering, memory profiling, CPU multithreading) | `05` |
| 6. Edge cases & risk mitigation (asset bloat, cache thrashing, pathfinding over large terrain) | `06` |
| Supporting: schemas, algorithms, budgets | `A`, `B`, `C` |

### Conventions

- **`MUST` / `SHOULD` / `MAY`** are RFC-2119. A violated `MUST` is a defect, not a style choice.
- **Budget boxes** are hard numeric ceilings enforced by CI. Exceeding one fails the build.
- **⚠ HONEST LIMITATION** marks a place where a requirement as literally stated is not achievable
  in real time on target hardware, and where the specification states the approximation instead.
  These are deliberate. A specification with none of them is a specification that has not been
  costed.

### Numbers that were derived, not asserted

The area budget, the structure and interior counts, the six-platform memory tables, the GPU and CPU
frame budgets, the install-size breakdown, the tick-rate amortisation, and the navmesh composition
were each computed and cross-checked so that every table sums to its stated total. Where a design
exceeded its budget, the overage is **recorded with its number and its resolution** rather than
quietly absorbed — see Appendix C §C.16 for the four documented overages (GPU 16.80 ms vs a
16.67 ms frame; physics 2.72 ms vs 2.40; audio 1.67 ms vs 1.60; crowd flow fields 120 MB vs a
64 MB pool) and exactly what was cut to close each one.

### The central architectural bet

Three decisions carry most of the weight, and they are stated here because everything else follows
from them:

1. **The simulation does not scale with the platform.** The Resident Continuum is 372 MB and
   4,127,000 records on *every* target, including Xbox Series S. Only presentation fidelity scales.
   A Series S player and a PC Ultra player get the same world; they get different pixels. This is
   what makes cross-platform co-op, shared QA reproduction, and a single deterministic replay
   format possible.
2. **Interiors are generated, not stored.** 35,836 of the 37,436 enterable interiors exist on disk
   as a 180-byte seed plus an override list — 6.5 MB total — and are solved at load by a constraint
   grammar with a 91.4% first-pass yield and 54 authored fallback templates. The marginal cost of
   the 19,200th apartment is 3.5 MB of *resident* memory and ~0 bytes of unique disk.
3. **Derived state is never persisted.** 2.1 GB of the 11.4 GB PS5 working set is recomputable —
   traffic densities, weather cell state, puddle depth, the SPL grid, coverage state, the navmesh,
   the GI cache, shadow pages, animation pose. That is why a save is 196 MB and not 2 GB, and why a
   save can be loaded by a different game version.

---

## Status

**Version 0.9 — architecture-complete pre-greenlight draft.** The system contracts, budgets, and
verification gates are defined. Not yet written, and deliberately out of scope for this repository
(see `docs/00-index.md` §0.5): the narrative script and mission-by-mission breakdown, the art
direction bible, the audio asset lists and music licensing plan, monetisation, and the platform
certification checklist.

Changes to architecture invariants require a written ADR signed by the Technical Director and the
relevant discipline lead. **Budgets are zero-sum by policy**: a ceiling may only be raised by an ADR
that names the bytes, milliseconds, or gigabytes being taken away from something else.
