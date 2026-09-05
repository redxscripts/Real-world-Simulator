# 02 — World Geography & Map Layout Plan

**Region:** New Vermillion (fictional jurisdiction; North Pacific–analogue coast)
**Anchor metropolitan area:** Vermillion City, population 4,127,000 (metro statistical area)
**Engine world extents:** `X ∈ [-11200, +11200] m`, `Y ∈ [-9300, +9300] m`, `Z ∈ [-120, +2480] m`
**Bounding footprint:** 22.4 km × 18.6 km = 416.6 km² · **Land fraction 60.5%** · **Land area 252.0 km²**
**Water area:** 78.0 km² (navigable, swimmable, and diveable to −68 m)
**Vertical span:** 2,600 m (deepest diveable point to highest summit)

---

## 2.1 Coordinate system, precision, and origin policy

This is decided first because it constrains every downstream system.

| Property | Specification |
|---|---|
| Authoritative world coordinate | `WorldPosition` = `double[3]` (F64), metres, right-handed, **+X east, +Y north, +Z up** |
| Authoritative world time | `WorldTime` = `double`, seconds since epoch `2031-01-01T00:00:00Z` (monotonic, never wall-clock) |
| Rendering transform | Per-cell `float3x4` (F32) relative to a **rebased origin**; GPU shaders receive only cell-local F32 |
| Physics | **Island-local** F32. Each physics island stores its own origin; bodies are simulated relative to it and rebased on island migration |
| Floating origin rebase | Triggered when the primary simulation source exceeds **4,000 m** from the current origin. Rebase is applied atomically at a frame boundary to *all* subsystems via `IOriginConsumer::OnRebase(delta)` |
| Sub-millimetre drift budget | ≤ 0.05 mm per rebase (guaranteed by F64 authoritative storage; F32 is render-only) |
| Navmesh / cost-field | F32, cell-local, 16-bit quantised within a 64 m cell (0.98 mm resolution) |
| Terrain heightfield | F32 relative to a per-tile base elevation, 16-bit quantised |

**Why F64 authoritative and F32 everywhere else.** At 11,200 m from origin, F32 has ~1.0 mm of
quantisation error; at 22,400 m it is ~2 mm. That is enough to cause visible jitter in
high-frequency suspension simulation and to break determinism across platforms with different
compiler contraction behaviour. The rule is therefore: **the simulation never stores a large
coordinate in F32.** Every subsystem that touches GPU or physics works in cell-local or
island-local space. `MUST NOT` be violated — see §6.5.2 for the divergence failure this prevents.

**Timezone & DST.** New Vermillion observes DST. All schedule data (§3.10) is authored in local
wall-clock time and **compiled to monotonic world time** at cook, using a timezone rule table with
explicit DST transitions. A 23-hour and a 25-hour day each occur once per simulated year. The
schedule solver `MUST` handle both without double-booking or missed shifts: activities are keyed to
monotonic intervals, and wall-clock recurrence rules are expanded at generation with
"skip/shift/anchor" resolution policies per activity class (a 07:00 shift start on a DST-spring-
forward day anchors to 06:00 or 08:00 per the resident's employer policy attribute, not silently).
This is a real bug class in every schedule-driven game and it is specified here because it is
invisible until it breaks a 40-hour playtest.

## 2.2 Area budget

Every square metre is accounted for. CI asserts the sum.

| Zone | Land area (km²) | % of land | Structures | Interior rate | Interiors | Struct/km² |
|---|---|---|---|---|---|---|
| Portside & Old Vermillion | 9.4 | 3.7% | 6,100 | 74% | 4,514 | 649 |
| Cathedral Quarter (Financial) | 6.8 | 2.7% | 3,900 | 82% | 3,198 | 574 |
| Riverton | 14.2 | 5.6% | 11,400 | 66% | 7,524 | 803 |
| Marlow Heights | 8.6 | 3.4% | 4,300 | 33% | 1,419 | 500 |
| Ashford & Eastern Belt | 23.0 | 9.1% | 12,900 | 41% | 5,289 | 561 |
| Halstead Bay Port | 11.2 | 4.4% | 2,400 | 26% | 624 | 214 |
| Halstead Bay Town | 7.4 | 2.9% | 5,200 | 59% | 3,068 | 703 |
| Cordova Flats Petrochemical | 9.8 | 3.9% | 1,800 | 18% | 324 | 184 |
| Cordova Flats Logistics | 5.6 | 2.2% | 2,600 | 31% | 806 | 464 |
| Suburban Belt | 28.0 | 11.1% | 17,600 | 32% | 5,632 | 629 |
| Meridian Valley | 41.0 | 16.3% | 7,900 | 41% | 3,239 | 193 |
| Kestrel National Forest | 33.0 | 13.1% | 2,700 | 24% | 648 | 82 |
| Sable Range | 26.0 | 10.3% | 1,150 | 21% | 242 | 44 |
| Anza Basin | 21.0 | 8.3% | 2,300 | 27% | 621 | 110 |
| Coastal Strip | 7.0 | 2.8% | 1,150 | 25% | 288 | 164 |
| **TOTAL** | **252.0** | **100%** | **83,400** | **44.9%** | **37,436** | **331** |

**Water area budget (78.0 km²):**

| Body | Area (km²) | Max depth | Player access |
|---|---|---|---|
| Vermillion Bay | 34.0 | 46 m | surface craft, diveable to 30 m |
| Halstead Harbour | 9.0 | 22 m | surface craft, diveable (turbid) |
| Meridian River + 6 tributaries | 6.0 | 11 m | small craft, swim, dive shallow |
| Sable Lake | 8.0 | 62 m | small craft, dive to 40 m (cold, low viz) |
| Nearshore reef shelf (to −68 m) | 21.0 | 68 m | dive only |

**Scale comparison:** GTA V's playable land area is ≈ 78 km² (the figure varies between 75.8 and
81 km² across sources because of differing coastline and water-inclusion conventions; we use the
land-only measurement for an apples-to-apples ratio). **252.0 / 78 = 3.23×**. Against RDR2 (≈ 75
km²) the ratio is 3.36×. If water is included on both sides (GTA V total world ≈ 137 km² including
the Pacific), MERIDIAN's total interactive footprint is 330.0 km², a ratio of 2.41× — this second
number is the one to use in any public comparison that includes water, and Legal must approve the
framing (§6.6.5).

### 2.2.1 Why 252 km² and not 400 km²

Scale beyond ~260 km² of hand-authored-equivalent land stops being a differentiator and starts
being a liability, for three quantified reasons:

1. **Traversal time.** At the region's 90 km/h average road speed, 252 km² (≈ 16 km mean diameter)
   is a 12-minute crossing. At 400 km² (≈ 20 km) it is 15 minutes. Players abandon worlds where
   traversal exceeds ~20 minutes without a fast-travel alternative, and fast travel erodes the
   sense of scale we bought.
2. **Content density.** Structure count per km² (§2.2) is what players read as "detail". Doubling
   area at constant density doubles cost linearly; doubling density at constant area costs
   superlinearly but reads as *more* world. We chose density.
3. **Streaming working set.** Resident memory scales with the *traversal-rate-weighted* streaming
   volume, not with total area (§5.8). A larger map with the same speed costs the same memory — but
   a larger map with the same *density* costs more in HLOD and terrain-clipmap tiers to keep the
   15 km horizon credible.

## 2.3 Biome classification and map layout

### 2.3.1 Schematic layout

```
                        NORTH  (+Y)
   ┌──────────────────────────────────────────────────────────────────────┐
   │  ▲▲▲ SABLE RANGE (26 km²)  ▲ Sable Peak 2,410 m  ▲ ski resort        │
   │ ▲▲▲  alpine/glacier  ▲▲▲  ~~~ Sable Pass 1,612 m (seasonal closure)  │
   │  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲│  ANZA BASIN (21 km²)  high desert     │
   │  KESTREL NAT. FOREST (33)   │  1,050 m plateau, salt flats, solar    │
   │  old-growth rainforest      │  ghost town, badlands escarpment →     │
   │  ▼ treeline 1,780 m         │  (region east boundary)                │
   │  ~~~~~~~~~~~~~~~~~~~~~~~~~~~│                                        │
   │  SUBURBAN BELT (28)  Bellweather · Sable Foothills · New Cordova     │
   │  Junction Springs ⌂  ~~~~~~~~│                                        │
   │                              │  MERIDIAN VALLEY (41)                  │
   │  ╔═ VERMILLION CITY (62) ═╗  │  orchards · vineyards · row crops      │
   │  ║ Marlow Heights (8.6)   ║  │  dairies · packing houses              │
   │  ║ Riverton (14.2)        ║≈≈≈ Meridian River (142 km, 34 m³/s)       │
   │  ║ Cathedral Qtr (6.8)    ║  │  Sable Lake 8 km² / 62 m               │
   │  ║ Portside (9.4)         ║  │                                        │
   │  ║ Ashford & Eastern (23) ║  │                                        │
   │  ╚════════⌂ VMI Airport ══╝  │                                        │
   │ ░░ VERMILLION BAY (34 km² water) ░░  Vermillion Bay Bridge 2,860 m    │
   │  HALSTEAD BAY (34 land+9 water)                                        │
   │  ⚓ port/containers · shipyard · town (7.4)                            │
   │  CORDOVA FLATS (15.4) ⚗ petrochemical · logistics · tidal flats      │
   │                                                                        │
   │ COASTAL STRIP (7)  Cape Moraine · dunes · Kestrel Reefs (offshore ↓)   │
   └──────────────────────────────────────────────────────────────────────┘
                        SOUTH (-Y)          WEST (-X) ← open ocean → EAST (+X)
```

*(Schematic only — not to scale in aspect. Authoritative layout lives in the World Partition
project's `RegionLayout.strata` and the GIS-derived `region.gpkg` geodatabase.)*

### 2.3.2 Biome classes

Classification uses a **Whittaker-style bioclimate envelope** (mean annual temperature × mean
annual precipitation) evaluated *per 64 m cell* from the weather model's 90-year climatology run
(§3.2.7), not authored by hand. This means the biome map is a **derived artefact** — if a designer
changes terrain and alters orographic lift, the biome classification and therefore the species
composition updates automatically.

| Biome class | Cells | Area (km²) | Mean T (°C) | MAP (mm/yr) | Dominant substrate | Canopy |
|---|---|---|---|---|---|---|
| **Temperate maritime rainforest** | 8,100 | 33.2 | 9.8 | 2,140 | andosol over basalt | 38–62 m, 0.82 closure |
| **Coastal scrub & dune** | 1,710 | 7.0 | 12.1 | 780 | entisol, aeolian sand | < 2 m, 0.15 |
| **Mediterranean oak woodland** | 3,980 | 16.3 | 13.4 | 620 | alfisol, gravelly loam | 8–16 m, 0.41 |
| **Montane conifer** | 4,620 | 18.9 | 7.2 | 1,310 | inceptisol, colluvium | 22–40 m, 0.74 |
| **Subalpine** | 2,050 | 8.4 | 3.9 | 1,180 | spodosol, scree | 3–9 m krummholz, 0.28 |
| **Alpine tundra / nival** | 2,930 | 12.0 | −0.8 | 1,440 (62% as snow) | gelisol, rock | none |
| **Cold semi-arid shrubsteppe** (Anza) | 5,130 | 21.0 | 11.6 | 265 | aridisol, saline | < 1.5 m, 0.09 |
| **Alluvial agricultural** (Valley) | 10,010 | 41.0 | 13.9 | 540 (68% irrigated) | vertisol/mollisol | crop-dependent |
| **Riparian corridor** | 1,540 | 6.3 | 12.4 | — | fluvial sediment | 18–30 m gallery |
| **Urban impervious** | 30,400 | 124.6 | +2.4 UHI offset | — | asphalt/concrete/roof | street trees only |
| **Estuarine / tidal flat** | 990 | 4.1 | 12.0 | — | saline mud | salt marsh |
| **Lentic / lotic water** | 19,060 | 78.0 | — | — | — | — |
| **Subaqueous reef & kelp** | (within water) | 21.0 | 6–14 | — | carbonate/basalt | kelp 12–24 m |

**Total accounted:** 252.0 km² land + 78.0 km² water. (Cell count × 0.004096 km² per 64 m cell.)

### 2.3.3 Biome transition grammar

Every biome boundary `MUST` be authored to one of six **transition types**. Each type has a
mandated width, an interpolation rule, and a set of validation checks. This is the mechanism that
prevents the "paint-by-numbers biome seam" that makes large maps feel like a quilt.

| Type | Example | Width | Interpolation rule | Validation |
|---|---|---|---|---|
| **T1 — Ecotone** | Montane conifer → subalpine at the treeline (1,780 m) | 180–600 m | Species frequency curves cross over a slope-and-exposure-modulated band; canopy closure decays as `1/(1+e^((elev-treeline)/62))`; krummholz form factor rises | No cell may contain two biome classes at > 60% frequency each for more than 40 m of contiguous run |
| **T2 — Edaphic** (substrate-driven) | Alfisol woodland → aridisol shrubsteppe across the Sable Pass | 250–1,200 m | Driven by the soil layer, not elevation; species swap keyed to `soil_water_holding_capacity` | Soil layer must be continuous; no authored overrides |
| **T3 — Hydrological** | Valley floor → alluvial fan → montane colluvium | 400–2,000 m | Substrate grain size follows a depositional model (fan apex angle 4–9°, sorting coefficient decreasing downslope) | Fan geometry generated from the DEM's flow accumulation, not hand-placed |
| **T4 — Anthropogenic hard edge** | Cordova Flats petrochemical → estuarine marsh | 0–30 m | A physical barrier (fence, berm, riprap, wall) is **required**; land value discontinuity is legal and expected | Barrier must be a real collider with a real material; no invisible edges |
| **T5 — Anthropogenic soft edge** | Riverton → Suburban Belt | 400–1,500 m | Parcel size, zoning, setback, building age, and street-tree species all ramp; **no single road may be the boundary** | Land-value gradient must be monotone across the band with ≤ 2 reversals |
| **T6 — Littoral** | Dune → beach → surf → reef crest → slope → drop-off | continuous | Substrate, water turbidity (Jerlov type), wave energy, and species all key off depth and exposure | Jerlov type transitions must match measured visibility targets (§2.9) |

**T5 is the one that matters most for player perception.** The failure mode in open-world games is a
single arterial road where "the city ends and the suburbs begin". Real urban gradients are
multivariate and slow: lot width, fence height, building age, roof form, street tree species,
driveway surface, presence of on-street parking, storefront vacancy, and litter density all shift
over 1 km. We author T5 bands with a **seven-channel gradient mask** (lot area, building age,
height storeys, impervious fraction, canopy fraction, land value, litter density), each an
independent 4 m-resolution float field. The procedural content generators (parcel subdivision §2.4,
façade grammar §4.3.2, set dressing §4.6) all sample these fields. Because the channels are
independent, you get the real thing: a block of 1920s walk-ups next to a 1978 strip mall next to a
2019 four-storey over a ground-floor laundromat, because the channels ramp at different rates.

## 2.4 Zoning, parcels, and the land-value field

### 2.4.1 Zoning taxonomy

Zoning is not flavour text. It is the **input to every procedural content generator** and to three
simulation systems (traffic demand, crime baseline, economy).

| Code | Name | FAR | Max height | Setback | Trip gen. (ITE-style, daily trips / 1,000 m² GFA) | Land value (rel.) |
|---|---|---|---|---|---|---|
| R-1 | Detached residential | 0.35 | 3 storeys | 6 m front / 2 m side | 9.5 | 0.55 |
| R-2 | Attached / low-rise residential | 0.9 | 4 storeys | 4 m / 1.5 m | 12.0 | 0.72 |
| R-3 | Mid-rise residential | 2.2 | 8 storeys | 3 m / 0 m | 14.5 | 1.10 |
| R-4 | High-rise residential | 6.5 | 34 storeys | 0 m | 16.0 | 1.85 |
| C-1 | Neighbourhood commercial | 0.6 | 2 storeys | 0–3 m | 42.0 | 0.80 |
| C-2 | Corridor commercial | 1.8 | 5 storeys | 0 m | 63.0 | 1.25 |
| C-3 | Central business | 9.0 | 74 storeys | 0 m | 71.0 | 3.40 |
| M-1 | Light industrial | 0.7 | 3 storeys | 8 m | 24.0 | 0.42 |
| M-2 | Heavy industrial | 0.5 | 4 storeys | 12 m | 19.0 | 0.31 |
| M-3 | Petrochemical / port | 0.3 | 6 storeys (process) | 25 m | 11.0 | 0.22 |
| MU-1 | Mixed use, low | 1.4 | 5 storeys | 0 m | 38.0 | 1.05 |
| MU-2 | Mixed use, mid | 3.2 | 12 storeys | 0 m | 51.0 | 1.60 |
| MU-3 | Mixed use, high | 7.8 | 40 storeys | 0 m | 66.0 | 2.75 |
| CI | Civic / institutional | 1.5 | 10 storeys | 6 m | 15.0 | 1.30 (non-market) |
| OS | Open space / park | 0.05 | 1 storey | — | 8.0 (park) | 0.00 |
| AG | Agriculture, irrigated | 0.03 | 1 storey | — | 2.1 | 0.14 |
| AG-2 | Agriculture, dry/pasture | 0.01 | 1 storey | — | 0.9 | 0.08 |
| AG-3 | Agriculture, marginal/range | 0.01 | — | — | 0.4 | 0.03 |
| FR | Forest reserve | 0.00 | — | — | 0.2 | 0.00 |
| RC | Resource conservation | 0.00 | — | — | 0.1 | 0.00 |
| T | Transit / ROW | — | — | — | (network) | 0.00 |
| U | Utility | 0.1 | varies | 15 m | 3.0 | 0.05 |

### 2.4.2 Parcel generation pipeline

Parcels are the atom of all urban content. Pipeline (offline, in the World Partition editor):

1. **Input:** DEM (2 m resampled from a 10 m synthetic base), hydrology graph, road centreline
   graph (2,640 km), zoning layer, the seven-channel T5 gradient masks (§2.3.3).
2. **Block formation:** road graph faces → city blocks. 41,200 blocks region-wide.
3. **Parcel subdivision:** recursive binary space partition per block, with lot-area target sampled
   from the zoning code's distribution (`R-1`: lognormal μ = 620 m², σ = 0.32) and orientation
   constrained to the block's frontage axis. Corner lots get a 1.18× area multiplier and a
   two-street frontage flag (drives duplex/façade treatment).
4. **Setback & envelope:** apply zoning setbacks → buildable envelope polygon → extrude by a
   height sampled from `FAR × lot_area / footprint_area`, clamped by zoning max height and by the
   **viewshed ordinance** layer (Marlow Heights and Cathedral Quarter both have protected view
   corridors; a building that violates one is rejected and re-solved — a real constraint in real
   cities and a source of authentic irregularity).
5. **Occupancy assignment:** each parcel is assigned a use (from zoning-conditional probabilities),
   then a **household or business** from the demographic model (§3.11), then a `ResidentRecord` set.
6. **Output:** `parcels.strata` — 187,400 parcels, 41 bytes each after compression = 7.7 MB.

**Result:** 187,400 parcels, of which 83,400 carry a structure (44.5% — vacant lots, parks,
surface parking, and ROW make up the rest). Vacant lots are **not** filler: they are generated at
the rate the zoning and land-value model implies, which produces the authentic pattern of a
half-empty block in Cordova Flats and zero vacant lots in Cathedral Quarter.

### 2.4.3 Land-value field and its consumers

A single 4 m-resolution scalar field (`land_value ∈ [0, 4.0]`), generated by a **gravity-model
accessibility computation**: value at a cell is a function of accessibility to employment centres
(weighted by job count), to transit nodes (weighted by headway), to amenity (parks, waterfront,
views), minus disamenity (noise contours from road/air/rail traffic, pollution from M-2/M-3 zones,
flood risk, slope). Then iterated to equilibrium over 40 passes so that value feeds back into
density feeds back into trips feeds back into noise.

**Consumers (7):** building age & condition generator · household income assignment · retail
tenant mix · crime baseline rate · police patrol allocation (§3.12.5) · resident clothing/vehicle
asset selection · municipal services quality (street lighting, road surface condition, refuse
collection frequency — all visible and all simulated).

**This is the highest-leverage authored data in the project.** One field, seven systems reading it,
and it is the reason Riverton and Bellweather feel different without a designer hand-placing the
difference.

## 2.5 District detail

### 2.5.1 Vermillion City (62.0 km², pop. 2,140,000, 5 boroughs)

**Portside & Old Vermillion** (9.4 km², pop. 186,000)
1870s–1930s street grid, 5–8 storey masonry, narrow 14 m ROW. Zoning C-3/MU-2/R-3. Contains:
Ferry Terminal (Class S interior), the Customs House (Class S), 41 working piers, the Fish Market
(pre-dawn operation — a real 03:30–09:00 activity cluster in the ledger), 3 historic hotel
interiors (Class A), and the Old Vermillion catacomb/storm-drain network (1.9 km walkable, 0 m
cellular coverage — a deliberate dead zone, §3.12.3). Verticality: −14 m (drains) to +42 m.
Crime baseline: high property, moderate violent. Jurisdiction: VC PD, 2nd Precinct.

**Cathedral Quarter** (6.8 km², pop. 94,000 daytime 410,000)
Financial and civic core. 8–74 storeys. Contains **Vermillion Tower** (312 m roof, 348 m spire,
74 floors, 61 passenger + 4 freight elevators, Class S interior on 9 of its floors, non-enterable
but rendered floors in between), the Cathedral of St. Brendan (Class S), City Hall (Class S, 6
public floors), the County Courthouse (Class S — the venue for §3.12.7 adjudication), Vermillion
Central Station (Class S, deep-bore metro at −38 m, commuter rail at −12 m, concourse at −6 m,
street at 0). **Population ratio 1 : 4.4 day-to-night** — the single largest diurnal population
swing in the region and the hardest test of the ledger (§3.9.6). Verticality: −38 m to +348 m —
a **386 m span in one district**, which is the driving requirement for the G4 mutable-state
cell size of 16 m (§5.8.1) and for the elevator/vertical-streaming system (§4.5.4).

**Riverton** (14.2 km², pop. 402,000)
Densest residential: 4–7 storey walk-ups, 1900–1965, 66% interior rate — **7,524 interiors, the
largest single interior concentration in the region** and the reason the stock-interior pipeline
must work. Zoning R-3/MU-1/C-2. Contains 22 public housing estates (each a bespoke-anchored
procedural complex with a distinct courtyard typology), 4 metro stations, 11 schools, 3 hospitals
(1 Level-I trauma centre, Class S), 180 ground-floor retail units. Crime baseline: highest violent
rate in the region, and — critically — the **police patrol allocation derived from land value
produces measurably slower response times here than in Marlow Heights.** This is not a bug and not
a political statement authored by a writer; it is an emergent property of the model, and Pillar 3
("causation must be legible") requires it be observable. Design review has approved this
deliberately. See §6.6.4 for the communications risk.

**Marlow Heights** (8.6 km², pop. 71,000)
Hillside affluent, 1–3 storeys on 900–2,400 m² lots, winding roads with 8–14% grades, retaining
walls, 33% interior rate (high wall/gate ratio → many structures are gatehouses, garages, and
pool houses rather than enterable homes). Zoning R-1/OS. Contains 2 protected view corridors, the
Marlow Country Club (Class A), and the region's highest land value (3.6). Crime baseline: low
street crime, high-value burglary. Verticality: +88 m to +204 m; the grade means the road network
has 41 switchbacks and 6 dead-ends — a pathfinding stress case (§6.3).

**Ashford & Eastern Belt** (23.0 km², pop. 683,000)
Postwar: R-2 grids, strip commercial on arterials, M-1 light industrial, and **Vermillion
International Airport (VMI)** — 3 runways (3,410 m / 2,740 m / 2,130 m), 2 terminals (Class S),
4 concourses, 68 gates, a 24-hour cargo operation, and the region's second-largest employment
centre (38,000 jobs). The airport is the anchor for §3.11's supply-chain model (air freight) and
for the air-support dispatch tier (§3.12.6). Verticality: −6 m (baggage) to +61 m (control tower).
Airport interiors: 14 Class S/A, 88 Class B/C.

### 2.5.2 Halstead Bay & Cordova Flats (34.0 km², pop. 412,000)

**Halstead Bay Port** (11.2 km²): 9 container berths, 2 Ro-Ro, 4 dry bulk, 1 liquid bulk. 148
gantry cranes and rubber-tyred yard cranes (12 articulated, physics-simulated at Tier 1 when
near). Container yard with 34,000 manifested containers — **each with a real consignee, contents
record, and destination in the economy model (§3.11.3)**. Stealing a container is not stealing a
box; it is removing 14 pallets of a specific SKU from a specific store's inbound supply, and the
store goes short 36 hours later. Dry dock (Class S, 340 m × 58 m, floodable — a full shallow-water
simulation case). Shipyard heritage cranes, a rail yard with 22 km of industrial track.

**Halstead Bay Town** (7.4 km², pop. 168,000): R-2/M-1/C-1, 59% interior rate, 3,068 interiors.
Working-class, union density 41% (drives labour-action events in the economy sim), 14 pubs, a
fishermen's co-op, 3 churches, the Halstead Mercantile tower (188 m, the region's second tallest).

**Cordova Flats Petrochemical** (9.8 km²): refinery (340,000 bbl/day nameplate — process units are
bespoke geometry with **simulated process state**: crude slate, throughput, unit outages, flare
stack activity tied to the economy model and visible from 12 km as a heat/light source). 18%
interior rate because most of it is fenced process area with control rooms and pump houses rather
than enterable buildings. **11.2 km of pipe rack** (walkable on some, climbable on most), 4 flare
stacks (up to 96 m), 62 storage tanks (of which 8 are enterable via roof hatches — a Class D
interior with a confined-space gas hazard modelled). Tidal flats with 2.8 m semi-diurnal range.
Jurisdiction: Port Authority Police (a separate agency with a distinct mandate and a
mutual-aid latency of 140 s to VC PD — §3.12.5).

**Cordova Flats Logistics** (5.6 km²): 2.6 km² of warehousing, 412 units, cross-dock operations
with real truck arrival distributions (Poisson, λ peaking at 04:00 and 15:00), 31% interior rate.

### 2.5.3 Suburban Belt (28.0 km², pop. 508,000, 4 municipalities)

Bellweather (master-planned 1994, 8.2 km², curvilinear streets with 3 hierarchical levels, HOA
covenants that generate visible uniformity), Sable Foothills (exurban 2,000 m²+ lots, 7.6 km²,
wildfire interface zone — the region's highest fire risk, §3.3.5), New Cordova (company town built
1957 for the refinery, 5.4 km², 61% of housing owned by one entity in the economy model), Junction
Springs (rail town, 6.8 km², freight yard, commuter rail terminus).

**17,600 structures at 32% interior = 5,632 interiors.** The low interior rate is authentic: single-
family homes in games are almost never enterable because each costs a bespoke interior. Ours are
procedural and record-seeded, so 32% is a *budget* decision (§4.4), not a technical limit. It is set
by the resident interior memory budget (640 MB, §4.4.3) and the density of simultaneous occupancy
the streaming scheduler can keep warm in a suburban traversal.

### 2.5.4 Meridian Valley (41.0 km², pop. 121,000)

The agricultural heart. 12.4 km² orchards (apple, pear, cherry, walnut — each with a distinct
canopy form, phenology, and machine-harvest pattern), 6.8 km² vineyards (5 estates, each Class A
interior for the winery), 9.2 km² row crops (tomato, alfalfa, corn, rice in the flooded bottoms —
rice paddies are a shallow-water simulation case), 7.6 km² dairy and pasture (2,400 head of cattle
simulated at Tier 3 with herd behaviour and a real grazing rotation that visibly changes pasture
height across the season), 5.0 km² of valley towns and processing (4 packing houses, 2 grain
elevators, 1 sugar beet processor, 3 farm-supply towns).

Irrigation: 148 km of canal and 312 km of drip/sprinkler lateral, driven by Sable Lake allocation.
**Water rights are simulated**: in a drought year (the weather model produces them at ~1-in-9
frequency over the 90-year climatology) the junior rights holders in the lower valley get curtailed,
crops fail, land value drops, and the packing houses lay off workers — which propagates into the
ledger's employment matrix and therefore into resident schedules, transit ridership, and crime
baseline. This is a **multi-year emergent narrative** with no authored content.

### 2.5.5 Kestrel National Forest (33.0 km²)

Old-growth temperate rainforest (14.2 km², canopy 38–62 m, 2,140 mm MAP), managed second-growth
(10.4 km², harvest units on a 34-year rotation with visible cut blocks and replant cohorts),
riparian corridors (4.2 km²), alpine meadow transition (4.2 km²).

2,700 structures at 24% = 648 interiors: ranger stations, fire lookouts (5, each Class B with a
360° view — a landmark navigation aid), logging camps, 2 abandoned mine adits (Class D, 340 m and
610 m walkable, no coverage, gas hazard), 19 backcountry cabins, a research station (Class A),
and the **Kestrel Fish Hatchery** (Class A, tied to the salmon run model).

**Verticality:** 240 m to 1,780 m (treeline). The canopy is a **navigable layer**: 12 km of
interconnected branch traversals in the old-growth stands, requiring a second navmesh at canopy
height (§6.3.4) and a distinct wind-exposure profile in the weather field (canopy-top wind is
2.4× understorey wind; this matters for sound propagation and for the player's climbing stability).

### 2.5.6 Sable Range (26.0 km²)

Subalpine (8.4 km²), alpine rock and glacier (9.8 km²), permanent snowfield and glacier (2.6 km²,
the Vermillion Glacier, retreating 11 m/yr in the sim's 90-year climatology), ski resort and legacy
mining (5.2 km²).

**Sable Peak, 2,410 m** — the region's high point and the source of the Meridian River's headwaters.
**Sable Pass, 1,612 m**, with a 2.6 km Summit Tunnel; the pass road **closes seasonally** from
~mid-November to ~late-April based on accumulated snowpack and avalanche hazard, computed from the
weather model — a hard, physical gate between the coast and Anza Basin that forces the player to
use the tunnel or the coastal highway. **Avalanche hazard** is simulated (Rutschblock-class
stability from snowpack layering, wind slab deposition, temperature gradient metamorphism) and
produces real events that bury road sections and require the county to plow — which the ledger
tracks as a work order.

**Sable Ridge Ski Area**: 41 pistes, 9 lifts (3 detachable quads, 2 fixed triples, 1 gondola
(8-person, 2.4 km), 2 surface), 480 m vertical, 62 ha skiable. Lifts are **fully simulated**:
loading/unloading cycles, haul rope speed, drive and emergency braking, and the ledger puts 1,200–
3,400 skiers on the mountain on a weekend — each with a skill level, a lift-ticket state, and a
chosen piste. This is a *tour de force* of the Resident Continuum (Tier 2 agents on moving carriers)
and one of the most expensive single features in the project (28 person-months). It is retained
because it is the most visible demonstration that the simulation is not a city-only trick.

1,150 structures at 21% = 242 interiors: ski lodge (Class A), base lodge, 4 lift shacks, 3 mine
buildings, the summit observatory (Class A, at 2,340 m — the region's highest interior), 2 rescue
huts, patrol rooms.

### 2.5.7 Anza Basin (21.0 km²)

Cold semi-arid shrubsteppe on a 1,050 m plateau east of Sable Pass — the **rain shadow** is real and
computed: 265 mm MAP against 2,140 mm on the windward forest, a 8.1× ratio produced by the
orographic model (§3.2.4), not authored.

5.4 km² salt flats (the Anza Dry Lake, 41 km² of which 5.4 km² is the hard pan — a flat, high-μ
surface used as a speed-test and landing area), 8.2 km² shrubsteppe (sagebrush, saltbush, juniper),
2.6 km² solar and infrastructure (a 340 MW PV plant with 612,000 tracked panels — each an instanced
mesh with a tracker angle driven by the sun position; the plant's output feeds the grid model),
4.8 km² badlands and the ghost town of **Anza City** (pop. 1881: 2,400; pop. 2031: 41 — all 41 in
the ledger, all with routines).

2,300 structures at 27% = 621 interiors. The ghost town's 88 standing structures are 71% enterable
(Class B/C) — a hand-authored procedural override because it is a landmark location.

**Cellular coverage:** 3 sites for the entire 21 km². **62% of the basin is below −110 dBm** and
28% is below −124 dBm (no service). This is the region's primary dead zone and the design intent
behind placing a protagonist here (§1.6): the deputy operates in a jurisdiction where a crime
cannot be reported by phone and radio repeaters have line-of-sight limits.

### 2.5.8 Coastal Strip (7.0 km²)

2.4 km² dune field (foredune 6–14 m, parabolic dunes up to 22 m, actively migrating 1.8 m/yr —
simulated as a slow terrain mutation with a corresponding mutation entry in the delta journal),
2.2 km² cliffs and **Cape Moraine** (a 148 m headland with a 1908 lighthouse, Class B interior, and
a seabird colony of ~24,000 individuals simulated at Tier 3 as a density field with embodied
individuals near the player), 1.6 km² beaches (7 named, each with a measured profile and a seasonal
berm cycle), 0.8 km² tidal flats.

1,150 structures at 25% = 288 interiors: beach houses, the lighthouse, 3 surf shops, a coast guard
station (Class A), 2 lifeguard towers per beach (Class D), the Cape Moraine visitor centre.

## 2.6 Verticality

Verticality is specified as three independent axes, because they have different streaming,
rendering, and simulation costs.

### 2.6.1 Terrain relief

| Feature | Elevation |
|---|---|
| Sable Peak (summit) | +2,410 m |
| Sable Ridge Ski Area summit | +2,340 m |
| Sable Pass (road high point) | +1,612 m |
| Summit Tunnel crown | +1,548 m |
| Anza Basin plateau | +1,050 m |
| Treeline (Kestrel/Sable) | +1,780 m |
| Marlow Heights crest | +204 m |
| Meridian Valley floor | +38 to +96 m |
| Vermillion City datum | +4 m |
| Cordova Flats (MHWS) | +2.8 m |
| Lowest land (tidal flat) | +0.4 m |
| Deepest diveable point (Kestrel drop-off) | −68 m |
| Deepest bathymetry in world bounds (non-interactive) | −120 m |

**Total terrain relief: 2,478 m.** Max single-view relief visible from the Sable Peak summit:
2,478 m over a 41 km line of sight (the Anza escarpment) — this is the scene that validates the
15 km horizon requirement and it is the reason the far-clipmap tier exists (§5.6.3).

### 2.6.2 Built verticality

| Structure | Below grade | Above grade | Total span | Notes |
|---|---|---|---|---|
| Vermillion Tower | −24 m (5 levels) | +348 m (spire) | 372 m | 74 floors; 9 enterable |
| Cathedral Quarter deep-bore metro | −38 m | — | 38 m | Deepest public space |
| Halstead Mercantile | −18 m | +188 m | 206 m | 44 floors |
| VMI Control Tower | −4 m | +61 m | 65 m | Class A interior at +54 m |
| Sable Ridge Gondola upper terminal | — | +2,340 m | — | Highest interior |
| Cordova Flats flare stack #3 | +2 m base | +96 m | 98 m | Climbable |
| Vermillion Bay Bridge pylon | −46 m (footing) | +212 m | 258 m | Climbable to +186 m |
| Cathedral Bypass road tunnel | −18 m | — | 18 m | 3.4 km, no coverage |
| Old Vermillion storm drains | −14 m | — | 14 m | 1.9 km walkable |

**Elevators:** 4,180 across the region, of which 1,880 are player-usable. Each is a **vertical
streaming source** (§4.5.4) — the car is a simulation source that moves vertically, so the
streaming scheduler must treat elevator shafts as high-priority pre-stream corridors. A 74-floor
ride at 6 m/s is 12 s of continuous vertical streaming, which is the single hardest streaming case
in the game. It is handled by pre-streaming the destination floor on call, not on travel (§5.8.5).

### 2.6.3 Subsurface

| Layer | Depth | Content | Coverage |
|---|---|---|---|
| Basements | −3 to −9 m | 68% of Cathedral Quarter and Portside buildings; parking, plant, retail | 12,400 units |
| Cut-and-cover metro | −12 m | Green & Yellow lines | 22 km |
| Deep-bore metro | −22 to −38 m | Red, Blue, Purple lines | 96 km |
| Road tunnels | −18 m | Cathedral Bypass (3.4 km), Halstead Harbour (1.9 km), Sable Summit (2.6 km), Cordova Utility (1.1 km) | 9.0 km |
| Storm drainage | −2 to −14 m | Downtown 100-yr design; Cordova Flats 10-yr design (floods) | 480 km |
| Utility tunnels | −4 to −8 m | Steam, chilled water, power, fibre — walkable in 34 km | 34 km |
| Mine adits | −0 to −210 m | 2 in Kestrel, 4 in Sable Range, 1 in Anza | 7 (2 walkable) |
| Archaeological / catacomb | −6 to −19 m | Old Vermillion, 1.9 km | 1 network |

**Total walkable subsurface: 681 km.** All of it is **zero cellular coverage** and most is **zero
radio coverage**, which makes the subsurface the region's dominant tactical space for §3.12
(escape, evasion, dead-zone crime) — a systemic consequence of the coverage model rather than a
level-design decision.

## 2.7 Transit network

| Mode | Extent | Vehicles | Headway (peak / base) | Notes |
|---|---|---|---|---|
| **Metro — Red** (Cathedral ↔ Ashford ↔ VMI) | 34 km, 26 stn | 96 cars | 2.5 / 6 min | Deep-bore, −28 m |
| **Metro — Blue** (Portside ↔ Halstead) | 28 km, 22 stn | 78 cars | 3 / 7 min | Harbour crossing, −22 m |
| **Metro — Green** (Riverton loop) | 19 km, 18 stn | 54 cars | 3 / 8 min | Cut-and-cover, −12 m |
| **Metro — Orange** (VMI ↔ Marlow ↔ Cathedral) | 24 km, 17 stn | 62 cars | 4 / 9 min | Airport people-mover branch |
| **Metro — Purple** (cross-town deep bore) | 31 km, 21 stn | 84 cars | 3 / 7 min | Deepest, −38 m at Central |
| **Metro — Yellow** (surface light metro to Bellweather) | 12 km, 8 stn | 28 cars | 5 / 12 min | At-grade, 14 level crossings |
| **Metro total** | **148 km, 112 stn** | **402 cars** | | 1.42M weekday boardings |
| **Bus** | 42 routes, 1,180 stops | 612 vehicles | 4–40 min | Diesel-electric; depot at Ashford |
| **Commuter rail** | 2 lines, 96 km, 19 stn | 44 cars | 15 / 30 min | VC Central ↔ Junction Springs ↔ Valley; ↔ Anza |
| **Ferry** | 3 routes | 9 vessels | 20 / 35 min | Pier 9 ↔ Halstead ↔ Cordova |
| **Cable car / funicular** | 2 | 6 cars | 8 min | Marlow Heights incline, Portside heritage line |
| **Airport (VMI)** | 3 runways, 68 gates | — | 42 movements/hr peak | Domestic + 8 international |
| **Freight rail** | 212 km | — | — | Port, refinery, valley agriculture |

**Player interaction with transit is full-fidelity:** buying a fare product (the ledger tracks
balance), boarding at any of 112 stations, riding in a simulated car with simulated occupants drawn
from the ledger (a Red-line car at 07:40 has 214 riders, each a Tier 2 agent with an actual origin
and destination from their schedule), and alighting. Trains run on a real timetable with real
delay propagation: a signal failure at Riverton Junction delays the following 6 trains by 2–9
minutes, and those delays appear in every affected resident's schedule, causing late arrivals at
work, which causes the ledger's employer model to log absences. **The transit network is not a fast-
travel menu; it is a system that the population depends on and that can fail.**

Fast travel exists separately (a map-based "commit to a journey" action) but it is **implemented as
a ledger time-advance plus a simulated journey** (§1.6): the player loses the journey's real
duration, encounters are possible, and every other resident's schedule advances correctly. Fast
travel that does not advance the world is a Pillar-1 violation and is prohibited.

## 2.8 Road network

| Class | Length (km) | Lanes | Speed limit | Geometry source |
|---|---|---|---|---|
| Freeway / motorway | 168 | 6–10 | 105 km/h | Hand-authored splines, 3 corridors |
| Expressway / limited access | 96 | 4–6 | 85 km/h | Hand-authored |
| Major arterial | 640 | 4–6 | 60 km/h | Semi-procedural from block graph |
| Minor arterial / collector | 810 | 2–4 | 45–50 km/h | Procedural |
| Local street | 1,022 | 2 | 30–40 km/h | Procedural (parcel frontage) |
| Rural highway | 218 | 2 | 90 km/h | Semi-procedural from terrain |
| Unpaved forest/mountain track | 380 | 1–2 | 30–60 km/h | Procedural from cost field |
| Service road / farm track | 142 | 1 | 30 km/h | Procedural |
| Industrial / port yard road | 104 | 2 | 25 km/h | Hand-authored |
| **Drivable total** | **2,640** | | | |
| Sidewalk / pedestrian path | 4,120 | — | — | Procedural from parcel frontage |
| Cycle path / greenway | 388 | — | — | Semi-procedural |
| Trail (hiking/equestrian) | 612 | — | — | Procedural from cost field + authored POIs |

**Key structures:** 41 bridges (longest: **Vermillion Bay Bridge**, 2,860 m suspension, 6 lanes +
pedestrian, +212 m pylon, −46 m footing, climbable), 4 tunnels (9.0 km), 38 grade separations, 212
signalised intersections (all with a **simulated signal controller**: fixed-time plans by time-of-
day, actuated detection on minor approaches, and a central traffic-management system that can
retiming plans for incidents — which the dispatch system uses to *green-wave* responding units
(§3.12.6), a genuinely emergent mechanic), 6 roundabouts, 1,480 unsignalised junctions.

**Traffic assignment:** the demand model is a **trip-generation → distribution → mode-split →
assignment** four-step process (§3.8.3) computed offline per time-of-day slice (16 slices/day),
then dynamically re-assigned near the player. Region-wide: 11.4M person-trips/day, 7.9M vehicle-
km/day, peak-hour network mean speed 34 km/h, off-peak 58 km/h.

## 2.9 Water and the underwater world

### 2.9.1 Surface water rendering and simulation

| Property | Specification |
|---|---|
| Wave model | **FFT (Tessendorf) displacement**, 256×256 grid, 4 cascaded patches (2 m / 16 m / 128 m / 1,024 m tile size) + **Gerstner** detail for near-field (< 20 m from camera) |
| Spectrum | Pierson-Moskowitz / JONSWAP with fetch-limited growth; **wind-driven from the weather field**, not a global "sea state" slider |
| Sea state range | Beaufort 0 (Cordova Flats slack tide) to Beaufort 8 (open ocean storm) |
| Refraction | Screen-space + depth-based; **spectral absorption by Jerlov water type** per region |
| Foam | Jacobian-based whitecap + shoreline interaction + vessel wake |
| Caustics | Projected from the displacement normal, 3-band, only in shallow (< 12 m) clear water |
| Vessel dynamics | 6-DoF rigid body with **added mass**, **hydrostatic restoring from the sampled heightfield**, drag, and wave impact impulses sampled at 40 points along the hull |
| Buoyancy objects | Any rigid body with a `MaterialRecord` density < 1,025 kg/m³ floats; debris from destruction does too |

**Jerlov water types by region** (drives absorption coefficients and visibility):

| Region | Type | Visibility (clear) | Notes |
|---|---|---|---|
| Open ocean beyond 5 km | IB | 32 m | Deep blue attenuation |
| Kestrel Reefs | II | 18–26 m | Seasonal: worse after storms (resuspended sediment) |
| Vermillion Bay interior | 5 | 4–9 m | Turbid, plume from Meridian River |
| Halstead Harbour | 7 | 2–5 m | Industrial turbidity, hydrocarbon film on surface |
| Sable Lake | IA (cold, low productivity) | 14–22 m | Thermocline at 18 m; below, 4 °C |
| Meridian River | 6 | 1–3 m | Glacial flour in spring melt |
| Cordova tidal flats | 9 | 0.3–1 m | Effectively opaque |

**Visibility is a simulation output, not an art setting.** It is computed from the weather model's
recent precipitation (runoff → sediment load), wind (mixing depth → resuspension), season
(phytoplankton bloom), and river discharge. A dive at the Kestrel Reefs in March after a 60 mm rain
event has 6 m visibility; the same dive in August has 24 m. Diving is therefore a **weather-
dependent activity**, which is a real and rarely simulated constraint.

### 2.9.2 Underwater zones (Kestrel Reefs, 21.0 km²)

| Zone | Depth | Terrain | Features | Fauna density (Tier 3 aggregate) |
|---|---|---|---|---|
| Intertidal / surge | 0 to −2 m | Rock platform, boulder | Tidepools (a real tidal model, §2.10), surge currents up to 1.4 m/s | High (invertebrates) |
| Kelp forest | −2 to −18 m | Sand/rock, 12–24 m kelp | Kelp is a **simulated flexible structure** (verlet chains, 800k instances), reduces visibility, attenuates sound, provides cover | Medium |
| Reef crest | −8 to −22 m | Carbonate/basalt framework | 41 named dive sites, 3 wrecks | Highest |
| Reef slope | −22 to −45 m | Rubble, then wall | Wall dive, overhangs, 2 swim-throughs | Medium-high |
| Drop-off | −45 to −68 m | Vertical wall to abyssal | **Depth limit**; nitrogen narcosis model above 40 m, decompression obligation modelled | Low |
| Abyssal shelf | < −68 m | — | **Non-interactive**, rendered only | — |

**Wrecks (3):** a 1944 Liberty-type cargo ship at −34 m (intact, 6 penetra­ble compartments, Class
D interiors), a 1978 trawler at −18 m (broken in two), and a **light aircraft at −11 m** (mission-
relevant). Each is a bespoke interior with an acoustic bake (underwater reverb is a distinct
propagation model: 1,480 m/s sound speed, no air-interface reflection, high-frequency absorption
~0.06 dB/m at 10 kHz — §3.7.7).

**Diving simulation:** the player has a real physiological model — gas supply (twin 12 L at 200 bar
= 4,800 L free gas), consumption scaled by exertion and depth (ρ × V̇), nitrogen uptake per a
three-compartment Haldane model with half-times of 5/20/75 min, no-decompression limits, mandatory
decompression stops on supersaturation, nitrogen narcosis above 40 m (a progressive input-latency
and visual-distortion effect), oxygen toxicity above 56 m equivalent, and thermal load from water
temperature vs wetsuit clo value. **This is the purest expression of Pillar 6** and it costs 6
person-months.

### 2.9.3 Tides

Semi-diurnal, from a harmonic approximation with 6 constituents (M2, S2, N2, K2, K1, O1) fitted to a
fictional but self-consistent reference station. Range: 2.8 m mean, 4.1 m spring, 1.2 m neap. Driven
by moon phase and solar declination from the astronomical clock (§2.10). Tide gates the Cordova
tidal flats (walkable at low water, 2 m deep at high), the Halstead dry dock, the Fish Market ramp,
and 14 beach-access paths. Tide level is written to the delta journal as a **derived field**, not a
mutable one — it is recomputed from world time and never needs persisting.

## 2.10 The simulated calendar and astronomical clock

MERIDIAN runs a **real astronomical model**, because six systems depend on it and any inconsistency
between them is immediately visible.

| Element | Model | Consumers |
|---|---|---|
| Solar position | Meeus low-precision ephemeris (accuracy 0.01°), latitude **41.6° N**, longitude **−124.2°**, UTC−8/−7 | Lighting, shadows, PV plant output, solar gain on buildings, plant phenology, diurnal activity rhythms, photography/exposure |
| Day length | 9 h 04 m (winter solstice) → 15 h 22 m (summer solstice) | Activity scheduling, street-lighting controllers, retail hours, crime opportunity |
| Lunar position & phase | Same ephemeris, phase from synodic month 29.53059 d | Night illumination (a real lighting source: full moon ≈ −12.7 mag, quarter ≈ −10.0), tide constituents, wildlife activity, agricultural planting (a real practice some residents follow) |
| Twilight | Civil/nautical/astronomical by solar depression angle (−6/−12/−18°) | The lighting system's sky model, AI vision thresholds, "dusk" activity transitions |
| Stellar field | Hipparcos-derived catalogue, ~9,200 stars to magnitude 6.5, with proper extinction from atmospheric model | Night sky rendering, navigation flavour, no gameplay dependency |
| Seasonal cycle | 4 meteorological seasons + 24 phenological transition points | Flora (leaf-out, flowering, senescence, leaf-fall as a **physical debris layer** that affects friction and fire), fauna (migration, breeding, hibernation), agriculture (planting/harvest), tourism, energy demand, road maintenance |
| Holidays | 14 fixed + 3 floating (incl. religious calendars across 6 traditions represented in the demographic model) | Retail closure, transit headway change, crowd events, police staffing (holiday pay → reduced overtime → slower response) |
| Elections | Municipal every 4 y, state every 4 y (offset 2 y) | Policy variables: police funding, transit frequency, zoning changes, fire budget — all of which feed the simulation |
| Time-of-day slices | 96 per day (15-min) for demand models | Traffic assignment, transit headway, retail hours, crowd density |

**Time scale:** 1 real second = 1 simulated second by default. The "rest" action advances time at
300× (ledger and weather tick at full rate, embodied agents are not instantiated). No other time
dilation is permitted except the physics catch-up policy (§5.4.2), which is capped at 3 substeps
and then dilates the *render* timeline, never the sim.

**Consequence:** a full simulated year is 31.5M seconds. The 90-year climatology run (§3.2.7) is
2.84 billion sim-seconds, executed offline at 1,200× real time on the build farm over 11 days —
this is a **build-farm capacity item** and it must be planned (§6.8, R-09). It is not re-run; the
output is a 340 MB baked climatology asset.

## 2.11 World boundary

The boundary is **diegetic and physical**, never an invisible wall. `MUST NOT` use a collision
plane that stops the player without a visible cause.

| Edge | Distance from region centre | Mechanism |
|---|---|---|
| **West — open ocean** | 40 km offshore | Sea state rises (Beaufort 6+ from the weather model's offshore fetch), fuel range becomes insufficient for return, and a maritime VHF warning plays. At 60 km the vessel's engine enters a fuel-starvation state. Aircraft: ATC refuses oceanic clearance without an IFR plan; at 120 km the aircraft's nav display shows "OUTSIDE CHARTED AREA" and the autopilot refuses. **Hard limit: 180 km offshore**, where the ocean impostor ends. |
| **East — Anza escarpment** | +11,200 m | A 1,900 m near-vertical fault escarpment with no route. Climbable in principle up to 1,200 m of it; above that, rock quality (a `MaterialRecord` with a low fracture toughness and an authored "unstable" flag) makes any anchor fail. There is no wall — there is a cliff you cannot climb. |
| **North — Sable Range headwall** | +9,300 m | Glaciated, 55–70° ice and mixed terrain above 2,200 m, with an **objective hazard model** (serac collapse, avalanche) that kills an unroped player at rates consistent with real alpinism. Two technical routes exist and are completable with the right equipment — a genuine 4-hour alpine objective for players who want it. Beyond the divide, non-interactive terrain impostor to 60 km. |
| **South — Cordova exclusion + salt pan** | −9,300 m | A 9 km fenced petrochemical exclusion zone (M-3, real fence colliders, real gates, real patrols, real signage, real trespass consequences under §3.12), then an 18 km salt pan with no road, no water, and a heat-load model that makes crossing on foot lethal above 38 °C without supplies. |

**Airspace:** a hard 3D boundary volume. Inside it: full flight simulation. Outside: the aircraft
continues to fly but the world stops streaming detail (Tier 3 and Tier 4 only), and at 180 km the
simulated flight ends. There is **no wall in the air**. The region reads as part of a continent
because the far-clipmap and impostor skyline extend 60 km in every direction (§5.6.3).

**Off-map consistency:** the ledger contains **9.4 million records** for the wider fictional state,
of which 4.127M are in-region. Off-region residents have records with `region = external` and are
never promoted above Tier 4. Their purpose is to make in-region residents' relationships, supply
chains, and news references point at places that exist — a cousin in Kettleford, a shipment from
Puerto Sarno. Cost: 340 MB of ledger storage, 0 runtime memory (never resident).
