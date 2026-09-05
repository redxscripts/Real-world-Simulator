# 04 — Interior & Detailing Strategy

**Scope:** 37,436 enterable interiors across 83,400 structures (44.9%), plus the exterior
detailing system that makes those structures credible from outside. This section answers three
questions the brief poses explicitly: **procedural vs bespoke**, **memory budget per interior**, and
**how zero loading screens is achieved**.

---

## 4.1 Principle: procedural for plan and bulk, bespoke for anchors

The failure mode of procedural interiors is that they look generated. The failure mode of bespoke
interiors is that there are 35,836 of them and they cannot be made. The resolution is not a
compromise between the two; it is a **division of labour by what each is good at**:

| Aspect | Method | Why |
|---|---|---|
| Floorplan geometry, room adjacencies, egress, structure, services | **Procedural (constraint solve)** | There is a correct answer, it is code-derivable, and no human should spend time on it |
| Building shell, façade, massing, roof | **Procedural (grammar) + modular kit** | Same, plus the kit gives art direction control at family level |
| Bulk furnishing and fixture placement | **Procedural (rule-based decorator)** | Volume is impossible otherwise; the rules carry the taste |
| **Occupant identity** — whose home is this, what do they own, what has happened here | **Procedural seeded from the `ResidentRecord`** | This is the differentiator (§4.6). No human can author 35,836 lives; the ledger already has them |
| **Anchor props** — the 1–4 objects per interior that carry its character | **Bespoke** | 2,400 authored assets reused 35,836 times with variation. This is where taste is concentrated |
| Signature and hero interiors | **Bespoke** | 1,600 interiors that carry narrative and landmark weight |
| **Signage, branding, menus, notices, labels** | **Bespoke-authored kit, procedurally distributed** | Text is the single highest-information-density visual channel and generated text is immediately recognisable as fake. 14,800 authored signage assets, procedurally placed by establishment type |

**The rule, stated as an invariant (`AI-07`):** *no procedural system may generate legible text,
legible faces, or legible brand marks.* Those three are the uncanny-valley tripwires. Everything
legible is authored; everything structural and volumetric is generated.

## 4.2 Taxonomy and counts

### 4.2.1 Budget classes

| Class | Count | Description | Resident ceiling | Examples |
|---|---|---|---|---|
| **S — Hero** | 180 | Fully bespoke, full acoustic bake (incl. convolution IR), full GI bake, high dynamic-light count, ≥ 60 interactive props | **220 MB** | Cathedral of St Brendan; Vermillion Central concourse; VMI Terminal 1; City Hall; County Courthouse courtrooms ×14; Halstead dry dock; Level-I trauma centre ED; 9 floors of Vermillion Tower; the refinery control room; Sable Ridge summit observatory |
| **A — Signature** | 1,420 | Bespoke kit + layout grammar, hand-tuned, baked GI, parametric reverb, 20–60 interactive props | **95 MB** | Nightclubs, hospitals wards, schools, police stations, fire stations, hotel lobbies/suites, museums, theatres, large restaurants, wineries, ski lodge, coast guard station, metro stations ×112 (a family of 9 archetypes) |
| **B — Stock commercial/civic/industrial** | 13,400 | Fully procedural plan + procedural dress + 1–2 archetype anchors, 8–20 interactive props | **34 MB** | Retail units, offices, restaurants, bars, gyms, workshops, warehouses, garages, medical/dental practices, banks, post offices, laundromats, church halls |
| **C — Stock residential** | 19,200 | Fully procedural plan + record-seeded dress, 4–8 interactive props | **16 MB** | Apartments, houses, mobile homes, farmhouses, residential hotel rooms, dormitories |
| **D — Utility/back-of-house** | 3,236 | Minimal procedural, 0–4 interactive props | **6 MB** | Stairwells, lift lobbies, plant rooms, refuse rooms, basements, crawl spaces, storage tanks, mine adits, tunnels, public WCs, service corridors |
| | **37,436** | | | |

### 4.2.2 Archetype families

35,836 procedural interiors resolve to **412 space-type archetypes** organised into **18
district-style families** (§4.3.4). The archetype is the unit of authoring effort: one authored
archetype definition (program, fixture schedule, finish palette, anchor set, lighting template,
acoustic class) serves an average of 87 interiors.

| Family | Archetypes | Interiors | Districts |
|---|---|---|---|
| F01 Victorian/Edwardian masonry | 34 | 6,120 | Portside, Riverton W |
| F02 Interwar walk-up | 28 | 5,480 | Riverton, Halstead Town |
| F03 Postwar mid-rise | 26 | 4,310 | Riverton E, Ashford |
| F04 Contemporary high-rise | 22 | 3,180 | Cathedral Quarter, Marlow |
| F05 Suburban detached (5 era bands) | 41 | 5,632 | Suburban Belt, Valley towns |
| F06 Strip commercial | 19 | 2,940 | Ashford, arterials region-wide |
| F07 Big-box / warehouse | 12 | 1,180 | Cordova Logistics, Ashford |
| F08 Industrial process | 16 | 1,024 | Cordova Petrochem, Halstead Port |
| F09 Civic / institutional | 24 | 1,410 | region-wide |
| F10 Healthcare | 14 | 880 | 3 hospitals, 41 clinics |
| F11 Education | 18 | 1,240 | 118 schools, 4 colleges |
| F12 Hospitality | 21 | 2,180 | hotels, motels, B&Bs |
| F13 Food service | 26 | 3,410 | restaurants, cafés, bars, fast food |
| F14 Retail small | 24 | 3,860 | region-wide |
| F15 Agricultural | 19 | 2,240 | Meridian Valley |
| F16 Rural / forest / resource | 17 | 1,140 | Kestrel, Sable, Anza |
| F17 Transit infrastructure | 14 | 1,690 | 112 metro stations, depots, tunnels |
| F18 Heritage / derelict | 18 | 1,110 | ghost town, abandoned industry, adits |
| | **412** | **44,056** | *(some interiors count in two families; the distinct total is 37,436)* |

## 4.3 The Interior Layout Grammar (ILG)

### 4.3.1 Structure

A **shape-and-graph grammar** with a constraint solver. Not a Wave Function Collapse — WFC produces
locally plausible tilings but cannot guarantee global constraints (egress travel distance, vertical
service continuity, structural span), and those constraints are what make a floorplan read as a
building rather than a maze.

```
INPUT:
  envelope          — the floor plate polygon (from the building shell, §4.3.2)
  storey_index      — 0..N
  floor_to_floor    — m (from the style family and era: 3.1 m residential 1930s, 3.9 m office
                        1970s, 4.6 m loft-conversion, 2.6 m modern residential)
  use_class         — from §2.4.1 zoning + the establishment record (§3.11.2)
  era               — construction year, from the building record
  socio_tier        — from land value (§2.4.3)
  structural_system — {load-bearing masonry | steel frame | RC frame | flat-plate PT |
                       light timber | heavy timber | tilt-up | steel portal}
  core_location     — for multi-storey: the lift/stair core position (fixed by the ground floor)
  occupant_records  — for residential: the ResidentRecord(s) of the household (§3.9.2)

STAGE 1 — SPACE PROGRAM
  From (use_class, era, socio_tier, envelope area) select a PROGRAM TEMPLATE: a list of
  (space_type, target_area, area_tolerance, count) tuples. 412 templates authored.
  Example — F05 "Suburban detached, 1994, 3-bed, socio_tier 0.62, 148 m²":
    entry_hall 6.5 ±25% · living 22 ±20% · dining 12 ±25% · kitchen 14 ±20% ·
    wc 2.2 ±20% · utility 5 ±30% · stair 4.5 · landing 6 ±30% ·
    bed_main 15 ±15% · ensuite 5.5 ±20% · wardrobe 3.2 · bed2 11 ±15% · bed3 10 ±15% ·
    bathroom 6 ±20% · airing_cupboard 1.1 · hall_upper 5 ±30%
  Template selection is stochastic within the family, weighted by envelope aspect ratio and
  by the household composition (a household with 3 children gets bed2+bed3+bed4; a single
  occupant gets a study instead of bed3).

STAGE 2 — PARTITION (slicing-tree solve)
  Represent the plan as a binary slicing tree: each internal node is a horizontal or vertical
  cut with a position; each leaf is a room. This representation:
    · guarantees rectangular-ish rooms (real rooms are)
    · makes area constraints a 1-D root-find per cut
    · keeps the search space small (a 15-room plan has ~2.4M tree topologies vs an intractable
      free-form space)
    · maps directly onto real construction: party walls, joist spans, and plumbing runs
  Enumerate topologies by a stochastic grammar biased toward the family's typology (a Georgian
  plan is double-pile and symmetrical; a ranch is single-pile and linear; a shotgun is
  single-pile and 1 room wide).

STAGE 3 — CONSTRAINT SOLVE (simulated annealing, 4,000 iterations, ~12 ms)
  HARD constraints (rejection):
    · egress: ≥ 2 exits where occupancy load > 49; travel distance to an exit ≤ 45 m
      (unsprinklered) / 60 m (sprinklered); dead-end corridor ≤ 6 m / 15 m
    · structure: clear span ≤ system limit (6 m light timber, 9 m flat-plate, 14 m PT,
      24 m steel portal); column grid alignment with the storey above and below (a column
      may not land in a doorway on the floor below — a real and visible defect if violated)
    · accessibility: a 0.915 m clear route through all public spaces; a 1.5 m turning circle
      in an accessible WC; threshold ≤ 13 mm (this is BOTH a building-code requirement and the
      game's own accessibility requirement, §6.6.6 — the same constraint serves both)
    · envelope: no room outside the envelope; window-bearing rooms must touch an exterior wall
  SOFT constraints (cost):
    · area error: Σ (A_actual − A_target)² / A_target²
    · adjacency: a weighted graph of preferred/required adjacencies per space type
      (kitchen↔dining +3.0, kitchen↔entry +1.2, bathroom↔bedroom +2.4, bathroom↔kitchen −4.0
       (a real code and hygiene constraint), wc↔entry +2.0, utility↔kitchen +2.6,
       boiler↔exterior_wall +1.8)
    · wet-wall alignment: bathrooms, WCs, kitchens, and utility must share a plumbing wall
      where possible, and must stack vertically across storeys. Weight +3.8.
      ⇒ This is what produces the characteristic vertical alignment of bathrooms in an
        apartment building, which is a real structural/economic fact and which players
        recognise subconsciously as correct.
    · daylight: habitable rooms need glazing ≥ 8% of floor area; a dual-aspect room is
      preferred (+1.4); a room 3 deep from a window is penalised (−2.2)
    · privacy gradient: public → semi-private → private along the circulation path (+2.0)
    · proportion: room aspect ratio within the family's range (a 1:4 room is penalised)
    · circulation efficiency: corridor area as a fraction of total, target 8–18% (+/−)

STAGE 4 — CIRCULATION INSERTION
  From the egress solution: corridors, stairs, lifts, lobbies. Stair geometry from a real
  building-code riser/tread solve: 2R + G = 610–650 mm (R = riser, G = going), max riser
  190 mm, min going 250 mm, 12–16 risers per flight, a landing at every change of direction.
  ⇒ Stairs in MERIDIAN have correct proportions, which is why walking up them with the
    motion-matched locomotion (§3.10.5) looks right. Getting stairs wrong is the single
    fastest way to make a procedural interior look fake.

STAGE 5 — SERVICES INSERTION
  Plumbing risers (from the wet-wall alignment), drainage falls (a 1:80 gradient, which
  constrains floor levels and produces the real step-down in a bathroom), HVAC duct routes
  through a 400–900 mm plenum above the suspended ceiling (which is why corridor ceilings are
  lower than room ceilings — a real and ubiquitous detail), electrical distribution boards,
  and a flue/chimney where the heating system requires one (era-dependent: pre-1960 = a
  chimney in every heating room, which is why old houses have chimneys in odd places).

STAGE 6 — FINISH SCHEDULE
  Per surface: a material from the family's palette, weighted by (room type, socio_tier, era,
  occupant tenure, occupant income). Emits a MaterialRecord ID per surface (§3.1) — which is
  what feeds the acoustic bake (§3.7.3's RT60 is computed FROM THIS SCHEDULE, not authored),
  the friction model, and the ballistic model. **Shoot a wall in a procedural apartment and
  the penetration result is correct for its actual construction**, because the construction
  is real.

STAGE 7 — VALIDATION
  A code-compliance checker re-verifies every hard constraint plus:
    · window-to-room alignment: every window on the exterior façade must correspond to a
      room that wants one. **A window opening into a closet is a CI-blocking defect.**
    · door swing clearance: no door may swing into another door's arc or block a required
      clear width
    · no unreachable space: a flood-fill from the entry must reach every room
    · no non-manifold geometry, no self-intersecting walls, no zero-area rooms
  On failure: re-solve with a different seed (max 6 attempts), then fall back to a
  hand-authored safe template for the family (3 per family = 54 fallbacks).
  Measured first-pass yield: 91.4%. Fallback rate at ship: 0.31%.
```

**Cook cost:** 37,436 interiors × avg 2.4 storeys = 89,850 floorplans × 12 ms = **18 minutes** of
single-core solve time, parallelised to **11 s on 96 cores**. The layout grammar is essentially
free at cook time. Its cost is entirely in authoring and tooling (§6.8, R-01).

### 4.3.2 The exterior: building shell and façade grammar

Interiors must match exteriors. This is solved by **sharing a constraint, not by post-hoc
reconciliation**.

```
Mode A — INSIDE-OUT (62% of structures: detached, suburban, rural)
  Generate the space program first → derive the massing:
    · footprint from the program's ground-floor area and the parcel's buildable envelope (§2.4.2)
    · storeys from the program's total area / footprint, capped by zoning (§2.4.1) and by the
      viewshed ordinance
    · roof form from (era, region, style family, snow load, span): a gable at 30–45° in the
      snow-load districts, a hip at 22–30° in the suburban belt, a flat/parapet in the urban
      core, a mansard in F01, a shed in F16
    · CHIMNEY/FLUE POSITION FROM THE ACTUAL HEATING APPLIANCE LOCATION (§4.3.1 Stage 5) — a
      chimney in the wrong place is the most visible procedural tell there is
    · window positions from the interior's daylight-needing rooms, on a module grid
    · entry position from the interior's entry_hall
    · garage/porch/deck from the program and the parcel orientation

Mode B — OUTSIDE-IN (31%: row buildings, apartments over retail, infill)
  The envelope is fixed by the block/parcel grammar (§2.4.2). The interior solves within it.
  Deep plans (> 16 m) get a double-loaded corridor, a light well, or a rear wing — all
  authentic responses, chosen by the family's typology.

Mode C — COMPOSITE (7%: megastructures — metro stations, airport, mall, hospital, stadium)
  Bespoke massing; procedural internal plan within a hand-authored envelope subdivision.

RECONCILIATION (all modes):
  The façade generator and the plan solver share a WINDOW SCHEDULE. It is solved as a
  constraint satisfaction iterated to a fixpoint (measured: 3.1 passes average, 6 max).
  Invariants asserted in CI:
    · every window maps to an interior room that wants daylight, OR to a void/shaft/plant room
    · every daylight-needing habitable room has ≥ 1 window on an exterior wall
    · every door on the façade maps to an interior circulation terminus
    · every chimney/flue/vent/soil-stack on the façade maps to an interior service element
    · every exterior wall thickness matches the interior LayerStack (§3.4.2)
    · floor levels visible from outside (a window head height, a spandrel) match the interior
      floor-to-floor
  ⇒ The building is ONE object with two representations, not two objects glued together.
```

### 4.3.3 Façade assembly and style families

Composition follows the **base–shaft–capital** structure that almost all real architecture uses,
even when it is denying it:

| Band | Height | Content |
|---|---|---|
| **Base** | 0 – 4.5 m (1–2 storeys) | Plinth/plinth course, ground-floor treatment (storefront, entry, garage, blank wall), a distinct material (stone or glazed brick against a brick shaft), awnings/canopies, signage band, utilities (meters, vents, downpipes, a hydrant), and the **street-level detail layer**: litter, staining, kick-plate wear at 0.9 m from 40 years of shoes, a worn path on the step |
| **Shaft** | 4.5 m – (H − 2 m) | A repeating **bay module**: window opening + surround + spandrel + pier, with a vertical ordering element (pilaster, engaged column, or a recessed bay). Bay width from the structural module (3.0–4.2 m modern, 2.4–3.3 m masonry). Variation by a per-bay seeded permutation: 8–14% of bays differ (an AC unit, a satellite dish, a boarded window, a different curtain, a balcony, a fire escape landing, a satellite-mounted camera) |
| **Capital** | top 2 m | Cornice/parapet/coping, a roof-edge treatment, gutters and downpipes, roof plant (era-dependent: none pre-1950, cooling towers post-1970, PV post-2010), and the **runoff staining origin** (§4.3.4) |

**18 style families × 6 era bands × 3 condition tiers = 324 façade kit configurations.** Each kit
is 40–190 modular pieces. Total kit: **14,200 façade assets**, 68 MB of geometry, 1.4 GB of
textures (heavily atlased and shared — §6.1).

### 4.3.4 Aging, patina, and the staining model

A building's condition is a scalar `c ∈ [0,1]` derived from `(age, land_value, owner maintenance
budget from §3.11.2, exposure, damage history from the delta journal)`. `c` drives:

| Effect | Model |
|---|---|
| **Water-runoff staining** | **The most important one.** Compute the actual runoff paths from the roof/parapet geometry: for each façade point, the upslope contributing area on the roof and the façade above it, using the same flow-accumulation algorithm as the hydrology system (§3.3.1) run on the building's geometry at 0.1 m resolution. Stain intensity ∝ contributing area × prevailing wind-driven rain exposure (from §3.2's 90-year climatology, per façade orientation) × (1 − c). Produces the **streaks below parapets, below sills, and below copings** that make a building look 40 years old, positioned correctly for its actual geometry and its actual orientation. A south-west face in the prevailing wind is dirtier than a north-east face — computed, not painted. |
| Biological growth | Algae/lichen on surfaces with (a) low solar exposure (from the solar model), (b) high moisture retention (material porosity + orientation), (c) low `c`. A 0–1 mask that tints and roughens. |
| Rust staining | Below every exposed ferrous fixing, lintel, or railing, as a vertical streak of length ∝ age × exposure. Positions come from the actual fixing layout in the kit. |
| Paint/coating failure | A per-face procedural failure mask (alligatoring, flaking, chalking) with area ∝ (1 − c)² and a preference for south-west faces (UV load) and for the base band (splash-back, impact). Reveals the layer below via the LayerStack (§3.4.2) — a 1920s building with a 1974 repaint and a 1998 repaint shows **three colour layers at a paint-failure edge**, which is a real forensic detail and which the §3.12.6 investigation system can actually use (paint transfer evidence). |
| Glazing condition | 4–11% of panes cracked/missing/boarded by `c`; a per-pane soiling factor reducing transmission 2–19%; condensation and interior blind states driven by the occupant record. |
| Graffiti | A district-modulated procedural decal system with **7 authored style families** (tag, throwie, piece, stencil, poster, sticker, political) whose density is driven by land value, surface type, visibility, and a **removal model**: the city's maintenance budget (§3.11.5) removes graffiti from high-value districts within 3–14 days and from low-value districts within 90–never. **The graffiti density map is therefore an accurate visual readout of municipal spending inequality** — emergent, uncomfortable, true. |
| Repair patches | Where `c` is mid-range, patches: a mismatched brick course, a different-colour panel, a modern window in an old opening. Material records differ from the parent, so the patch is ballistically and acoustically distinct. |

### 4.3.5 Street-level detail layer

The 0–3 m band is where a player spends 95% of their attention. It gets its own generator with a
density budget:

| Element | Density (urban core) | Source |
|---|---|---|
| Litter | 0.4–14 items/m² by district cleanliness (from the §3.11.5 refuse budget) | 240 authored items, procedurally scattered with a wind-driven accumulation bias into corners, kerb lines, and against obstacles (a **deposition model** using the §3.2 wind field — litter collects where the wind drops it) |
| Street furniture | 1 per 18 m of frontage | 1,140 kit assets: benches, bins, bollards, signs, signals, hydrants, post boxes, planters, bike racks, bus shelters, newsracks, ATMs, vending |
| Signage | 1 per 6 m of commercial frontage | 14,800 authored signs (§4.1 rule: all legible text is authored), placed by establishment type and validated against the establishment record — **the sign on a shop says what the shop actually sells, because the shop record is real** |
| Utility | per parcel | Meters, downpipes, vents, cable runs, junction boxes, manholes (with real covers that can be lifted → a Class D interior), street-light columns (each with a luminaire state from §3.11.4) |
| Vegetation | 62 street trees/km of frontage (§3.3.2), plus planters, weeds in cracks (a **crack-colonisation model** driven by surface age and material) | Species from the T5 gradient mask (§2.3.3) |
| Parked vehicles | from the ledger's vehicle ownership and the parking model | §4.3.6 |
| Surface wear | per cell | A **desire-path and wear model**: paths are worn into grass where the pedestrian flow field (§3.8.4) has high density off-pavement; kerbs are chipped at junction radii; manhole covers are polished by tyres; steps are dished at their centres. All driven by actual simulated traffic and pedestrian counts, not painted. |

### 4.3.6 Parking

A real subsystem because it is the largest single determinant of street appearance in a car-
dependent region. 412,000 parking spaces: on-street 88,000, surface lots 142,000, structured
128,000, private/residential 54,000. Occupancy is driven by the §3.8.2 demand model per time slice
and by a **search-for-parking behaviour** in T1/T2 agents (a cruising-for-parking model that
contributes 11–28% of local traffic in Cathedral Quarter at peak — a real and well-documented
phenomenon, and one that makes downtown traffic feel correct).

Parked vehicles are **ledger-backed**: every parked car belongs to a resident or an establishment,
with a `VehicleRecord` (§3.5) that has real state — a registration expiry date, an inspection
sticker, damage history from the delta journal, a fuel level, and a lock state. A car parked in the
same place for 30 days accumulates a flat tyre, a dust layer, a parking ticket, and eventually a
tow — all real ledger events. **The player can identify a resident's car, learn their schedule from
it, and know when they are home by whether it is there.**

## 4.4 Memory budget per interior

### 4.4.1 Accounting model

An interior's memory is split between **shared pools** (which it allocates against but does not
add to) and **private data** (which is additional). Getting this wrong is how projects discover at
month 40 that interiors cost 4× the estimate.

```
SHARED (already budgeted in §5.10.1, an interior draws from these):
  · Virtual Texture pool (2,950 MB PS5, §5.10.1) — textures
  · Geometry pool (1,800 MB)             — virtualized clusters
  · Audio bank pool (520 MB total)       — interior-specific banks
  · Shared Interior Kit Atlas (180 MB)   — the 4,200 props and 615 assemblies for the
                                           3 resident style families

PRIVATE (the per-interior delta, charged against the 640 MB interior budget):
  · Baked GI: irradiance probes, AO, practical-light irradiance, sky-visibility
  · Acoustic probe set (§3.7.1) + convolution IRs (Class S only)
  · Dynamic light records
  · Set-dressing instance data + per-object mutable state
  · Floorplan graph + room graph (needed at runtime for §3.7.3's room detection and for
    navigation)
  · Interior-specific audio bank references
  · Bundle shell (the exterior shell geometry paired to the interior, §4.5.1)
```

### 4.4.2 Per-class allocation

| Component | Class S | Class A | Class B | Class C | Class D |
|---|---|---|---|---|---|
| **VT texture working set** *(draws from shared pool)* | 96.0 MB | 38.0 MB | 11.2 MB | 9.8 MB | 2.4 MB |
| **Virtualized geometry clusters** *(shared pool)* | 42.0 MB | 16.0 MB | 5.0 MB | 2.6 MB | 1.0 MB |
| Baked GI (probes @ SH2 48 B, AO, lightmap-equiv) | 18.0 MB | 8.0 MB | 2.4 MB | 0.62 MB | 0.14 MB |
| Acoustic probes (3 m lattice, 4.2 KB each) | 6.2 MB | 3.4 MB | 1.1 MB | 0.35 MB | 0.08 MB |
| Convolution IRs (4.8 s stereo 48 kHz, partitioned) | 5.5 MB | — | — | — | — |
| Dynamic light records (≤ 220 / 60 / 24 / 6 / 2) | 12.0 MB | 5.0 MB | 1.6 MB | 0.04 MB | 0.01 MB |
| Set-dressing instances + mutable object state | 24.0 MB | 16.0 MB | 9.0 MB | 0.58 MB | 0.06 MB |
| Floorplan + room graph | 4.1 MB | 2.6 MB | 1.4 MB | 0.31 MB | 0.09 MB |
| Interior audio banks | 12.2 MB | 6.0 MB | 2.3 MB | 1.6 MB | 0.72 MB |
| **PRIVATE SUBTOTAL** | **82.0 MB** | **41.0 MB** | **17.8 MB** | **3.5 MB** | **1.1 MB** |
| Shared-pool draw | 138.0 MB | 54.0 MB | 16.2 MB | 12.4 MB | 3.4 MB |
| **TOTAL RESIDENT CEILING** | **220 MB** | **95 MB** | **34 MB** | **16 MB** | **6 MB** |

*(Numbers rounded to the budget ceiling; CI asserts the ceiling per class, not the breakdown.)*

**Note the composition shift.** Class C (a residential apartment) is **78% shared-pool draw** — it
costs almost nothing private, because its textures and geometry are the kit's. Class S is 37%
private, because it is bespoke. This is precisely why the archetype/kit strategy (§4.2.2) is what
makes 37,436 interiors affordable: **the marginal cost of the 19,200th apartment is 3.5 MB of
private memory and ~180 bytes of disk.**

### 4.4.3 Global interior budget (640 MB)

| Slot | Budget | Content |
|---|---|---|
| Shared Interior Kit Atlas | **180 MB** | 3 resident style families (the loaded districts'); 4,200 props, 615 assemblies, atlased. Evicted/replaced on district change with a 6 s cross-fade |
| Active interior (worst case Class S) | **220 MB** | The interior the player is inside |
| Pre-warm ring (4 slots × ≤ 40 MB private) | **160 MB** | Speculatively loaded bundles at reduced fidelity (§4.5.2). Capped at 40 MB private per slot regardless of class — a pre-warmed Class S loads its geometry and GI but at a −1.5 mip texture bias |
| Bundle shells (paired exterior shells) | **40 MB** | §4.5.1 |
| Eviction hysteresis reserve | **40 MB** | Recently-evicted bundles held for 2.0 s against ping-pong (§6.2) |
| **TOTAL** | **640 MB** | |

### 4.4.4 Disk cost

| Item | Size | Notes |
|---|---|---|
| Shared interior kit (4,200 props, meshes + textures) | 21.4 GB | Deduplicated by hash across all 37,436 interiors (§5.9.3) |
| Façade kits (14,200 assets) | 18.2 GB | |
| Hero/signature bespoke geometry and textures (1,600 interiors) | 44.6 GB | The largest single interior disk item |
| **Procedural interior definitions (35,836)** | **6.5 MB** | **180 bytes each: seed, archetype ID, program-template ID, occupant record refs, override list.** Interiors are **generated at load, not stored.** |
| Baked GI (37,436 interiors) | 3.9 GB | |
| Acoustic probes (1.94M probes) | 8.1 GB | |
| Convolution IRs | 0.4 GB | |
| Signage kit (14,800 authored) | 2.8 GB | |
| Anchor prop kit (2,400 × 3 condition variants) | 6.2 GB | |
| **Interior total** | **105.6 GB** | of a 180 GB install (§6.1) |

**The 6.5 MB line is the architectural payoff.** Storing 35,836 authored interiors at even 120 KB
each would be 4.3 GB of level data and — far worse — 4.3 GB of *data that must be versioned,
reviewed, and rebaked*. Generating them costs 18 minutes of cook time and 6.5 MB of overrides. This
is the difference between shipping and not shipping (§6.8, R-01).

### 4.4.5 Platform scaling

| Platform | Interior budget | Adjustments |
|---|---|---|
| PS5 / XSX | 640 MB | Reference |
| PS5 Pro / next | 700 MB | +1 pre-warm slot, +0.5 mip on the active interior |
| XS Series S | 380 MB | Kit atlas 110 MB (2 families), pre-warm ring 2 slots × 24 MB, active ceiling shifted one class down (S→A, A→B, C→D). Class D stays D. **Interior count is unchanged** — only fidelity. |
| PC Low (8 GB VRAM) | 480 MB | Class shift as Series S; VT pool 2,200 MB |
| PC Mid (12 GB VRAM) | 640 MB | Reference |
| PC High (16 GB+) | 900 MB | +2 pre-warm slots, full-mip active interior, 4 kit families resident |

**Invariant (`AI-08`):** interior *count*, *layout*, *occupant identity*, and *set-dressing
content* `MUST` be identical on every platform. Only texture resolution, GI quality, dynamic light
count, and pre-warm depth may differ. A Series S player and a PC Ultra player explore the same
apartment with the same photograph on the same wall.

## 4.5 Zero loading screens

### 4.5.1 The interior bundle

```
InteriorBundle = an ATOMIC streaming unit containing:
  · the interior volume (floorplan, geometry, GI, acoustics, set dressing, lights)
  · its EXTERIOR SHELL (the building's façade geometry, roof, and immediate ground plane) —
    because the shell must be resident whenever the interior is, or entering produces a
    hole in the world behind you
  · its portal volumes (the door/window trigger geometry and their acoustic portals)
  · its room graph and floorplan graph (needed at runtime for §3.7.3 and navigation)
  · its mutable state slab (§5.8.4)
Loaded together. Evicted together. NEVER partially resident — a partially resident bundle is the
source of every "I could see into an unfinished room" bug.
```

### 4.5.2 Placement classes and the portal render

Interiors cannot always live at their real world position: a basement is bigger than its building,
a metro station is 400 m long under a block that has other buildings on it, and a row house's
interior must not intersect its neighbour's. Three placement classes:

| Class | Share | Placement | Transition |
|---|---|---|---|
| **P1 — In-situ** | 62% | Interior geometry occupies the real building footprint, inside the shell. Requires: interior volume ⊆ shell volume, and no overlap with any neighbouring interior. | Direct. No portal render. The camera simply moves through the door. **Cheapest and best.** |
| **P2 — Undercroft** | 31% | Interior placed in a per-district **offset volume** at `Z − 400 m` within the same world cell range, so it shares the cell's streaming and origin-rebase behaviour but does not intersect anything. | **Portal dual-view render.** The door is a portal plane. While the portal is in the frustum, the renderer draws the offset interior with a camera translated by `+400 m Z`, into a stencil-masked region. The door frame occludes the discontinuity. Cross-fade the camera to the interior space over 0.18 s as the player crosses the plane — invisible because the player's view is entirely inside the door frame at that moment. |
| **P3 — Detached megastructure** | 7% | Metro stations, airport terminals, malls, hospitals, stadiums: placed in a dedicated world-cell range with their own local origin. | Portal dual-view, plus an **interior sub-streaming system** (§4.5.4) because a Class S megastructure exceeds the 220 MB ceiling for a single resident set — it streams internally by wing/concourse. |

**Portal render cost:** one additional frustum cull + draw for the offset volume, only while the
portal is visible and only for P2/P3. Measured: **0.42 ms GPU** for a typical P2 residential
transition (one extra interior's worth of shading). Budgeted in §5.11.2.

**Lighting continuity across a portal** is the hard part, and getting it wrong is instantly
visible. Solution:
- The interior's baked GI includes an **entry-irradiance probe** at the portal that stores the
  exterior's incident radiance distribution (SH3, 12 coefficients) **baked for 24 sun positions ×
  5 sky conditions = 120 samples**, interpolated at runtime by the actual sun/sky state. The
  interior's GI therefore responds to the actual weather and time of day at its door, without
  any runtime global trace into the interior.
- The **exposure adaptation** (§5.5.6) uses a single global autoexposure with a 0.42 s
  adaptation time constant — matching the human pupillary light reflex (0.2–1.0 s). Walking from
  a 40,000 lux street into a 200 lux interior produces a real, correctly-timed adaptation ramp.
  This is not a loading mask; it is physiology, and it does 60% of the work of hiding the
  transition.
- **Light spill**: the interior's portal emits onto the exterior ground (a projected decal from
  the entry-irradiance probe), and the exterior's daylight spills in. Both directions, both
  correct.

### 4.5.3 Predictive pre-streaming

```
For each portal within 40 m of a simulation source, at 4 Hz:
  P(entry) = w_dist · exp(−d/6.5)
           · w_align · max(0, cos θ between player velocity and the portal normal)^2.2
           · w_gaze  · (0.35 + 0.65 · max(0, cos φ between camera forward and portal bearing)^3)
           · w_path  · 1.0 if the player's current navigation path intersects the portal
           · w_intent· 2.4 if an interaction prompt is targeted, 3.1 if a mission objective is
                       behind it, 1.0 otherwise
           · w_hist  · 1.8 if the player entered this portal in the last 6 min
           · w_vehicle · 0.06 if the player is in a vehicle moving > 12 m/s (they are driving
                       past, not entering) — EXCEPT for garages and drive-throughs, which get
                       a dedicated check
  normalised to [0,1]

  P > 0.35 → HIGH priority stream, full fidelity, into a pre-warm slot
  P > 0.08 → LOW priority stream, −1.5 mip texture bias, coarse geometry, no dynamic lights
  P ≤ 0.08 → not streamed

  Budget: 4 pre-warm slots (6 on PC-High, 2 on Series S), 40 MB private each.
  Lead time target: 8–12 s at walking speed, 3–5 s at vehicle speed into a garage.
```

**Vehicle case:** driving into a garage at 12 m/s gives 3.3 s from 40 m. The predictor raises its
scan radius to 140 m for vehicle sources and adds a **road-graph corridor bias**: garages and
parking structures on the player's predicted route (from the §3.8.1 macro traffic model's link
sequence) are pre-warmed at 300 m. This is the same mechanism as §5.8.5's highway-spine
pre-streaming.

### 4.5.4 Vertical streaming (elevators and multi-storey)

A 74-floor elevator ride at 6 m/s is 12.4 s of continuous vertical movement through 74 potential
bundles. Naive handling is a guaranteed hitch.

```
· An elevator CAR is a SIMULATION SOURCE (§5.8.2) from the moment it is called. The call itself
  is the intent signal — no prediction needed.
· On call: the destination floor's bundle is streamed IMMEDIATELY at high priority. This is the
  single highest-value streaming optimisation in the interior system, because the intent is
  known with certainty 12–90 s in advance (the ledger's elevator demand model, §3.9.6, knows
  where the car is going before the player does).
· While the car is in the shaft: a sliding window of ±2 floors around the car is resident, plus
  the origin and destination. The shaft itself is a Class D bundle (6 MB) resident for the ride.
· Floors passed are NOT loaded. The view through the shaft (where the shaft has a glazed section,
  as Vermillion Tower's does for 18 floors) is rendered from a LOW-FIDELITY floor proxy: a
  pre-baked impostor per floor (a 512×512 albedo + normal + emissive card, 1.1 MB each, 74 floors
  = 81 MB, resident only for glazed-shaft buildings — 4 in the region). At 6 m/s the parallax
  error of an impostor at 4 m distance is below the perceptual threshold, and the emissive card
  correctly shows lit and unlit floors driven by §3.9.6's occupancy.
· On arrival: the destination bundle is already resident (streamed at call time). Cross-fade.
```

**Stairs:** a stairwell is a single Class D bundle containing all floors' landings (a stairwell is
geometrically repetitive, so one bundle covers it). The doors onto each floor are portals into that
floor's bundle, pre-warmed at 2 floors of separation by the player's vertical velocity (a real
predictable signal: 0.4 m/s of climb = floor 3 in 8 s).

### 4.5.5 The vestibule mask (worst-case fallback)

When prediction misses — the player sprints at an unpre-warmed door, or a vehicle enters a garage
at 40 m/s — the system needs a cover. It is **diegetic**:

```
1. The door animation is lengthened from 0.28 s to 0.55 s (a heavier door — a physically
   plausible variation, and door mass IS varied in the authored data, so this is not a lie).
2. Exposure adaptation is forced to its slowest time constant (0.85 s) for this transition.
   Justified: the eye does adapt more slowly from a bright exterior.
3. A 0.15-alpha, 0.2 s vignette-darken at the frame edges, reading as the door frame.
4. The stream is issued at MAXIMUM priority with a temporary governor override (§5.11.3):
   texture pool bias −1 mip region-wide for 0.6 s, cloud pass dropped to low-res only,
   crowd cap halved. This buys ~140 MB of instantaneous I/O headroom and ~2.4 ms of frame time.
5. If STILL not resident at the moment of crossing: the door is locked, and the player gets a
   "the door is stuck" interaction prompt requiring a second attempt (a 0.9 s delay).
   Frequency target: < 0.02% of transitions.
```

**Target:** vestibule mask used in **< 0.4%** of transitions; step 5 used in **< 0.02%**. Measured
by the bot harness (§1.4 USP-4, §6.7).

### 4.5.6 Portal culling (why unobserved interiors are free)

```
An interior bundle contributes ZERO rendering cost when:
  · no portal to it is within the view frustum, OR
  · the player is outside the portal's VIEW VOLUME (a baked convex hull extending 12–40 m from
    the portal, sized by the interior's sightline analysis — you cannot see into a 200 m²
    warehouse from 40 m away through a 1 m door, so its view volume is 12 m; you can see into a
    small shop from 40 m, so its view volume is 40 m)
  · AND the player is not inside it

Enforcement: a two-level portal hierarchy. A building's bundles are grouped under a
BUILDING PORTAL SET; the set is rejected by a single AABB frustum test before any individual
portal is tested. 41,200 blocks ⇒ 41,200 building-set tests (a GPU instance-cull, ~0.03 ms),
of which ~140 survive to per-portal testing (~0.01 ms).

Additionally: interior geometry is EXCLUDED from the exterior's occlusion structures (HZB,
BVH for GI/acoustics/ballistics) unless the bundle is resident. An unresident interior does not
cast a shadow, does not block a bullet, and does not reflect. It does not exist. This is correct
behaviour, and it is what makes 37,436 interiors free.

EXCEPTION: the shell always exists (it is part of the exterior). A bullet that hits a wall hits
the wall's LayerStack (§3.4.2) whether or not the interior behind it is resident. If the bullet
penetrates and the interior is not resident, the bundle is force-streamed at maximum priority
BEFORE the projectile's exit is resolved — the projectile's simulation is suspended for up to
6 frames. Frequency: measured at 0.7 events/hour of play. This is the correct behaviour and it
is the reason the LayerStack and the streaming system must share a contract.
```

## 4.6 Set dressing seeded by the Resident Record

This is USP-4's substance. It is specified in detail because "procedural set dressing" usually
means "random prop scatter", which is exactly what produces the fake look.

### 4.6.1 Inputs

```
From the occupant ResidentRecord(s) (§3.9.2):
  income, wealth, debt, tenure, residence_duration, household_composition, ages,
  occupations, education_proxy, OCEAN, ethnicity_id, languages, religions,
  health.conditions, health.medications, health.injuries, health.mobility,
  assets.possessions (a 64-entry list of owned object archetypes with condition and
                      acquisition date),
  assets.wardrobe, assets.vehicles,
  relationships (who visits, who lives here, who is estranged),
  history.witnessed_events, history.player_interactions,
  employment (uniform? tools? work-at-home?),
  psychology.openness (novelty seeking), conscientiousness (tidiness),
  state_vector (fatigue, intoxication, stress — affects the CURRENT state of the room)

From the world:
  §3.2 weather (windows open/closed, a dehumidifier running, snow on the doormat),
  §3.11.2 establishment inventory (what is on the shelves of a shop — a real stock number),
  §3.11.5 news/events (a newspaper headline, a flyer, a protest poster),
  §2.10 calendar (seasonal decoration, holiday state, a birthday card on the mantel with the
                  correct date),
  the delta journal (damage, theft, prior visits by the player)
```

### 4.6.2 The decorator

```
STAGE 1 — STYLE INFERENCE
  style_vector ∈ Δ^33 (a distribution over 34 style archetypes) from a linear model on
  (income, age cohort, openness, tenure, residence_duration, occupation, region, ethnicity,
   land_value). Constrained: a household's style vector persists over time with a slow drift
  (0.04/yr), because people do not redecorate abruptly. A record that has lived here 19 years
  has a style vector 19 years accumulated — including furniture acquired in 2012 that is now
  dated. **Dated furniture is the single strongest age cue in a real interior.**

STAGE 2 — FURNITURE LAYOUT (per room)
  A rule-based layout with REAL proxemics and circulation:
    · primary circulation route: 0.9 m clear width from the entry to every habitable room,
      never obstructed (except by a deliberate "cluttered household" state at low
      conscientiousness + high household density)
    · focal point: a seating arrangement faces its focal element (TV, fireplace, view, table)
      within ±22°
    · conversation distance: seats in a social grouping are 1.2–2.4 m apart and within a 90°
      arc (Hall's proxemics: the social zone). A household of 4 does not have 4 chairs in a
      row; it has a sofa and two chairs at 1.8 m facing a coffee table.
    · surface affordance: objects are placed on surfaces that support them (a table takes a
      bowl, not a floor), with a **clearance and grouping model** — objects cluster (a desk
      cluster: monitor, keyboard, mug, papers, pen, lamp within a 0.6 m radius) rather than
      distributing uniformly. Uniform distribution is the #1 procedural-set-dressing tell.
    · reach and use: frequently used objects are within 1.4 m of their use point and at
      0.7–1.5 m height; rarely used objects are high, low, or in a back room
    · symmetry breaking: an authored 8–14% of placements are deliberately slightly askew
      (a chair pushed back from a table, a book left open face-down, a mug on a coaster-
      adjacent surface) with the magnitude scaled by (1 − conscientiousness)

STAGE 3 — LIFE EVIDENCE (the differentiator)
  Conditional object injection, each rule tied to a record field:
    · household has children aged 2–11 → toys (age-appropriate, from the age distribution),
      child-height furniture, stair gates if < 3 y, artwork on the fridge (authored child-
      drawing assets, 340 of them, selected by child age — a 4-year-old's drawing and a
      9-year-old's drawing are visually distinct and correctly so), height marks on a
      doorframe if residence_duration > 4 y
    · health.medications non-empty and age > 62 → a pill organiser on the kitchen counter,
      a medication list on the fridge, a mobility aid by the bed
    · health.injuries present → the injury's aid (a crutch, a sling on a chair back)
    · assets.vehicles non-empty → keys in a bowl by the entry, a parking permit on the
      counter, a garage door remote, work boots by the door
    · employment.occupation == 51-2031 (transit operator) → a uniform on a hook, a
      high-vis jacket, a transit authority mug, a timetable printout, a union badge
    · union_member → union stickers, a bargaining-committee flyer, a strike fund notice
    · relationships includes a child with strength < −0.4 (estranged) → NO photographs of
      that child, while photographs of other children are present. The absence is the detail.
    · history.player_interactions includes a positive event → a memento of it
    · assets.possessions includes a musical instrument → an instrument, a stand, sheet music,
      and (from conscientiousness) either a case or an uncase'd instrument on a stand
    · a pet in the household (pets are ledger entities) → a bowl, a bed, a scratched
      doorframe at the correct height for the animal, a litter tray or a dog door
    · tenure == 'rent' and land_value low → a landlord's notice, a repairs-request log,
      unfixed defects (a stain, a broken blind) that a tenant would not repair
    · residence_duration < 90 days → boxes, unhung pictures leaning against a wall, a
      mattress without a frame
    · state_vector.intoxication > 0.4 at 22:00 → open bottles, an ashtray if a smoker
    · §2.10 season → a coat by the door (winter), a fan (summer, if no AC per §3.11.4),
      seasonal decoration within 6 days of a holiday, removed within 9 days after
    · §3.2 weather at generation time → windows open (mild, dry) or closed (rain, cold,
      high pollen), a doormat state, a damp coat

STAGE 4 — WEAR AND CONDITION
  Every placed object has a condition scalar from (acquisition_date, quality tier from income,
  use frequency from the room's traffic, household density, conscientiousness). Worn objects
  get a condition variant (4 authored per anchor prop). Textile fade is computed from the
  ACTUAL solar exposure of the surface (§2.10's sun path × the window's orientation × the
  object's distance from it) — a sofa 1.2 m from a south-west window has a faded right arm and
  not a faded left arm. **This is free once the sun model and the window schedule exist, and
  it is the detail that makes an interior look lived-in rather than dressed.**
  A wear path on the floor (a 4–11% gloss reduction along the circulation route from Stage 2),
  and a "rug shadow" where a rug was moved.

STAGE 5 — ANCHOR PLACEMENT
  1–4 hand-authored anchor props per interior (§4.1), selected by (family, archetype, style
  vector, socio_tier) with a per-record seed. Anchors carry the identity: a specific worn
  armchair, a specific espresso machine, a specific framed photograph (with an authored subject
  consistent with the household composition), a specific poster.
  Rule: an anchor MUST be visible from the room's primary viewpoint (a visibility solve on the
  Stage-2 layout). An anchor nobody sees is wasted authoring.

STAGE 6 — VALIDATION
  · no object intersects another, a wall, or the circulation route (a swept-volume check)
  · no object floats or sinks (a support-surface check)
  · every door and window opens without collision
  · the room's occupancy load (from the furniture count) does not exceed its code limit
  · **a "readability" check**: from the entry viewpoint, the set dressing must convey the
    occupant's (income tier, age band, household type) with ≥ 70% classifier accuracy against
    the actual record. This is an automated proxy for §1.4 USP-4's blind test, run on 100% of
    interiors at cook time using a trained classifier. **Interiors that fail are flagged for
    designer review** — 3.1% fail at current tuning, ~1,160 interiors, which is a real QC
    workload and is budgeted in §1.8.
```

**Determinism:** the decorator is seeded by `hash(resident_id, parcel_id, storey_index,
generator_version)`. The same interior generates identically on every run and every platform. No
storage. Overrides (a designer's hand-tweak) are stored as a sparse patch list (avg 2.4 entries per
interior, 96 bytes each = 8.6 MB region-wide).

**Cook cost:** Stage 1–6 at 340 ms per interior × 35,836 = 3.4 hours single-core, **2.1 min on 96
cores**. Free.

## 4.7 Interior lighting

### 4.7.1 Strategy

Interiors do **not** use the exterior's global illumination solution. Tracing GI into 37,436
interiors at runtime is unaffordable, and it is unnecessary: interiors are enclosed, their lighting
is dominated by static geometry and practical fixtures, and the dynamic element (daylight through
the portal) is bounded.

```
LAYER 1 — BAKED STATIC (offline)
  · irradiance probes: SH order 2 (9 coefficients × RGB = 108 B, quantised to 48 B) at a
    1.5 m lattice in Class S/A, 3.0 m in B/C/D. Interpolated trilinearly at runtime with a
    normal-direction bias.
  · baked AO: per-vertex, 8-bit.
  · practical-light baked contribution: every fixture that is ON in the "default" state is
    baked into the probe field. This is the 90% case (a shop at opening hours, a home at
    evening).
  · bake cost: 340 core-hours region-wide (a path tracer at 64 samples/probe with a
    denoiser). Incremental per bundle on change (§5.14.3).

LAYER 2 — PORTAL DAYLIGHT (runtime, cheap)
  · the entry-irradiance probe at each portal, baked for 120 sun/sky states (§4.5.2),
    interpolated by the ACTUAL current sun position and sky condition from §3.2/§5.7.
  · a **portal light**: a rectangle area light at the portal whose radiance is the
    interpolated exterior radiance, contributing to the interior's dynamic lighting and to
    the light spill onto the exterior ground. 1 per portal, ≤ 6 per bundle.
  · a **sun patch**: where direct sun enters, a projected quad with the portal's shape,
    positioned by the actual sun vector and clipped by the interior geometry. Animated
    correctly across the day — a sun patch that moves across a kitchen floor over 3 hours is
    one of the most convincing interior details available and it costs 0.06 ms.

LAYER 3 — DYNAMIC PRACTICALS (runtime)
  · fixtures whose state differs from the baked default: a light the player switches on, a
    TV, a desk lamp, an oven light, a torch. Shadow-casting only for ≤ 2 per view (a hard
    cap; the rest are non-shadow-casting with a baked-approximation falloff).
  · count ceilings: Class S ≤ 220 records (≤ 40 active, ≤ 2 shadowed), A ≤ 60/16/1,
    B ≤ 24/8/1, C ≤ 6/4/1, D ≤ 2/1/0.
  · flicker, colour temperature (2,200 K incandescent to 6,500 K daylight, from the fixture's
    era — a 1974 house has 2,850 K tungsten and a 2024 apartment has 4,000 K LED, and the
    difference is immediately legible), and a failure state from §3.11.4.

LAYER 4 — EMISSIVE & SCREENS
  · TVs and monitors display AUTHORED content (a fictional broadcast schedule from §3.11.5's
    news model — the TV in a resident's home is playing what the news model says is on, at the
    time the schedule says). 340 authored broadcast segments, 6 min average, streamed at
    1.2 Mbps.
  · an appliance panel, an exit sign, a neon sign (with a real transformer hum in §3.7.6 and a
    real flicker spectrum).
```

### 4.7.2 Exposure and adaptation

A single global autoexposure with a **physiologically-calibrated adaptation curve**:
- Luminance measurement: a 1/8-res downsample chain (6 mips) → a weighted average with a
  centre bias (0.62) and a bright-region rejection (the 88th percentile, not the mean, to avoid
  blowing out when looking at a window).
- Adaptation: `L_adapt(t+Δt) = L_adapt(t) + (L_target − L_adapt(t))·(1 − e^(−Δt/τ))` with
  `τ_dark = 0.85 s` (dark adaptation, slow) and `τ_light = 0.24 s` (light adaptation, fast).
  Real human values are 0.2–1.0 s for the pupil and up to 22 min for full rhodopsin
  regeneration; the long tail is represented by a second, slower term with τ = 42 s and a
  0.18 weight, which produces the correct "walking into a dark room and slowly seeing more"
  experience.
- **Range:** the renderer supports 0.0008 cd/m² (starlight, dark-adapted) to 120,000 cd/m²
  (direct sun). A 1:1.5×10⁸ range. Tone mapping is a filmic curve with a night-vision
  branch: below 0.03 cd/m² the response shifts toward scotopic (a desaturation and a
  blue-green shift following the Purkinje effect), which is both correct and a strong
  atmospheric tool.

## 4.8 Interior simulation

An interior is not a set. It has state.

| System | Model | Tick |
|---|---|---|
| **Occupancy** | From §3.9.6: a count per room, driven by the ledger's building worker/resident roster and their schedules. Drives acoustics (crowd absorption, §3.7.3), lighting controllers, HVAC load, restroom demand, and elevator demand. | 1/60 Hz |
| **HVAC** | A 1-node thermal model per zone: `C·dT/dt = Q_internal(occupants 75 W each + equipment + solar gain through the portal from §4.7.1) − Q_hvac − U·A·(T_in − T_out from §3.2)`. Setpoint from the establishment record; a **failure state** when §3.11.4's energy model sheds load. Outputs: a temperature (which affects §3.14's thermal comfort and clothing), an audible HVAC noise floor (§3.7.5's ambient), and a **smoke/fire damper state** (§3.3.5). | 1/60 Hz |
| **Lighting controllers** | Per-luminaire state from (occupancy, time-of-day, a photocell reading from §3.2's exterior irradiance, a manual override, a failure state, and a budget-driven maintenance state from §3.11.4). A real controller: an office at 18:30 has a 40% occupancy-swept floor where the lights have been switched off bank by bank. | 1 Hz |
| **Plumbing** | Water demand from occupancy; a supply pressure from the municipal model; **a failure state** (a burst pipe → a flood event → a shallow-water solve (§3.3.3) inside the interior → damage to the delta journal → a maintenance work order → a closed establishment). Real and reachable. | 1/60 Hz |
| **Air quality** | CO₂ from occupancy (a crowded bar reaches 1,800 ppm, which affects §3.14's cognition), CO from a gas appliance or a running vehicle in an attached garage (**a lethal confined-space hazard, modelled, and a real cause of accidental death in attached-garage houses**), smoke infiltration from §3.3.5, radon in basements on the Anza Basin's granite, and methane in the Cordova Flats tanks. Class D interiors have a **confined-space entry protocol** in the interaction system. | 1/60 Hz |
| **Appliances** | Per-establishment and per-household appliance states: a fridge compressor cycle (an audible 42 s on / 18 min off — a real sound design anchor for "this is a home"), a TV (§4.7.2), a cooker, a washing machine (a 47-min cycle that the resident's schedule accounts for), an alarm clock. Each is an audio emitter and an energy load. | 1 Hz |
| **Security** | Lock state, alarm state, camera coverage (§3.12.6's 14,200 cameras), a doorbell, a window state. A burglary target's security level is a real derived quantity from the occupant's income and the district's crime rate — **a well-secured house in Marlow Heights takes 4.2× longer to enter than an unsecured one in Riverton East, and makes more noise doing it, which feeds §3.7.5 and therefore §3.12.** | 1 Hz |

## 4.9 Interior navigation

Interiors have their own navmesh, generated at cook time per bundle:
- 8 m tiles within the bundle, 0.25 m voxel resolution.
- **Off-mesh links** for: doors (with an open/close cost and a lock state), windows (climbable
  if the sill height and the record's mobility allow), stairs (per-step links with a slope cost),
  counters (vault links), and furniture (a "squeeze behind" link where the clearance is 0.4–0.6 m).
- Dynamic carving for destruction (§3.4): a breached wall creates a **new** off-mesh link; a
  collapsed floor removes one. Re-carve at 8 m tile granularity, async, ≤ 4 tiles/frame
  (§6.3.5).
- The room graph (§4.3.1 output) is the **coarse** layer: agents path room-to-room on the graph,
  then locally on the navmesh. This is what lets a T1 agent in a 40-room hospital find its way
  without a fine navmesh query.

## 4.10 Validation and CI gates

| Gate | Assertion | Failure action |
|---|---|---|
| `INT-01` | Every interior's private memory ≤ its class ceiling (§4.4.2) | Build fails |
| `INT-02` | Global interior residency in the 12 golden paths (§6.7) ≤ 640 MB | Build fails |
| `INT-03` | Zero windows mapping to a non-daylight room (§4.3.2) | Build fails |
| `INT-04` | Zero egress violations (§4.3.1 Stage 3 hard constraints) | Build fails |
| `INT-05` | Zero unreachable rooms (flood-fill from entry) | Build fails |
| `INT-06` | Zero set-dressing object intersections (§4.6.2 Stage 6) | Build fails |
| `INT-07` | Accessibility route present in 100% of public interiors (§4.3.1) | Build fails |
| `INT-08` | ≥ 70% occupant-classifier accuracy on the readability check (§4.6.2) across a 2,000-interior sample | Warn at 70–75%, fail < 70% |
| `INT-09` | Bot traversal of 2,500 doors: zero loading screens, zero frames > 33 ms | Build fails |
| `INT-10` | Fallback-template usage ≤ 1.0% of stock interiors | Warn |
| `INT-11` | Vestibule-mask usage ≤ 0.4% of transitions in the bot harness | Warn at 0.4%, fail at 1.0% |
| `INT-12` | Determinism: the same seed generates byte-identical set dressing across 3 platforms | Build fails |
| `INT-13` | No unlit void: every interior has ≥ 1 light source or an authored dark state with an entry-irradiance probe | Build fails |
| `INT-14` | Interior count == 37,436 and the class distribution matches §4.2.1 ±2% | Build fails |
