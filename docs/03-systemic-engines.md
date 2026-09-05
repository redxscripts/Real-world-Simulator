# 03 — Systemic World Engines

This section specifies the simulation. It is organised so that each subsection names: **inputs**,
**model**, **outputs**, **tick rate**, **budget**, **failure modes**, and **prior art**. Where the
brief's literal requirement is not achievable in real time, the subsection carries an
**⚠ HONEST LIMITATION** block stating exactly what is approximated and how the approximation is made
undetectable.

Dependency order matters. §3.1 is the foundation and must be implemented first (§6.8, R-02).

```
                        ┌────────────────────────────┐
                        │  §3.1 MATERIAL ONTOLOGY    │  one record, 8 consumers
                        └─────────────┬──────────────┘
        ┌──────────────┬──────────────┼──────────────┬──────────────┐
        ▼              ▼              ▼              ▼              ▼
  §3.2 WEATHER   §3.4 DESTRUCTION §3.5 VEHICLE  §3.6 BALLISTICS §3.7 AUDIO
        │              │              │              │              │
        ├──────────────┴──────┬───────┴──────────────┴──────────────┤
        ▼                     ▼                                     ▼
  §3.3 HYDRO/ECO/FIRE   §3.8 TRAFFIC (macro→meso→micro)      §3.13 PERCEPTION
        │                     │                                     │
        └──────────┬──────────┴──────────────────┬──────────────────┘
                   ▼                             ▼
        §3.9 RESIDENT CONTINUUM ◄──────► §3.12 CRIME→DISPATCH→COURT
                   │                             ▲
                   ▼                             │
        §3.10 SCHEDULES  ──► §3.11 ECONOMY ──────┘
```

---

## 3.1 The Material Ontology

### 3.1.1 Principle

**One physical surface, one record, eight consumers.** No subsystem owns a private material table.
This is architecture invariant `AI-02`.

The failure mode this prevents is well documented in shipped titles: drywall that stops a rifle
round but not a whisper; asphalt that is slippery for cars and grippy for feet; a wooden door that
sounds like metal when shot and breaks like glass. Each of those is a symptom of two systems
disagreeing about what a surface is. Players cannot name the inconsistency but they feel it, and
it is the single most common reason "realistic" claims read as false.

### 3.1.2 `MaterialRecord` — authoritative field set

```yaml
MaterialRecord:
  id:                mat_concrete_cast_200          # stable string key, never reused
  version:           14                             # bumped on any field change; drives rebake
  display_name:      "Cast concrete, 200 mm"
  class:             ceramic_brittle                # enum, selects deformation model family

  # --- Bulk mechanical ---
  density_kg_m3:     2400
  youngs_modulus_GPa: 30.0                          # E
  poisson_ratio:     0.20
  yield_strength_MPa: null                          # brittle: no yield plateau
  ultimate_strength_MPa: 40.0                       # compressive
  tensile_strength_MPa: 3.8                         # governs spall / crack initiation
  fracture_toughness_MPa_m05: 0.9                   # K_IC
  hardness_HB:       null
  mohs_hardness:     6
  strain_rate_sensitivity: 0.04                     # d(ln σ)/d(ln ε̇) — matters for impacts
  bulk_sound_speed_m_s: 3650                        # √(K/ρ) — used by impact solver & audio

  # --- Contact / tribology ---
  friction_static:   {dry: 0.72, wet: 0.48, icy: 0.09, muddy: 0.38, oily: 0.14}
  friction_kinetic:  {dry: 0.61, wet: 0.41, icy: 0.06, muddy: 0.31, oily: 0.11}
  rolling_resistance_coeff: 0.014                   # for tyres on this surface
  restitution:       {low_v: 0.18, high_v: 0.09}    # velocity-dependent
  permeability_m_s:  1.0e-8                         # hydraulic conductivity → puddle drainage
  porosity:          0.12                           # water retention → wetness decay rate
  soil_hydrologic_group: null                       # A/B/C/D for AG cells (NRCS SCS-CN)

  # --- Thermal ---
  specific_heat_J_kgK: 880
  thermal_conductivity_W_mK: 1.7
  melting_point_K:   null
  ignition_temp_K:   null                            # non-combustible
  emissivity:        0.91
  albedo:            0.35

  # --- Acoustic (six octave bands: 125, 250, 500, 1k, 2k, 4k Hz) ---
  absorption_coeff:  [0.02, 0.02, 0.03, 0.04, 0.07, 0.09]
  scattering_coeff:  [0.10, 0.10, 0.12, 0.15, 0.18, 0.22]
  transmission_loss_dB: [52, 55, 58, 60, 62, 64]    # measured/derived STC curve
  stc_rating:        56

  # --- Ballistic ---
  ballistic:
    areal_density_kg_m2: 480                        # ρ × t
    penetration_model:  brittle_composite
    resistance_curve:                               # residual-penetration capacity vs obliquity
      normal_0deg_mm_equiv:   240
      obliquity_exponent:     1.35                  # R(θ) = R0 / cos(θ)^n
      critical_ricochet_deg:  {fmj: 62, ap: 71, hp: 55, shotgun_pellet: 48}
      spall_cone_half_angle_deg: 32                 # behind-side spall
      spall_mass_fraction:   0.14
      secondary_fragment_velocity_frac: 0.28

  # --- Deformation / destruction ---
  deformation:
    model:            fracture_voronoi              # enum: dent_basis | fracture_voronoi |
                                                    #       fibre_split | crack_graph |
                                                    #       heightfield_displace | soft_body | none
    voronoi_cell_target_m: 0.35                     # pre-fracture granularity
    cohesion_J_m2:    42                            # edge energy to separate
    rebar_layer:      true                          # reveal a secondary material on breach
    debris_lifetime_s: 240                          # before compaction into a rubble pile
    dust_emitter:     {type: mineral, albedo: 0.72, size_um: 4.5, mass_frac: 0.09}

  # --- Fire ---
  fire:
    fuel_model:       null                          # Anderson 13-class fuel model, if vegetative
    heat_of_combustion_MJ_kg: null
    fuel_load_kg_m2:  null
    surface_area_ratio_m2_m3: null
    moisture_of_extinction_pct: null

  # --- Optical (bound to the shading model, not duplicated) ---
  shading_model:    opaque_dielectric_conductor_0
  btdf_profile_id:  prof_concrete_rough_03
  subsurface_id:    null

  # --- Provenance ---
  source:           "ACI 318 Table 4.3 + measured absorption (ISO 354 chamber, 2029-03) + NIJ 0101.06 test series"
  confidence:       {mechanical: high, acoustic: measured, ballistic: measured}
```

### 3.1.3 The eight consumers

| # | Consumer | Fields read | Read frequency |
|---|---|---|---|
| 1 | Renderer / BRDF | `shading_model`, `btdf_profile_id`, `albedo`, `emissivity`, `subsurface_id` | Per-frame (cached in GBuffer materialID) |
| 2 | Physics solver | `friction_*`, `restitution`, `density` | Per contact, 120–1000 Hz |
| 3 | Destruction | `class`, `deformation.*`, `tensile_strength`, `fracture_toughness`, `bulk_sound_speed` | Per impact event |
| 4 | Ballistics | `ballistic.*`, `density`, `areal_density`, `strain_rate_sensitivity` | Per projectile intersection, up to 12 per projectile |
| 5 | Acoustics | `absorption_coeff`, `scattering_coeff`, `transmission_loss_dB` | Per traced ray bounce (15 Hz) |
| 6 | Hydrology | `permeability_m_s`, `porosity`, `soil_hydrologic_group` | Per cell per hydrology tick (1/60 Hz) |
| 7 | Fire | `fire.*`, `ignition_temp_K`, `thermal_conductivity`, `specific_heat` | Per cell per fire tick (1/30 Hz) |
| 8 | AI perception | `transmission_loss_dB` (auditory occlusion), `albedo` (visual contrast), `permeability` (track retention) | Per perception query |

### 3.1.4 Population strategy

2,412 records at ship. Provenance discipline:

| Tier | Count | Source | Examples |
|---|---|---|---|
| **Measured** | 318 | Published test data, standards bodies, or in-house measurement | NIJ ballistic test series, ISO 354 absorption chamber data, ASTM friction tests, tyre-surface μ from published skid-test data |
| **Engineered** | 894 | Derived from material class + specification (ACI, AISC, Eurocode, SAE) | Structural steel grades, concrete strengths, glazing assemblies, timber species × grade |
| **Estimated** | 1,200 | Class-average with per-record overrides | Interior finishes, fabrics, plastics, composites, soils |

**Rule:** any material that appears in a ballistics, destruction, or acoustics **validation test**
(§6.7) `MUST` be in the Measured tier. Estimated-tier materials may not be used for surfaces that
the player shoots, crashes into, or listens through during a mission. This is enforced by a cook-
time lint.

### 3.1.5 Runtime representation

- Authored as YAML in `content/materials/`, 2,412 files, ~11 MB source.
- Cooked to a **flat binary array-of-structures-of-arrays**, 484 bytes/record hot + 1.6 KB cold =
  1.2 MB hot (always resident) + 4.0 MB cold (streamed on demand by destruction/ballistics).
- Materials are referenced by **16-bit index** into the cooked array. A 16-bit materialID is packed
  into the GBuffer alongside the visibility-buffer triangle ID — this is why the visibility buffer
  is 64-bit (§5.5.6).
- `MUST NOT` be mutable at runtime. Damage does not change a material; it changes the **layer stack**
  at a position (§3.4.2), e.g. `concrete_cast_200` → `{concrete_cast_200 @ 60%, rebar_steel_A615 @
  40%}` on a spalled face. Layer stacks are per-cell mutable state, not per-material.

---

## 3.2 Weather, Microclimates, and the Atmosphere

### 3.2.1 Architecture: three coupled scales

A single global weather state is what every shipped game has, and it is the reason weather never
feels local. MERIDIAN runs **three nested scales** with distinct tick rates and solvers.

| Scale | Solver | Grid | Tick | Purpose |
|---|---|---|---|---|
| **Synoptic** | Pattern generator + front advection | 6 "air masses" over a 400 km domain | 1/3600 s (hourly) | Sets the macro regime: ridge, trough, frontal passage, marine layer, heat event, atmospheric river |
| **Mesoscale** | Semi-Lagrangian advection + diffusion on a 2D field | 512 m × 512 m, 44 × 36 cells over the world bounds | 1/30 s | Carries T, RH, precip rate, wind, pressure, cloud base/ceiling, fog, visibility |
| **Microscale** | Per-cell modifier evaluation (no solve — a function) | 64 m × 64 m, evaluated lazily on query | on demand | Applies orography, UHI, canyon channelling, sea breeze, katabatic drainage, canopy shelter, urban canyon shading |

The microscale layer is **not a grid that is simulated**; it is a **function evaluated on query**
from the mesoscale cell plus static baked modifiers. That distinction is what makes 15 km of
microclimate affordable. Storing and advecting a 64 m weather grid over 252 km² would be 61,523
cells × 11 fields × 8 bytes = 5.4 MB per timestep — cheap to store but expensive to advect stably
at 64 m resolution (CFL condition at 15 m/s wind ⇒ Δt < 4.3 s ⇒ fine, actually) — but the real
cost is that most of those cells are never queried. Lazy evaluation wins by ~40× on CPU.

### 3.2.2 `WeatherCell` state (mesoscale)

```yaml
WeatherCell:                # 44 × 36 = 1,584 cells, 96 bytes each = 152 KB total, always resident
  pressure_hPa:      1013.4
  temperature_C:     14.2         # at 2 m AGL, reduced to cell mean elevation
  dewpoint_C:        9.8
  relative_humidity: 0.74
  wind_u_ms:        -3.2          # east component at 10 m AGL
  wind_v_ms:         6.1          # north component
  wind_gust_factor:  1.38         # from Pasquill stability class
  stability_class:   D            # Pasquill-Gifford A–F
  precip_rate_mm_h:  0.0
  precip_type:       none         # none|rain|drizzle|snow|sleet|freezing_rain|hail
  cloud_cover:       0.62         # 0–1 total
  cloud_base_m:      1180
  cloud_top_m:       3400
  cloud_liquid_water_path_kg_m2: 0.18
  fog_lwp_kg_m2:     0.0          # separate from cloud: ground-level
  visibility_m:      18400
  solar_irradiance_W_m2: 412      # derived from cloud + airmass
  boundary_layer_height_m: 820
```

### 3.2.3 The synoptic driver

Six air masses, each with `(origin, T, RH, dewpoint, thickness, velocity, frontal_character)`.
Generated from a **Markov regime model** fitted to the 90-year climatology (§3.2.7) so that:

- Regime frequencies match the target climate: 34% marine-influenced, 22% continental high, 14%
  frontal passage, 11% cutoff low, 8% heat low / offshore (a foehn event — see §3.2.5), 6%
  atmospheric river, 5% convective.
- **Persistence is realistic**: regime autocorrelation at lag 1 day is 0.71, at lag 3 days 0.38.
  Weather that changes every 20 minutes of play reads as random; weather that persists for two
  in-game days and then changes decisively reads as weather.
- Seasonal modulation: the regime transition matrix is a function of day-of-year (4 harmonic terms).

Frontal passage produces the specific, recognisable sequence the player learns: **cirrus thickening
→ altostratus → warm-front stratiform rain (4–9 h, 1–3 mm/h) → warm sector (mild, humid, southerly)
→ cold front (30–90 min of heavy convective rain, 8–25 mm/h, wind shift 60–110°, temperature drop
4–9 °C) → post-frontal (clearing, cold, gusty northwesterly, excellent visibility)**. This sequence
is emergent from the model, not scripted, and it is the thing that makes USP-2 feel real to a
player who has ever watched a front come through.

### 3.2.4 Orographic precipitation and the rain shadow

**Model:** linear-theory orographic precipitation (Smith & Barston 2004 class), simplified to a
steady-state upwind/leeward asymmetry solver:

```
P(x,y) = Re[ F{ ρ_w · U · q_s · Ĥ(k) · Ĉ(k) } ]
  where Ĉ(k) = exp(−k·d) / (1 − i·k·τ_c·U)      # uplift factor with time delay τ_c = 2000 s
        Ĥ(k) = FFT of terrain height field h(x,y)
        q_s  = saturated specific humidity at the LCL
        U    = wind speed component normal to the terrain gradient
        d    = horizontal scale of the response (~9 km for this region)
```

Implemented as: (a) an offline bake of `Ĥ(k)` for the full 22.4 × 18.6 km terrain at 512 m
resolution (a 44×36 complex array — 12 KB), and (b) a runtime per-tick evaluation that is an
elementwise multiply + inverse FFT on a 44×36 grid. Cost: **31 µs per tick**, 1/30 Hz ⇒ negligible.

Then a **fallback scavenging term** removes moisture from the air mass downwind of precipitation,
which is what actually creates the rain shadow — without depletion, a linear model just makes it
rain on both sides.

**Result and validation:** the Sable Range windward (Kestrel) face receives 2,140 mm/yr and the
leeward Anza Basin 265 mm/yr. Ratio **8.08×**. Real-world analogue: the Cascades windward/leeward
ratio is 5–10× (e.g. Quinault 2,300 mm vs Yakima 200 mm). Target envelope: **5–12×**. This is the
primary validation of §3.2 (see §1.4 USP-2 falsifiable test (a)).

**Cloud-base interaction:** the LCL is computed as `LCL ≈ 125 · (T − T_d)` metres. When the terrain
exceeds the LCL, cloud forms *at the terrain* — which is the mechanism that puts the Sable Range
above a cloud deck at 1,180 m while the valley is clear. Cloud base is a per-cell field; the
renderer's volumetric cloud system (§3.2.6) samples it directly. A player standing on Sable Peak
can be **inside** the cloud, with visibility 40 m, while the valley is sunlit. That is a real
mountain experience and it is not achievable with a global weather state.

### 3.2.5 Microclimate modifiers

Seven modifiers, each a function of static baked data + the mesoscale cell. Evaluated on query.

| # | Modifier | Model | Baked data | Range |
|---|---|---|---|---|
| **M1** | **Elevation lapse** | `T(h) = T_cell − Γ·(h − h_cell)`, Γ = 6.5 K/km (saturated) or 9.8 K/km (dry), selected by whether the parcel is saturated; Γ transitions smoothly across the LCL | Terrain height | −16.0 K at Sable Peak |
| **M2** | **Orographic wind speedup** | `U(x) = U_cell · S(x)`, S from a Bernoulli/continuity constriction factor computed from the local cross-section narrowing, plus a ridge-crest speedup of up to 1.9× and a lee recirculation zone of 0.2–0.5× extending 3–8× the hill height downwind | Terrain gradient + curvature field (2 m res) | 0.2× – 1.9× |
| **M3** | **Urban Heat Island** | Nocturnal cooling rate ∝ **sky view factor** Ψ_s (Oke); daytime storage ∝ impervious fraction × thermal admittance. Plus anthropogenic heat flux Q_f from the energy-demand model (§3.11.4), plus evapotranspiration deficit. `ΔT_UHI = a·(1−Ψ_s) + b·Q_f − c·E + d·(1−imperv)`, calibrated to give **≤ +3.4 K** at night in Cathedral Quarter, **≤ +1.1 K** daytime, and **< +0.5 K** on nights with U > 5 m/s (ventilation). | Ψ_s per 64 m cell (baked from building geometry, 4-bit), impervious fraction, building thermal admittance class | −0.3 K to +3.4 K |
| **M4** | **Canyon wind channelling** | Per-district **baked flow field** from offline steady RANS (4 m resolution, 16 wind directions, 6 districts = 192 core-hours one-time). Runtime: interpolate by wind direction (circular lerp between the two bracketing baked directions), blend with M2. Produces the real effect: a 6 m/s regional wind becomes 11 m/s in the Cathedral Quarter's 22 m-wide, 90 m-tall canyons (venturi) and 0.8 m/s in sheltered courtyards. | 6 districts × 16 directions × (4 m grid, 3 velocity components, 16-bit) = 240 MB on disk, 34 MB resident for loaded districts | 0.08× – 2.1× |
| **M5** | **Sea breeze / marine layer** | A **coastal front** solver: when land–sea ΔT exceeds ~2.5 K by mid-morning, a marine boundary layer advances inland at 1–4 m/s to a penetration depth of 8–35 km, capped by the Sable Range. Carries cloud base 300–700 m, T depression 4–8 K, visibility reduction. Retreats at dusk. Seasonal: strongest May–August, absent Nov–Feb. | Coastline geometry, prevailing wind, sea surface temperature (a separate 1-D slab model with 340-day thermal inertia) | −8 K, +100% RH |
| **M6** | **Katabatic drainage & radiation fog** | At night under stability class E/F with U < 2 m/s, cold air drains downslope along the terrain gradient and pools in topographic lows. Fog forms where the pooled layer saturates. Modelled as a **downslope flux accumulation** on the flow-accumulation grid (already computed for hydrology, §3.3.1 — free reuse) with a pooling depth from local terrain concavity. | Flow accumulation grid, terrain concavity | +12 K inversion strength, visibility 20 m |
| **M7** | **Canopy shelter** | Under a canopy with closure > 0.5: wind reduced to 0.15–0.35× at 2 m (exponential decay with a canopy-specific extinction coefficient), T range damped by ±2.5 K, RH raised 6–14%, solar irradiance reduced to 2–18% (sunflecks handled as a high-frequency spatial noise term). | Canopy closure + height per 4 m cell (from the vegetation layer) | 0.15× wind |

**Modifier M6 is the reason Meridian Valley fogs.** 62% of clear, calm, high-humidity autumn nights
produce valley fog by 04:00 that burns off between 09:00 and 11:30 as the inversion breaks. It does
not form on windy nights (M2/M3 ventilate) or in the urban core (M3 keeps the air warm). This
produces the §1.4 USP-2 validation criterion (c) for free.

### 3.2.6 Volumetric clouds, fog, and precipitation rendering

**Cloud model:** 3D density field from a Worley-noise cellular base modulated by a Perlin detail
layer, with density driven per-cell by `cloud_cover`, `cloud_base_m`, `cloud_top_m`, and
`cloud_liquid_water_path`. Beer-Lambert + single-scattering with a **Henyey-Greenstein phase
function** (two-lobe, g₁ = 0.65 forward, g₂ = −0.45 back, weight 0.7) and **powder/beer's-powder
term** for the self-shadowing that makes cloud bases look heavy. Multi-scattering approximated by
6 octaves of progressively wider, cheaper scatter samples (the Hillaire/Horizon-Zero-Dawn approach).

**Two-pass raymarch:**
1. **Low-res pass:** ¼ screen resolution, 8–12 adaptive steps between the near and far cloud
   bounds, hierarchical step-skipping on empty space via a **pre-integrated density min/max
   froxel** structure (a 3D texture at 64×64×32 over the loaded region, rebuilt at 4 Hz).
2. **High-res cone pass:** only on pixels whose reprojected history is invalid (disocclusion,
   camera motion above threshold, first frame after a regime change) — typically 8–18% of pixels.
   24–32 steps.
3. **Temporal:** reprojection with motion vectors, 16-frame history, variance-clipped.

Budget: **1.4 ms** (GPU) at 1440p internal on the reference PC-Mid; **1.7 ms** on PS5 (lower
clock, same work). Governor tiers at §5.11.3.

⚠ **HONEST LIMITATION:** we do not raymarch clouds over the full 15 km horizon at high fidelity.
Beyond 6 km, clouds switch to a **projected 2D cloud-shape texture** on a paraboloid dome with the
same density/colour solve, cross-faded between 5.5 and 7 km. At cloud base 1,180 m and a 6 km
horizontal distance the viewing angle is < 11° above the horizon, where the parallax error of a
dome projection is below the 1 px silhouette-discontinuity threshold of §1.4 USP-6. Validated by the
40-viewpoint horizon capture test.

**Ground fog:** a separate, thinner volumetric layer whose density is driven by M6's pooled
inversion depth and the cell's `fog_lwp`. Rendered in the same pass with a distinct phase function
(no forward lobe — fog is nearly isotropic) and a lower step count. Fog **interacts with the
acoustics system**: high humidity above 60% reduces high-frequency atmospheric absorption
(~0.02 dB/m at 4 kHz dry vs ~0.008 dB/m saturated), so sound carries further in fog. That coupling
is a real atmospheric-acoustics effect and it is free once both systems read the same field.

**Precipitation rendering:** GPU particle system, 60,000 rain streaks max within a 40 m camera box,
streak length ∝ drop terminal velocity (9.1 m/s for a 2.5 mm drop) + wind vector, **collision
against the terrain and building heightfields only** (not full scene depth — a 4× saving that is
undetectable because splash decals are spawned from the heightfield hit). Snow: 24,000 flakes, 6
crystal types, flutter from a curl-noise wind field, accumulates into a **snowpack layer** (§3.3.4)
that is a real terrain mutation, not a texture.

**Dynamic puddling and surface wetness** — see §3.3.3, which is part of the hydrology system
because it is the same water.

### 3.2.7 The 90-year climatology bake

Everything above has parameters (a, b, c, d in M3; the Markov transition matrix; regime
frequencies; phenological dates; snowpack; fire weather indices). They are not hand-tuned. They are
**fitted**.

Process:
1. Author a target climate description for New Vermillion: marine west-coast, Köppen **Csb** in the
   city/valley/coast, **Cfb** in the forest, **Dsb** at elevation, **BSk** in Anza Basin. Monthly
   normals from a real-analogue station composite (latitude 41.6° N, coastal, with a 2,400 m range
   within 90 km — a genuinely rare and interesting climate, matching the Pacific Northwest).
2. Run the full weather model offline at 1,200× real time for 90 simulated years = 2.84 G
   sim-seconds. Build-farm cost: 11 days × 96 cores = 25,300 core-hours.
3. Fit: (a) the Markov transition matrix by maximum likelihood on the regime sequence; (b) M3's
   coefficients by least squares against the target UHI envelope; (c) phenological dates by matching
   the target growing-degree-day accumulation (base 5 °C, 1,420 GDD to apple bloom); (d) fire-weather
   indices against a target 97th-percentile event frequency.
4. Ship the output as a 340 MB `climatology.strata` containing per-cell monthly normals, return
   periods (2/10/50/100-year precipitation and wind), and the fitted parameters.

**Why this matters for gameplay:** return periods let design specify "the 100-year storm that floods
the Cordova underpass" as a *real* event with a *real* frequency (1% per year ⇒ the player sees it
with probability 26% over a 30-in-game-day playthrough) rather than as a scripted mission. And it
lets QA test against a known distribution.

### 3.2.8 Weather tick budget

| Item | Cost | Rate |
|---|---|---|
| Synoptic regime step | 12 µs | 1/3600 s |
| Mesoscale advection (semi-Lagrangian, 1,584 cells × 11 fields) | 184 µs | 1/30 s |
| Orographic FFT solve | 31 µs | 1/30 s |
| Boundary-layer / stability update | 44 µs | 1/30 s |
| Microscale modifier query (lazy, per consumer) | 2.1 µs/query, ~9,000 queries/frame | per frame |
| Cloud froxel rebuild | 0.18 ms GPU | 4 Hz |
| **Total CPU** | **≈ 19 µs/frame amortised + 19 ms/frame for microscale queries** | — |

The microscale query cost is the real one: 9,000 queries/frame at 2.1 µs = 18.9 ms, which is over
budget on its own. Mitigation: (a) queries are batched by system and **cached per 64 m cell for
4 frames** (weather does not change meaningfully in 66 ms) — reduces to ~1,900 unique
queries/frame = 4.0 ms; (b) that 4.0 ms is spread across 4 worker threads = **1.0 ms/frame wall
clock**; (c) the cache is invalidated immediately on cell change, so there is no staleness the
player can detect. **Net: 1.0 ms CPU across the weather system.** Within the §5.11.1 simulation
budget of 3.0 ms across 3 threads.

---

## 3.3 Hydrology, Ecology, Fire, and Phenology

### 3.3.1 Hydrology

Water is a single conserved quantity, not a visual effect. The same water rains, puddles, runs off,
infiltrates, floods, evaporates, and irrigates.

**Model chain per 64 m cell, at 1/60 Hz:**

```
precip (mm/h, from §3.2)
  → interception          I = min(P, S_canopy), S_canopy = 0.2 · LAI · (1 − e^(−0.5·P/S))
  → throughfall           P' = P − I
  → snow partition        if T < 0.5 °C: → snowpack (§3.3.4); if 0.5–2.0 °C: mixed
  → infiltration          Green-Ampt:  f = K_s · (1 + ψ·Δθ·M / F)
                            K_s from MaterialRecord.permeability_m_s (or soil_hydrologic_group)
                            ψ = wetting-front suction (m), Δθ = moisture deficit, M = depth
                            F = cumulative infiltration (state)
  → surface runoff        NRCS SCS Curve Number:  Q = (P − 0.2·S)² / (P + 0.8·S),
                            S = 254·(100/CN − 1)
                            CN from land use × hydrologic soil group × antecedent moisture condition
                            (AMC I/II/III from the prior 5-day rainfall total)
  → surface storage       shallow-water solve (§3.3.3)
  → soil moisture         3-layer bucket (0.15 m / 0.6 m / 2.0 m), Richards-style flux between
                            layers, field capacity & wilting point per soil class
  → lateral subsurface    topographic-index (TOPMODEL-style) transmissivity on the flow-accumulation grid
  → groundwater           unconfined aquifer, 1 layer, transmissivity from substrate
  → baseflow              linear reservoir, recession coefficient k = 0.94/day
  → channel routing       Muskingum:  O_{t+1} = C1·I_{t+1} + C2·I_t + C3·O_t
                            C1 = (Δt − 2Kx)/(2K(1−x) + Δt), etc. K = 2.4 h, x = 0.25 for the
                            Meridian River mainstem; per-reach values from slope & geometry
  → streamflow            Q (m³/s) per reach; 1,412 reaches
  → floodplain            where Q exceeds bankfull, water spreads onto the 2 m floodplain grid
                            using the same shallow-water solver
```

**Outputs consumed by gameplay:**
- River stage (m) at 88 gauging points → bridge clearance, fordability (a 0.6 m ford at base flow
  is a 1.9 m ford after a storm — a real navigation constraint), boat navigation, fisheries.
- Flood extent → Cordova Flats underpass closure, valley road closures, property damage entries in
  the ledger, insurance claims in the economy model, and a **mud layer** on receded floodplain
  (a terrain mutation with a `mud_sat` material override, μ = 0.31).
- Soil moisture → tyre and foot friction on unpaved surfaces, dust generation (dry) vs mud
  (saturated), fire behaviour (§3.3.5), agricultural yield.
- Groundwater → well levels for the 412 rural properties that use wells (a ledger attribute; a
  drought year drops wells and forces water hauling, which changes resident schedules).

**Budget:** 61,523 cells at 1/60 Hz, but **only loaded-region cells** (a 3 km radius macro-region =
28 km² = 6,836 cells) run the full chain; the rest run a **1/3600 Hz aggregate** on a 512 m grid
(1,584 cells). Cost: 0.62 ms CPU per second of sim time at 1/60 Hz ⇒ **10 µs/frame**, plus 0.04 ms
for the shallow-water solve (§3.3.3). The hydrology system is *cheap* because the expensive part
(shallow water) is spatially restricted.

### 3.3.2 Ecology — flora

1,240 plant species in the design bible; **340 individually modelled** (each with a mesh set, a
phenology curve, a growth form, and a material record); the remaining 900 are represented as
**aggregate groundcover/understorey types** (38 types) — an honest scope decision, because
individual meshes for 1,240 species at 3 LOD-equivalents is 3,720 mesh sets and the disk budget
(Appendix C) cannot carry it.

Placement is **derived, not authored**:
1. Per-64 m-cell **habitat suitability index** per species, from: biome class (§2.3.2), soil class,
   slope, aspect, elevation, canopy closure, moisture regime (from §3.3.1's soil moisture
   climatology), disturbance history (fire, logging, grazing, flood), and land use.
2. **Competition resolution**: per cell, species are ranked by suitability × a shade-tolerance
   interaction matrix; the top N (by growth form: 1 canopy tree, 3 subcanopy, 8 shrub, 12
   herbaceous) get allocation, with frequency proportional to suitability.
3. **Spatial pattern**: not uniform. Individual-based placement via a **Thomas cluster process** for
   clonal species (aspens, many shrubs) and a **hard-core Strauss process** for competition-
   limited trees, both seeded by a hash of the cell coordinate so the result is deterministic and
   reproducible without storage.
4. **Density targets:** old-growth stand 340–620 stems/ha (dbh 40–210 cm), managed second growth
   1,100–1,800 stems/ha at age 22, orchard 420 trees/ha on a 4.8 × 5.0 m grid, vineyard 3,300
   vines/ha, urban street trees 62/km of frontage.

Total **instanced vegetation individuals: 148.6 million**, of which ~2.1M are rendered at any time
within the loaded region. Rendering is a **separate path from virtualized geometry** (§5.5.7):
instanced meshes with vertex-shader wind animation, because virtualized geometry does not support
per-vertex world-position offset in the cluster-culling path. Canopy trees get 3 mesh variants +
4 rotation/scaling permutations; grasses and forbs use a **card-and-cross billboard** system with a
per-species BTDF profile at > 30 m, transitioning to geometry < 30 m.

**Phenology:** each species has a curve over the growing-degree-day axis: bud swell → leaf-out →
flower → fruit set → senescence → leaf-fall → dormancy. Leaf-fall is a **physical event**: leaves
become a 0.02–0.14 m debris layer on the ground (a terrain material override with μ_dry = 0.44,
μ_wet = 0.22, high flammability in autumn, and a distinctive sound). 11,400 kg of leaf litter per
hectare in the deciduous stands; the sim tracks a per-cell litter mass that decays by decomposition
(half-life 14 months, temperature-dependent) and is consumed by fire (§3.3.5).

### 3.3.3 Hydrology ↔ rendering ↔ friction: puddles and surface wetness

This is the requirement most likely to be faked, so it is specified precisely.

**Two coupled representations:**

**(A) Surface wetness** — a scalar `w ∈ [0,1]` per surface point, stored in a GBuffer channel and
in a world-space **4 m-resolution wetness texture** over the loaded region (28 km² = 7.2M texels ×
1 byte = 7.2 MB). Update at 2 Hz:

```
dw/dt =  k_rain · P · sky_exposure                       # accumulation
       − k_evap · (T, U, RH, solar) · (1 − shelter)      # evaporation (Penman-style)
       − k_drain · permeability · w                      # drainage into substrate
       + k_runon · upstream_flux                         # water arriving from upslope
       + k_splash · nearby_impact                        # vehicle/pedestrian splash
```
`sky_exposure` is a **baked per-texel sky-visibility term** (like a sky-facing AO) plus a runtime
screen-space sky-occlusion for dynamic geometry. Eaves, awnings, bridges, and canopies stay dry
while the street floods — a detail players notice instantly and that requires the sky-visibility
bake, which is why it is specified.

`w` drives the shading: roughness remapped `roughness ← lerp(roughness_dry, 0.04, w·0.85)`,
specular F0 raised, albedo darkened by a per-material `wet_albedo_scale` (0.55–0.78 depending on
porosity — porous materials darken more, which is the physical reason wet concrete is darker than
wet steel), plus a **thin-film interference** term on hydrophobic surfaces (waxed car paint shows
beading: a procedural bead normal map whose density ∝ (1 − porosity)).

**(B) Standing water** — a **2D shallow-water solver** on a 0.5 m grid over a 128 m radius around
each simulation source (256×256 = 65,536 cells):

```
∂h/∂t + ∇·(h·u) = P·sky_exposure − infiltration − evaporation
∂(h·u)/∂t + ∇·(h·u⊗u) = −g·h·∇(z_b + h) − τ_b/ρ
  z_b = terrain + a "micro-relief" field (kerb heights, camber, ruts, the soil deform layer §3.4.6)
  τ_b = Manning bed shear: τ_b = ρ·g·n²·|u|·u / h^(1/3), n from MaterialRecord (asphalt 0.013,
        grass 0.035, gravel 0.030, mud 0.045)
```
Solved with **20 Gauss-Seidel/Jacobi iterations on GPU compute**, 10 Hz, cost **0.28 ms GPU**.
Beyond the 128 m radius, standing water is represented by the static wetness field only (no flow),
which is undetectable because flow is only observable at close range.

Output: a **puddle depth texture** (16-bit, 0–200 mm) sampled by:
- the renderer (refraction, reflection, and a foam/ripple normal where depth changes),
- the vehicle tyre model (aquaplaning, §3.5.6),
- the pedestrian system (splash events, gait change, avoidance steering — residents walk *around*
  puddles, which requires the puddle depth texture to be an input to pedestrian cost fields, §6.3),
- the audio system (splash Foley triggered by contact with depth > 3 mm),
- the soil deform layer (rain erases tyre tracks and footprints at a rate ∝ cumulative rainfall).

**Aquaplaning onset** — Horne's empirical relation:

```
V_p [km/h] = 16.66 · √(p_tire [kPa] / 6.895)      # from V_p[mph] = 10.35·√(p[psi])
```
A tyre at 220 kPa (32 psi) aquaplanes at **94 km/h**; at 180 kPa, 85 km/h; at 260 kPa, 102 km/h.
Onset is gradual: above 0.7·V_p the effective μ degrades as `μ_eff = μ_wet · (1 − (v/V_p)^4)` and
the tyre's aligning torque collapses (the driver feels the steering go light — which is the
*correct* cue and is the reason this matters for game feel). Puddle depth modulates V_p: below
2.5 mm, no effect; 2.5–10 mm, the model above; above 10 mm, V_p reduced by up to 22%.

**Budget summary:** wetness 7.2 MB texture + 0.06 ms GPU; shallow water 131 KB grid + 0.28 ms GPU +
0.02 ms CPU; total **0.34 ms GPU, 0.02 ms CPU**.

### 3.3.4 Snowpack

Per-cell, 1/3600 Hz: SWE (mm), depth (m, from SWE/density), density (50–550 kg/m³, aging),
layered profile (5 layers for avalanche stability), liquid water content, albedo (0.85 fresh →
0.45 aged → 0.32 dirty/melt), thermal conductivity. Melt by a **temperature-index method with an
energy-balance correction**: `M = DDF · max(0, T − T_base) + SR_term`, DDF = 3.2 mm/°C/day for
open, 2.1 mm/°C/day under canopy (canopy interception and shading). Rain-on-snow events add latent
heat and produce the region's largest floods — a real Pacific-Northwest mechanism.

Snow is a **terrain mutation**: the renderer adds a snow-height layer to the terrain and to
horizontal surfaces (a baked "snow-catch" mask per building/prop that determines where snow
accumulates and where it slides off), the friction model swaps to `icy`/`snowy` entries in the
material record, and the hydrology routes melt to the channel network. **Snowploughs are ledger
entities**: the county issues plough work orders based on accumulation, and the player can watch a
plough clear Sable Pass over 40 minutes of real time, with the cleared lane persisting as a mutable
state entry.

### 3.3.5 Fire

**Spread model: Rothermel (1972).** This is the standard and it is what real fire behaviour
analysts use.

```
R = (I_R · ξ · (1 + φ_w + φ_s)) / (ρ_b · ε · Q_ig)        [m/s]

I_R = Γ_max · w_n · ε_min · M_A · S_T                        reaction intensity  [kJ/m²/s]
Γ_max = (C·β^(-A) / (β·(1 + B)))                             max reaction velocity, β = ρ_b/ρ_p
ξ   = exp((0.792 + 0.681·√(σ_SA))·(β + 0.1)) / (192 + 0.2595·σ_SA)   propagating flux ratio
φ_w = C_w · B_w · β^(-E_w) · U^B_w                            wind factor (U in m/s, midflame)
φ_s = 5.275 · β^(-0.3) · tan²(θ)                              slope factor
ε   = exp(-138/σ_SA)                                          effective heating number
Q_ig = 250 + 111,600·M_f                                      heat of preignition [kJ/kg]
```
with `ρ_b` bulk density, `w_n` net fuel load, `M_f` fuel moisture, `σ_SA` surface-area-to-volume
ratio, `M_A` moisture damping, `S_T` mineral damping.

**Fuel:** each vegetative `MaterialRecord` carries an **Anderson 13-class fuel model** assignment
(§3.1.2 `fire.fuel_model`) plus fuel load, surface-area-to-volume ratio, and moisture of
extinction. Fuel moisture is **derived from the weather field** (fine fuels equilibrate in 1–10 h
from T/RH via an EMC curve; duff and woody fuels on a 10–100 h lag; live fuel from seasonal cure
and the hydrology model's soil moisture). This is the coupling that makes fire weather real: the
same stand burns at 0.4 m/s flame-front speed after rain and 6.1 m/s during a foehn event.

**Derived outputs:**
- Fireline intensity (Byram): `I = H · w · R` [kW/m], H = heat of combustion (kJ/kg), w = fuel
  load (kg/m²).
- Flame length: `L = 0.0775 · I^0.46` [m].
- Crown-fire initiation (Van Wagner): requires surface intensity `I > I_0 = (0.010·CBH·(460 + 25.9·M_live))^(3/2)`
  and wind above a critical value.
- Spotting: ember transport distance from fireline intensity and wind (a stochastic model, 0.1–
  2.4 km), creating spot fires ahead of the front.
- Smoke: a particulate plume from the burn rate, advected by the mesoscale wind field, which
  **feeds back into §3.2's visibility and solar irradiance** — a large fire darkens the region
  downwind, cools it, and raises its PM₂.₅, and residents respond (health advisories in the ledger,
  school closures, schedule changes).

**Suppression:** 7 fire stations in-region (VC FD ×4, Halstead FD, County FR ×2, CAL-analogue
forestry ×1 seasonal). Response modelled identically to police dispatch (§3.12.5) with a separate
CAD, separate apparatus (engines, ladder trucks, tenders, bulldozers, 2 helitankers, 1 air tanker
on 15-minute alert from VMI during red-flag warnings), and separate jurisdiction. **Sable Foothills
is the wildland-urban interface** and the region's highest structural-loss risk; a fire there is a
multi-day event that evacuates ledger residents, closes roads, destroys structures permanently
(written to the delta journal), and drops land values.

**Simulation cost:** fire runs on the 64 m cell grid within 2 km of any ignition, at 1/30 Hz,
with an 8-neighbour spread solve. A 400 ha active fire = 6,250 cells ⇒ 0.09 ms per tick. Capped at
3 concurrent fires (a design and budget decision: more than 3 makes the smoke plume budget and the
destruction budget both blow).

### 3.3.6 Ecology — fauna

**312 vertebrate species.** 44 terrestrial mammals, 187 birds, 41 fish, 12 reptiles/amphibians,
28 marine mammals and diving birds. Population dynamics per species per **habitat patch** (the
region is partitioned into 1,940 patches by roads, rivers, and land use — patch connectivity is a
real fragmentation metric that the sim tracks).

**Population model** (per species × patch, 1/86400 Hz — daily):

```
N_{t+1} = N_t · exp( r · (1 − N_t/K_patch) ) − M_predation − M_harvest − M_vehicle − M_other
K_patch = HSI(patch) · area · seasonal_multiplier
```
- `HSI` = habitat suitability index from §3.3.2's inputs (vegetation structure, water, cover, food).
- `r` = intrinsic growth rate (elk 0.28/yr, deer 0.42/yr, coyote 0.51/yr, black bear 0.19/yr).
- `M_vehicle` is **emergent**: every embodied or Tier-2 agent (including player) collision with an
  animal is logged and subtracted. On the region's 2,640 km of road with 7.9M vehicle-km/day, deer-
  vehicle collisions run at 41/day — a number that falls out of the traffic model and the population
  model agreeing, not a number anyone authored.
- **Migration**: 3 species (elk, mule deer, sandhill crane analogue) have seasonal range shifts
  driven by snow depth (§3.3.4) and forage availability — elk descend from the Sable Range to the
  valley winter range when snow exceeds 0.4 m, which visibly changes what the player encounters by
  season.

**Trophic cascades — two authored-complete, emergent-behaviour chains:**

1. **Terrestrial:** grey wolf (reintroduced in the sim's year −14) → elk browsing pressure →
   riparian willow and aspen recruitment → beaver colonisation → pond formation → stream
   morphology, water table, and songbird habitat. Removing wolves (the player can, and there is a
   legal hunting season with tags issued by the state model) produces measurable riparian
   degradation over 3–5 simulated years.
2. **Marine:** sea otter → purple urchin → kelp forest extent → reef fish abundance → diving
   visibility and the fishery yield the Halstead co-op depends on. Otter population is 1,240; a
   decline (from a sim's oil spill event at Cordova Flats, or from orca predation pressure) causes
   an urchin barren and the loss of 3 of the 41 named dive sites.

**Why include cascades at all:** because they are the strongest available demonstration of Pillar 1
and Pillar 3 at a timescale the player can perceive within a 40-hour playthrough, and they cost
almost nothing beyond systems already required (population model, hydrology, vegetation, economy).

⚠ **HONEST LIMITATION:** individual animals are only embodied within 400 m of a simulation source
(cap: 220 individuals). Outside that they exist as **patch population counts and a density field**;
a herd of 340 elk in a distant meadow is a Tier-3 density aggregate that renders as instanced
billboards with VAT locomotion and rehydrates into individuals as the player approaches. The
rehydration boundary is at 400 m with 80 m of hysteresis, and herd composition is drawn
deterministically from the patch record so the same herd is the same herd.

### 3.3.7 Ecology tick budget

| Item | Cost | Rate |
|---|---|---|
| Hydrology full chain (loaded region, 6,836 cells) | 0.62 ms/s sim | 1/60 Hz |
| Hydrology aggregate (1,584 cells) | 0.04 ms/s sim | 1/3600 Hz |
| Shallow water (GPU) | 0.28 ms GPU | 10 Hz |
| Vegetation state (litter, phenology advance) | 0.11 ms/s sim | 1/3600 Hz |
| Fauna population (312 × 1,940 patches, sparse: 148k active pairs) | 0.31 ms/s sim | 1/86400 Hz |
| Fire (per active fire) | 0.09 ms/tick | 1/30 Hz |
| Snowpack | 0.06 ms/s sim | 1/3600 Hz |
| **Total** | **≈ 0.03 ms/frame amortised CPU + 0.28 ms GPU at 10 Hz** | |

Ecology is nearly free at runtime. Its cost is **offline bake and authoring**, not frames.

---

## 3.4 Destruction & Deformation

### 3.4.1 Principle

Deformation is selected **by material class**, not by object type. `MaterialRecord.class` and
`deformation.model` (§3.1.2) determine which of seven models runs. A concrete barrier and a
concrete wall deform the same way because they share a class; a car door and a car roof deform the
same way but differently from the door's steel reinforcement bar, because they are different
records.

### 3.4.2 The layer stack

Mundane but essential: a surface point is not one material, it is a **stack**.

```
LayerStack(position) = [ {material_id, thickness_mm, state}, ... ]   # outermost first
Example — a 1998 Riverton walk-up exterior wall:
  [ plaster_render_12   (12 mm, intact)
  , brick_clay_100      (100 mm, cracked 0.34)
  , cavity_air_50       (50 mm)
  , insulation_mineral_90 (90 mm, compressed 0.0)
  , cmu_block_190       (190 mm, intact)
  , plaster_gypsum_13   (13 mm, intact) ]
```
The ballistics solver (§3.6.4) walks the stack front-to-back consuming residual penetration. The
acoustics solver computes composite transmission loss from the stack (mass-air-mass resonance
included, which is why a cavity wall has worse low-frequency TL than the mass law predicts — a real
effect and a real difference the player hears). The destruction solver mutates stack entries:
`brick_clay_100 → breached`, exposing `cavity_air_50`, then `insulation_mineral_90 → scorched`.

Layer stacks are **authored per building type** (312 wall assemblies, 94 roof assemblies, 61 floor
assemblies, 148 glazing assemblies, 74 vehicle panel assemblies) and assigned to geometry at cook
by a material-region paint layer. **Not per-texel** — that would be unaffordable. Per-region, with
412 regions per average building.

### 3.4.3 The seven deformation models

**(1) `dent_basis` — ductile sheet metal (vehicle bodies, cladding, ductwork)**

Offline: each deformable panel is partitioned into **240–680 dent zones**. For each zone, a basis of
**6 displacement fields** is generated from a reduced-order model of a shell-element solve
(PCA over 400 sampled impact configurations spanning energy 0.5–80 kJ, direction ±45°, and
obliquity 0–70°). Basis vectors are 3-float displacements at the panel's vertex resolution,
quantised to 12 bits and compressed.

Runtime: on impact, select the affected zones by proximity, decompose the impact impulse onto the
basis (a ≤ 6×6 solve), and blend. Add **crease lines** from a precomputed fold-network per panel
(a hinge appears where the real thing hinges — the A-pillar, the beltline, the wheel arch).
Memory: **48–130 KB per unique panel set**, 340 vehicles × avg 22 panels ⇒ 2,640 panel sets,
1.9 GB on disk, **110 MB resident** (streamed with the vehicle).

Cost per impact: 4.2 µs CPU. Deterministic (no RNG in the blend).

This is the technique class used by RAGE for vehicle deformation; the delta here is that the basis
is derived from an actual shell-element ROM rather than hand-sculpted, and that the *same* basis
system serves architectural cladding.

**(2) `fracture_voronoi` — concrete, masonry, stone, cast iron**

Offline: pre-fracture the mesh into Voronoi cells at `voronoi_cell_target_m` (0.35 m for concrete,
0.12 m for brick, 0.9 m for rock). Build a **cohesion graph**: each shared face has a cohesion
energy `cohesion_J_m2 × face_area × strength_variance(seed)`.

Runtime: impact energy distributes into the graph; edges whose absorbed energy exceeds their
cohesion break; connected components below a size threshold become **rigid debris bodies** (Tier 3
physics, §5.4.4) with the parent's material record. Behind the fracture surface, a **secondary
layer is revealed** (`rebar_layer: true` ⇒ exposed rebar geometry + a `rust_stain` decal + a
`concrete_spalled` material override on the fresh surface, which has a different albedo, roughness,
and acoustic absorption than the weathered face — the real reason a broken wall looks broken).

**Spall** specifically: brittle materials under a compressive stress wave spall on the *far* face.
Modelled by propagating the impact's stress wave through the layer stack at `bulk_sound_speed_m_s`
and, if the reflected tensile wave at a free surface exceeds `tensile_strength_MPa`, ejecting a
**spall cone** at `spall_cone_half_angle_deg` from the far face with mass `spall_mass_fraction` of
the impacted volume. A 7.62 mm round hitting a 200 mm concrete wall from outside ejects a cone of
fragments **inside** — which is why cover facing the wrong way is dangerous, and which almost no
game models.

**(3) `fibre_split` — timber**

Fracture is anisotropic: wood fails in bending/tension perpendicular to grain at ~1/12 of its
parallel-to-grain strength. Each timber member carries a **grain direction** and is modelled as an
Euler-Bernoulli beam with a fibre stress distribution across its section; failure initiates on the
tension face and propagates along the grain, producing **splinters** (elongated debris with a
length:width ratio of 4:1 to 14:1, matching real fracture) and a **hinged partial failure** (a
splintered post that still carries load is a distinct state from a severed one — and the player can
see it, which is what makes wooden cover feel different from concrete cover).

**(4) `crack_graph` — glass, glazed tile, brittle thin panels**

A precomputed **radial + concentric crack graph** per glazing assembly. Projectile or blunt impact
selects a nucleation point; cracks propagate along graph edges with probability weighted by
distance from nucleation, impact energy, and the assembly's temper (annealed glass produces large
shards and stays in frame; tempered glass dices into 6–10 mm cubes and collapses; laminated glass
spiders but retains — all three behaviours are distinct and all three are in the region's 148
glazing assemblies).

Penetration punches a hole whose radius scales with projectile diameter and energy
(`r_hole = r_proj · (1 + 0.42·ln(E/E_ref))`), and the shard field around it is removed from the
collision and visual meshes.

**(5) `heightfield_displace` — soil, mud, sand, snow, asphalt-with-base**

Per-region **deformation heightfield** (§3.4.6) — see below, because it is the most-used model.

**(6) `soft_body` — fabric, rubber, foliage, tarps, awnings, body tissue**

PBD/XPBD volume or shell elements, 200–2,400 particles per instance, 3–4 solver iterations at
60 Hz. LOD: beyond 25 m, soft bodies are replaced by a **mode-reduced proxy** (the first 12
modes of a modal analysis, integrated as a 12-DoF spring system at 20 Hz) — a 30× saving that is
undetectable at that distance.

**(7) `none` — rigid, undeformable within the game's energy range**

Granite outcrop, structural steel at low energy, armour plate. These still produce **sparks, sound,
ricochets, and decals**, because "no deformation" is not "no response".

### 3.4.4 Structural failure and building collapse

⚠ **HONEST LIMITATION.** Real-time finite-element structural analysis of a 74-storey building is
not achievable at 60 FPS on any current hardware. We do not pretend otherwise. What we do instead
is a **quasi-static load-path solver on a baked structural graph**, which is both affordable and
produces failures that a structural engineer would recognise.

**Offline bake per structure:**
- A **structural graph**: nodes = load-bearing elements (columns, shear walls, core walls,
  transfer beams, floor diaphragms, foundation), edges = load paths. A 74-storey tower has ~2,900
  nodes. Total region-wide: 14.2M nodes, 88 MB (quantised, compressed).
- Each node stores: `demand_ratio` (utilization under gravity + code-specified lateral load),
  `tributary_area`, `redundancy_count` (how many alternate load paths exist if this node is
  removed), and `material_id`.
- Offline we compute, for each of the **top 64 most-consequential node removals**, the resulting
  redistribution and the set of nodes that then exceed utilization 1.0. This is a **precomputed
  progressive-collapse sensitivity map**.

**Runtime:**
1. Damage events (explosion, impact, fire, structural member shot) reduce node capacity.
2. At 5 Hz, the quasi-static solver recomputes demand redistribution on the affected subgraph only
   (typically < 400 nodes) via a fixed 6-iteration relaxation.
3. Nodes exceeding utilization fail → become debris → their demand redistributes → repeat, up to 12
   propagation generations per tick (a hard cap to prevent a runaway).
4. If **global stability** (fraction of gravity load reaching foundation) falls below 0.62, the
   structure enters **collapse**: a parameterised sequence generated from the structural graph —
   floor-by-floor pancake for moment frames, hinging for shear-wall structures, progressive for
   transfer-level failures. Near-camera debris (within 40 m) is real rigid bodies (capped at 600);
   everything else is a **particle + rubble-pile** representation, and the settled result is written
   to the delta journal as a **rubble pile heightfield** with a new set of navigable surfaces.

**Cost:** 0.4 ms CPU during an active collapse, 0 otherwise. **Budget:** ≤ 1 concurrent collapse
(a hard design cap — a second collapse during the first is queued and begins after the debris
settles, which is also more realistic because a real collapse monopolises everything).

**Design consequence:** 8 structures in the region are collapse-capable as authored set-pieces
(a parking structure, a 6-storey unreinforced-masonry block in Portside, the Cordova Flats
cracking unit, 2 warehouses, a bridge span, a mine headframe, a lift shaft). The other 83,392
structures support **local failure only** (wall breach, floor penetration, roof hole, stair
collapse) and never global collapse. This is stated plainly because pretending otherwise is how
projects die.

### 3.4.5 Destruction budget

| Item | Cap | Notes |
|---|---|---|
| Concurrent destruction jobs | 24 | Job-graph priority queue; overflow is coalesced |
| Live debris rigid bodies | 3,000 region-wide, 600 within 40 m of camera | Beyond: static rubble impostors |
| Debris geometry resident | 40 MB | Streamed from a 340 MB pool |
| Dent-basis panel sets resident | 110 MB | Per-vehicle streaming (§3.4.3) |
| Active deformation heightfields | 6 regions × 512×512 × 2 B = 3.1 MB | §3.4.6 |
| Active structural graphs | 40 subgraphs, 6 MB | |
| Particles from destruction | 120,000 | GPU sim, shared pool with VFX (§5.10.1) |
| Concurrent collapses | 1 | Hard design cap |

### 3.4.6 Soil and mud displacement

The most-used deformation model, so it gets its own specification.

**Representation:** a per-region **deformation heightfield**, 0.25 m resolution, 128 m × 128 m
tiles, 16-bit signed displacement (±2.0 m range, 0.06 mm resolution), with an accompanying
**compaction scalar** (0–1) and a **material override ID** (16-bit, for exposed subsoil, churned
mud, scuffed turf). 6 regions resident (3.1 MB).

**Writers:**
- **Tyres**: per-contact-patch displacement from vertical load, slip (longitudinal ⇒ rutting and
  ejecta; lateral ⇒ scarring), and soil bearing capacity (from `MaterialRecord.permeability` and
  the hydrology model's soil moisture). A loaded logging truck on saturated ground leaves a
  0.18 m rut; a passenger car on dry ground leaves nothing measurable. **Correct.**
- **Tracks**: tracked vehicles displace by a different model (uniform bearing pressure, no rutting,
  but a distinct grousers pattern) — a bulldozer's blade is a **heightfield sculpting tool** with a
  real earthworks volume calculation: cut and fill are conserved, and the sim reports the moved
  volume in m³. Clearing a firebreak (§3.3.5) moves 1,400 m³ and takes 55 minutes, which is why it
  is a real decision rather than a button.
- **Footprints**: biped contact at 0.02–0.06 m depth in soft substrate, 0 in hard. Persist and
  **are readable**: the §3.12.6 investigation system can lift a footprint and match it to a shoe
  record in the ledger (every resident owns shoes with a sole pattern).
- **Projectiles and explosions**: craters. A 40 mm HE round in dry loam ⇒ 0.42 m diameter, 0.18 m
  deep, 0.031 m³ ejecta. Ejecta is conserved and redeposited as a lip.
- **Rain erosion**: the hydrology model's overland flow erodes the deformation heightfield at a
  rate ∝ shear stress, washing out tracks over 4–40 h depending on intensity. **This is a gameplay
  clock**: a crime committed in mud in the valley is traceable for about a day.

**Readers:** terrain collision (the heightfield is a collision modifier), rendering (a
displacement + material override in the terrain shader), navigation cost (§6.3 — ruts slow
vehicles and are avoided by pedestrian cost fields), tyre grip (churned mud has μ = 0.31 vs the
parent soil's 0.44), and track evidence (§3.12.6).

### 3.4.7 Determinism

Destruction `MUST` be deterministic given the same inputs. Rules:
- No `rand()` — a per-event seeded PCG32 with the seed derived from `(impact_id, cell_hash)`.
- Debris body creation order is sorted by a stable key (fragment index, not pointer).
- Fracture-graph edge break order is sorted by (edge_id), never by iteration order of an unordered
  container.
- The debris body cap is enforced by a deterministic priority (distance-to-source, then body_id),
  never by arrival order.
- Every destruction event is written to the delta journal (§5.8.4) as an **event**, not as a
  resulting state, so a replay reconstructs it exactly.

---

## 3.5 Vehicle Dynamics

### 3.5.1 Scope

340 vehicles: 168 passenger cars, 44 light trucks/SUVs, 38 commercial trucks (incl. 3 articulated
with a real fifth-wheel and trailer sway dynamics), 22 motorcycles, 18 emergency, 14 agricultural,
12 construction, 9 marine, 8 rail, 7 rotorwing/fixedwing.

Each is specified as a **data sheet, not a handling curve**. Authoring per vehicle: mass and
inertia tensor, centre of gravity, 4 (or 6/8) wheel hardpoints with full suspension geometry, tyre
specification, powertrain specification (engine map, forced induction, transmission, differential,
driveline inertias), brake specification, aero specification, and a `MaterialRecord` per panel.

**Total authored vehicle data: 41 MB.** Average authoring time per vehicle: 3.1 person-days for a
passenger car, 9.4 for an articulated truck, 22 for a rotorwing.

### 3.5.2 Simulation tiers

| Tier | Vehicles | Rate | Model |
|---|---|---|---|
| **V0 — Hero** | ≤ 2 (player + player-adjacent) | **1,000 Hz** tyre & powertrain, 240 Hz chassis | Full §3.5.3–§3.5.8 |
| **V1 — Near** | ≤ 8 | 500 Hz tyre, 120 Hz chassis | Full model, 4-iteration contact solve |
| **V2 — Traffic** | ≤ 180 | 60 Hz | Simplified: kinematic single-track (bicycle) model + a lateral-load-transfer approximation + the same tyre friction surface sampled at a coarser grid |
| **V3 — Distant** | ≤ 2,400 | 10 Hz | Point-mass along a road-graph edge with speed from the macro traffic model (§3.8.3) |
| **V4 — Abstract** | ~9,000 in-region | 1 Hz | Density in the macro traffic model; no individual state |

**Tier promotion is hysteresis-gated** (V3→V2 at 240 m, V2→V3 at 310 m; V2→V1 at 70 m, V1→V2 at
95 m) and by **relevance** (a vehicle the player is looking at, has targeted, or is on a collision
course with is promoted regardless of distance).

⚠ **HONEST LIMITATION:** V2's bicycle model does not reproduce transient slip dynamics. The
boundary is at 240 m, where a vehicle subtends < 8 px at 1440p and its transient behaviour is
inaudible and invisible. The V1↔V2 handoff interpolates lateral offset, yaw rate, and speed over
0.4 s using a cubic blend so there is no discontinuity in position or heading.

### 3.5.3 Tyre model

**Semi-empirical, Magic-Formula class (Pacejka MF-Tyre 6.2 structure), with combined slip, camber
thrust, load sensitivity, transient relaxation, and a three-node thermal model.**

Pure-slip lateral:
```
F_y = D_y · sin( C_y · arctan( B_y·φ_y − E_y·(B_y·φ_y − arctan(B_y·φ_y)) ) ) + S_Vy
φ_y = α + S_Hy                       (α = slip angle, S_Hy = horizontal shift)
D_y = μ_y · F_z                        (μ_y = μ(T, v, surface, wear, pressure) — see below)
C_y = a1                              (shape factor, ≈1.30 lateral)
B_y = K_yα / (C_y · D_y)               (K_yα = cornering stiffness, N/rad)
E_y = (a5 + a6·dF_z) · (1 + a7·sign(φ_y))
K_yα = F_z · (a3·sin(2·arctan(F_z/(a4·F_z0)))) · (1 + a8·γ²)   # load & camber sensitivity
```
Pure-slip longitudinal: same form with `C_x ≈ 1.65`, slip `κ = (ω·r_e − v_x)/max(v_x, ω·r_e)`.

**Combined slip** via the MF interaction model — an elliptical weighting:
```
σ = √( (κ/σ_x*)² + (tan α/σ_y*)² )
F_x = (κ/σ_x*)/σ · F_combined(σ),   F_y = (tan α/σ_y*)/σ · F_combined(σ)
```
This is what makes a tyre under combined braking-and-turning lose lateral grip — the single most
important feel characteristic in a driving model and the one most games get wrong by treating the
axes independently.

**Friction coefficient** `μ` is a product of five terms, all read from the `MaterialRecord` of the
surface and the tyre state:
```
μ = μ_0(surface, condition) · f_T(T_tread) · f_v(v_slip) · f_p(p, load) · f_w(wear)

f_T: bell curve peaking at T_opt (street: 78 °C; performance: 92 °C; winter: 42 °C),
     μ(20 °C)/μ(T_opt) = 0.86, μ(130 °C)/μ(T_opt) = 0.71, μ(−10 °C)/μ(T_opt) = 0.34 (summer tyre)
f_v: mild decline, μ(v) = μ_0·(1 − 0.0086·ln(1+v/20))
f_p: pressure sensitivity — underinflation increases contact patch length but reduces lateral
     stiffness: μ ∝ (p/p_ref)^0.12 for p < p_ref, ∝ (p/p_ref)^(-0.05) above
f_w: wear state (tread depth 8 mm → 1.6 mm legal min → 0 mm): wet μ falls 0.94 → 0.71 → 0.58 of
     dry-normalised value; dry μ barely changes until the carcass overheats
```
`surface condition` is `dry | wet | icy | muddy | oily | snow_packed | snow_loose | gravel | sand |
leaf_litter | painted_marking | manhole_cover | tram_rail`. Each is an entry in the material record
(§3.1.2 `friction_*`) and the current condition comes from §3.3.3's wetness/puddle fields, §3.3.4's
snowpack, and the road-surface material of the specific lane cell. **A painted zebra crossing at
μ_dry 0.42 vs the asphalt's 0.72 is a real hazard and it is simulated** — braking on the white
lines in the wet is genuinely worse, and players who learn that have learned something true.

**Three-node thermal model** (core / carcass / tread):
```
C_c·dT_c/dt = P_hyst(carcass→core) − Q_amb(core)
C_k·dT_k/dt = P_flex + P_slip·(1−χ) − P_hyst(k→c)
C_t·dT_t/dt = P_slip·χ − P_hyst(t→k) − Q_contact(t→road) − Q_conv(t→air) − Q_spray(t→water)
P_slip = |F_x·v_slip_x| + |F_y·v_slip_y|         (frictional power)
P_flex = 2π·ω·M_rolling_resistance               (hysteresis from rolling resistance)
Q_spray = h_wet · A · (T_t − T_puddle) · min(1, h_water/2.5mm)     # the reason tyres cool in the wet
χ = 0.62 (tread share of slip power)
```
Capacities: C_t ≈ 1,800 J/K, C_k ≈ 5,400 J/K, C_c ≈ 9,200 J/K (an 11 kg tyre). A hard 6-lap
circuit run takes tread temperature from 18 °C to 96 °C in about 4 minutes — the correct timescale.

**Transient relaxation**: slip force does not appear instantaneously; the contact patch has a
relaxation length σ ≈ 0.3 m. Modelled as a first-order lag on the slip state:
`dφ/ds = (φ_demand − φ)/σ`. At 30 m/s this is a 10 ms lag — small, but it is what produces the
correct yaw-response phase and the absence of the "on-rails" feeling of instantaneous-slip models.

**Alignment torque and self-aligning behaviour**: `M_z` from the same magic formula with a separate
set of coefficients plus a pneumatic trail term. This drives steering-wheel feel (force feedback on
supported wheels) and the "steering goes light" cue at aquaplaning onset (§3.3.3).

**Cost:** 4 tyres × (18 magic-formula evaluations + 3 thermal nodes + relaxation) = ~2,400 flops
per wheel per substep. At 1,000 Hz for 2 hero vehicles: 19.2 MFLOP/s. **Negligible.** The
expensive part is not the tyre model; it is the **contact/constraint solve** (§3.5.7).

### 3.5.4 Suspension geometry (kinematic, not raycast)

`MUST NOT` use raycast "fake" suspension. Each wheel's position is solved from actual hardpoints.

**Per-wheel hardpoint set** (17 points for a double-wishbone front, 13 for a MacPherson strut,
19 for a multilink rear, 11 for a solid axle with Panhard/leaf):
```
chassis_frame:  upper_front, upper_rear, lower_front, lower_rear, pushrod_chassis,
                rocker_pivot, damper_chassis, spring_chassis, tie_rod_inner,
                antirollbar_chassis, bumpstop_top
unsprung:       upright_top, upright_bottom, hub_centre, tie_rod_outer, pushrod_wheel,
                antirollbar_link, bumpstop_bottom
```
**Solver:** at each 500 Hz substep, solve the linkage constraint system
`C(q) = 0` (13–19 constraints, 6·n_bodies unknowns) by **Newton-Raphson with 3 iterations**,
warm-started from the previous substep. Converged residual < 1e-5 m.

**Derived quantities, all real and all affecting behaviour:**
- Camber, toe, caster, kingpin inclination, scrub radius — as *functions of travel*, not constants.
- **Camber curve**: a double wishbone designed for −0.8°/100 mm of travel gains negative camber in
  compression, which is what keeps the contact patch flat in a corner. A MacPherson strut gains
  positive camber, which is why it understeers harder at the limit. The difference is simulated and
  the player feels it without knowing why.
- **Bump steer**: toe change with travel, from the tie-rod's instant centre mismatch. A worn or
  badly-designed suspension steers itself over bumps.
- **Roll centre height** and its migration with travel → **jacking forces** and roll-steer.
- **Anti-dive %** and **anti-squat %** from the instant-centre geometry → how much load transfer is
  reacted geometrically vs through the springs.
- **Ackermann error** at large steer angles.
- **Bushing compliance**: each bushing is a 6×6 stiffness/damping matrix. Under load the suspension
  *deflects*, producing compliance steer (the dominant cause of passive rear-axle toe-in under
  cornering load, i.e. real rear-axle stability).
- **Damper curves**: separate compression and rebound, each with low-speed and high-speed regimes
  (a 4-parameter curve: `F = c1·v + c2·v² ` below the knee, `F = F_knee + c3·(v − v_knee) + c4·
  (v−v_knee)²` above), plus **fade**: damper oil temperature rises with dissipated power
  (`P = F·v`), viscosity falls as `μ(T) = μ_ref·exp(−0.021·(T − T_ref))`, and the curve scales
  down. A car descending 14 km of Sable Pass switchbacks on its brakes loses 22% of its damping by
  the bottom. **This is the kind of detail that makes a mountain road feel like a mountain road.**
- **Progressive bump stops**: `F = k_bs · max(0, x − x_contact)^1.8`, with a gas-filled
  hydro-pneumatic variant for trucks (a real, distinctly different feel).
- **Anti-roll bars**: torsional, with a rate and a lever ratio; **active** anti-roll bars on 4
  vehicles (hydraulic, 1,400 Nm, 20 Hz control loop) as a genuine handling differentiator.
- **Air suspension** on 11 vehicles: variable spring rate and ride height with a compressor,
  reservoir, and a 40 s height-change time — and a **failure mode** (a leaking bag, which the
  player can acquire by damaging a corner).

**Cost:** 4 wheels × (19 constraints × 3 Newton iterations) ≈ 3,200 flops/substep at 500 Hz for
V0/V1 ⇒ 1.6 MFLOP/s per vehicle, 16 MFLOP/s for 10 vehicles. Fine. The Newton solves are
cache-friendly and vectorisable across wheels (SoA layout, §5.2.3).

### 3.5.5 Powertrain

**Engine:** a **mapped torque surface** `T_map(RPM, throttle)` (a 64×32 table, bilinear) with
real-time physical corrections — not a quasi-dimensional cycle simulation, which would cost 100×
more for no perceptible benefit.

```
T_actual = T_map(RPM, θ) · ρ_ratio · f_knock · f_temp · f_transient · f_altitude
  ρ_ratio   = ρ_intake / ρ_ref,  ρ_intake from §3.2 (T, p, RH) → a hot day at Anza Basin
              (1,050 m, 34 °C, 88 kPa) costs a naturally aspirated engine 12.4% of its power.
              At Sable Pass (1,612 m) it costs 17.1%. This is why the valley truck struggles
              on the pass and the turbo car does not.
  f_knock   = 1 − 0.32·max(0, K − K_limit)/K_limit ; K = knock index from compression ratio,
              intake T, cylinder pressure, and fuel octane (RON 87/91/95/100 in the region's
              4 fuel grades). Knock pulls timing; sustained knock damages the engine (a ledger
              entry on the vehicle record).
  f_temp    = cold-start enrichment & friction: 0.82 at −10 °C, 1.00 at 88 °C coolant,
              0.94 at 118 °C (overheat derate)
  f_transient = throttle-rate enrichment/deceleration (a 40 ms first-order term)
```

**Forced induction** (38% of the fleet): a **shaft-dynamics turbo model**.
```
J_tc · dω_tc/dt = P_turb − P_comp − P_bearing
P_turb = ṁ_exh · c_p · T_exh_in · η_t · (1 − (p_amb/p_exh)^((γ−1)/γ))
P_comp = ṁ_air · c_p · T_amb · ((p_boost/p_amb)^((γ−1)/γ) − 1) / η_c
J_tc ≈ 1.1e-5 kg·m²   ⇒  spool time constant 0.28–1.15 s depending on turbine A/R
wastegate: PID on p_boost, with an actuator lag of 80 ms
intercooler: ε = 0.62–0.78 effectiveness, with HEAT SOAK — ΔT_rise accumulates with charge air
              mass flow and decays with ram air; 4 hard launches raise intake T by 26 °C and cost
              8.1% power, recovering over 90 s. A real and perceivable effect.
```
Also modelled: **turbo flutter/surge** on abrupt throttle lift (an audio event driven by the
compressor operating point crossing the surge line), **lag** after a gear change (the operating
point moves off the map), and **altitude compensation** (a wastegate turbo over-boosts at altitude
until the compressor hits its speed limit).

**Clutch:** torque capacity `T_c = μ_c(p, T) · R_eff · F_clamp(pedal)` with a friction-vs-slip
curve that has a negative-slope region (the source of clutch judder and of the ability to stall),
temperature from dissipated power `P = T_c · Δω`, and a wear state that reduces clamp load. A
burnout produces real fade, real smoke (a particle event tied to the temperature), and real
permanent wear written to the vehicle record.

**Gearbox:** ratios, final drive, shift time (manual 280–620 ms with a driver-skill modifier;
torque-converter automatic 180–420 ms with a lock-up strategy; DCT 40–110 ms with pre-selection;
CVT a continuous ratio with a belt-torque limit), torque interruption during the shift, and
**shift logic**: an automatic's controller selects gears from a 3-input map (throttle, speed,
brake) with adaptive learning — it holds a gear downhill under engine braking, which is the
behaviour a real transmission has and which games almost never reproduce.

**Differential:** open (torque to the lower-μ side — the reason one-wheel-spin happens on a snowy
shoulder), limited-slip (preload torque + ramp angles for power and coast, e.g. 40 N·m preload,
45° power ramp, 30° coast ramp → a locking torque curve), locking (driver-selected), viscous (a
shear-rate-dependent coupling with a thermal state), and electronic (a clutch pack with a 40 ms
actuation and a torque-vectoring controller that reads yaw-rate error). Full-time 4WD with a
centre differential and a transfer case (high/low range, 2.72:1 low) on 22 vehicles.

**Driveline inertia and torsion:** shafts have inertia and finite stiffness, producing **shuffle**
(2–6 Hz) and **clunk** (a 40 ms transient) on tip-in/tip-out. Modelled as a 3-inertia torsional
system (engine – gearbox/halfshaft – wheel) integrated at 1,000 Hz. This is the single highest-
value addition to powertrain feel after the tyre model and it costs 600 flops.

**Cooling system:** radiator (effectiveness vs airflow, which depends on vehicle speed and cooling-
fan state), coolant loop thermal mass (14 L, 4.18 kJ/kg·K ⇒ 58 kJ/K), oil cooler, and an
**electric fan with a thermostat**. Overheating is a real failure state: a blocked radiator
(debris, a collision that bends the core support) causes a slow temperature rise, a warning at
112 °C, a limp mode at 118 °C (fuel cut on 2 cylinders), and head-gasket failure at 126 °C
— written permanently to the vehicle record. Towing a trailer up Sable Pass at 34 °C ambient in a
vehicle with a marginal cooling system is a **planning problem**, and the sim makes it one.

### 3.5.6 Brake system

```
Per axle:
  E_dissipated = ∫ F_brake · v dt        (split front/rear by a bias + ABS + load-transfer model)
  T_rotor:  C_r·dT_r/dt = P_in − h_conv·A_r·(T_r − T_air) − h_cond·(T_r − T_hub) − Q_water_spray
    C_r = m_rotor · c_p = 8.4 kg · 460 J/(kg·K) = 3,864 J/K   (cast iron vented disc, front)
    h_conv = 12 + 0.62·√v  W/(m²·K)   (forced convection on a rotating disc), A_r ≈ 0.42 m²
  T_pad:    separate node, coupled to the rotor by conduction at the interface
  T_fluid:  caliper → hose → line, with a 40 s lag; boiling point from the fluid grade
            (DOT 3: 205/140 °C dry/wet; DOT 4: 230/155; DOT 5.1: 260/180; the region's
             vehicles have a fluid grade and an age-derived moisture content that lowers the
             wet boiling point over time — a maintenance state in the vehicle record)
  μ_pad(T): street organic  μ peaks at 220 °C (0.42), fades to 0.19 at 520 °C
            semi-metallic   μ peaks at 340 °C (0.44), fades to 0.26 at 680 °C
            ceramic         μ flat 0.40–0.46 from 100 to 800 °C
```

**Validation figure:** stopping a 1,650 kg vehicle from 200 km/h dissipates 2.55 MJ. With a 62%
front bias, each front rotor absorbs 0.79 MJ ⇒ **ΔT = 205 K** on an 8.4 kg cast-iron disc. Four
consecutive stops from 200 km/h with 20 s between them puts the front rotors at ~610 °C, the pads
into fade, and the fluid at risk of boiling. **This is measured and correct**, and it means a
player who brakes hard down Sable Pass in a car with street pads experiences genuine, progressive
brake fade with a soft pedal and increasing stopping distance — the exact failure that kills people
on real mountain roads, and the reason the pass has 6 runaway-truck ramps (which exist in the map,
are signed, and work).

**ABS:** a real controller. Per-wheel slip-ratio target (λ* ≈ 0.12 for dry asphalt, 0.22 for
loose gravel, 0.08 for ice — the controller does not know which it is on, which is why ABS on
gravel *increases* stopping distance, as it does in reality). Hydraulic modulation at 15 Hz with a
pump-back, an accumulator, and a pressure-reduction phase. Audible and tactile (pedal pulsation on
supported force-feedback devices).

**ESC / stability control:** a yaw-rate and sideslip reference model (`r_ref = v·δ/(L + K·v²)`),
error → selective single-wheel braking with a torque request, plus engine torque intervention.
Three-mode ladder per §3.5.9.

### 3.5.7 Contact and constraint solving

The chassis is a rigid body (or 3 bodies for an articulated truck: tractor, trailer, and a virtual
fifth-wheel coupling). Contacts: 4–18 tyre patches, plus chassis/bumper/suspension-member contacts
with the world.

**Solver:** Sequential Impulses with **warm starting**, 240 Hz for V0/V1, 4–8 iterations (adaptive:
iterations increase while the max residual exceeds a threshold, capped at 8). Continuous collision
detection (conservative advancement, 2 substeps max) for any body exceeding 42 m/s or for
wheel-to-kerb contacts at high speed — the case that produces tunneling through a barrier at
250 km/h if you don't.

**Tyre contact patch**: not a point. The patch is a **rectangle computed from load and pressure**
(length from `L = √(2·R·δ)` where δ is deflection; width from the tyre section), subdivided into
8 longitudinal × 3 lateral elements, each with its own slip, μ, and pressure (a parabolic-ish
distribution with a wear-driven modifier). This is what produces the difference between a
hard-cornering front tyre's inner-edge wear and its outer-edge wear, and the **progressive** loss of
grip as the patch saturates from one edge inward — the feel characteristic that separates a real
model from a magic formula with a point contact.

### 3.5.8 Aerodynamics

```
F_drag  = ½·ρ·C_d·A·v_rel²          v_rel = vehicle velocity MINUS the local wind vector (§3.2)
F_lift  = ½·ρ·C_l·A·v_rel²          C_l typically +0.28 (front) / +0.14 (rear) for a sedan —
                                    i.e. it lifts, and at 250 km/h a sedan loses 210 N of front
                                    axle load, measurably lightening the steering
F_side  = ½·ρ·C_s·A_side·v_wind²    with a yaw-dependent C_s table
M_yaw   = from the side-force centroid offset — the CROSSWIND MOMENT
```
**Crosswind:** the wind vector comes from §3.2 including modifier M2 (ridge speedup) and M4
(canyon channelling). Crossing the Vermillion Bay Bridge (2,860 m, +78 m deck, exposed) in a
foehn event with a 22 m/s crosswind produces a 1,340 N side force on a boxy van and a real
yaw moment the player must correct for. The bridge has wind screens over 1,180 m of its length,
and the sim knows it — behind a screen the side force drops 68%. **This is authored world detail
that only exists because the systems are coupled.**

**Drafting/slipstream:** vehicles within 30 m and ±2.5 m lateral of a leader experience reduced
`C_d·A` (up to −42% at 8 m for a car behind a truck) and a lateral "suction" moment when pulling
alongside. Implemented as an analytical wake model (a velocity deficit cone with a Gaussian lateral
profile), not a CFD solve. Cost: 0.9 µs per vehicle pair within range, capped at 40 pairs.

**Dirty air:** the same wake model applies to rotorcraft downwash (a real hazard for a following
helicopter and for ground personnel) and to a motorcycle following a truck (buffeting from a
turbulence-intensity term, which produces a visible and audible rider instability).

### 3.5.9 The assist ladder

Three levels, per Pillar "authenticity with accessibility". **The assist changes only the input
and the control layer, never the underlying model.** This is non-negotiable: an assist that
substitutes a simpler model produces a vehicle that feels different, and players who switch
difficulty notice immediately.

| Level | What it does | What it does NOT do |
|---|---|---|
| **0 — Authentic** | Nothing. Full model, no intervention. ESC off, ABS off if the vehicle is pre-1998 (era-correct), no traction control. | — |
| **1 — Assisted** | ESC active at full authority; ABS active; a low-pass filter on steering input (τ = 28 ms); a torque-limiting traction control; brake-force blending to prevent rear lock-up under heavy pedal. | Does not reduce engine power, does not change tyre μ, does not reduce brake fade, does not prevent aquaplaning. |
| **2 — Guided** | Level 1 plus: an automatic counter-steer assist bounded at 30% of the required correction, speed-sensitive steering-rate limiting, and a "stability envelope" that reduces throttle request near the limit (not torque — the driver's request). | Does not prevent all spins. Does not change the physics. Does not remove consequences of a crash. |

Also era-correct: a 1974 vehicle has no ABS, no ESC, no crumple zones designed to modern standards
(a real crash-severity difference in §3.5.10), bias-ply tyres (a distinctly worse transient
response), a carburettor (which floods on a steep descent and stalls — a real failure mode on Sable
Pass), and a manual choke.

### 3.5.10 Crashworthiness and occupant injury

Collisions are resolved by the physics solver; the **occupant model** is a separate system.

```
Per occupant (player + NPC):
  Δv of the occupant's compartment over the crash pulse (from the solver, 1 kHz sampled)
  Crash pulse shape from the impacted structure: material class, mass, crush depth, and
    the structural graph's energy absorption (§3.4.4)
  Restraint state: belt on/off, belt pretensioner fired, airbag deployed (a 12–22 ms event with
    a real deployment threshold of Δv > 18 km/h frontal / 24 km/h offset)
  Occupant kinematics: a 3-segment (head/torso/pelvis) lumped-mass model with belt and bag
    interaction, integrated over the 120 ms crash pulse
  Injury risk: AIS-level probability curves vs chest acceleration, head impact criterion (HIC15),
    femur load, neck tension/moment
```
Outputs: an **injury set** on the resident record (fracture, laceration, concussion, internal
haemorrhage, amputation, death) with recovery timelines that the ledger tracks — a hospitalised
resident is absent from their schedule for 3 days to 14 weeks, their employer logs the absence,
their household income drops, their insurance claim enters the economy model, and a police report
is filed (§3.12.6) with a real collision-reconstruction record (skid marks from the soil deform
layer, vehicle rest positions, debris field, EDR "black box" data from the vehicle record).

**This is the darkest system in the game and it is specified because it is required for Pillar 3.**
If crashing has no consequence beyond a respawn, driving is a video game. If crashing has a
consequence that the world records and that people in the world are affected by, driving is a
decision. Rating exposure is addressed in §6.6.3; the *design* decision is that injury is not
shown graphically in detail — it is reported, felt in the player's own embodiment state (§3.14),
and consequential.

**Player death:** permitted, and handled by §5.8.4 (world-state-preserving respawn at a hospital
with a real injury recovery period, a real medical bill in the economy model, and a real gap in the
ledger where the player's life continued without them). No "wasted" screen. No time rewind.

### 3.5.11 Vehicle budget

| Item | Cost |
|---|---|
| V0 hero (2 vehicles) full model | 1.14 ms CPU (1,000 Hz substeps amortised over a 16.67 ms frame = 16 substeps × 71 µs) |
| V1 near (8 vehicles) | 0.42 ms CPU |
| V2 traffic (180 vehicles, bicycle model) | 0.28 ms CPU |
| V3 distant (2,400, point mass) | 0.06 ms CPU |
| Chassis contact solve (V0+V1) | 0.31 ms CPU |
| **Total vehicle systems** | **2.21 ms CPU across 2 physics workers** |

Within §5.11.1's 2.4 ms physics budget across 2 threads ⇒ 1.2 ms/thread. **2.21/2 = 1.11 ms/thread.**
Passes with 8% headroom. This budget is the reason V2's cap is 180 and not 400 (§5.11.3 governor
drops it to 120 then 80 under pressure).

---

## 3.6 Ballistics & Terminal Effects

### 3.6.1 Weapon catalogue

96 weapons: 34 pistols/revolvers, 9 SMGs, 14 rifles, 8 shotguns, 6 DMRs, 5 sniper, 4 machine guns,
3 grenade launchers, 2 RPG, 4 melee (ballistic-adjacent: thrown), 7 non-lethal (taser, beanbag,
tear gas, baton). Each specified as data:

```yaml
WeaponRecord:
  id: wpn_rifle_762x51_m40a1_analog
  cartridge: cart_762x51_m80
  action: gas_piston_short_stroke          # affects cycling time, fouling, malfunctions
  barrel_length_mm: 508
  twist_rate_mm: 305                       # 1:12" — stabilises up to ~11 g projectiles
  muzzle_velocity_ms: 838                  # ±1.4% shot-to-shot (σ), from a barrel-length curve
  muzzle_energy_J: 3340
  rate_of_fire_rpm: 700
  recoil:
    impulse_Ns: 8.9
    direction_deg: {yaw_mean: 0.4, yaw_sigma: 1.8, pitch_mean: 62, pitch_sigma: 4.1}
    recovery_time_ms: 92
  malfunctions:
    rounds_between_stoppages: 4200         # MRBS
    types: {fte: 0.34, ftf: 0.11, double_feed: 0.06, stovepipe: 0.19, case_sep: 0.02,
            out_of_battery: 0.08, magazine_fault: 0.20}
    modifiers: {dirty: ×2.4, cold: ×1.3, sand: ×6.1, wet: ×1.8, damaged_mag: ×3.2}
  traceability: 0.61                       # probability that a fired case + striation match links
                                           # to this specific weapon in §3.12.6 forensics
  optics: [mount_picatinny]
  suppressor_compatible: true
  suppressor_delta: {velocity_ms: +18, sound_dB: −26, flash: −0.9, report_signature: 'subsonic_crack_only'}
```

**Malfunctions are simulated** because a weapon that never jams is a toy. The rate is era- and
maintenance-correct: a well-maintained modern rifle has an MRBS of 4,200; a 1960s surplus rifle in
Anza Basin dust has an effective MRBS of ~690. Cleaning is an action; neglect is a state on the
weapon record.

### 3.6.2 External ballistics

**Point-mass with a standard drag function (G1 or G7), integrated at 2 kHz.**

```
State: x, v (3-vectors), t, spin (for drift)
d v/dt = −(ρ_air/(2·m)) · C_d(v_mach) · A · |v| · v  +  g  +  a_wind  +  a_magnus  +  a_coriolis

Re-expressed with the ballistic coefficient:
  retardation = (ρ/ρ_0) · G(v_mach) / BC
  BC = SD / i,   SD = m/d²  (sectional density),  i = form factor
  G(v) = the G1 or G7 standard drag function (tabulated, 96 Mach points, cubic interpolation)

ρ_air = p/(R_d·T_v),  R_d = 287.05 J/(kg·K),  T_v = virtual temperature (humidity-corrected)
  → sampled from §3.2 at each integration step. A 900 m shot from Sable Ridge (1,840 m ASL,
    81 kPa, 4 °C) into the valley has 18.4% less drag than the same shot at sea level, and the
    trajectory is visibly flatter. Snipers in the game read a ballistic calculator that reports
    this, because it is real.
```

**Aerodynamic jump and spin drift:** a right-hand-twist barrel imparts spin; the gyroscopic
precession under gravity-induced yaw produces a **spin drift** of 0.14–0.42 m at 800 m for a
7.62 mm — small, but it is the difference between a hit and a miss at that range and it is the
detail that makes long-range shooting a *skill* rather than a *lookup*. Also: **yaw of repose**,
**Coriolis** (0.06–0.18 m at 800 m, direction-dependent — included because it is 40 flops), and
**crosswind deferral** (the wind's effect is concentrated in the first third of flight because the
bullet has less time to be deflected later; a 4.5 m/s crosswind gives 0.31 m of drift at 500 m for
a .308-class round and 1.94 m at 1,000 m).

**Validation table (target ±2% against published retardation data):**

| Round | MV (m/s) | BC (G1) | Drop @ 300 m (m, 100 m zero) | Drop @ 800 m (m) | TOF @ 800 m (s) | Drift @ 800 m, 4.5 m/s (m) |
|---|---|---|---|---|---|---|
| 9×19 FMJ 8.0 g | 360 | 0.142 | 0.86 | 12.4 | 2.16 | 3.9 |
| 5.56×45 M855 4.0 g | 940 | 0.304 | 0.42 | 5.1 | 1.28 | 1.6 |
| 7.62×39 7.9 g | 730 | 0.275 | 0.71 | 8.4 | 1.52 | 2.6 |
| 7.62×51 M80 9.5 g | 838 | 0.404 | 0.62 | 6.9 | 1.34 | 2.1 |
| .338 Lapua 16.2 g | 880 | 0.640 (G7 0.322) | 0.44 | 4.6 | 1.13 | 1.3 |
| 12 ga slug 28.4 g | 480 | 0.060 | 1.31 | — (max eff. 120 m) | — | — |

### 3.6.3 Interior ballistics (why muzzle velocity varies)

Barrel-length curve per cartridge (a 5.56 mm round from a 508 mm barrel gives 940 m/s; from a
267 mm SBR, 812 m/s, with a 62% increase in muzzle flash and a 9 dB increase in report — a real
and audible consequence). Chamber pressure curve (a 3-point Pieper fit), propellant burn rate,
case volume, and **temperature sensitivity** (a cartridge at −12 °C on Sable Ridge gives 3.1% lower
velocity than at +24 °C, moving the 800 m point of impact 0.21 m). Ammunition lot variance
(σ = 1.4% on velocity).

**Why include this:** because it makes loadout a *decision* with observable consequences, and it
costs nothing at runtime (all precomputed tables).

### 3.6.4 Terminal ballistics and penetration

```
On each surface intersection (from the ray/trace, up to 12 per projectile):
  1. Read LayerStack(position, normal) (§3.4.2)
  2. For each layer, front to back:
       θ = angle of incidence from the layer normal
       R_layer = resistance_curve.normal_0deg_mm_equiv × (thickness/200)^0.5 / cos(θ)^n
       E_required = R_layer × f(projectile_type, v_impact, mass, diameter, construction)
       if E_residual < E_required:
            if θ > critical_ricochet_deg[type]:  → RICOCHET
                 new direction = reflect(v, n) with a randomised perturbation from the surface
                 roughness; residual velocity = |v|·(0.34–0.71) by material class
                 the projectile is now UNSTABLE (yaw 8–40°) and its drag coefficient rises 2.4×
                 → a ricochet is a genuinely unpredictable, low-energy, tumbling hazard
            else: → STOP (embed); transfer momentum to the struck body; spawn a deformation event
       else: → PENETRATE
            E_residual -= E_required × (1 + 0.18·(behind_spall_energy/E_required))
            spawn the layer's deformation model (§3.4.3) with the absorbed energy
            spawn the behind-face spall cone if the layer is brittle (§3.4.3 model 2)
            reduce projectile mass by the fragmentation fraction (HP: 0.42, FMJ: 0.06, AP: 0.00)
            reduce velocity by the momentum-conserving residual
            the projectile may now be sub-fragmenting: spawn 2–9 secondary fragments for HP
  3. Terminal: if the target is a body, run §3.6.5
```

**Penetration reference matrix** (calibrated to published NIJ 0101.06 and MIL-STD test data where
available; design-approved feel curve where not — stated explicitly so nobody mistakes the table
for a physics claim):

| Barrier | 9×19 FMJ | 5.56 M855 | 7.62×51 M80 | 12 ga 00 buck | .50 BMG |
|---|---|---|---|---|---|
| 13 mm gypsum drywall (single leaf) | pen, minor loss | pen | pen | pen, pattern opens | pen |
| 2× 13 mm drywall + 90 mm stud cavity | pen | pen | pen | 1–3 pellets pen | pen |
| 100 mm clay brick | **stop @ > 30° obliquity** | pen @ < 15°, ≤ 40 m | pen | stop | pen |
| 200 mm cast concrete | stop | stop @ > 60 m | stop @ > 120 m | stop | **pen** |
| 6 mm mild steel plate | stop | pen @ ≤ 100 m | pen @ ≤ 300 m | stop | pen |
| 12 mm AR500 armour plate | stop | stop | stop | stop | stop @ > 200 m |
| 6 mm automotive glass (laminated windscreen) | pen, deflected 4–11° | pen | pen | **stop** (pellets flatten) | pen |
| 4 mm tempered side glass | pen | pen | pen | pen | pen |
| Car door (0.8 mm skin + trim + 1.2 mm beam) | pen (beam may stop) | pen | pen | stop at beam | pen |
| Engine block (cast Al, 14 mm) | stop | pen @ < 150 m | pen | stop | pen |
| 300 mm oak trunk | stop @ > 0.4 m penetration depth | pen to 0.6 m | pen to 0.9 m | stop @ 0.15 m | pen through |
| NIJ IIIA soft armour | stop | **pen** | **pen** | stop (blunt trauma: AIS 2–3) | pen |
| NIJ III hard plate | n/a | stop (blunt trauma) | stop | stop | pen |
| NIJ IV ceramic | n/a | stop | stop | stop | stop |

**Behind-armour blunt trauma** is modelled: a stopped round still delivers momentum, producing a
backface deformation depth that maps to an injury probability. This is why body armour is not
invincibility, which is a real and important property.

### 3.6.5 Terminal effects on bodies

⚠ **Design decision stated plainly:** we model trauma **physiologically**, not visually. There is no
dismemberment system, no gore slider as a gameplay mechanic, and no lingering on injury. The model
exists so that combat has **graduated, legible consequences** — Pillar 3 — and so that armour
choice, calibre choice, and shot placement are real decisions.

```
Body segmentation: head, neck, thorax (with lung/heart/great-vessel subregions), abdomen (with
  solid-organ and hollow-viscus subregions), pelvis, ×2 arms (upper/forearm/hand), ×2 legs
  (thigh/lower leg/foot). 22 segments, each with: tissue layer stack, bone presence and strength,
  major-vessel presence, and a vital-organ flag.

On hit:
  E_deposited = ½·m·(v_in² − v_out²)   (v_out = 0 for a non-penetrating hit)
  Wound channel volume ≈ E_deposited / (tissue_specific_energy)   — a permanent cavity estimate
  Temporary cavity ≈ 3.1× permanent for rifle velocities (> 600 m/s), ≈ 1.2× for handgun
    → the temporary cavity is what causes the difference between a rifle and a pistol wound,
      and it is only significant above ~600 m/s. Below that, permanent cavity dominates.
      This is the actual ballistic literature and it is why "hydrostatic shock" is not modelled
      as a magic pistol effect.
  Bone: if E_deposited/segment > bone fracture threshold → fracture, plus 2–9 secondary
    fragments from the bone acting as a projectile (a real and significant wounding mechanism)
  Vessel: if a major vessel segment is lacerated → haemorrhage rate 0.9–3.4 L/min depending on
    the vessel; otherwise capillary/venous bleeding at 0.02–0.35 L/min, modified by
    arterial pressure and by clotting (a 6-minute onset)

Physiological state (per agent, integrated at 1 Hz):
  blood_volume (5.0 L adult baseline)
  consciousness:  from cerebral perfusion pressure; loss at ~30% blood volume or a head hit
                  above threshold; a graded GCS-analogue score
  shock_index:    HR/SBP
  mobility:       from fracture locations and pain
  pain:           a scalar that degrades aim, sprint, and vault performance (§3.14)
  incapacitation: a state with a timer derived from the injury set
  recovery:       hours-to-weeks, written to the ResidentRecord
```

**Consequence chain:** a shot wound is not a hit marker. It produces an injury that the ledger
tracks; the injured party goes to hospital (a Class S interior with a real ED workflow, staffed by
ledger residents on shift); an ambulance is dispatched under the same CAD as police (§3.12.5) with
a separate priority scheme; a police report is filed; if the player caused it, evidence exists
(bullet, casing with firing-pin impression ⇒ traceability 0.61, blood DNA, CCTV); and the injured
person's household, employer, and social graph all register the event. **One shot can generate 40
downstream ledger entries.** That is the game.

### 3.6.6 Ballistics budget

| Item | Cost |
|---|---|
| Projectiles in flight (cap 240) at 2 kHz | 0.24 ms CPU/frame amortised |
| Penetration resolves (cap 96/frame, avg 3 layers) | 0.11 ms CPU |
| Terminal effects | 0.04 ms CPU |
| Tracer/visual (GPU) | 0.05 ms GPU |
| **Total** | **0.39 ms CPU, 0.05 ms GPU** |

**Note on hitscan:** `MUST NOT` be used. Every projectile is a simulated body with a 2 kHz
trajectory. At 60 FPS that is 33 integration steps per frame — a 900 m/s round crosses 15 m per
frame, and hitscan would let it pass through a 200 mm wall it should have stopped at, or miss a
1.8 m tall pedestrian it should have hit. Sub-frame collision is done by **swept segment tests
against the BVH** (§5.5.4), not by point sampling. Cost is in the BVH traversal, budgeted above.

---

## 3.7 Audio: Ray-Traced Spatial Acoustics

### 3.7.1 Architecture

A four-stage hybrid, because pure runtime ray tracing cannot produce low-frequency diffraction and
pure baking cannot respond to a wall the player just demolished.

```
STAGE 1 — OFFLINE PROBE BAKE (wave acoustics, low frequency)
   FDTD (finite-difference time-domain) solve on a 60 mm grid per interior and per street canyon,
   1,024 source positions, 40,000 timesteps → impulse responses → extracted per probe:
     · 6-octave-band RT60 (125 Hz – 4 kHz)
     · edge-diffraction kernel per portal (doors, windows, corridor mouths) — this is the part
       geometric acoustics cannot do, and it is what makes sound bend around a corner
     · SH-order-3 directional encoding of early-reflection energy
   Probe lattice: 3.0 m in interiors, 8.0 m in street canyons, 24 m outdoors.
   1.94M probes region-wide, 4.2 KB each = 8.1 GB on disk, 240 MB resident (streamed per cell).
   Bake cost: 340 core-hours per 1,000 interiors → 12,200 core-hours total (§6.8, R-09).

STAGE 2 — RUNTIME GEOMETRIC TRACE (mid/high frequency, 15 Hz)
   For each of ≤ 24 "acoustically significant" sources:
     · 12 specular reflection rays against the same BVH used for GI (§5.7.2)
     · 4 diffraction probes (the nearest baked portal edges with an unoccluded path)
     · 1 direct-path occlusion ray → transmission loss through the LayerStack
   Per bounce: path length ⇒ delay; Σ material absorption ⇒ per-band gain; scattering coeff ⇒
   a diffusion term fed to the reverb tail
   Cost: 24 × 17 rays × 15 Hz = 6,120 rays/s. BVH traversal 4.1 µs/ray ⇒ 0.025 ms CPU. Trivial.
   The DSP is the cost, not the trace.

STAGE 3 — SYNTHESIS
   · Direct path: HRTF-convolved (SOFA dataset, 128-tap FIR, 2 per voice), with ITD and ILD
   · Early reflections (≤ 8, from Stage 2): discrete delayed+filtered+spatialised taps
   · Late reverberation: a 12-delay-line FDN with a Hadamard mixing matrix, per-band absorption
     one-pole filters, feedback gain g_b = 10^(−3·T_line/RT60_b) driven by Stage 1/2 RT60
   · Convolution reverb: only for 3 "hero spaces" concurrently (Cathedral of St Brendan,
     Vermillion Central concourse, the dry dock), partitioned IR with 2,048-sample blocks,
     GPU FFT. IR length 4.8 s (cathedral RT60 = 6.2 s at 500 Hz).

STAGE 4 — MIX & OUTPUT
   48 kHz, 512 concurrent voices (96 with the full DSP chain, 416 with a 2-band EQ + gain +
   simple spatialiser), 4-band parametric per bus, 14 buses, loudness-normalised to
   ITU-R BS.1770-4 with a −23 LUFS dialogue anchor, dynamic range modes
   (Full / Night / Dialogue-Boost) as an accessibility requirement.
```

### 3.7.2 Occlusion and transmission

Occlusion is **not** a binary. The direct-path ray hits the BVH; the hit yields a `LayerStack`
(§3.4.2); transmission loss is computed from the stack including **mass-air-mass resonance** for
cavity assemblies:

```
Single leaf (mass law):    TL(f) = 20·log10(m″·f) − 47                    [m″ = kg/m²]
Double leaf (cavity):      TL(f) = TL_1 + TL_2 + 20·log10(f/f_0) − correction
                           f_0 = (1/2π)·√(s·(1/m″_1 + 1/m″_2)),  s = cavity stiffness
Flanking:                  add a flanking path loss term from the baked probe (real buildings
                           leak around partitions; this is why an STC 56 wall gives you 48 dB
                           of privacy, not 56)
```
Applied per octave band ⇒ a low-frequency thump through a wall is much louder relative to a
high-frequency hiss, which is the single most important perceptual cue for "there is someone in the
next room". **Bass through walls** is the reason this system exists.

Real values, verified against the material database:

| Assembly | STC | Audible effect at 1 m source, 90 dB SPL |
|---|---|---|
| 13 mm gypsum on 90 mm studs (single leaf) | 33 | Normal speech audible and intelligible |
| 2× 13 mm gypsum on 90 mm studs + mineral wool | 50 | Speech audible, not intelligible |
| 100 mm clay brick | 45 | Loud speech faintly audible |
| 200 mm cast concrete | 56 | Very loud speech barely audible; a door slam is a thump |
| 6 mm annealed glazing | 31 | Speech fully audible |
| 24 mm laminated IGU | 42 | Speech audible, muffled |
| Vehicle door (skin + trim + beam) | 28 | Conversation outside is clearly audible inside |
| Solid-core timber door, 44 mm | 32 | — |
| Rated fire door, steel, 60 min | 44 | — |

### 3.7.3 Dynamic reverb from room geometry

RT60 is **not** a per-room authored value. It is computed at runtime from the room the listener is
in:

```
V = room volume (from the interior cell's baked volume, or from a runtime flood-fill for
    destroyed/modified geometry)
S_i, α_i = surface areas and absorption coefficients per material per octave band
A_b = Σ S_i · α_i  +  Σ (occupant_absorption × n_occupants)   ← people absorb ~0.4 sabins each
                        + furniture_absorption (from the set-dressing manifest, §4.6)

Sabine:    RT60_b = 0.161 · V / A_b
Eyring:    RT60_b = 0.161 · V / (−S_total · ln(1 − ᾱ_b))     ← used when ᾱ > 0.2 (correct branch)
```
Then the FDN's 12 per-band feedback gains are set from `RT60_b`. **A crowd changes a room's
acoustics.** An empty nightclub and the same nightclub with 340 people in it differ by ~1.4 s of
RT60 at 1 kHz because the occupants absorb 136 sabins. That is a real effect, it is free once the
occupant count is available (it is — from §3.9), and it is the kind of thing that makes a space
feel inhabited.

**Room detection:** the listener's containing volume is found from the interior cell's baked
**room graph** (rooms, corridors, and their portal connections). Sound from an adjacent room takes
the portal path (diffraction from Stage 1) rather than the through-wall path, unless the wall is
thin — the solver picks whichever is louder, per band. This is why an open door down a corridor is
a completely different acoustic event from a closed one.

### 3.7.4 Outdoor propagation

Outdoors there is no reverb, but there is:
- **Geometric spreading**: spherical (6 dB/doubling) near-field, transitioning to **cylindrical**
  (3 dB/doubling) under a temperature inversion or in a street canyon — a real effect that makes
  sound carry much further on a still, cold night (M6's katabatic inversion, §3.2.5). The
  transition is driven by the weather field's stability class.
- **Atmospheric absorption**: ISO 9613-1 per-band coefficients as a function of T, RH, and p —
  all from §3.2. At 4 kHz, 20 °C, 20% RH: 0.22 dB/m. At 90% RH: 0.11 dB/m. **Sound carries twice
  as far in fog.** Real, coupled, free.
- **Ground effect**: interference between the direct and ground-reflected path, with the ground
  impedance from the material record (grass is absorbent ⇒ less interference; concrete and water
  are reflective ⇒ strong comb filtering and much longer carry over water). A gunshot across
  Vermillion Bay is audible at 3.1 km; the same gunshot across the forest floor is audible at
  480 m.
- **Wind refraction**: a downwind gradient bends sound toward the ground (increasing carry),
  upwind bends it up (creating a shadow zone). Driven by §3.2's wind profile.
- **Barriers**: terrain and building diffraction from the Stage-1 probe lattice plus runtime
  knife-edge computation for the near field.
- **Foliage attenuation**: 0.10–0.28 dB/m above 2 kHz in the old-growth stands (§2.5.5), from the
  canopy closure field. Low frequencies pass essentially unattenuated. This is why you can hear a
  chainsaw through a forest but not a voice.

### 3.7.5 The SPL → AI perception coupling

**This is the architectural payoff of the whole audio system** and it is why acoustics is specified
in the simulation section rather than as a presentation concern.

Every acoustic solve produces, as a by-product, an **A-weighted sound pressure level field** over
the loaded region: a 4 m-resolution scalar grid (`spl_grid`, 8-bit, 0–140 dB(A)) updated at 5 Hz
plus per-agent queries at higher resolution for nearby sources.

Consumers:
- **§3.13 perception**: an agent detects a sound if `SPL_at_agent − ambient_noise_floor ≥
  detection_threshold(SPL_at_agent, agent_state)`. The ambient noise floor is itself computed
  from traffic, wind (from §3.2), precipitation, HVAC, and crowd density — so a gunshot during a
  freight-train pass in Cordova Flats is masked, and the same gunshot at 03:00 in Marlow Heights
  wakes 340 people.
- **§3.12 crime reporting**: the `T_perceive` term is driven by this, not by distance.
- **Wildlife**: a 92 dB(A) event within 240 m flushes birds (a Tier-3 flock scatter event),
  startles ungulates (a herd displacement that persists 4–40 min in the patch record), and is
  logged as a disturbance event in the ecology model. Repeated disturbance shifts a patch's habitat
  suitability index down — **the player's shooting range habit can empty a valley of elk over a
  season.**
- **Resident behaviour**: noise complaints are a real ledger event class. A resident whose
  apartment exceeds the municipal night-time limit (45 dB(A) at the façade, per the §2.4.1 noise
  ordinance derived from the road/air/rail noise contours) generates a complaint, which generates a
  non-emergency police call, which enters the dispatch queue. **A loud car at 02:00 in Riverton
  produces a real, if trivial, police response.**

### 3.7.6 Audio event catalogue

| Category | Count | Notes |
|---|---|---|
| Foley footsteps | 2,340 | 14 surface conditions × 9 footwear classes × 6 gait states × ~3 variants |
| Vehicle (per vehicle) | 180–340 | Engine: a **granular synthesis** source driven by RPM, load, and the induction/exhaust pressure traces from §3.5.5 — not sample playback. This is required because a turbo's boost state, a cold start, and a knock-derated engine all produce states a sample bank cannot cover. Plus tyre noise (a filtered noise source driven by surface μ and speed — the sound of gravel vs asphalt vs wet asphalt is *computed*, not sampled), wind noise (from the aero model), driveline (shuffle/clunk events from §3.5.5), and impact. |
| Weapons (per weapon) | 60–110 | Report: a near-field crack + a far-field boom with distinct propagation (the boom carries further and is the part heard at 1.2 km); mechanical cycle sounds; suppressor variants; indoor/outdoor variants **derived from the acoustic solve**, not separately authored |
| Ambience beds | 1,140 | Per biome × time-of-day × weather state, streamed, 32 kbps Opus |
| Material impacts | 8,900 | 2,412 materials × impact classes, deduped to 8,900 by material class |
| Destruction | 2,100 | Per deformation model × energy band |
| Wildlife | 4,600 | 312 species × call types × seasonal variants; triggered by the ecology model's activity pattern |
| Urban | 3,200 | HVAC, traffic, sirens (from dispatch state), construction (from ledger work orders), crowd walla (from local agent density) |
| **Total** | **≈ 23,100 assets** | **4.8 GB cooked** (Opus 48 kbps ambience/music, ATRAC9/ADPCM one-shots) |

**Music:** 74 minutes of scored adaptive music in 9 stems, 340 transition points. **Deliberately
restrained** — a world this detailed does not need a score telling the player how to feel. Music
triggers on 14 authored narrative conditions and otherwise stays silent.

### 3.7.7 Underwater acoustics

A distinct propagation model, not a filter on the air model:
- Sound speed 1,480 m/s (vs 343) ⇒ wavelength 4.3× longer ⇒ diffraction around the player's own
  head is different, and the HRTF set is **swapped for an underwater set** (measured data shows
  localisation collapses to a front/back confusion; we reproduce that, which is genuinely
  disorienting and is the correct behaviour).
- Absorption ~0.06 dB/m at 10 kHz, ~0.006 dB/m at 1 kHz ⇒ **only low frequencies carry**, so a
  boat engine 400 m away is a distinct low thump while a diver's exhalation 3 m away is bright.
- Surface and bottom reflection: a **Lloyd's mirror** interference pattern from the sea surface,
  with attenuation from the bottom material (sand 0.6 dB/grazing angle, rock 0.15).
- No reverb in open water; strong reverb inside a wreck (Stage 1 probes are baked for the 3 wrecks).
- The player's own breathing and regulator flow are the dominant local sounds, and their rate is
  driven by the §3.14 exertion model — **the audio system is a physiological readout.**

### 3.7.8 Audio budget

| Item | Cost |
|---|---|
| Voice mixing (512 voices, SIMD AVX2/NEON, 48 kHz) | 0.82 ms across 2 threads |
| FDN reverb (3 instances) + HRTF convolution | 0.31 ms |
| Convolution reverb (3 hero spaces, GPU FFT) | 0.18 ms GPU |
| Acoustic ray trace (Stage 2, 15 Hz) | 0.025 ms amortised |
| SPL grid update (5 Hz) | 0.09 ms amortised |
| Granular engine synthesis (≤ 6 vehicles) | 0.24 ms |
| Event/trigger logic | 0.18 ms |
| **Total** | **1.67 ms CPU across 2–3 audio workers, 0.18 ms GPU** |

Within §5.11.1's 1.6 ms audio budget ⇒ **1.04% over.** Resolution: reduce concurrent full-DSP voices
from 96 to 88 (a governor tier, §5.11.3) which recovers 0.11 ms. Documented here rather than
silently, per Appendix C's zero-sum policy.

---

## 3.8 Traffic

### 3.8.1 The three-tier coupling

No single traffic model works at all scales. Microscopic simulation of 7.9M vehicle-km/day is
impossible; macroscopic simulation cannot reproduce a single car's lane change. MERIDIAN runs three
coupled tiers with explicit handoff hysteresis — the same architectural pattern as §3.9's Resident
Continuum, applied to vehicles.

| Tier | Model | Extent | Rate | Entities |
|---|---|---|---|---|
| **Macro** | **Cell Transmission Model** (a discrete Lighthill-Whitham-Richards solve) on the road graph | Whole region, 2,640 km in 12,400 links | 1 Hz | 12,400 link densities |
| **Meso** | **Queue-based / speed-density** per link with vehicle "packets" | Loaded macro-region (3 km radius, ~340 km of road) | 4 Hz | ~9,000 packets |
| **Micro** | **IDM car-following + MOBIL lane-change**, fully embodied | 240 m radius + any road in the player's predicted corridor | 60 Hz (V2) / 120 Hz (V1) | ≤ 180 (V2) + ≤ 10 (V1) |

**Cell Transmission Model:**
```
n_i(t+1) = n_i(t) + (Δt/Δx)·(f_{i−1}(t) − f_i(t))
f_i(t)   = min( D_i(t), S_{i+1}(t), Q_i )
D_i      = v_i · n_i                    (demand: free-flow speed × density)
S_i      = w · (N_i − n_i)              (supply: backward wave speed × remaining capacity)
Q_i      = capacity                     (from lanes, speed limit, signal state)
v_i      = V_free,i · (1 − n_i/N_i)^a   (Greenshields-type speed-density)
w        = backward wave speed ≈ 5.8 m/s (21 km/h) — the speed at which a shockwave propagates
```
This is what real dynamic traffic assignment uses, and it produces **shockwaves**: a bottleneck at
the Cordova Flats bridge sends a queue propagating backwards at 21 km/h, which is visible from a
helicopter as a growing line of brake lights. That is not an authored event; it is the model working.

**IDM (micro):**
```
dv/dt = a·[ 1 − (v/v_0)^δ − (s*(v, Δv)/s)² ]
s* = s_0 + v·T + (v·Δv)/(2·√(a·b))
  a = 1.4 m/s²  (max accel, per vehicle class: truck 0.8, sports car 2.6)
  b = 2.0 m/s²  (comfortable decel; emergency 8.0)
  s_0 = 2.0 m   (min gap)
  T = 1.5 s     (desired headway; per-driver personality ⇒ 0.9–2.4 s, drawn from the ResidentRecord)
  δ = 4         (acceleration exponent)
  v_0 = desired speed = speed_limit · (1 + driver_aggression), aggression ∈ [−0.12, +0.34]
```

**MOBIL (lane change):**
```
incentive:  ã_c(t) − ã_n(t)  ≥  p·(ã_c'(t) − ã_n'(t)) + Δv_th
  ã = acceleration of the subject; ã_n = of its current leader; primes = after the change
  p = politeness factor ∈ [0, 1], drawn per driver from the ResidentRecord personality
    (p ≈ 0.0 for an aggressive driver = pure selfish lane change; p ≈ 0.9 for a courteous one)
safety:     b_safe ≥ ã_c'(t) ≥ −b_safe   (b_safe = 4.0 m/s²)
```
**Driver parameters come from the ledger.** A 61-year-old transit worker on their commute has
`T = 1.9 s, p = 0.72, aggression = −0.04`. A 23-year-old in a modified coupe at 01:40 on a Friday
has `T = 0.95 s, p = 0.11, aggression = +0.28`. Traffic therefore has **personality structure**
that varies by district, time of day, and day of week — the rush-hour traffic on the Vermillion
Freeway is measurably more polite than the Friday-night traffic in Portside, and that is emergent
from the population model, not authored.

### 3.8.2 Demand: the four-step model

Offline, per time-of-day slice (16 slices/day):
1. **Trip generation** — from §2.4.1's ITE-style rates × floor area × land use, modulated by the
   ledger's actual occupancy (a household of 4 generates more trips than a household of 1; the
   region's 1.62M households have a real size distribution).
2. **Trip distribution** — gravity model with a friction factor from the travel-time matrix
   (computed by Contraction Hierarchies, §6.3.2), constrained to match the ledger's actual
   home–work pairs.
3. **Mode split** — a multinomial logit over (auto, transit, walk, cycle) with utilities from
   travel time, cost, and the resident's transit-subscription and vehicle-ownership attributes.
   Region-wide: 71% auto, 19% transit, 7% walk, 3% cycle. Riverton: 41/44/11/4 (a genuinely
   transit-dependent district).
4. **Assignment** — user equilibrium (Wardrop) by Frank-Wolfe, 60 iterations, converged to a
   relative gap < 0.4%. Output: link flows and speeds per slice.

**Result:** 11.4M person-trips/day, 7.9M vehicle-km/day, peak-hour network mean speed 34 km/h,
off-peak 58 km/h, V/C > 0.9 on 11% of arterial links in the peak.

### 3.8.3 Dynamic reassignment and incident response

The baked assignment is a **prior**. At runtime:
- Incidents (crash, roadblock, flood, fire, collapse, plough operation) reduce link capacity.
- The CTM propagates the queue.
- The meso tier re-routes packets via a **dynamic user-optimal** assignment on the affected
  subnetwork only (≤ 900 links), at 1 Hz, 4 Frank-Wolfe iterations, cost 0.31 ms.
- **Signal control responds**: the central traffic-management system (a real ledger entity with
  212 controllers) can retiming plans, and dispatch (§3.12.6) can request a **green wave** for a
  responding unit. This reduces emergency-vehicle ETA by 14–22% in dense urban — a real
  operational practice and a real gameplay-visible benefit to the police.

**Handoff hysteresis** between tiers (the thing that prevents visible pop):
- Macro→Meso: on entering the loaded macro-region; packets are spawned from the link density with
  speed drawn from the link's speed-density curve, positioned by a stratified sample along the link
  (never clustered), and given a destination drawn from the baked OD matrix.
- Meso→Micro: at 240 m or on relevance promotion; the packet's single vehicle is replaced by a
  `ResidentRecord`-backed embodied vehicle (the driver is a real person with a real trip purpose,
  drawn from the packet's OD flow — **which means the traffic around the player is populated by
  people who were already going somewhere**), with lateral offset, speed, and heading interpolated
  over 0.4 s.
- Micro→Meso at 310 m; Meso→Macro on leaving the macro-region.
- **Invariant:** the total vehicle count in each link is conserved across every handoff within
  ±1 (a CI assertion). If 340 vehicles are in a link at the macro tier and the player drives
  through it, they must encounter ~340 vehicles, not 40 and not 900. **This is the test that
  catches every traffic-pop-in bug** and it is run on every commit.

### 3.8.4 Pedestrian traffic

Same three-tier architecture on the 4,120 km sidewalk network, with a **flow-field** solution
instead of CTM:
- Macro: density per sidewalk link, 1 Hz.
- Meso: packets of 1–8 pedestrians, 4 Hz, with a group-affiliation flag (couples and families walk
  together and at the slower member's pace — a real and visible behaviour).
- Micro: individual agents with **ORCA** (optimal reciprocal collision avoidance) at 15 Hz, ≤ 340
  concurrent. ORCA rather than pure steering because it produces the correct reciprocal behaviour
  at crossing points: two people meeting head-on step aside in a coordinated way rather than
  doing the "awkward dance". Cap: 340 (governor tiers: 240, 160, 96).

### 3.8.5 Traffic budget

| Tier | Cost |
|---|---|
| Macro CTM (12,400 links, 1 Hz) | 0.07 ms amortised |
| Meso (9,000 packets, 4 Hz) | 0.11 ms amortised |
| Micro vehicles (180 @ 60 Hz + 10 @ 120 Hz) | 0.34 ms |
| Micro pedestrians (340 ORCA @ 15 Hz) | 0.19 ms |
| Dynamic reassignment (1 Hz) | 0.31 ms amortised |
| Pedestrian flow-field recompute (GPU, 1 Hz) | 0.12 ms GPU |
| **Total** | **1.02 ms CPU, 0.12 ms GPU** |

---

## 3.9 The Resident Continuum

### 3.9.1 The problem

The brief requires that "every visible NPC has a routine-driven life cycle rather than simple
random despawning/spawning." Taken literally over a 4.1M population and 252 km², this is
impossible at 60 FPS: 4.1M agents at even 1 Hz with 500 bytes of state is 2 GB and 4.1M updates.

The resolution is not to reduce the requirement. It is to recognise that the requirement is really
two requirements:
- **(R1)** Every resident has a **persistent identity and a coherent life** that continues whether
  or not they are rendered.
- **(R2)** When a resident is rendered, their behaviour is **continuous with** their persistent
  life — they were going somewhere, from somewhere, for a reason.

R1 is a **data** problem (affordable at 4.1M). R2 is a **fidelity** problem (affordable at ~4,000).
The Resident Continuum is the architecture that satisfies both.

### 3.9.2 `ResidentRecord`

```yaml
ResidentRecord:                      # 4,127,000 records; 312 bytes hot + 1.9 KB cold
  id:                res_0x2A71F4C9  # stable 64-bit, never reused, never reordered
  identity:
    given_name_id:   18422           # index into a culturally-consistent name pool
    family_name_id:  6410
    sex:             f
    birth_year:      1969
    ethnicity_id:    7               # drives name pool, language, some social graph structure
    languages:       [en, es]
  household_id:      hh_0x41B2       # 1,620,000 households
  relationships:     [ {other: res_..., type: spouse|child|parent|sibling|partner|friend|
                        colleague|rival|creditor, strength: −1.0..1.0, since: worldtime} ]  # ≤ 24
  residence:
    address_parcel:  pcl_0x8E412
    unit:            3B
    tenure:          rent            # rent|own|social|shelter
    since:           worldtime
    rent_or_mortgage_monthly: 1140
  employment:
    employer_id:     emp_0x1204      # 148,000 establishments
    occupation_code: 51-2031         # a real SOC-style code: transit operator
    shift_pattern:   {type: rotating_4on_2off, start: 05:40, end: 14:20, break: 30}
    hourly_wage:     31.40
    since:           worldtime
    tenure_months:   187
    union_member:    true
    performance:     0.68            # affects promotion/scheduling events
  economics:
    income_annual:   68400
    debt:            [{type: auto, balance: 14200, apr: 0.071, payment: 388}]
    savings:         4120
    credit_score:    664
  assets:
    vehicles:        [veh_0x...]     # each a full VehicleRecord with its own state (§3.5)
    possessions:     [ ... ]         # drives interior set dressing (§4.6) — a 64-entry list of
                                     # owned object archetypes with condition & acquisition date
    wardrobe:        [ ... ]         # drives clothing selection (§3.10.4)
  health:
    conditions:      [{code: E11.9, since: ..., severity: 0.4}]   # an ICD-10-analogue subset
    mobility:        0.86            # 0–1, affects gait, transit use, stairs
    medications:     [ ... ]
    injuries:        [ {segment: ..., ais: 2, since: ..., recovery_end: worldtime} ]  # from §3.6.5
    bmi:             27.4
  psychology:                            # OCEAN + 3 derived
    openness:        0.41
    conscientiousness: 0.78
    extraversion:    0.33
    agreeableness:   0.71
    neuroticism:     0.44
    risk_tolerance:  0.28                # derived: f(conscientiousness, neuroticism, age, income)
    law_attitude:    0.82                # derived: f(agreeableness, prior victimisation, employment)
    altruism:        0.55
  history:
    prior_victimisation: 2
    prior_arrests:     0
    prior_convictions: 0
    witnessed_events:  [ {event_id, worldtime, memory_fidelity: 0.0..1.0} ]   # ≤ 12, decaying
    player_interactions: [ {worldtime, type, valence} ]                      # ≤ 32, decaying
  current:                               # the hot 312 bytes
    tier:            T4
    activity:        at_work
    position:        {cell: 0x..., local_offset: ...}   # or link_id + s for on-network
    state_vector:    [ 12 floats: mood, fatigue, satiation, warmth, stress, intoxication,
                       bladder, urgency, awareness, compliance, health_acute, plan_progress ]
    plan_cursor:     14                  # index into the compiled schedule (§3.10)
    next_transition: worldtime + 2840
```

**Generation:** 4,127,000 records are generated offline by a **synthetic population model**:
1. Census tracts (412) with authored age × sex × ethnicity × household-type × income distributions
   consistent with the land-value field (§2.4.3) and the zoning.
2. Household formation: a headcount is partitioned into 1.62M households by a family-structure
   model (single, couple, couple+children, single-parent, multigenerational, shared non-family)
   with tract-specific rates.
3. Housing assignment: households bid on the 187,400 parcels (§2.4.2) by an affordability rule
   (rent ≤ 32% of income) with a preference for proximity to employment and to the household's
   social graph. **This produces real segregation patterns and real commute patterns** — the
   model does not author either.
4. Employment assignment: a **148,000-establishment × 23-sector** employment matrix by district,
   matched to the working-age population by occupation code with an education/skill proxy.
   Unemployment is 5.8% regionally but 3.1% in Marlow Heights and 14.2% in Riverton East — again
   emergent, not authored.
5. Social graph: 4–24 ties per person, formed by household, workplace co-location, school cohort,
   and neighbourhood proximity, with a homophily parameter per attribute.
6. Psychology: OCEAN sampled from age- and sex-conditioned distributions with household
   correlation (siblings correlate at r ≈ 0.22 on conscientiousness) — enough structure that
   families feel like families.

**Generation cost:** 41 minutes on 96 cores. **Output:** 1.1 GB compressed (`ledger.strata`).

### 3.9.3 The five tiers

| Tier | Range from a simulation source | Cap | Representation | AI rate | Animation | Rendering | Collision |
|---|---|---|---|---|---|---|---|
| **T0 — Embodied** | ≤ 64 m | 40 | Full agent: behaviour tree + utility layer + GOAP-lite planner | **60 Hz** decision, 120 Hz locomotion | Full: motion-matched, IK (foot/hand/look/spine), facial rig, cloth PBD | LOD0–1, virtualized-geometry clothing + skinned path | Full capsule + limb colliders |
| **T1 — Detailed** | 64–300 m | 260 | Full agent, simplified planner | 15 Hz | Motion-matched, foot IK, no facial, cloth → bone-driven sway | LOD2–3 | Capsule |
| **T2 — Crowd** | 300 m–2 km | 3,000 | Agent + ORCA-lite, no planner (follows plan route) | 5 Hz | **VAT** (vertex animation texture) locomotion from a 240-frame bake per gait archetype | LOD4–5, instanced, ≤ 6 material slots | Capsule in the crowd solver only |
| **T3 — Aggregate** | 2 km – loaded macro-region | ~180,000 | **Density on the road/sidewalk graph.** No individual state. Movement is a flow-field advection of counts. | 1 Hz | none | Instanced billboards with VAT, spawned from density at the render boundary | none |
| **T4 — Ledger** | region-wide, off-screen | 4,127,000 | **Record only.** Advanced by the abstracted stochastic ledger tick. | 1 Hz (in-session) or via Downtime Simulation (§5.11.4) | none | none | none |

**Tier counts at steady state:** 40 + 260 + 3,000 + ~180,000 + ~3.94M = 4.13M. Total resident
runtime memory: **380 MB** (§3.9.7).

### 3.9.4 The ledger tick (T4)

T4 is the whole point of the architecture, so it is specified in detail.

At 1 Hz, each T4 record advances by an **abstracted transition** — not a full behaviour simulation.
The record's `activity` is a state in a **per-record compiled schedule** (§3.10), and the tick does:

```
for each record whose next_transition <= now:
    1. Resolve the current activity's outcome:
         - work:      did they attend? P = f(health, fatigue, household disruption, transit delay)
                      absence ⇒ employer log ⇒ wage deduction ⇒ economics update
         - transit:   did the service run? from §3.8's actual state; delay ⇒ schedule slip
         - social:    did the counterpart attend? (both records are checked — a meeting that one
                      party missed is a real ledger event and can decay a relationship)
         - sleep:     duration and quality ⇒ fatigue state
    2. Stochastic perturbation: sample from the activity's errand distribution
         (a 12% chance per day of an unplanned errand: pharmacy, bank, school pickup, fuel)
    3. Weather response (§3.2): if precip > 4 mm/h and the activity is outdoor-leisure,
         substitute an indoor alternative; if visibility < 200 m and the activity requires
         driving a mountain road, defer
    4. News/event response: region-level events (a fire, a transit strike, a curfew, an election)
         modify the activity choice distribution for the affected population
    5. Advance relationships: 1 Hz relationship decay/reinforcement is applied in BATCH once per
         sim-hour, not per tick (a 4.1M × 24 edge update is 98M operations/hour — batched and
         sparsified to only edges with |Δstrength| > threshold, which is 3.1% of edges)
    6. Recompute derived economics once per sim-day: income, expenditure, debt service, savings
    7. Set next_transition
```

**Cost:** 4.127M records × 1 Hz, but only **11.4% of records transition in any given second**
(the average activity duration is 8.8 s of sim time... no — the average activity duration is
~52 minutes, so ~0.032% transition per second). Let me be precise: mean activity duration 52 min
⇒ transition rate = 4.127M / 3,120 s = **1,323 transitions/second**. At 4.8 µs each (a state
machine step plus 2 PRNG draws plus a write to the hot array), that is **6.4 ms/s of sim time =
0.106 ms/frame**. Plus the batched hourly relationship pass (98M × 3.1% = 3.0M edges, at 40 ns
each = 122 ms once per sim-hour = 0.002 ms/frame amortised) and the daily economics pass (4.1M ×
0.9 µs = 3.7 s once per sim-day = 0.0043 ms/frame amortised).

**Total ledger tick: 0.113 ms/frame.** For 4.1 million simulated lives. That is the number that
makes USP-1 real. It is affordable because the ledger tick does almost nothing per record, and
because the vast majority of records are simply *waiting*.

⚠ **HONEST LIMITATION.** T4 records are not "living" in any rich sense. They advance through a
compiled schedule with stochastic perturbation. They do not have emergent thoughts, they do not
improvise, and two T4 residents with identical attributes and schedules will behave identically
modulo PRNG. **The claim is continuity and coherence, not richness.** Richness exists only at T0–T1
(300 concurrent agents). The design contract is: no player should ever be able to distinguish a T4
resident's life from a fully simulated one *by observing its consequences*, and the falsifiable
test in §1.4 USP-1 is the check on that claim.

### 3.9.5 Tier promotion and demotion (rehydration)

The hard part. A T3 aggregate is a density; a T2 agent is an individual. Where does the individual
come from?

```
PROMOTION (T4 → T3 → T2 → T1 → T0):
  Trigger: record enters the promotion radius of a simulation source, OR is referenced by a
           higher-tier agent's plan (a T0 agent's scheduled meeting partner is force-promoted
           regardless of distance — you cannot have a conversation with a density field).
  T4→T3: no work; the record contributes to its link's density.
  T3→T2: INSTANTIATION. Select records from the link's density using a stratified, deterministic
         sample (seeded by (link_id, sim_hour)) such that the sampled records' aggregate
         attributes (age, sex, occupation, income distribution) match the link's OD-flow profile
         within a χ² tolerance. Instantiate a T2 agent per selected record at a position sampled
         along the link by the same stratification. **The identity is not invented at this point;
         it is drawn from the population that the macro model says is on this link.** This is the
         mechanism that makes R2 true.
  T2→T1: allocate the full agent struct, evaluate the plan cursor, warm the animation state from
         the VAT phase so the gait phase is continuous, allocate a capsule in the crowd solver.
  T1→T0: allocate planner, IK, facial rig, cloth; run a 0.25 s "context settle" where the agent
         evaluates its immediate surroundings (obstacles, other agents, its goal) — invisible
         because it happens at 64 m.

DEMOTION: the reverse, with a 0.4 s interpolation of position/heading/gait-phase into the lower
         tier's representation, and a **state write-back** to the record so that a subsequent
         promotion continues from the same point.

HYSTERESIS: promotion radii are 90% of demotion radii (T0 64/71 m, T1 300/330 m, T2 2 km/2.2 km).
            A record may not change tier more than once per 2.0 s (dwell time).
            A record may not change tier more than 4 times per 60 s (oscillation guard) — if it
            does, it is pinned at the higher tier for 60 s.
```

**The invariant, asserted in CI:** for any record, the sequence of `(tier, position, activity)`
over time must be **continuous within tolerance**: position discontinuity ≤ 0.6 m at T2→T1 and
≤ 0.1 m at T1→T0; activity must not change across a promotion except by the record's own schedule.
A test harness runs 400 hours of simulated play with random camera teleportation and asserts zero
violations. **This test is the single most important QA asset in the project** — it is what
prevents the "NPC that was there and now is a different NPC" bug that breaks the whole premise.

### 3.9.6 The diurnal swing problem

Cathedral Quarter: 94,000 residents, 410,000 daytime population (§2.5.1). At 08:30 the district
needs ~310,000 people present. Obviously only ~3,300 can be instantiated (T0+T1+T2 caps). The rest
are T3 density on the sidewalk graph and T4 records "at work" inside buildings.

The mechanism that makes this read as full:
1. **Interior occupancy is a T4 aggregate.** A building with 4,200 workers has an occupancy count,
   not 4,200 agents. The count drives: window lighting (§4.7), HVAC noise (§3.7.6), elevator
   demand (§2.6.2), restroom/water demand (the utility model), and lunchtime egress.
2. **Lunchtime egress is a scripted-but-derived event**: at 12:00–13:30 the model releases 34% of
   a building's occupancy onto the sidewalk graph as T3 density, with a **stagger** derived from
   the building's floor count and elevator capacity (a 74-storey tower cannot release 1,400 people
   in 5 minutes; it takes 34 minutes through 61 elevators at 32 s per cycle — computed, and it
   produces the real observed pattern of a lobby queue spilling onto the pavement).
3. **Transit surge**: at 07:30–09:00 and 16:30–18:30 the T3 density on the metro links rises to
   the modelled load factor (1.42 standees/m² peak on the Red line). Boarding a train at peak puts
   the player in a car with 214 T2 agents — the cap is reached, and the governor (§5.11.3) reduces
   interior detail and disables cloth to hold 60 FPS. **This is the game's hardest perf scene and
   it is a designed one**, with a specific budget allocation (§5.11.4 "hard scene" table).
4. **Night**: the district empties to 94,000, street density drops 78%, and the character of the
   place changes — fewer T2 agents, more loitering behaviour (a distinct activity class), more
   crime opportunity (§3.12.2's baseline is time-modulated by street population and lighting).

### 3.9.7 Resident memory budget

| Component | Bytes | Notes |
|---|---|---|
| T0 agent structs (40 × 18.4 KB) | 0.74 MB | Behaviour tree state, planner, animation instance, IK, cloth |
| T1 agent structs (260 × 6.2 KB) | 1.61 MB | |
| T2 agent structs (3,000 × 1.1 KB) | 3.30 MB | SoA, hot fields only |
| T3 density graph (180,000 cell-entries × 24 B) | 4.32 MB | Road + sidewalk link densities |
| **T4 hot array (4,127,000 × 312 B)** | **1,287 MB** — ⚠ | **Does not fit. See below.** |
| T4 cold storage (disk, 1.9 KB/record) | 7.8 GB on disk | Never resident |

⚠ **The T4 hot array does not fit in the 380 MB budget.** Resolution — the actual design:

The `current` block is **not** resident for all 4.1M records. It is split:
- **Active set:** records with `next_transition ≤ now + 300 s`, i.e. about to do something. Mean
  activity duration 52 min ⇒ 4.127M × (300/3120) = **397,000 records active**. Kept in a
  **min-heap by `next_transition`** with the hot 312-byte block resident: 397,000 × 312 B =
  **124 MB**.
- **Dormant set:** the remaining 3.73M records keep only `(id, tier, activity, next_transition,
  position_quantised)` = **40 bytes**, in a flat array sorted by `next_transition`:
  3.73M × 40 B = **149 MB**. Their full hot block is paged in from the cold store on activation
  (a 4 KB read from NVMe at ~90 µs, amortised over 1,323 activations/s = 0.12 ms/s — free).
- **Total: 273 MB.** Plus the T0–T3 tiers (9.97 MB) and the relationship-graph adjacency (a
  compressed sparse row, 4.127M × 9.4 mean edges × 6 B = **233 MB**) — which also does not fit.
- **Relationship graph resolution:** only edges with `strength > 0.15` are resident (38% of
  edges) = 89 MB. Weak ties are paged on demand, which is correct because a weak tie only matters
  when one of its endpoints is instantiated — and at that point the endpoint's edge list is paged
  in as part of promotion.
- **Final total: 9.97 + 124 + 149 + 89 = 372 MB.** Within the 380 MB budget with 8 MB headroom.

This is stated at this level of detail because "4.1 million persistent residents" is a claim that
either survives arithmetic or does not. It survives, but only with a two-level residency design and
a paged relationship graph — not with a naive record array.

---

## 3.10 Schedules, Routines, and Behaviour

### 3.10.1 The plan structure

Every resident has a **compiled day plan**: an ordered list of `(activity, location_ref,
t_start, t_end, transit_leg?, social_group?, priority, fallback)` entries. Generated offline for a
**representative week** (7 days × 6 schedule archetypes = 42 templates per record class), then
**compiled at runtime** into a per-record cursor.

Activity classes (34):
```
sleep · personal_care · meal_preparation · meal_eating · meal_purchase · commute_out ·
commute_return · work_shift · work_break · work_travel · school · childcare · household_chore ·
shopping_grocery · shopping_retail · shopping_service · errand_medical · errand_financial ·
errand_civic · errand_religious · leisure_solo_home · leisure_social_home · leisure_bar ·
leisure_cafe · leisure_park · leisure_sport_active · leisure_sport_spectate · leisure_gym ·
leisure_cultural · leisure_walk · leisure_drive · sleep_daytime (shift workers) ·
care_dependent · involuntary_unemployed_jobsearch
```
Each has: a **location requirement** (home / workplace / a POI of a category / outdoors / any), a
**duration distribution**, a **co-presence requirement** (solo / household / social group /
strangers), an **energy/fatigue delta**, a **cost** (money, from the economics model), and a
**time-of-day validity window**.

### 3.10.2 Generation

Per record, from its attributes:
1. **Fixed anchors first**: work shift (from `employment.shift_pattern`), school (from age), sleep
   (from age and shift — a night-shift worker's sleep is 07:30–14:30 and this reshapes their
   entire week), childcare (from household composition).
2. **Commute legs**: from home parcel to workplace parcel, by mode chosen from the §3.8.2 mode-
   split logit evaluated with *this record's* attributes and *this time's* actual travel time.
   A resident who owns a car and lives in Bellweather drives; one who lives in Riverton above a
   metro station takes the Red line. The mode is not random — it is the same model that assigns
   region-wide traffic demand, so **individual behaviour and aggregate traffic agree**.
3. **Maintenance activities**: meals (3/day, with a home/away split by income and work schedule),
   personal care, chores (a weekly distribution), grocery (1.8×/week, at a store chosen by a
   gravity model from the retail availability layer — which is itself driven by the supply chain,
   §3.11.3, so **a store that has run out of stock sees its customers go elsewhere**).
4. **Discretionary time**: the residual is filled by a **utility-ranked choice model**:
   `U(a) = β_attr · attr(a, record) + β_social · social_opportunity(a) + β_cost · (−cost) +
   β_fatigue · (−fatigue_delta) + β_weather · weather_fit(a, §3.2) + ε_Gumbel`.
   The Gumbel error term makes it a logit, so the choice is stochastic but *structured* — an
   extravert with high openness and disposable income has a genuinely different leisure
   distribution from a fatigued night-shift worker with two children, and the difference is
   consistent day to day because the βs are per-record.
5. **Compile**: the week's plan is flattened into a monotonic-time cursor array with DST handled
   per §2.1.

### 3.10.3 Runtime behaviour at T0/T1

A compiled plan gives *what* and *where*; the behaviour layer gives *how*.

```
Layer 1 — PLAN EXECUTOR:  reads the cursor, issues goals ("go to parcel X, activity Y")
Layer 2 — UTILITY LAYER:   12 continuously-evaluated drives (fatigue, hunger, bladder, thermal
                           comfort, social, safety, urgency, curiosity, compliance, intoxication,
                           pain, task focus). Each produces a score; a score exceeding the current
                           plan activity's priority + a hysteresis band triggers an INTERRUPTION.
                           This is what makes an agent leave a queue to find a toilet, or abandon
                           a commute to shelter from a downpour.
Layer 3 — BEHAVIOUR TREE:  ~340 nodes, shared across all agents, data-driven from the activity
                           class. Sequence/selector/parallel/decorator.
Layer 4 — GOAP-LITE PLANNER: for activities requiring multi-step interaction (make a meal:
                           go to fridge → take ingredient → go to hob → place → wait → take →
                           go to table → sit → eat → clear). 14 actions, ~40 preconditions/effects,
                           A* over the action space with a 64-node budget per replan. Replans on
                           interruption or environment change (the fridge is empty because the
                           supply chain failed ⇒ replan to "go shopping").
Layer 5 — LOCOMOTION/ANIM: motion matching (§3.10.5) + IK layers + a physics-driven "active
                           ragdoll" for stumbles, shoves, and falls.
```

**Social behaviour**: agents in the same `social_group` (from the relationship graph) coordinate —
they walk at the slowest member's pace, they face each other when stationary (a conversational
formation solver: an F-formation from proxemics theory, with the geometry depending on group size
2/3/4+), they synchronise activity (both order coffee), and they have a **shared plan** where one
member's interruption propagates. Groups are generated from the relationship graph and are a major
source of the "these people know each other" feeling that single-agent systems cannot produce.

### 3.10.4 Appearance selection

Clothing is chosen per record per day from their `wardrobe` list, by:
```
score(garment) = w_weather · weather_fit(§3.2: T, precip, wind, solar)
               + w_occasion · dress_code(activity, workplace)
               + w_identity · self_expression(OCEAN.openness, occupation, age cohort)
               + w_income   · brand_tier(economics.income) vs garment.tier
               + w_state    · cleanliness, condition (a garment degrades; laundry is an activity)
               + w_availability · is it clean and at home?
```
`weather_fit` uses a real **clo insulation model**: total insulation = garment clo + trapped air
layer; required clo = f(T_air, wind (wind-chill), activity metabolic rate, humidity). A resident
whose required clo exceeds their wardrobe's maximum in a cold snap **shivers, walks faster, seeks
shelter, and shortens outdoor activities** — and if they cannot, they accrue a cold-exposure health
state. Conversely, a heat event produces visible behavioural adaptation: shade-seeking, reduced
midday outdoor activity, increased retail/café occupancy, and — for the vulnerable population
(elderly, no air conditioning, top-floor apartments) — a **real excess-mortality event** that the
ledger records and the news reports. §3.11.4's energy model determines who has air conditioning.

This is not cosmetic. It is the mechanism by which the weather system becomes visible in human
behaviour, which is Pillar 1.

### 3.10.5 Motion matching

**Database:** 44 hours of mocap, 1,240 clips, 41 subjects (sex × age × build × mobility
stratified), covering: 9 gaits × 8 speeds × 6 directions, 14 stair variants, 34 sit/stand, 22
carry variants, 18 social gestures, 12 fatigue states, 9 intoxication states, 14 injuries/limps,
plus per-occupation movement signatures (a transit worker's gait differs from a construction
worker's — a real and observable distinction, and 340 clips are occupation-tagged).

**Representation:** per clip per frame, a **feature vector** of 62 floats: 6 trajectory points
(future 0.3/0.6/1.0/1.5/2.0 s position + velocity), 6 joint positions (pelvis, feet, hands, head),
6 joint velocities, plus 4 scalar state tags (grounded, seated, carrying, intoxicated).

**Search:** a **VP-tree over quantised feature vectors** (16-bit per dimension, 124 bytes/frame),
with an early-exit bounding search. Database size: 44 h × 30 fps × 124 B = **5.9 GB raw**,
ACL-compressed to **1.21 GB** on disk, **340 MB resident** (a streamed subset — the resident set
is chosen by the currently instantiated population's attribute distribution, so a district with no
children does not page in child animation).

**Runtime cost:** 1 search per T0/T1 agent per frame at 60 Hz. 300 agents × 24 µs = 7.2 ms — over
budget. Mitigations: (a) T1 agents search at 20 Hz, not 60 Hz (their motion is slow enough that a
50 ms pose latency is invisible at > 64 m); (b) T0 agents search at 60 Hz but the search is
**budgeted to 900 VP-tree visits** with a graceful degradation to the best-so-far; (c) 4 worker
threads. Result: **(40 × 24 µs × 60 + 260 × 24 µs × 20) / 4 threads = 0.175 ms/frame/thread.**
Plus pose blending and IK: **2.2 ms total across 3 animation workers**, within §5.11.1's budget.

**Pose post-processing:** inertialisation blending (a 5-frame exponential decay on the pose delta
at a transition, which eliminates pops without the latency of a full blend), 3 IK layers (foot
placement on terrain/props with a 2-bone solve + pelvis drop; hand attachment for props, rails,
door handles, and grab points; look-at with a 3-joint spine+neck chain and a saccade model —
**the saccade model matters**: real human gaze is a series of 20–200 ms fixations separated by
30–80 ms saccades, and smooth-tracking gaze reads as robotic), plus a physics-driven **active
ragdoll** layer (PD-controlled joint targets with a 60 Hz solver) that blends in on stumbles,
shoves, and impacts so a character who is hit reacts physically without losing authored intent.

### 3.10.6 Civilian survival instincts and reactive behaviour

The brief calls for "realistic civilian survival instincts". Specification:

```
THREAT APPRAISAL (per T0/T1 agent, 15 Hz):
  threat_sources = { gunfire (SPL + bearing from §3.7.5), explosion, collision impact,
                     brandished weapon (visual, §3.13), aggressive approach, fire (§3.3.5),
                     flood, structural collapse, animal, vehicle at speed on a collision course }
  threat_score(s) = severity(s) · proximity(s) · certainty(s) · novelty(s)

  certainty is the interesting term: a sound behind a wall with 56 dB of TL gives low certainty
  of a gunshot vs a door slam — the agent's appraisal is genuinely uncertain, and the correct
  human response to uncertainty is ORIENTATION, not flight. So:

RESPONSE LADDER (Flinch → Orient → Assess → Act), with the Act branch depending on
risk_tolerance, altruism, and the presence of dependents:
  0. FLINCH (0–180 ms): a startle reflex. Universal, uncontrolled. Head duck, shoulder raise,
     blink. This is the first 180 ms and it is the detail that sells the whole system, because
     it happens BEFORE any decision.
  1. ORIENT (180–900 ms): head and torso turn toward the source, gaze saccades to it, locomotion
     halts or slows. Others nearby copy the orientation within 300–700 ms — **social referencing**,
     a real and powerful emergent behaviour: one person's flinch becomes a crowd's attention.
  2. ASSESS (0.5–4 s): the utility layer's `safety` drive spikes; the agent queries:
       - escape routes (a 4 Hz flow-field toward the nearest "safe" node: a doorway, a vehicle,
         away from the bearing, behind cover)
       - dependents in the social group (a parent's utility for `protect_dependent` outweighs
         `flee` by 3.4× — parents go TOWARD danger to collect a child, and this is authored
         deliberately because it is the true behaviour)
       - others' behaviour (conformity: if 60% of nearby agents are fleeing, the flee utility
         gets a +2.1 conformist bonus; if they are standing and watching, so does `watch`)
  3. ACT: one of
       FLEE          (68% base, modulated by risk_tolerance and prior_victimisation)
       FREEZE        (11% — tonic immobility; the agent stops, crouches, and is unresponsive to
                      social interaction for 3–40 s. Real, and rarely modelled.)
       HELP          (7% base × altruism; a medical-trained agent (3.1% of the population, from
                      the occupation matrix) has a 4.4× higher help rate — a nurse runs TOWARD it)
       SHELTER       (9% — move behind cover or into a building and stay)
       RECORD        (4% — hold up a phone. Produces a real ledger artefact: footage, which can
                      become evidence in §3.12.6 or a news item in §3.11.5. A modern behaviour
                      and a systemic one.)
       CONFRONT      (1%, gated by risk_tolerance > 0.8 and a perceived capacity advantage)
       IGNORE        (variable — a night-shift worker in Cordova Flats who has heard industrial
                      bangs for 20 years has a raised novelty threshold and often does not react
                      to a sound that would panic a visitor. This is authored as a per-record
                      `habituation` attribute accumulated from prior exposure.)
```

**Crowd dynamics under threat:** a **panic parameter** per crowd that rises with threat score and
local density and decays at 0.4/s. Above 0.6, ORCA's collision-avoidance politeness drops to 0
(agents shove), movement speed rises to 1.5× normal up to a density limit, and above 2.4 persons/m²
the model switches to a **continuum crowd flow** (a pressure-based solve that produces real crush
behaviour at > 5 persons/m²). Crush injury is a real outcome at a stadium egress or during an
evacuation, and the region has 3 venues where it can occur. **Design review has approved this**
because a fire evacuation that cannot injure anyone is not a fire evacuation. It is also the reason
§3.12.6 has crowd-control as a police tactic with real trade-offs.

---

## 3.11 Economy, Supply Chain, and Institutions

### 3.11.1 Why an economy model exists

Not as a feature. As a **source of authored-feeling content that costs nothing to author**. Every
shelf in every store, every container in every yard, every truck on every road, every job posting,
every vacant storefront, and every resident's income is a consequence of the economy. Without it,
all of that must be hand-placed, and 37,436 interiors cannot be hand-placed (§1.8).

### 3.11.2 Structure

```
SECTORS (23):  agriculture · food processing · fishing · forestry · mining · energy ·
               water & waste · construction · manufacturing · petrochemical · logistics ·
               wholesale · retail food · retail general · hospitality · transport · finance ·
               professional services · public administration · education · health · arts · other

ESTABLISHMENTS (148,000): each with {sector, parcel, floor_area, employment, operating_hours,
               inventory_capacity, supplier_links[], customer_catchment, rent, revenue_state}

FLOW GRAPH:    a directed graph of (establishment → establishment) commodity flows with
               {commodity_id, weekly_volume_t, mode (road/rail/sea/air/pipeline), lead_time}
               412,000 edges. Derived from an input-output matrix (a 23×23 Leontief table
               authored once, regionally calibrated) × establishment sizes.

HOUSEHOLDS (1.62M): demand side. Consumption by category from income, with an Engel curve
               (food share falls as income rises) and a per-category price elasticity.

GOVERNMENT:  revenue (property tax from §2.4.3's land value, sales tax, income tax, port fees)
             → budget allocation (police, fire, transit, roads, schools, parks, health)
             → service levels, which feed back into land value and quality of life
```

### 3.11.3 The supply chain as gameplay surface

Commodity flows are **physical**: a flow of `(Cordova Logistics WH-14 → Riverton Grocery #22,
commodity: canned tomatoes, 2.4 t/week, mode: road, lead_time: 18 h)` corresponds to an actual
manifested container or pallet in the world (§2.5.2's 34,000 containers, each with a consignee and
contents record), an actual truck with an actual driver record on an actual schedule, and an actual
inventory level at the destination.

```
Inventory dynamics at each establishment, 1/3600 Hz:
  stock[c] += arrivals[c] − sales[c] − spoilage[c]
  arrivals[c] = Σ over inbound flows with delivery_time <= now, gated by whether the delivery
                actually happened (the truck arrived, was not stolen, was not destroyed, the
                driver was not arrested, the road was not closed by flood or fire)
  sales[c]    = demand(c, catchment population, price, substitute availability, season,
                       day-of-week, time-of-day)
  stock[c] == 0 ⇒ OUT OF STOCK ⇒ the shelf is empty (a real visual state driven by a real
                inventory number) ⇒ customers substitute (a competitor gains demand) or travel
                further (a change in their schedule and in traffic demand) or forgo
```

**Consequences the player can cause and observe:**
- Steal a container from Halstead Yard ⇒ 36 h later, 4 stores are short on that SKU ⇒ shelves are
  visibly empty ⇒ a store manager's schedule includes an emergency order ⇒ a rival store gains
  demand ⇒ a delivery truck that should not be there is on the road.
- Burn a packing house in Meridian Valley ⇒ the orchards it served have no outlet ⇒ fruit is not
  harvested (a visible change in the orchard's phenology-driven harvest activity) ⇒ 240 ledger
  residents lose seasonal work ⇒ household incomes drop ⇒ Riverton East retail demand falls ⇒
  3 storefronts go vacant over 6 months ⇒ land value drops ⇒ crime baseline rises.
- Close Sable Pass (a fire, an avalanche, a player-caused collapse) ⇒ Anza Basin's inbound
  logistics lead time rises from 40 min to 3.4 h ⇒ fuel prices rise 11% in Anza ⇒ the ghost town's
  41 residents' schedules shift ⇒ a fuel station runs dry.
- Blockade the Cordova Flats refinery (a labour action, generated by the union-density attribute
  and a wage negotiation model) ⇒ regional fuel supply falls 62% ⇒ stations ration ⇒ queues form
  (a real crowd event with real traffic consequences) ⇒ bus headways lengthen ⇒ commute times rise
  ⇒ 41,000 residents are late to work ⇒ the ledger logs absences.

**None of this is authored.** All of it is the model running. This is the highest-leverage content
strategy in the project.

### 3.11.4 Energy and utilities

A **load model** at 1/3600 Hz: residential (heating/cooling degree days from §3.2, occupancy from
§3.9, appliance profiles), commercial (operating hours × floor area × sector), industrial (process
schedule), street lighting (a controller driven by solar depression angle, §2.10, with per-luminaire
failure states that accumulate and are repaired by a ledger work order — so a street can be dark
because nobody has fixed it, which affects crime opportunity and pedestrian behaviour), and water/
wastewater (demand from occupancy, capacity from infrastructure, and a **failure mode**: the
Cordova Flats storm system is designed for a 10-year event and floods on the 12-year one).

Generation: the Anza solar plant (340 MW, output from actual irradiance and panel soiling), 2 gas
plants (480 MW), 3 hydro on the Meridian River (210 MW, output from §3.3.1's discharge — **a
drought year reduces hydro output 41% and the region imports power at 3.2× the price**), and 1
wind farm offshore (180 MW, from §3.2's wind). A **blackout is possible** during a heat event with
a plant outage, and a blackout changes everything: no street lighting (crime baseline +180%), no
traffic signals (§3.8.3 falls back to a 4-way-stop rule, which the traffic model handles and which
produces a real 34% reduction in network speed), no metro (142,000 stranded riders, all ledger
records needing alternative plans), no refrigeration (food spoilage, a health event), and no
cellular sites after 4 h of battery backup (§3.12.3's coverage model degrades — **a blackout
creates dead zones, which is a systemic interaction nobody authored**).

### 3.11.5 Institutions, news, and public opinion

```
PUBLIC OPINION: per district × per issue (safety, economy, housing, environment, policing,
                transit), a scalar in [−1, 1] advanced at 1/86400 Hz by:
                  + lived experience (the district's residents' ledger events: victimisation,
                    service failures, employment changes)
                  + news coverage (weighted by salience and reach)
                  + social propagation (through the relationship graph — a 2-hop diffusion with
                    decay, so a widely-witnessed event propagates further than a private one)
                  − decay toward the district's baseline (half-life 42 days)

NEWS: 4 outlets (2 metro dailies, 1 broadcaster, 1 hyperlocal blog) with a story-selection model
      (salience × novelty × proximity × outlet bias). A story is generated from a ledger event
      class and affects: public opinion, policy variables, and the visibility of specific events.
      The player can be named in a story if the evidence in §3.12.6 identifies them, which
      changes how they are perceived region-wide — and therefore how NPCs react to them.

POLICY: the elected officials (on the §2.10 election cycle) set 18 policy variables
        (police budget, patrol allocation weighting, response-mode policy, transit frequency,
         zoning change rate, fire budget, tax rates, housing subsidy, environmental enforcement).
        These feed directly into §3.12.5 (police capacity), §3.8 (transit headway), §3.3.5
        (suppression capability), §3.11.2 (government budget), and §2.4.3 (land value via
        zoning). A player who terrorises Riverton for a month can cause a council election to
        turn on policing, which changes the patrol allocation, which changes response times —
        for everyone, in every district, for four years.
```

---

## 3.12 Crime, Reporting, Dispatch, Response, Investigation, Adjudication

The pipeline. Every stage is specified with its failure modes, because **the failure modes are the
game**.

### 3.12.1 Crime event taxonomy

```
CrimeEvent:
  id, worldtime_start, worldtime_end
  class:        41 codes (a UCR-analogue Part I/II structure):
                homicide · assault_aggravated · assault_simple · robbery · burglary ·
                larceny · motor_vehicle_theft · arson · rape · vandalism · drug_possession ·
                drug_trafficking · weapons_unlawful · dui · hit_and_run · fraud · embezzlement ·
                trespass · disorderly · prostitution · gambling · family_offence ·
                resisting · impersonating_officer · brandishing · discharge_firearm_city ·
                reckless_driving · street_racing · stolen_property · ...
  severity:     1–5 (drives priority, sentencing, and public opinion weight)
  location:     parcel + cell + indoor/outdoor + jurisdiction_id
  offender_ids: [] (may be unknown — the normal case)
  victim_ids:   []
  witness_ids:  [] with per-witness {perceived_at, certainty, memory_fidelity}
  evidence:     [] (a list of EvidenceItem, §3.12.6)
  status:       undetected | perceived | reported | dispatched | on_scene | resolved_cleared |
                resolved_uncleared | adjudicated
  police_record_id: null → set on report
```

**Baseline crime rate** is *derived*, per §3.12.2, not authored. The region generates ~1,840
crime events/day at baseline, of which ~41% are perceived by someone, ~34% are reported, ~29%
receive a dispatched response, and ~18% are cleared. Those percentages are outputs, and they match
real metropolitan clearance rates — which is the point.

### 3.12.2 Baseline generation

```
λ(cell, class, t) = base_rate(class)
                  · land_value_factor(§2.4.3)
                  · population_density_factor(§3.9's T3 density, at time t)
                  · opportunity_factor(time-of-day, street population, natural surveillance —
                     i.e. how many windows overlook this cell: a real CPTED metric computed
                     from building geometry)
                  · lighting_factor(from the street-lighting state, §3.11.4 — a dark cell has
                     a 2.1× burglary rate; this couples to the blackout scenario)
                  · guardianship_factor(police patrol presence in the last 30 min, from §3.12.5)
                  · routine_activity_factor(the convergence of motivated offenders, suitable
                     targets, and absent guardians — computed from the ledger: who is here,
                     what do they own, who is watching)
                  · prior_event_factor(a 30-day decaying memory of events in this cell)
                  · weather_factor(§3.2: rain suppresses street crime 34%, heat events above
                     32 °C raise violent crime 22%, both from published effect sizes)
                  · seasonal_factor(day-of-year, 2 harmonics)
```
Events are drawn as a non-homogeneous Poisson process at 1/60 Hz over the loaded region (T3+
cells run at 1/3600 Hz and produce a coarser event stream that is refined if the player
approaches — meaning a crime that happened off-screen is *retroactively specified* within the
bounds of what the ledger already implies, which is the honest way to do off-screen events).

⚠ **HONEST LIMITATION on retroactive specification:** a crime the player did not witness is
generated from the baseline model at the time the player *could* have witnessed it, not at the
time it "actually happened". This is unavoidable — simulating 1,840 fully embodied crimes/day
region-wide is not affordable. The constraint that keeps it honest: the generated event must be
**consistent with every already-committed ledger entry** (no resident can be a victim at a time
their record says they were at work in another district; no crime can occur inside a building whose
occupancy record is empty). A CI test asserts this over 30 simulated days. Where consistency cannot
be achieved, **the event is not generated** — the world declines to produce a crime it cannot
support. This is the correct failure direction.

### 3.12.3 Stage 2 — Reporting, with a real communications model

The brief asks specifically for "granular crime-reporting delays (distance to phone/police,
cellular coverage dead zones)". Full model:

```
T_report = T_decide + T_reach_comm + T_call_setup + T_queue + T_calltaker + T_dispatch_parse

T_decide (witness decides to report):
  base = 3–25 s by severity (a homicide in progress: 2–6 s; a burglary discovered after the
         fact: 60–400 s of assessment first)
  × (1 + 0.35·log2(n_other_witnesses))      BYSTANDER DIFFUSION — a real effect: with 6 other
                                             witnesses and no acquaintance, the delay roughly
                                             doubles. Modelled explicitly.
  × 0.6  if the witness group are acquainted (diffusion collapses in a cohesive group)
  × f(law_attitude)                          0.5× at 0.95, 2.4× at 0.20 — a resident with low
                                             law_attitude may not report at all (P = 0.31 of
                                             non-report at law_attitude < 0.3)
  × f(risk_tolerance, neuroticism)
  × f(prior_victimisation)                   prior victims report 1.6× faster
  + P(fear of reprisal) ⇒ non-report         if the offender is known to the witness and is
                                             violent, P(non-report) = 0.42. Real, and the reason
                                             organised crime in the sim is durable.

T_reach_comm:
  IF witness.carrying_mobile (P = 0.92 overall; 0.98 age<40, 0.71 age>75; 0.34 for the
     unhoused population; occupation-dependent for shift workers on site)
  AND coverage(witness.position, witness.carrier) >= −110 dBm      (§3.12.3.1 below)
  AND battery > 4%
  AND device_intact (not destroyed — a robbed witness cannot call)
     THEN T_reach_comm = 1.5–4 s (retrieve and unlock)
  ELSE
     path to the nearest working COMM POINT from the POI graph:
       landline in a residence or business (requires entry or being let in — a real obstacle),
       a business with a phone and staff present, a police call box (7 in the region, all in
       Portside, all pre-1994), an emergency roadside phone (34, all on freeways), a
       transit station's help point (112), a fire alarm pull station (1,880 — reports a fire,
       not a crime, and brings the fire service, which is sometimes the wrong outcome and is
       a real player option)
     T_reach_comm = path_time + entry_time + availability_wait
     IF no comm point within 25 min travel ⇒ T_reach_comm = ∞ ⇒ the crime is NOT REPORTED
       and enters the 'detected but unreported' state, which can still generate a
       witness statement if police canvass later (§3.12.6)
```

#### 3.12.3.1 Cellular coverage model

Not a hand-painted mask. A **baked RF propagation model**:

```
For each of 412 base stations (macro 148, micro 186, in-building DAS 78) and 3 carrier networks:
  P_rx(dBm) = P_tx + G_tx + G_rx − PL(d) − L_terrain − L_building − L_foliage − L_rain

  PL(d)     = log-distance:  PL(d0) + 10·n·log10(d/d0) + X_σ
              d0 = 100 m, n = {2.0 free, 2.9 suburban, 3.6 urban, 4.3 dense urban canyon,
                               1.8 in-building line of sight}, X_σ = 6–9 dB lognormal shadowing
  L_terrain = knife-edge diffraction (Lee approximation) over the 2 m DEM:
              ν = h·√(2·(d1+d2)/(λ·d1·d2))
              L = 6.9 + 20·log10( √((ν−0.1)² + 1) + ν − 0.1 )   dB
              with up to 2 diffraction edges on the path (a 2-edge recursive solve)
              → this is what makes the Sable Range's leeward valleys and the Anza Basin's
                badlands genuine dead zones without anyone painting one
  L_building= entry loss {12 dB suburban, 18 dB urban, 25 dB dense urban/low-E glazing}
              + interior partition loss from the LayerStack of walls traversed (§3.7.2's
                transmission loss, reused — one model, two consumers)
  L_foliage = ITU-R P.833:  A = 0.2·f^0.3·d^0.6 dB, d = path length through canopy,
              f = 1,900 MHz ⇒ 0.2·1900^0.3·d^0.6 ≈ 1.86·d^0.6 dB/m through the old-growth
              stands. At 60 m of forest: 22 dB. At 200 m: 45 dB — service lost.
  L_rain    = negligible below 20 GHz; included for completeness

  Then: coverage_class = P_rx vs sensitivity (−104 dBm voice / −110 dBm data / −124 dBm none)
        and a HANDOVER model between cells, with a dropped-call probability when a moving
        agent crosses a weak-coverage boundary at speed.
```

**Bake:** at 8 m resolution over 252 km² = 3.94M sample points × 412 stations is too dense;
instead bake a **32 m coverage grid** (246,000 cells × 3 carriers × 1 byte class = 738 KB) plus
a **2 m grid within 400 m of any simulation source** (resident, 8-bit, 3 carriers, ~1.8 MB).
Total: **~3 MB, 0.4 ms bake per loaded cell region, done at cook time** (2.4 core-hours total).

**Verified outcomes** (targeted design, produced by the physics):
- 100% of the metro tunnels and 100% of the 681 km of subsurface (§2.6.3): **no coverage**
  (18–25 dB of concrete + 20 m depth). Except 78 in-building DAS sites which give coverage on 9
  metro platforms and 2 tunnel sections — a real infrastructure investment the transit authority
  made, visible in the budget model.
- 62% of Anza Basin below −110 dBm; 28% below −124 dBm (§2.5.7).
- 41% of Kestrel National Forest below −110 dBm (foliage + terrain).
- 96.4% of the urban core has voice service; 88.1% has data.
- Sable Range above 1,900 m: 12% coverage (a single macro site at the ski area plus line-of-sight
  from the valley).
- **Inside a Cordova Flats storage tank** (a Class D interior, §2.5.2): −38 dB of steel ⇒ no
  coverage. Entering a tank is entering a dead zone, physically.

**Radio (police/emergency) coverage is a separate model** with different frequencies (700/800 MHz
⇒ better diffraction and foliage penetration, worse building penetration at higher bands), 34
sites, and 9 repeaters. Radio dead zones exist where cellular does not (deep subsurface, tank
interiors) and vice versa (a 700 MHz site covers Anza Basin better than 1,900 MHz cellular does).
**A deputy in Anza Basin may have no cellular but have radio; a player in a metro tunnel has
neither.** This asymmetry is a designed tactical space and it falls out of the physics.

#### 3.12.3.2 The call itself

```
T_call_setup = 2–6 s (dial, ring)
T_queue      = PSAP modelled as an M/M/c queue with ERLANG-C:
                 offered traffic A = λ_calls × h_mean_handle   [Erlangs]
                 c = 6 call-taker positions for VC (4 Halstead, 2 County, 1 Port Authority,
                     1 Anza Basin — a single dispatcher for 21 km² and 41 people)
                 P(wait) = C(c, A) = [A^c/c! · c/(c−A)] / [Σ_{k<c} A^k/k! + A^c/c! · c/(c−A)]
                 E[wait] = P(wait) / (c·μ − λ)
               Baseline: λ = 0.9 calls/s region-wide, h = 84 s ⇒ A = 75.6 Erlangs, c = 6 ⇒
                 ⚠ overloaded. Real PSAPs run at A/c ≈ 0.7–0.85, so: c = 118 positions
                 region-wide across 5 centres, A = 75.6, utilisation 0.64, E[wait] = 3.4 s.
               During a mass-casualty or high-salience event, λ spikes 8–40× for 4–20 min
                 (every witness with a phone calls). At λ = 12 calls/s, A = 1,008 Erlangs,
                 utilisation 8.5 ⇒ **the queue diverges.** The model clamps at a 180 s wait and
                 generates 'abandoned call' events, which become 'unverified report' entries
                 that a dispatcher may call back. THIS IS REAL: a major incident produces a
                 911 meltdown, and reports during it are delayed by minutes, not seconds.
                 A player who causes a large incident thereby degrades the response to their
                 own incident — an emergent, perverse, and completely realistic outcome.
T_calltaker  = 20–70 s (a structured protocol: location, nature, weapons, injuries, suspect
               description, caller safety, callback number). Duration depends on caller state:
               an intoxicated or panicking caller takes 1.8× longer and produces a lower-fidelity
               location, which can send units to the wrong address.
T_dispatch_parse = 8–40 s (CAD recommends, dispatcher accepts/overrides)
```

### 3.12.4 Stage 3 — Dispatch (CAD simulation)

```
CAD state:
  UNITS: 412 sworn patrol units region-wide across 7 agencies, plus 68 fire apparatus,
         34 EMS units, 12 transit police, 9 Port Authority, 6 Sheriff marine, 4 air units,
         3 K9, 2 SWAT teams, 1 negotiation team, 6 detectives on call.
  UNIT STATE MACHINE:
    off_duty → available → assigned → en_route → on_scene → transport → available
                       ↘ clear_without_action ↗      ↘ meal_break → available
                       ↘ out_of_service (fuel, maintenance, court appearance, training)
  SHIFT MODEL (real):
    3 watches with 10-hour tours: 07:00–17:00, 15:00–01:00, 23:00–09:00 (overlapping
    "power hours" 15:00–17:00 and 23:00–01:00 where staffing peaks — a real deployment pattern)
    Shift change at 07:00 and 17:00: a 30-minute overlap with a **roll-call/briefing period**
    during which 40% of units are out_of_service. ⇒ Response times degrade 41% for 30 minutes,
    twice a day, predictably. **A player who learns this has learned something true.**
    Meal breaks: a 30-minute window mid-tour, staggered, during which a unit is available but
    with a 4-minute additional delay.
    Fatigue: units on their 3rd consecutive night tour have a 1.6× longer on-scene processing
    time and a 2.4× higher rate of use-of-force escalation (from published fatigue research).
    Overtime budget: the agency has a monthly OT cap from §3.11.5's policy variables. When
    exhausted, minimum staffing applies and units are not held over ⇒ response degrades at
    month end. **The police get slower on the 28th because the budget ran out.**
  ASSIGNMENT ALGORITHM (per incoming call):
    1. Classify → priority (P1–P5) and required unit type/capability
    2. Candidate set: units with {status ∈ {available, en_route_to_lower_priority},
       jurisdiction == call.jurisdiction OR mutual_aid_eligible, capability ⊇ required}
    3. Score = w1·street_ETA (from §6.3's CH router on the actual road graph, traffic-aware,
               with signal preemption if the unit is L&S)
           + w2·priority_mismatch + w3·workload_balance (a unit that has handled 6 calls this
               tour is deprioritised)
           + w4·specialty_fit
    4. Assign the top N by priority (P1: 2–4 units; P2: 1–2; P3: 1; P4: queued; P5: none)
    5. Notify over radio ⇒ the radio channel has CAPACITY: 412 units on 6 channels. A channel
       with > 14 active transmissions/min produces comms degradation: garbled messages,
       repeated transmissions, and a 3–11 s delay on acknowledgements. During a major incident
       the channel saturates and units stop coordinating — a real failure mode, modelled.
```

**Seven jurisdictions with real friction:**

| Agency | Sworn | Beat count | Primary mandate | Mutual-aid latency to VC PD |
|---|---|---|---|---|
| Vermillion City PD | 1,840 | 41 | City of VC | — |
| Halstead Bay PD | 312 | 9 | Halstead Bay | 105 s |
| County Sheriff | 604 | 22 | Unincorporated + valley + forest | 130 s |
| State Highway Patrol | 148 | 6 corridors | Freeways, state routes | 88 s |
| Port Authority Police | 96 | 4 zones | Port, Cordova Flats, harbour | 140 s |
| Transit Police | 74 | system-wide | Metro, rail, bus | 96 s |
| Anza Basin Marshal Service | 11 | 1 | Anza Basin, ghost town | 340 s |

Mutual aid requires a **request, an authorisation, and a handoff**, each with a delay. A pursuit
that crosses from VC into County jurisdiction produces a visible handoff: the VC units may or may
not continue (policy-dependent — a "no pursuit across jurisdiction without supervisor approval"
rule exists in VC PD and is enforced), the Sheriff is notified, and there is a 130 s window of
reduced coverage. **This is a designed exploit surface and it is honest about being one.**

### 3.12.5 Stage 4 — Response, five tiers

```
TIER 1 — SINGLE UNIT
  Trigger: P3/P4, or P2 with no second unit available
  Behaviour: travel (L&S for P1/P2 with signal preemption), arrive, approach tactically
    (a real approach pattern: stop 1.5 car lengths back and offset, observe for 4–8 s, exit on
     the kerbside, approach on the driver's side or the passenger side per agency policy and
     threat assessment), make contact, assess.
  Outcome space: warning · citation · arrest · report only · clear without action · escalate

TIER 2 — 2–3 UNITS, CONTAINMENT
  Trigger: P1/P2, or an in-progress crime with a suspect present
  Behaviour: pincer approach from separate axes (computed by the CAD, which knows both units'
    ETAs and assigns approach bearings to cut off egress), contact/cover roles (one officer
    makes contact, one covers — a real and universally taught doctrine), a 30 m reactive
    perimeter.

TIER 3 — CORDON
  Trigger: a barricaded suspect, a hostage event, an armed suspect in a defined area, a major
    crime scene, a hazardous materials release, or an evacuation
  Behaviour:
    · establish an INNER perimeter (hot zone, 40–120 m) and an OUTER perimeter (cold zone,
      150–400 m) from the actual street network: the CAD computes a minimum edge cut on the
      road graph that separates the incident cell from the network (§6.3's graph is reused —
      a max-flow/min-cut solve on ~900 links, 0.9 ms)
    · close each perimeter edge with a unit, a barricade vehicle, a portable barrier, or a
      spike strip. Each closure is a REAL MUTATION of the road graph: capacity → 0, which the
      macro traffic model (§3.8.1) sees immediately, so civilian traffic reroutes, queues
      propagate backwards at 21 km/h, and the player can watch the diversion happen from a
      rooftop.
    · establish a command post (an actual vehicle and an actual incident commander record with
      a span-of-control limit of 6 direct reports — exceeding it degrades coordination, which
      is a real ICS principle and a real failure mode at large incidents)
    · divert pedestrian flow (the §3.8.4 flow-field gets the perimeter as an obstacle)
    · request a supervisor (a sergeant, +3–9 min)
    · establish a media line if public opinion salience is high (§3.11.5)
    · hold for 20 min – 8 h depending on the situation

TIER 4 — NEGOTIATION
  Trigger: hostage, barricade with a communication channel, suicidal subject, or a suspect who
    has responded to contact
  Behaviour: a negotiation state machine driven by a SUBJECT STATE model:
    stress ∈ [0,1] advanced at 1 Hz by:
      + time elapsed without resolution (0.004/s)
      + perceived threat (snipers visible, armoured vehicle, flashbangs, a breach noise)
      + media presence (a camera visible raises stress 0.11)
      + demands refused
      − rapport built (a real mechanic: the negotiator's dialogue choices affect rapport, and
        rapport is a persistent value across the incident)
      − time spent talking (0.002/s)
      − physiological needs met (food, water, a phone, a cigarette — real negotiation tactics
        that buy time and lower stress)
      − fatigue (a subject who has been awake 30 h has a lower stress ceiling and a higher
        volatility)
    OUTCOMES at stress thresholds:
      stress < 0.25 for 120 s ⇒ high probability of peaceful surrender (0.71)
      stress 0.25–0.70        ⇒ prolonged; negotiable; outcomes branch on rapport
      stress > 0.85           ⇒ escalation risk 0.34/min: a hostage is harmed, or a breakout
                               attempt, or self-harm
      A SWAT breach is available at any time and resolves the incident in 4–11 s with a
        casualty probability that depends on stress, subject armament, and breach quality —
        and it is USUALLY WORSE than negotiating. The model is calibrated to published
        outcome data for barricade incidents, where patience outperforms entry.
    The negotiation team is 1 unit region-wide, on a 45-minute callout ⇒ for the first
    45 minutes, the incident is handled by a patrol supervisor with no negotiation training,
    and the outcomes are worse. **Resource scarcity is real.**

TIER 5 — SPECIALISED
  SWAT (2 teams, 45-min callout, 14 operators each, armoured vehicle 62 min)
  AIR SUPPORT (4 units: 2 VC PD helicopters, 1 Sheriff, 1 news — the news helicopter is not
    police and its presence raises subject stress and public salience)
    · flight is gated by WEATHER MINIMA: 300 m ceiling and 3 km visibility for VFR over a
      congested area, plus a 12 m/s crosswind limit for the skid gear. §3.2 determines
      whether air support can fly. During a marine layer event (M5, cloud base 300–700 m),
      air support is grounded for 3–7 hours. **A player who learns the marine layer schedule
      has learned how to rob a bank.**
    · sensor payload: FLIR (a real thermal model — a human at 33 °C skin against a 14 °C
      background is 8.4× above the noise floor and detectable through light foliage but not
      through a roof; a person who has been stationary in contact with cold ground for 20 min
      has a reduced signature and is genuinely harder to find), spotlight (2.4 Mcd, which
      also *reveals the aircraft's position* and raises subject stress), and a 40× daylight
      optic.
  K9 (3 units, 22-min callout): tracking uses the soil deform layer's footprints (§3.4.6) and
    the scent model — scent disperses downwind at 1.8× the wind speed, is destroyed by heavy
    rain (> 8 mm/h washes a track in 40 min), and persists 6–72 h depending on temperature and
    humidity, all from §3.2 and §3.3.1. **A crime committed in the rain and investigated 4 h
    later has no scent track.** A crime in the Anza Basin's dry cold has one for 3 days.
  DRONE (8 units, 12-min callout, 34-min endurance, 400 m ceiling, wind limit 11 m/s)
  MARINE (6 units, Halstead + Port Authority)
  HAZMAT (2 units, Cordova Flats)
  BOMB SQUAD (1 unit, 55-min callout)
```

### 3.12.6 Stage 5 — Investigation and evidence

```
EvidenceItem types and their generation:
  ballistic:  fired case (with firing-pin impression and ejector-mark striation) ⇒ linked to a
              specific WeaponRecord with probability = weapon.traceability (0.31–0.88 by type
              and condition). A revolver leaves no case. A suppressed weapon leaves one but
              no report signature. Two crimes fired from the same weapon are LINKED by the
              national database with a 34-day lag (a real backlog) unless a local examiner
              prioritises it.
  projectile: a recovered bullet ⇒ class characteristics (calibre, rifling twist and land/groove
              count) narrow the weapon set; individual characteristics require a comparison
              sample, which requires a suspect.
  biological: blood (⇒ DNA, a 6-week lab backlog or a 9-day priority), hair, saliva, skin cells
              under fingernails. Degrades: DNA in direct sun and rain loses viability with a
              half-life of 4.1 days (from §3.2's UV and precip).
  impression: footwear (⇒ matched to a shoe record in the ledger's wardrobe list — every
              resident owns shoes with a sole pattern) and tyre (⇒ matched to a VehicleRecord),
              both read from the soil deform layer (§3.4.6) and BOTH ERASED BY RAIN (§3.4.6).
  digital:    CCTV. The region has 14,200 cameras (public 1,880, transit 2,400, commercial
              6,900, private residential 3,020) with a baked coverage volume. Camera footage
              has a retention period (7–45 days by owner type), a resolution, a field of view,
              and a LOW-LIGHT PERFORMANCE — an infrared camera at night gives a
              face-recognition probability of 0.24 vs 0.71 in daylight. A player who commits a
              crime at 03:00 in a rainstorm in a district with poor camera coverage is
              substantially safer, and this is computable from the map.
              Also: ANPR (automatic number plate recognition) on 412 arterial gantries and all
              freeway exits, with a 90-day retention. Cell-site analysis (a warrant-required
              query that places a phone within 300–1,200 m of a base station at a time —
              useless in a dead zone, which is a defensive use of §3.12.3.1).
  physical:   fibres, glass fragments (refractive-index matched to a specific glazing assembly),
              paint transfer (matched to a specific vehicle colour code — 1 of 340 vehicles
              shares it), tool marks, ligature, accelerant residue (arson).
  testimonial: witness statements. Each witness's account is generated from their
              {perceived_at, certainty, memory_fidelity} and DEGRADES: memory_fidelity decays
              at 0.11/day and is corrupted by (a) post-event information (a witness who has
              spoken to another witness converges on a shared, possibly wrong, account —
              a real contamination effect), (b) stress at encoding (high stress narrows
              attention to the weapon — "weapon focus effect" — so a witness to an armed robbery
              reliably describes the gun and unreliably describes the face), and (c) the
              cross-race effect, modelled honestly as a fidelity penalty when the witness and
              subject differ in the witness's familiarity category. **A witness description is
              not a fact; it is evidence with a known error distribution**, and the sim treats
              it that way.
  forensic_limit: every evidence type has a collection probability (a scene processed by 2
              technicians recovers 0.61 of available evidence; by a rushed single technician
              under a P1 workload, 0.34) and a lab backlog (DNA 6 weeks, ballistics 11 days,
              toxicology 3 days, digital 9 days). Backlogs are a function of §3.11.5's budget.

INVESTIGATION STATE MACHINE (per CrimeEvent, 1/3600 Hz):
  scene_secured → scene_processed → evidence_logged → leads_generated →
    suspect_identified? → interview/lineup → charge_filed? → adjudication
    OR → uncleared (0.82 of burglaries, 0.62 of robberies, 0.31 of assaults, 0.19 of homicides
         remain uncleared — matching real clearance rates, and emerging from the evidence model
         rather than being set as a target)
  A cold-case review runs at 1/2,592,000 Hz (30 days) and can reopen if new evidence appears
  (e.g. the same weapon is recovered in an unrelated crime).
```

### 3.12.7 Stage 6–7 — Arrest, booking, adjudication

```
ARREST: requires probable cause, which is a MODELLED QUANTITY:
  P_cause = f(direct observation by an officer, witness identification confidence, physical
              evidence linking the subject, ANPR/CCTV placement, admission)
  An arrest with P_cause < 0.42 is a false arrest ⇒ the case is dropped at review, the subject
  is released, and the agency accrues a liability event (§3.11.5) that affects policy.
  ⇒ The police DO make mistakes, and they are penalised for them by the model.

BOOKING: 40–90 min at a station (a real interior, Class B, with a booking workflow that runs
  and that the player experiences if arrested: property inventory — the player's possessions
  become a ledger record and are returnable; photograph; fingerprints ⇒ entered into the
  database, which means the player is now IDENTIFIABLE for all future crimes, permanently.
  **Getting fingerprinted is a one-way door** and it is one of the game's most consequential
  systemic decisions.)

BAIL/RELEASE: a bail schedule by charge, an ability-to-pay check against the player's ledger
  economics, a pretrial-services risk assessment, and a release decision. A player with no money
  and a P1 charge spends 40–180 days in custody — during which the world continues (§3.9.4's
  ledger runs) and their life degrades (employment lost at 9 consecutive unexcused absences,
  housing lost at 2 missed payments, relationships decay). **Time in custody is a real cost
  measured in the world, not a fade to black.**

ARRAIGNMENT → PRETRIAL → ADJUDICATION:
  court dates on the §2.10 calendar, in the County Courthouse (Class S interior, §2.5.1), with
  14 courtrooms running a real docket (412 cases/day region-wide, of which 94% resolve by plea).
  Outcome = f(evidence strength, defence quality (a function of the player's money and of the
  public defender's caseload — 148 cases per attorney, which is 2.6× the recommended standard
  and which produces worse outcomes, emergent from §3.11.5's budget), prosecution priority,
  prior record, judicial disposition).
  Sentence → a correctional record. Incarceration removes the character from the world for its
  duration, with the ledger continuing.

CONSEQUENCE PERSISTENCE: a conviction is a permanent record entry affecting employment (a
  criminal-record check is part of the establishment hiring model), housing (a landlord
  screening model), credit, firearm eligibility, voting, travel, and — critically — how police
  treat the character on every future contact (a prior record raises the escalation probability
  in Tier 1–2 encounters by 2.4×, which is a modelled, documented, and uncomfortable real-world
  dynamic that design review has approved including because omitting it would be a worse
  distortion).
```

### 3.12.8 Budget

| Item | Cost | Rate |
|---|---|---|
| Crime event generation (loaded region) | 0.021 ms | 1/60 Hz |
| Perception evaluation for crime events | 0.14 ms | 1/60 Hz |
| Report model (active reports) | 0.03 ms | event-driven |
| CAD simulation (412 patrol + 114 other units) | 0.41 ms | 1/60 Hz |
| Coverage query (lazy, cached) | 0.06 ms | per query |
| Response AI (embodied officers, ≤ 18) | 0.52 ms | 1/60 Hz |
| Cordon graph solve (max-flow on 900 links) | 0.9 ms | event-driven |
| Negotiation state machine | 0.01 ms | 1 Hz |
| Investigation state machines (≤ 4,000 active cases) | 0.18 ms | 1/3600 Hz |
| Evidence decay / retention sweep | 0.09 ms | 1/86400 Hz |
| **Total** | **1.37 ms CPU** | |

Within §5.11.1's simulation budget. **The single most expensive item is the embodied response AI**
(0.52 ms for 18 officers), which is why the governor's first response to pressure is to reduce
simultaneous Tier-3 cordon units from 8 to 4 (a change that is *diegetically explained* by
understaffing — a governor action with an in-world justification, which is a design pattern we
apply throughout §5.11.3).

---

## 3.13 Perception

Shared by NPCs, wildlife, and police. One model, three consumers.

### 3.13.1 Vision

```
P(detect | target, observer, environment, Δt) = 
    P_resolved · P_contrast · P_attention · P_recognition · (1 − Π(1 − p_i·Δt))

P_resolved:  angular size of the target's critical dimension vs the observer's acuity.
             Johnson-criterion style: detection requires ≥ 1.5 "resolution cells" across the
             minimum dimension, recognition ≥ 6, identification ≥ 12.
             resolution_cell = 1/(60 · acuity) degrees; acuity = 1.0 nominal, degraded by
             age (0.72 at 70 y), intoxication (0.6 at 0.12 BAC), fatigue, and uncorrected
             vision (34% of the population wears correction, which is a wardrobe/asset state —
             an observer who has removed their glasses at home is functionally myopic).

P_contrast:  L_target vs L_background at the observer, both computed from the ACTUAL RENDERED
             LUMINANCE (§5.7's lighting output, sampled from a 1/8-res luminance buffer at
             10 Hz — this is a real cross-system dependency and it means that a player standing
             in shadow at night is genuinely harder to see, computed rather than faked), plus
             atmospheric extinction over the distance (from §3.2's visibility), plus
             precipitation scatter, plus the observer's dark adaptation state (a 22-minute
             rhodopsin regeneration curve; an observer who has just looked at a bright surface
             is temporarily night-blind — walking out of a lit shop into a dark street costs
             an NPC 40–90 s of visual capability).

P_attention: from the activity class (a driver has a narrow, road-biased attention cone; a
             person on a phone has 0.34× peripheral attention — and 71% of adults on the
             sidewalk at any time are on a phone, from a ledger behaviour distribution; a
             security guard on a fixed post has a high-attention narrow cone; a sleeping person
             has 0.02× visual attention but retains 0.6× auditory), plus a gaze direction with
             a saccade model (§3.10.5).

P_recognition: is the target a known entity? Requires the observer's relationship graph to
             contain the target, OR a prior `witnessed_events` entry with sufficient fidelity.
```

### 3.13.2 Hearing

Directly from §3.7.5's SPL field:
```
detect ⟺ SPL_at_observer(A-weighted) − noise_floor − hearing_threshold(observer) > 0
noise_floor = max over { traffic (§3.8), wind (§3.2), precipitation, HVAC, crowd, industrial }
              computed per cell, 5 Hz
hearing_threshold = 0 dB nominal; degrades with age (a 4 kHz notch, +14 dB at 70 y), with
              occupational exposure (a Cordova Flats refinery worker has a 22 dB loss at
              4 kHz — a real and specific audiometric signature, and it means they will not
              hear a suppressed pistol but will hear a low-frequency impact), and temporarily
              after a loud event (a temporary threshold shift of 18–42 dB decaying over
              40 min–16 h — **the player experiences this too**, §3.14)
```

### 3.13.3 Smell and other senses

Included where they matter: smoke (from §3.3.5's plume, detectable at 0.4–6 km downwind — a real
early-warning channel for fire), gas (a mercaptan leak at Cordova Flats), decomposition, food
(a restaurant's kitchen exhaust is a POI attractant), and **vibration** (a freight train at 400 m
is felt before it is heard; a structural collapse is felt at 900 m). Each is a scalar field or an
event query, and each costs nothing beyond a lookup.

---

## 3.14 Player embodiment

Pillar 6. The player character is a simulated body, not a camera.

```
Locomotion:   mass 78 kg, inertia, momentum with a 0.34 s acceleration lag, a gait selection
              from the motion-matching DB (§3.10.5) driven by desired speed and terrain, foot
              placement IK on terrain and props with a reach envelope, a stumble model when a
              foot placement fails, and a fall model (an active ragdoll with a protective
              reflex — hands out, a 0.42 s brace window that reduces injury by 61% if the
              player is not already at the fall-damage threshold).
Carry:        a 2-hand / 1-hand / holstered state per item; a load model (a 32 kg pack shifts
              the CoM, changes the gait, raises the metabolic cost 41%, and affects climbing);
              a grip-fatigue model (a ledge hang has a 34–180 s limit by load and by the
              character's strength attribute).
Reach:        a real reach envelope from the shoulder joint — an object 0.78 m away at 1.4 m
              height requires a step or a lean, and the animation system produces the lean.
              No magnetic pickup. No object teleportation to the hand.
Breathing:    driven by metabolic demand (an O₂ uptake model from activity intensity), altitude
              (§3.2's pressure: at Sable Pass, 1,612 m, VO₂max falls 11%; at Sable Peak,
              2,410 m, 19% — a real altitude penalty that makes the summit an objective),
              exertion debt, and stress (an audible breath rate that also affects aim stability
              and, underwater, gas consumption §2.9.2).
Thermal:      a 2-node (core, skin) thermoregulation model with clothing clo (§3.10.4),
              metabolic heat, and environmental exchange from §3.2 (T, wind ⇒ wind chill,
              humidity ⇒ heat index, solar load, and water immersion which is 25× more
              conductive than air). Outputs: shivering, sweating, a performance penalty, and —
              at the extremes — hypothermia and heat illness as ledger-tracked health states.
              Sable Peak in winter without appropriate kit is lethal in 90 minutes. That is
              approximately correct and it makes the mountain a real objective.
Nutrition:    caloric and hydration state over multi-day play. A deficit produces measurable
              performance decline (strength −8%, endurance −14%, cognition −11% at a 24 h fast)
              and residents have the same model, which is why a stranded NPC's condition
              degrades believably.
Injury:       the §3.6.5 body model, applied to the player. Limb-specific: a fractured forearm
              prevents two-handed weapon use; a thigh fracture prevents sprinting; a rib
              fracture makes every breath audible and degrades aim. Pain is a scalar that
              degrades fine motor control. Recovery is real time, tracked in the ledger, and
              accelerable only by actual medical care (a hospital, a splint, analgesics — each
              with its own effects, including opioid analgesics producing dependence over a
              14-day course, which is modelled because it is true and because not modelling it
              would be a distortion).
Senses:       the §3.13 model applied to the player, including dark adaptation (a real
              22-minute curve, presented as a rendering exposure adaptation that matches it),
              temporary threshold shift after loud events (§3.13.2), and the underwater
              auditory collapse (§3.7.7).
```

**Design consequence:** the player cannot do anything an NPC cannot do, and every capability the
player has is a consequence of the same models. This is what makes the world's reaction to the
player coherent — the player is a peer entity, not a privileged one.
