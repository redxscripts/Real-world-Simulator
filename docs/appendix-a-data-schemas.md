# Appendix A — Authoritative Data Schemas

Schemas for records **not** already inlined in §3. Where a schema is inlined in the body
(`MaterialRecord` §3.1.2, `ResidentRecord` §3.9.2, `WeatherCell` §3.2.2, `CrimeEvent` §3.12.1,
`WeaponRecord` §3.6.1), the body is authoritative and this appendix carries only the storage
layout.

Conventions: types are `u8/u16/u32/u64`, `i*`, `f32/f64`, `bool`, `str<N>` (a fixed N-byte UTF-8
field), `ref<T>` (a 64-bit handle), `enum[N]`, `arr<T,N>`, `vec3f`, `vec3d`. All records are
`#pragma pack(1)`-equivalent with explicit alignment comments. All are versioned by a `u16 version`
field at offset 0; a version mismatch is a hard load failure, never a silent reinterpretation.

---

## A.1 `DistrictZone` — zoning and gradient authoring (§2.4)

```
DistrictZone                                    # 41,200 blocks; 214 B each = 8.8 MB
  u16   version
  u32   zone_id                                 # stable, never reused
  str<24> name                                  # "Riverton East"
  u16   borough_id                              # ref into 15 districts (§2.5)
  enum  zone_code        : u8                   # §2.4.1's 22 codes (R-1 … U)
  u16   jurisdiction_id                         # §3.12.4's 7 agencies + sub-precinct
  u16   census_tract_id                         # §3.9.2's 412 tracts
  f32   land_value_mean                         # the §2.4.3 field's zone average
  f32   far                                   # floor area ratio
  u8    max_storeys
  f32   setback_front_m
  f32   trip_gen_rate                           # ITE-style daily trips / 1,000 m² GFA (§2.4.1)
  f32   noise_limit_day_dBA                     # §3.7.5's ordinance
  f32   noise_limit_night_dBA
  u16   fire_station_id                         # §3.3.5's suppression
  u16   police_beat_id                          # §3.12.4
  f32   impervious_fraction                     # §3.3.1's CN input
  u8    soil_hydrologic_group                   # A/B/C/D (NRCS)
  u8    biome_class                             # §2.3.2's 13 classes
  u8    style_family                            # §4.2.2's F01–F18
  u8    coverage_carriers : bitmask<3>          # §3.12.3.1's 3 carrier networks
  f32   gradient_mask[7]                        # §2.3.3's T5 seven channels at the zone centroid:
                                                # lot_area, building_age, height_storeys,
                                                # impervious, canopy, land_value, litter_density
  u32   parcel_count
  u32   structure_count
  u32   interior_count
  u32   resident_count                          # from §3.9.2's household assignment
  u32   employment_count                        # daytime, from §3.11.2
  vec2f centroid
  arr<u32, 8> boundary_block_ids                # a ring, for the T5 gradient solver
```

**Derived at cook, not stored:** the 4 m-resolution gradient mask fields (7 × f32 over 252 km² at
4 m = 7 × 15.75M × 4 B = 441 MB raw → quantised to 8-bit and RLE-compressed = **38 MB**, streamed
per G1 cell). The zone record stores only the centroid values; the fields are the authority.

## A.2 `ParcelRecord` (§2.4.2)

```
ParcelRecord                                    # 187,400 parcels; 41 B compressed = 7.7 MB
  u16   version
  u32   parcel_id
  u32   block_id
  u16   zone_id
  f32   area_m2
  f32   frontage_m
  u8    frontage_street_count                   # 1, or 2 for a corner lot (a 1.18× area multiplier)
  u8    structure_present : bool
  u32   structure_id                            # if present
  u16   household_id                            # §3.9.2, or 0xFFFF for non-residential
  u32   establishment_id                        # §3.11.2, or 0
  u8    parking_spaces
  f32   land_value                              # the §2.4.3 field sampled at the centroid
  u16   buildable_envelope_hash                 # an index into the envelope polygon set
```

## A.3 `StructureRecord` (§2.5, §4.3)

```
StructureRecord                                 # 83,400; 268 B = 22.4 MB
  u16   version
  u32   structure_id
  u32   parcel_id
  u8    style_family                            # F01–F18
  u8    era_band                                # 0–5 (§4.3.3)
  u8    structural_system                       # §4.3.1's 8 systems
  u8    placement_class                         # §4.5.2's P1/P2/P3
  u8    storeys_above
  i8    storeys_below                           # §2.6.3
  f32   height_m
  f32   floor_to_floor_m
  f32   footprint_area_m2
  f32   gfa_m2                                  # gross floor area
  f32   condition                               # §4.3.4's c ∈ [0,1]
  u32   interior_bundle_id                      # 0 if not enterable
  u8    interior_class                          # §4.2.1's S/A/B/C/D, or 0
  u16   wall_assembly_id                        # §3.4.2's 312 assemblies
  u16   roof_assembly_id                        # §3.4.2's 94
  u16   glazing_assembly_id                     # §3.4.2's 148
  u32   structural_graph_root                   # §3.4.4, or 0 if not collapse-capable
  u8    collapse_capable : bool                 # 8 structures region-wide (§3.4.4)
  u16   occupancy_load                          # a building-code quantity
  u16   parking_capacity
  u32   worker_count                            # §3.9.6's occupancy aggregate
  u32   resident_household_count
  f32   sky_view_factor                         # §3.2.5 M3's UHI input, 4-bit quantised
  u16   noise_contour_dBA                       # from the §2.4.3 traffic/air/rail model
  u16   elevator_count                          # §2.6.2
  u8    protected_landmark : bool
  vec3d position
  f32   orientation_rad
```

## A.4 `InteriorCell` / `InteriorBundle` (§4.4, §4.5.1)

```
InteriorBundle                                  # 37,436; 180 B = 6.5 MB (the §4.4.4 disk line)
  u16   version
  u32   bundle_id
  u32   structure_id
  u8    budget_class                            # S/A/B/C/D (§4.2.1)
  u8    placement_class                         # P1/P2/P3 (§4.5.2)
  u8    archetype_id_lo                         # §4.2.2's 412 archetypes
  u16   archetype_id
  u16   style_family
  u16   program_template_id                     # §4.3.1 Stage 1's 412 templates
  u64   generation_seed                         # hash(resident_id, parcel_id, storey, gen_version)
  arr<ref<ResidentRecord>, 4> occupant_ids      # ≤ 4 households per bundle
  u16   floorplan_graph_chunk                   # a streamed chunk ref (the §4.3.1 Stage 2 output)
  u16   room_graph_chunk
  u16   gi_probe_chunk                          # §4.7.1
  u16   acoustic_probe_chunk                    # §3.7.1
  u16   audio_bank_set_id
  u8    portal_count
  arr<u32, 4> portal_volume_ids                 # §4.5.6
  u8    entry_irradiance_probe_id               # §4.5.2's 120-state bake
  u16   override_patch_offset                   # §4.6.2's sparse patch list (avg 2.4 entries)
  u8    override_patch_count
  u32   shell_asset_hash                        # the paired exterior shell (§4.5.1)
  vec3d placement_origin                        # P1 = the structure position;
                                                # P2 = structure position with Z − 400 m;
                                                # P3 = a dedicated cell-range origin
  u16   private_memory_ceiling_MB               # §4.4.2, asserted by CI gate INT-01
```

## A.5 `LayerStack` (§3.4.2)

```
LayerStack                                      # 615 building + 74 vehicle-panel assemblies
                                                # (§3.4.2); 34 B + 8 B/layer
  u16   version
  u16   assembly_id
  u8    layer_count                             # ≤ 8
  arr<Layer, 8>:
      u16 material_id                           # §3.1.2's index
      u16 thickness_mm
      u8  state : enum                          # intact | cracked | breached | scorched |
                                                # spalled | perforated | missing
      u8  state_param                           # 0–255, the state's magnitude (crack density,
                                                # perforation radius, compression)
  f32   composite_stc                           # computed at cook from the layers, incl. the
                                                # mass-air-mass resonance (§3.7.2)
  f32   composite_R_value                       # thermal (§4.8's HVAC)
  u16   ballistic_r0_mm_equiv                   # the normal-incidence aggregate (§3.6.4)
```

Mutable state (a breached layer) lives in the G4 slab (§5.8.4), not in the assembly record.

## A.6 `VehicleRecord` (§3.5)

```
VehicleRecord                                   # 340 authored specs; 6.4 KB each = 2.2 MB
  u16   version
  u32   vehicle_id
  str<32> marque                                # §6.6.1: fictional, 34 marques
  str<32> model
  u16   model_year
  u8    class                                   # §3.5.1's 10 classes
  # --- mass properties ---
  f32   mass_kg
  f32   inertia[6]                              # Ixx Iyy Izz Ixy Ixz Iyz
  vec3f cog                                     # relative to the chassis frame
  f32   payload_capacity_kg
  # --- wheels & suspension ---
  u8    wheel_count                             # 4, 6, 8, 2 (motorcycle), 0 (marine/rail)
  arr<WheelSpec, 8>:
      vec3f hardpoints[19]                      # §3.5.4's hardpoint set
      f32   spring_rate_N_m
      f32   damper_curve[8]                     # §3.5.4's 4-parameter comp/rebound curves
      f32   bumpstop_rate_N_m, bumpstop_contact_m
      f32   antiroll_rate_N_m_rad
      u16   bushing_stiffness_id                # a 6×6 matrix ref
      f32   track_m
  # --- tyres ---
  u16   tyre_spec_id                            # §3.5.3's coefficient set
  f32   tyre_pressure_kpa_ref
  f32   tyre_mass_kg
  # --- powertrain ---
  u16   engine_spec_id                          # the T_map(RPM, throttle) 64×32 table + corrections
  u8    forced_induction                        # none | turbo | supercharged | twin-turbo
  f32   turbo_J_kg_m2, turbo_AR, wastegate_kpa  # §3.5.5
  u16   gearbox_spec_id                         # ratios, shift times, controller map
  u8    drivetrain                              # FWD | RWD | AWD | 4WD-part-time | 4WD-full | 6×6
  u16   diff_spec_id                            # open | LSD(preload, ramps) | locking | viscous |
                                                # electronic
  f32   driveline_inertia[3]                    # the §3.5.5 3-inertia torsional system
  # --- brakes ---
  arr<BrakeSpec, 4>:
      f32   rotor_mass_kg, rotor_diameter_mm, rotor_thickness_mm
      u8    pad_compound                        # §3.5.6's 3 μ(T) curves
      f32   caliper_piston_area_m2
  f32   brake_bias_front
  u8    abs_present, esc_present, tc_present    # §3.5.9's era-correctness
  # --- aero ---
  f32   cd, cd_a_m2, cl_front, cl_rear, cs_yaw_table[9]
  f32   side_area_m2, side_force_centroid_x
  # --- crashworthiness ---
  f32   crush_stiffness_N_m[6]                  # per impact direction (§3.5.10)
  u8    restraint_set                           # none | belt | belt+pretensioner | belt+airbag
  u8    ncap_era_class                          # 0–5, a decade-indexed structural standard
  # --- panel deformation ---
  arr<u16, 24> panel_assembly_ids               # §3.4.2's 74 vehicle panel assemblies
  arr<u16, 24> dent_basis_set_ids               # §3.4.3 model 1

VehicleState                                    # per instantiated vehicle; 412 B
  u32   vehicle_id
  ref<ResidentRecord> owner_id
  str<12> registration_plate
  u32   registration_expiry_tick
  u8    inspection_sticker_valid : bool
  f32   odometer_km
  f32   fuel_level_pct, fuel_grade              # §3.5.5's 4 octane grades
  f32   coolant_temp_K, oil_temp_K
  arr<f32, 4> rotor_temp_K, pad_temp_K, fluid_temp_K
  arr<f32, 4> tyre_temp_core_K, tyre_temp_tread_K, tyre_pressure_kpa, tyre_wear_mm
  arr<u8, 24> panel_deformation_state           # 0–255 per panel (§3.4.3)
  u8    lock_state, alarm_state, immobiliser_state
  u16   damage_history_count
  f32   clutch_wear, brake_pad_remaining_mm
  u8    coolant_system_integrity                # §3.5.5's blocked-radiator failure
```

## A.7 `DispatchUnit` and `DispatchOrder` (§3.12.4–§3.12.6)

```
DispatchUnit                                    # 412 patrol + 114 other; 148 B each
  u16   version
  u32   unit_id
  u8    agency_id                               # §3.12.4's 7 jurisdictions
  u8    unit_type                               # patrol | traffic | k9 | swat | air | marine |
                                                # negotiator | detective | supervisor | fire_engine |
                                                # fire_ladder | fire_tender | ems_ambulance |
                                                # ems_paramedic | hazmat | bomb | plough |
                                                # bulldozer | helitanker | air_tanker
  u8    status                                  # §3.12.4's state machine
  u8    capability_bitmask                      # 16 bits: air | marine | k9 | swat | hazmat |
                                                # negotiator | traffic_investigation | forensic |
                                                # breaching | aerial_flir | drone | armoured |
                                                # medical_advanced | crowd_control |
                                                # language_es | language_zh
  u32   assigned_call_id                        # or 0
  u16   watch_id                                # §3.12.4's 3 watches
  u16   tour_index                              # consecutive tours worked (a fatigue input)
  u32   meal_break_until_tick
  u32   out_of_service_until_tick
  f32   workload_tour                           # calls handled this tour (§3.12.4's balancing)
  u8    radio_channel_id                        # §3.12.4's 6 channels + a capacity model
  ref<ResidentRecord, 2> officer_ids            # 1 or 2 per unit
  ref<VehicleRecord> vehicle_id
  vec3d position
  f32   heading
  f32   speed

DispatchOrder                                   # ≤ 400 concurrent; 96 B each
  u16   version
  u32   order_id
  u32   crime_event_id                          # §3.12.1
  u8    priority                                # P1–P5 (§3.12.4)
  u8    response_mode                           # lights_sirens | silent | no_response |
                                                # lights_only
  u8    tier                                    # §3.12.5's 1–5
  u32   reported_at_tick, dispatched_at_tick
  f32   report_delay_s                          # §3.12.3's computed total (for validation)
  arr<u8, 6> report_delay_components            # T_decide, T_reach_comm, T_call_setup,
                                                # T_queue, T_calltaker, T_dispatch_parse
  u8    report_channel                          # mobile | landline | roadside_phone |
                                                # transit_help_point | fire_pull | walk_in |
                                                # officer_observed | anpr_alert | cctv_alert
  u8    coverage_class_at_report                # §3.12.3.1
  u16   psap_center_id                          # §3.12.3.2's 5 centres, 118 positions
  arr<u32, 8> assigned_unit_ids
  u8    jurisdiction_id, is_mutual_aid : bool
  u32   mutual_aid_requested_tick, mutual_aid_authorised_tick
  u8    cordon_state                            # none | computing | inner_only | full | stood_down
  u32   cordon_perimeter_link_count
  u8    air_support_requested : bool, air_support_denied_weather : bool
  u8    negotiation_state                       # §3.12.5 Tier 4
  f32   subject_stress, subject_rapport
  u8    outcome                                 # cleared | uncleared | warning | citation |
                                                # arrest | clear_no_action | escalated |
                                                # surrendered | breached | abandoned
```

## A.8 `EvidenceItem` (§3.12.6)

```
EvidenceItem                                    # ≤ 4,000 active cases × avg 4.2 items
  u16   version
  u64   evidence_id
  u32   crime_event_id
  u8    type                                    # ballistic_case | ballistic_projectile |
                                                # biological_blood | biological_dna |
                                                # biological_hair | impression_footwear |
                                                # impression_tyre | digital_cctv |
                                                # digital_anpr | digital_cellsite |
                                                # physical_fibre | physical_glass |
                                                # physical_paint | physical_toolmark |
                                                # physical_accelerant | testimonial |
                                                # forensic_comparison | documentary
  vec3d position
  u32   deposited_tick
  f32   integrity                               # 0–1, decayed by §3.2's UV/precip and by time
  f32   decay_half_life_days                    # DNA in sun/rain: 4.1 d; a footprint in mud:
                                                # the next rain event; a casing: never
  u32   source_ref                              # a WeaponRecord, VehicleRecord, ResidentRecord,
                                                # camera_id, or shoe/tyre pattern id
  f32   link_probability                        # §3.12.6's traceability for ballistics, a
                                                # match confidence for the rest
  u8    collected : bool, collected_by_unit_id
  u32   lab_submitted_tick, lab_result_tick     # the backlog model (§3.12.6)
  u8    admissible : bool                       # a chain-of-custody flag; §3.12.7
  f32   probative_weight                        # §3.12.7's outcome input
```

## A.9 `GovernorRule` (§5.11.3)

```
GovernorRule                                    # 26 knobs; 64 B each, shipped as data
  u16   version
  u8    knob_id                                 # §5.11.3's 1–25 + §6.3.5's 26
  u8    activation_order                        # the ladder position (a design decision)
  u8    perceptual_cost_class                   # invisible | low | medium | high
  u8    diegetic : bool                         # §5.11.3's '*' — an in-world justification exists
  u8    step_count                              # how many discrete steps
  arr<f32, 6> step_values                       # the value at each step
  arr<f32, 6> measured_saving_ms                # per step, measured not estimated
  u8    trigger_dimension                       # frame_time | gpu_time | cpu_time | memory |
                                                # thermal | io | thrash | navmesh_backlog
  u16   trigger_pool_id                         # for memory triggers, one of §5.10.1's 21 pools
  f32   engage_threshold                        # e.g. 0.92 for a memory utilisation
  f32   release_threshold                       # e.g. 0.84 — always below engage (hysteresis)
  u16   min_dwell_ms                            # 4,000 (§5.11.3's anti-oscillation)
  u16   lockout_ms                              # 60,000 on an oscillation detection
  u32   scene_tag_exclusions                    # a bitmask; HS-07 excludes knob 15 (§5.11.4)
```

## A.10 `DeltaJournalEntry` and the slab layout (§5.8.4)

```
DeltaJournalEntry                               # 48 B average; ~2.4M events/day of play
  u16   version
  u32   tick                                    # the authoritative WorldTime tick (§2.1)
  u32   cell_id                                 # the G4 16 m cell
  u16   entity_ref                              # a slab-relative index
  u8    mutation_type                           # prop_transform | damage | layer_stack |
                                                # litter | puddle_residue | graffiti | snow_depth |
                                                # burn_state | disturbance | light_state |
                                                # lock_state | evidence | rubble_pile |
                                                # structure_destroyed | terrain_displace
  u8    settle_class                            # protected | long | medium | short | ephemeral
  u32   settle_at_tick                          # §5.8.4's World Settle Pass
  u8    payload[32]                             # mutation-specific, a tagged union

StateSlab                                       # ≤ 64 KB hard cap; sparse, copy-on-write
  u16   version
  u32   cell_id
  u16   entry_count
  u16   snapshot_generation                     # incremented on each compaction
  u32   snapshot_offset, snapshot_length        # into the compacted snapshot arena
  u32   journal_offset, journal_length          # into the per-cell journal ring
  u8    priority_class                          # drives eviction (§6.2.3)
  u8    protected_entry_count                   # never settled, always persisted

Compaction: every 600 ticks, per cell —
    snapshot' = apply(snapshot, journal[0..k])
    journal'  = journal[k..]
  where k is chosen so snapshot' ≤ 18 KB and journal' ≤ 4 KB.
  A slab exceeding 64 KB after compaction triggers the overflow ladder (§6.4 #9).
```

## A.11 `StreamingCellManifest` (§5.8.1)

```
CellManifest                                    # one per cell per grid; 68 B; 1.73M cells = 118 MB
  u16   version
  u8    grid                                    # G0–G4 (§5.8.1)
  u8    z_band                                  # §6.3.4; 0 for surface
  u32   cell_index                              # a Morton code within the grid
  u32   chunk_count
  u64   chunk_ids_offset                        # into the manifest's chunk-id array
  u32   total_bytes                             # the §6.2.3 cost function's `size`
  u16   pso_set_id                              # §5.12.4's prewarm set
  u16   derived_structures_bitmask              # navmesh | coverage_fine | acoustic_probes |
                                                # gi_sdf | structural_graph | flow_field
  f32   derived_build_ms                        # the §6.2.3 cost function's `derived_build_time`
  f32   perceptual_penalty                      # the §6.2.3 cost function's weight
  u8    spine_link : bool                       # §5.8.5's highway spine
  u8    no_evict_candidate : bool               # a mission anchor or landmark
  u16   hysteresis_band_pct                     # §6.2.2 #2's boundary band
```

## A.12 Storage summary

| Record | Count | Size each | Total | Resident |
|---|---|---|---|---|
| `MaterialRecord` (§3.1.2) | 2,412 | 484 B hot + 1.6 KB cold | 5.0 MB | 1.2 MB hot, cold streamed |
| `ResidentRecord` (§3.9.2) | 4,127,000 | 312 B hot + 1.9 KB cold | **7.8 GB cold store** (a 1.29 GB hot block if fully resident — it is not) | **273 MB** (§3.9.7: a 124 MB active-set heap + a 149 MB dormant 40 B array) |
| `WeatherCell` (§3.2.2) | 1,584 | 96 B | 152 KB | all |
| `DistrictZone` (A.1) | 41,200 | 214 B | 8.8 MB | all |
| Gradient mask fields (§2.3.3) | 15.75M × 7 | 8-bit RLE | 38 MB | per G1 cell |
| `ParcelRecord` (A.2) | 187,400 | 41 B | 7.7 MB | per cell |
| `StructureRecord` (A.3) | 83,400 | 268 B | 22.4 MB | per cell |
| `InteriorBundle` (A.4) | 37,436 | 180 B | 6.5 MB | all (manifest only) |
| `LayerStack` (A.5) | 689 (615 building + 74 vehicle panel) | ~98 B | 68 KB | all |
| `VehicleRecord` (A.6) | 340 | 6.4 KB | 2.2 MB | all |
| `VehicleState` (A.6) | ≤ 340 active + 9,000 T4 | 412 B | 3.9 MB | 3.9 MB |
| `DispatchUnit` (A.7) | 526 | 148 B | 78 KB | all |
| `DispatchOrder` (A.7) | ≤ 400 | 96 B | 38 KB | all |
| `CrimeEvent` (§3.12.1) | ~1,840/day, ≤ 4,000 active | 224 B | 0.9 MB | active set |
| `EvidenceItem` (A.8) | ≤ 16,800 | 88 B | 1.5 MB | active set |
| `GovernorRule` (A.9) | 26 | 64 B | 1.7 KB | all |
| `StateSlab` (A.10) | ~66,600 allocated | ≤ 64 KB | — | 192 MB (§5.8.4) |
| `CellManifest` (A.11) | 1,734,172 | 68 B | 118 MB | all (a 118 MB always-resident index is the price of 5 grids; it is 0.9% of the PS5 budget and it is what makes the scheduler O(candidates) rather than O(cells)) |
| Structural graphs (§3.4.4) | 14.2M nodes | 6.2 B | 88 MB | 6 MB (40 subgraphs) |
| Navmesh (§6.3.6) | — | — | 1.25 GB (disk) | 63 MB |
| Acoustic probes (§3.7.1) | 1.94M | 4.2 KB | 8.1 GB (disk) | 240 MB per loaded region |
| Coverage volumes (§3.12.3.1) | 246,000 × 3 | 1 B | 738 KB coarse + 2 m fine | 3 MB |
| Climatology (§3.2.7) | — | — | 340 MB (disk) | 22 MB |

**Ledger total: 7.8 GB on disk, 273 MB of records resident** — the two numbers that make USP-1
real. The 273 MB sits inside the **372 MB ledger pool** (§5.10.1), which additionally holds 9.97 MB
of T0–T3 agent structs and density graph and 89 MB of relationship-graph adjacency (the 38% of
edges with `strength > 0.15`; weak ties page in on endpoint promotion). Naive residency would need
1,287 MB for the hot array plus 233 MB for the graph — 1.52 GB, which does not fit on any target.
The two-level residency design and the paged relationship graph are what make 4,127,000 persistent
residents arithmetically possible, and the 372 MB line is **identical on every platform** (§C.2.1).
**World index total (manifests + zones + parcels + structures + bundles): 163 MB disk, 163 MB
resident** — the fixed cost of addressing a 252 km² world, and 1.3% of the PS5 budget.
