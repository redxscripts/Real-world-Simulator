# Appendix B — Load-Bearing Algorithms

Pseudocode for the thirteen algorithms on which the architecture rests. Language is deliberately
C-like and explicit about determinism-relevant details (ordering, seeding, accumulator shape),
because those are the parts that are expensive to get wrong later (§5.4.3).

---

## B.1 Cell streaming scheduler with cost-aware eviction

Implements §5.8.2, §5.8.3, §6.2.3. Runs at phase P5 (§5.3), ≤ 1.20 ms across 2 workers.

```
CONSTANTS
  GRID[]        = { G0:4096m, G1:1024m, G2:256m, G3:64m, G4:16m }          # §5.8.1
  RADIUS[]      = { G0:always, G1:3200, G2:620, G3:220, G4:96 }            # metres
  MAX_INFLIGHT  = 32 (console) | 48 (PC)
  MIN_DWELL     = 2.0 s
  SCORE_CACHE_TTL = 4 frames                                                # §5.8.3 step 3
  EVICT_BATCH   = true                                                      # §6.2.3

fn scheduler_tick(frame):
    # ---- 1. update sources (§5.8.2) ----
    for s in SIM_SOURCES sorted_by s.id:                # DETERMINISTIC ORDER
        s.pos      = authoritative_position(s.entity)
        s.vel      = velocity(s.entity)
        s.altitude = s.pos.z - terrain_height(s.pos.xy)

        # altitude / speed policy (§5.8.5(a))
        if s.altitude > 400 or |s.vel| > 62:
            s.profile = AIRBORNE     # G4→0, G3→0, G2→1400 shells-only, G1→8000
        elif s.altitude > 100 or |s.vel| > 40:
            s.profile = FAST
        else:
            s.profile = NORMAL

        # velocity-scaled radius and ellipsoidal bias (§6.2.2 #1)
        s.r_scale = 1.0 + clamp(0.9 * |s.vel| * 6.0 / RADIUS[G3], 0, 3.5)
        s.axis    = normalize(s.vel) if |s.vel| > 3 else UNIT_Z

    # ---- 2. enumerate candidates (spatial hash per grid) ----
    candidates = []
    for g in [G1, G2, G3, G4]:                          # G0 is handled separately (§5.6.3)
        for s in SIM_SOURCES:
            r = RADIUS[g] * s.r_scale * s.profile_radius_mult[g]
            for cell in hash_query(g, s.pos, r, s.axis, s.r_scale):
                if cell.z_band not in z_bands_for(s, g): continue      # §5.8.5(c)
                candidates.push((g, cell, s))

    # ---- 3. score (cached, 10 Hz effective) ----
    for (g, cell, s) in candidates:
        if cell.score_valid and cell.source_delta[s.id] < 4.0 m:
            continue                                                 # reuse the cached score
        cell.score[s.id] = score_cell(g, cell, s)
        cell.source_delta[s.id] = 0
    # the score cache invalidation is the mitigation for §5.8.3 step 3's 3.4 ms → 0.32 ms

    # ---- 4. merge VT / audio / animation page requests into ONE queue (§6.2.2 #6) ----
    queue = PriorityQueue()
    for (g, cell, s) in candidates where cell.score[s.id] > 0:
        queue.push(Priority(classify(cell.score[s.id]), cell))
    for req in vt_feedback_buffer():                                 # §5.6.2
        queue.push(Priority(P3, req.page))
    for req in audio_animation_requests():
        queue.push(Priority(P4, req.page))
    queue.push_all(journal_offload_requests(), Priority(P5))

    # ---- 5. issue I/O (§5.9.4) ----
    issued = 0
    inflight_chunk_set = {}                                          # dedup (§6.2.2 #7)
    while issued < MAX_INFLIGHT and not queue.empty():
        item = queue.pop()
        if item.priority == P5 and any_outstanding(P0..P2): break    # §5.9.4 throttle
        chunks = item.chunks.filter(c => c not in inflight_chunk_set
                                       and c not in chunk_cache)
        if chunks.empty(): continue
        # coalesce adjacent chunks up to 256 KB, floor at 16 KB (§5.9.2)
        for req in coalesce(chunks, max=256_KB, min=16_KB):
            io_submit(req, item.priority)
            inflight_chunk_set.add_all(req.chunks)
            issued += 1

    # ---- 6. process completions ----
    for c in io_completions():                                       # ordered by completion
        decompress_offthread(c)                                      # DPU / IO coprocessor / 2 workers
        for sub in c.subsystems:
            sub.registrar.stage(c)                                   # staged, not applied
    drain_command_buffer()                                           # at P1-commit (§5.2.6) —
                                                                     # a cell becomes visible at a
                                                                     # deterministic frame boundary
    for c in completed_cells:
        prewarm_pso(c.pso_set_id, on=BACKGROUND_POOL)                # §5.12.4
        build_derived(c)                                             # navmesh link-up, coverage,
                                                                     # acoustic registration, GI SDF
        c.resident_since = now()

    # ---- 7. evict (one batch pass, §6.2.3) ----
    for pool in MEMORY_POOLS:
        if pool.utilisation < pool.target: continue
        evict_batch(pool)


fn score_cell(g, cell, s) -> f32:
    d = distance_metric(cell, s)                # ellipsoidal if s.r_scale > 1
    r = RADIUS[g] * s.r_scale * s.profile_radius_mult[g]
    need   = clamp(1 - (d - r) / r, 0, 1) ** 1.4
    vis    = 1.0 if cell.in_frustum(s.camera) else 0.15
    pred   = predicted_need_20s(cell, s)                            # §5.8.5's heat map
    play   = cell.gameplay_importance           # 3.0 anchor | 1.2 POI | 0.8 landmark | 0
    vel    = 1.6 if dot(cell.dir_from(s), s.axis) > 0.6 and |s.vel| > 12 else 1.0
    spine  = 1.35 if cell.spine_link and aligned(cell, s) else 1.0  # §5.8.5
    cost   = cell.total_bytes / MB_PER_SECOND_THROUGHPUT
    age    = 0.8 if (now() - cell.resident_since) < MIN_DWELL else 0 # §6.2.2 #2
    return (W_NEED*need + W_VIS*vis + W_PRED*pred + W_PLAY*play) * vel * spine - W_COST*cost - age


fn evict_batch(pool):
    # GreedyDual-Size with Frequency (§6.2.3). ONE pass, not one item at a time.
    L = pool.clock                                   # the aging clock
    evictable = pool.items.filter(i =>
                    (now() - i.resident_since) >= MIN_DWELL          # min dwell
                and not i.in_no_evict_set                            # §6.2.3 constraints
                and not i.has_live_tier0_physics_island
                and not i.is_player_containing_bundle)
    if evictable.empty():
        telemetry(POOL_OVERCOMMITTED); return                        # a governor memory response
    for i in evictable:
        i.prio = L + (i.freq_60s_decayed * reload_cost(i)) / i.size_bytes
    evictable.sort_ascending_by(prio, then by id)                    # DETERMINISTIC TIE-BREAK
    freed = 0
    for i in evictable:
        if pool.utilisation_after(freed) < pool.release_threshold: break
        freed += pool.release(i)                                     # atomic for a bundle (§4.5.1)
        pool.decommit(i.pages)                                       # §5.2.5, returns memory to OS
    pool.clock = max(L, evictable.last().prio)
    if freed < pool.bytes_needed * 0.94: telemetry(EVICTION_YIELD_LOW)   # §6.2.1's gate


fn reload_cost(cell) -> f32:                                          # milliseconds
    return  cell.total_bytes / measured_drive_throughput_bytes_per_ms
          + decompression_time(cell)
          + cell.derived_build_ms
          + cell.perceptual_penalty
```

## B.2 Cluster selection, culling, and raster-path assignment

Implements §5.5.4–§5.5.6. GPU compute, persistent threads.

```
fn cull_pass_A(frame):                                                # uses the PREVIOUS HZB
    # Level 1: instances
    for i in parallel_over(resident_instances):                       # ~380,000
        aabb = i.bounds_world
        if not frustum_test(aabb, view):                 continue     # rejects ~74%
        if hzb_occluded(aabb, prev_hzb):                 continue     # rejects ~61% of survivors
        if projected_size_px(aabb) < 0.25:               continue     # rejects ~9%
        dag_stack.push(i.dag_root)

        # Level 2: DAG traversal, front-to-back
        while not dag_stack.empty():
            n = dag_stack.pop_front()                                 # ordered by distance
            if hzb_occluded(n.bounds_world, prev_hzb):   continue
            eps_px = (n.geometric_error * screen_h)
                     / (2 * distance(n, camera) * tan_half_fov_y)       # §5.5.4
            if eps_px <= ERROR_THRESHOLD:                             # 0.5 px default;
                for c in n.clusters:                                  # governor knob 15 range
                    if not frustum_test(c.bounds, view):    continue   # 0.35 – 1.6 px
                    if hzb_occluded(c.bounds, prev_hzb):    continue
                    if backcone_culled(c.normal_cone, view): continue  # rejects ~44%
                    # ---- raster path assignment (§5.5.4) ----
                    area_px2 = c.screen_area_px2 / c.triangle_count
                    path     = HW if area_px2 >= CROSSOVER_PX2 else SW # 32.0 default
                    visible_list.append({
                        cluster_offset: c.offset,
                        instance_id:    i.id,
                        path:           path,
                        material_runs:  c.material_run_count,          # ≤ 4 (§5.5.2 step 5)
                        variant_index:  variant_buffer[c.offset]       # §5.5.8 mechanism 1
                    })
                else:
                    for child in n.children sorted_by distance:        # a node is a parent in the
                        dag_stack.push(child)                          # DAG, not a tree (§5.5.2)
    build_hzb(current_depth)                                          # from pass A's output

fn cull_pass_B(frame):                                                # uses the CURRENT HZB
    for r in pass_A_occlusion_rejects:                                # ~12% of pass A's rejects
        if not hzb_occluded(r.bounds, current_hzb):
            visible_list.append(r)


fn software_raster(tile):                                             # §5.5.5, one workgroup/tile
    shared local_depth[32][32]  = MAX_UINT32
    shared local_payload[32][32]= 0
    for tri in tile.triangle_list:                                    # from the binning pass
        # fixed-point 16.16 edge functions — MATCHES the HW top-left fill rule exactly (§6.5.2)
        e0, e1, e2 = edge_functions_fp16(tri.v0, tri.v1, tri.v2)
        if signed_area_fp16(e0, e1, e2) <= 0: continue                # backface
        for y in lane_y_range(tri.bbox ∩ tile):                       # 32 lanes, one span each
            x_start, x_end = scanline_span_fp16(y, e0, e1, e2, TOP_LEFT_RULE)
            for x in [x_start, x_end]:
                z = interpolate_depth_fp16(x, y, tri)
                if z < local_depth[y][x]:
                    local_depth[y][x]   = z
                    local_payload[y][x] = tri.cluster_slot << 7 | tri.index_within_cluster
    # ONE atomic per surviving pixel, not per pixel-per-triangle (§5.5.5's 7.4× reduction)
    for (x, y) in tile where local_depth[y][x] < MAX_UINT32:
        atomic_min_u64(visibility_buffer[gx(x), gy(y)],
                       (u64)local_depth[y][x] << 32 | local_payload[y][x])


fn material_resolve(frame):                                           # §5.5.6
    histogram = zero[64]                                              # the hard bin cap
    for px in parallel_over(pixels, stride=1):
        v = visibility_buffer[px]
        if v == 0: continue
        run = lookup_material_run(v.cluster_slot, v.triangle_index)    # ≤ 4 runs per cluster
        atomic_add(histogram[run.material_shader_id], 1)
    offsets = exclusive_prefix_sum(histogram)
    for px in parallel_over(pixels):
        v = visibility_buffer[px]
        if v == 0: continue
        run = lookup_material_run(v.cluster_slot, v.triangle_index)
        slot = atomic_add(offsets[run.material_shader_id], 1)
        bin_pixel_lists[run.material_shader_id][slot] = px
    for bin in parallel_over(non_empty_bins):                          # ≤ 64, typically 11–19
        shade_bin(bin)                                                 # writes the GBuffer (§5.5.6)
```

## B.3 Resident tier promotion / demotion

Implements §3.9.3–§3.9.5. The invariant tested by the promotion-continuity harness (§6.7.2).

```
CONSTANTS
  TIER_RADIUS_UP   = { T0:64,  T1:300,  T2:2000,  T3:macro_region }
  TIER_RADIUS_DOWN = { T0:71,  T1:330,  T2:2200,  T3:macro_region + 200 }   # 10% hysteresis
  DWELL            = 2.0 s
  OSC_GUARD_COUNT  = 4
  OSC_GUARD_WINDOW = 60 s
  OSC_GUARD_PIN    = 60 s

fn continuum_tick(frame):                                             # 5 Hz for tier evaluation
    sources = SIM_SOURCES sorted_by id
    for rec in ACTIVE_RECORDS sorted_by next_transition, then id:     # DETERMINISTIC
        d = min_distance_to_any_source(rec.position, sources)
        src = argmin_source(rec.position, sources)

        target = tier_for_distance(d, rec, src)

        # force-promotion by plan reference (§3.9.5)
        if rec.id in REFERENCED_BY_INSTANTIATED_PLAN:                 # e.g. a T0 agent's
            target = max(target, T1)                                  # scheduled meeting partner

        # dwell and oscillation guards
        if (now() - rec.last_tier_change) < DWELL: target = rec.tier
        if rec.tier_change_count_in(OSC_GUARD_WINDOW) >= OSC_GUARD_COUNT:
            target = max(target, rec.tier)                            # pin at the higher tier
            rec.pinned_until = now() + OSC_GUARD_PIN
        if now() < rec.pinned_until: target = max(target, rec.tier)

        if target == rec.tier: continue

        # ---- transition ----
        if target > rec.tier:   promote(rec, target)
        else:                   demote(rec, target)
        rec.last_tier_change = now()
        rec.tier_change_history.push(now())


fn promote(rec, target):
    while rec.tier < target:
        match rec.tier:
          T4 -> T3:  # no work: the record contributes to its link's density (§3.9.3)
                     rec.tier = T3

          T3 -> T2:  # INSTANTIATION — identity drawn, not invented (§3.9.5)
                     link = road_or_sidewalk_link_at(rec.position)
                     sample = stratified_deterministic_sample(
                                 link.id, sim_hour(),                # the seed
                                 link.density_profile,               # the OD-flow attribute
                                 count = link.micro_quota)
                     # χ² tolerance check on the aggregate attributes (§3.9.5)
                     assert chi2_within_tolerance(sample, link.od_flow_profile)
                     agent = instantiate_T2(rec, position=sample.position_for(rec.id))
                     agent.gait_phase = vat_phase_from(rec.position, agent.speed)
                     rec.tier = T2

          T2 -> T1:  agent = upgrade_T2_to_T1(rec)
                     # WARM the animation state from the VAT phase so the gait is continuous
                     agent.anim_state = blend_from_vat(agent.gait_phase, tolerance=0.1)
                     crowd_solver.add_capsule(agent)
                     agent.plan_cursor = rec.plan_cursor              # CONTINUITY
                     rec.tier = T1

          T1 -> T0:  agent = upgrade_T1_to_T0(rec)
                     agent.planner = allocate_planner()
                     agent.ik = allocate_ik_rig()
                     agent.cloth = allocate_cloth(800..2500 particles by garment)
                     context_settle(agent, duration=0.25 s)           # invisible at 64 m
                     rec.tier = T0

        # continuity assertion (§3.9.5) — the CI-tested invariant
        assert |new_position - old_position| <= POS_TOL[rec.tier]     # 0.6 m at T2→T1,
        assert new_activity == old_activity                           # 0.1 m at T1→T0


fn demote(rec, target):
    while rec.tier > target:
        match rec.tier:
          T0 -> T1:  interp_out(rec.agent, from=T0_repr, to=T1_repr, over=0.4 s, cubic)
                     write_back_state(rec.agent -> rec)               # §3.9.5's state write-back
                     release_planner_ik_cloth(rec.agent)
                     rec.tier = T1
          T1 -> T2:  interp_out(...); crowd_solver.remove_capsule(rec.agent)
                     rec.plan_cursor = rec.agent.plan_cursor
                     rec.tier = T2
          T2 -> T3:  # fold the individual back into the link density
                     link = road_or_sidewalk_link_at(rec.position)
                     link.density += 1
                     link.attribute_histogram.add(rec.attributes)     # so the aggregate stays
                     release_agent(rec.agent)                         # consistent with who left
                     rec.tier = T3
          T3 -> T4:  rec.tier = T4                                   # record only
```

## B.4 Crime report delay with cellular coverage and PSAP queueing

Implements §3.12.3. The falsifiable test is §1.4 USP-5(a,b).

```
fn compute_report_delay(crime, witness) -> ReportResult:
    # ---- T_decide: the witness decides whether and when to report (§3.12.3) ----
    base = base_reaction_s(crime.severity)                  # 2–6 s for an in-progress homicide;
                                                            # 60–400 s for a discovered burglary
    n_others = count_witnesses_excluding(crime, witness)
    acquainted = relationship_graph_are_acquainted(witness, crime.witness_ids)

    bystander = 1.0
    if n_others > 0:
        bystander = acquainted ? 0.60 : (1.0 + 0.35 * log2(n_others + 1))   # DIFFUSION OF
                                                                            # RESPONSIBILITY
    attitude = attitude_factor(witness.law_attitude)        # 0.5× at 0.95, 2.4× at 0.20
    victim   = witness.prior_victimisation > 0 ? 1/1.6 : 1.0
    psych    = psych_factor(witness.risk_tolerance, witness.neuroticism)

    T_decide = base * bystander * attitude * victim * psych

    # non-report probability (§3.12.3)
    if crime.offender_id in witness.relationships and crime.severity >= 3:
        if pcg32(hash(crime.id, witness.id, 'fear')) < 0.42:
            return ReportResult(NOT_REPORTED, reason='fear_of_reprisal')
    if witness.law_attitude < 0.30:
        if pcg32(hash(crime.id, witness.id, 'attitude')) < 0.31:
            return ReportResult(NOT_REPORTED, reason='low_law_attitude')

    # ---- T_reach_comm: the communications model (§3.12.3) ----
    has_mobile = witness.assets.contains('mobile_phone')
                 # P = 0.92 overall; 0.98 age<40; 0.71 age>75; 0.34 unhoused; occupation-modified
    cov_class  = query_coverage(witness.position, witness.carrier)      # §3.12.3.1, B.12
    battery_ok = witness.device_battery > 0.04
    device_ok  = not witness.device_destroyed

    if has_mobile and cov_class >= COVERAGE_VOICE and battery_ok and device_ok:
        T_reach  = uniform(1.5, 4.0, seed=hash(crime.id, witness.id, 'reach'))
        channel  = 'mobile'
    else:
        cp = nearest_comm_point(witness.position, poi_graph)
        # comm points: a landline in a residence/business (requires entry or being let in),
        # 7 police call boxes, 34 roadside phones, 112 transit help points,
        # 1,880 fire pull stations (report a FIRE, not a crime)
        if cp == null or cp.travel_time_min > 25:
            return ReportResult(NOT_REPORTED, reason='no_comm_point_within_25min')
        T_reach = cp.path_time_s                                     # from §6.3's L0/L2 query
                + cp.entry_time_s                                    # a door, a lock, a business
                                                                     # being open
                + cp.availability_wait_s                             # is anyone home / open?
        channel = cp.type

    # ---- T_queue: the PSAP as an M/M/c queue with Erlang-C (§3.12.3.2) ----
    c = psap_positions[crime.jurisdiction_id]                        # 118 region-wide across 5
    lam = psap_call_arrival_rate(crime.jurisdiction_id, now())       # baseline 0.9/s region-wide;
                                                                     # 8–40× during a mass incident
    h = 84 s                                                         # the mean handle time
    A = lam * h / 60                                                 # offered traffic, Erlangs
    util = A / c
    if util >= 1.0:
        T_queue = 180 s                                              # the model CLAMPS; the queue
        abandoned = true                                             # diverges — a real 911 meltdown
    else:
        P_wait = erlang_c(c, A)
        T_queue = P_wait / (c / h - lam)
        abandoned = (pcg32(hash(crime.id, 'abandon')) < P_wait * 0.18)

    # ---- T_calltaker and T_dispatch_parse (§3.12.3.2) ----
    state_mult = 1.0 + 0.8*witness.intoxication + 1.2*(witness.stress > 0.8 ? 1 : 0)
    T_calltaker = uniform(20, 70, seed=...) * state_mult
    location_fidelity = 1.0 / state_mult                             # a panicking caller gives a
                                                                     # worse address (§3.12.3.2)
    T_parse = uniform(8, 40, seed=...)

    total = T_decide + T_reach + uniform(2,6) + T_queue + T_calltaker + T_parse

    return ReportResult(REPORTED, delay_s=total,
        components=[T_decide, T_reach, uniform(2,6), T_queue, T_calltaker, T_parse],
        channel=channel, coverage_class=cov_class, psap=crime.jurisdiction_id,
        abandoned=abandoned, location_fidelity=location_fidelity)


fn erlang_c(c: int, A: f64) -> f64:
    # C(c,A) = [A^c/c! · c/(c−A)] / [Σ_{k<c} A^k/k! + A^c/c! · c/(c−A)]
    # computed in log-space to avoid overflow at c = 118, A = 1008 (§3.12.3.2's spike case)
    log_num = c*log(A) - log_factorial(c) + log(c/(c-A))
    sum = 0.0
    for k in 0..c-1: sum += exp(k*log(A) - log_factorial(k))
    log_last = c*log(A) - log_factorial(c) + log(c/(c-A))
    denom = sum + exp(log_last)
    return exp(log_num) / denom
```

## B.5 Tyre magic formula with combined slip, thermal, and brake fade

Implements §3.5.3 and §3.5.6. Runs at 1,000 Hz for V0/V1 vehicles (§3.5.2).

```
fn tyre_solve(w, dt, surface, puddle_depth, air):        # w = a wheel state, SoA across 4 wheels
    # ---- 1. slip quantities ----
    v_x = max(|w.v_longitudinal|, 0.5)                            # a floor to avoid a 0/0 at rest
    kappa = (w.omega * w.r_effective - w.v_longitudinal) / v_x    # longitudinal slip ratio
    alpha = atan2(w.v_lateral, v_x) - w.steer_angle               # slip angle (rad)
    gamma = w.camber                                              # from §3.5.4's kinematic solve

    # ---- 2. the friction coefficient (§3.5.3) ----
    cond = surface_condition(surface, puddle_depth)
           # dry | wet | icy | muddy | oily | snow_packed | snow_loose | gravel | sand |
           # leaf_litter | painted_marking | manhole_cover | tram_rail
    mu0 = surface.material.friction_kinetic[cond]                 # THE MATERIAL ONTOLOGY (§3.1)
    f_T = bell(w.tread_temp_K, w.tyre_spec.T_opt,
               w.tyre_spec.mu_ratio_cold, w.tyre_spec.mu_ratio_hot)
    f_v = 1 - 0.0086 * ln(1 + |w.v_slip| / 20)
    f_p = (w.pressure / w.spec.p_ref) ** (w.pressure < w.spec.p_ref ? 0.12 : -0.05)
    f_w = wear_curve(w.tread_depth_mm, cond)                      # 8mm→0.94, 1.6mm→0.71,
                                                                  # 0mm→0.58 in the wet
    mu  = mu0 * f_T * f_v * f_p * f_w

    # ---- 3. aquaplaning (§3.3.3, Horne) ----
    if cond == 'wet' and puddle_depth > 2.5 mm:
        Vp_kmh = 16.66 * sqrt(w.pressure_kPa / 6.895)
        Vp_kmh *= 1.0 - 0.22 * clamp((puddle_depth - 10 mm)/90 mm, 0, 1)
        ratio = clamp(w.speed_kmh / Vp_kmh, 0, 1)
        mu *= 1 - ratio**4                                        # gradual onset
        w.aligning_torque_scale = 1 - clamp((ratio - 0.7)/0.3, 0, 1)   # "the steering goes light"

    # ---- 4. the Magic Formula, lateral and longitudinal (§3.5.3) ----
    Fz = w.vertical_load                                          # from §3.5.4's suspension solve
    Dy = mu * Fz + w.spec.S_Vy
    Cy = w.spec.a1                                                # ≈ 1.30
    Ky = Fz * w.spec.a3 * sin(2*atan(Fz/(w.spec.a4*w.spec.Fz0))) * (1 + w.spec.a8*gamma*gamma)
    By = Ky / (Cy * Dy)
    Ey = (w.spec.a5 + w.spec.a6*(Fz/w.spec.Fz0)) * (1 + w.spec.a7*sign(alpha))
    phi_y = alpha + w.spec.S_Hy
    Fy_pure = Dy * sin(Cy * atan(By*phi_y - Ey*(By*phi_y - atan(By*phi_y))))

    Dx = mu * Fz + w.spec.S_Vx
    Cx = w.spec.b1                                                # ≈ 1.65
    Bx = w.spec.b2 * sin(2*atan(Fz/(w.spec.b3*w.spec.Fz0))) / (Cx * Dx)
    Ex = (w.spec.b5 + w.spec.b6*(Fz/w.spec.Fz0)) * (1 + w.spec.b7*sign(kappa))
    phi_x = kappa + w.spec.S_Hx
    Fx_pure = Dx * sin(Cx * atan(Bx*phi_x - Ex*(Bx*phi_x - atan(Bx*phi_x))))

    # ---- 5. combined slip: the elliptical interaction model (§3.5.3) ----
    sigma_x_star = Bx * kappa / (mu * Fz)      # normalised
    sigma_y_star = By * tan(alpha) / (mu * Fz)
    sigma = sqrt(sigma_x_star^2 + sigma_y_star^2)
    if sigma < 1e-6:
        Fx, Fy = 0, 0
    else:
        F_combined = combined_curve(sigma, mu, Fz, w.spec)
        Fx = (sigma_x_star / sigma) * F_combined
        Fy = (sigma_y_star / sigma) * F_combined

    # ---- 6. transient relaxation (§3.5.3): a 0.3 m relaxation length ----
    ds = |w.v_longitudinal| * dt                                  # travelled distance
    w.phi_state.x += (phi_x - w.phi_state.x) * (1 - exp(-ds / w.spec.sigma_relax))
    w.phi_state.y += (phi_y - w.phi_state.y) * (1 - exp(-ds / w.spec.sigma_relax))

    # ---- 7. aligning torque (steering feel, force feedback) ----
    Mz = magic_formula_torque(alpha, Fz, mu, w.spec) * w.aligning_torque_scale
       + pneumatic_trail(kappa, alpha, Fz) * Fy

    # ---- 8. the three-node thermal model (§3.5.3) ----
    P_slip = |Fx * w.v_slip_x| + |Fy * w.v_slip_y|
    P_flex = 2*PI * w.omega * (surface.material.rolling_resistance_coeff * Fz * w.r_effective)
    chi = 0.62                                                    # the tread share of slip power
    Q_contact = h_contact(surface) * w.contact_area * (w.T_tread - surface.temperature)
    Q_conv    = (12 + 0.62*sqrt(w.speed)) * w.outer_area * (w.T_tread - air.T)
    Q_spray   = puddle_depth > 2.5mm
                ? h_wet * w.contact_area * (w.T_tread - puddle_temp) * min(1, puddle_depth/2.5mm)
                : 0
    w.T_tread  += dt * (P_slip*chi - P_hyst(w.T_tread, w.T_carcass) - Q_contact - Q_conv - Q_spray)
                       / w.spec.C_tread                           # 1,800 J/K
    w.T_carcass+= dt * (P_flex + P_slip*(1-chi) - P_hyst(w.T_carcass, w.T_tread)
                        - P_hyst(w.T_carcass, w.T_core)) / w.spec.C_carcass     # 5,400 J/K
    w.T_core   += dt * (P_hyst(w.T_carcass, w.T_core) - Q_amb(w.T_core, air))
                       / w.spec.C_core                            # 9,200 J/K
    # saturation and failure (§6.5.4)
    w.T_tread = clamp(w.T_tread, 190, 480)
    if w.T_carcass > 420: emit_event(TYRE_DELAMINATION, w)        # a blowout
    return {Fx, Fy, Mz}


fn brake_solve(axle, dt, vehicle, air):                           # §3.5.6
    P_in = brake_force[axle] * vehicle.speed                      # the dissipated power
    h_conv = 12 + 0.62*sqrt(vehicle.speed)                        # a rotating disc
    Q_conv = h_conv * axle.rotor_area * (axle.T_rotor - air.T)
    Q_cond = h_interface * (axle.T_rotor - axle.T_hub)
    Q_water= axle.puddle_immersion * h_wet * axle.rotor_area
                        * (axle.T_rotor - axle.puddle_temp)
    axle.T_rotor = clamp(axle.T_rotor + dt*(P_in - Q_conv - Q_cond - Q_water)/axle.C_rotor,
                         250, 1480)                               # C_rotor = 8.4kg × 460 J/kg·K
    axle.T_pad   += dt*(pad_coupling(axle.T_rotor, axle.T_pad) - pad_convection(axle.T_pad))/axle.C_pad
    axle.T_fluid += dt*(fluid_coupling(axle.T_caliper, axle.T_fluid))/axle.C_fluid   # a 40 s lag

    mu_pad = pad_friction_curve(axle.pad_compound, axle.T_pad)    # organic peaks at 220 °C (0.42),
                                                                  # fades to 0.19 at 520 °C
    if axle.T_fluid > axle.fluid_grade.boiling_point_wet:
        emit_event(VAPOUR_LOCK, axle); mu_pad *= 0.12             # a soft pedal, a real failure
    if axle.T_rotor > 1100: emit_event(ROTOR_STRUCTURAL_FAILURE, axle)
    return mu_pad


fn abs_controller(w, dt, surface):                                # §3.5.6
    lambda_target = slip_ratio_target(surface.condition)          # 0.12 dry asphalt,
                                                                  # 0.22 loose gravel, 0.08 ice
    # the controller does NOT know which surface it is on — which is why ABS on gravel
    # INCREASES stopping distance, as it does in reality
    if w.slip_ratio > lambda_target + 0.04: reduce_pressure(w, rate=15 MPa/s)
    elif w.slip_ratio < lambda_target - 0.02: increase_pressure(w, rate=8 MPa/s)
    # hydraulic modulation at 15 Hz with pump-back and an accumulator
```

## B.6 Puddle shallow-water solve

Implements §3.3.3(B). GPU compute, 10 Hz, 0.28 ms for a 256×256 grid.

```
GRID: 0.5 m spacing, 256×256 per simulation source, radius 128 m
STATE: h[]  (water depth, m), hu[], hv[] (discharge, m²/s)
STATIC: z_b[] = terrain_height + micro_relief
        micro_relief = kerb_heights + road_camber + ruts_from(§3.4.6 deform layer)
        n[]   = Manning coefficient from MaterialRecord:
                asphalt 0.013 | concrete 0.012 | grass 0.035 | gravel 0.030 | mud 0.045
        K[]   = infiltration capacity from MaterialRecord.permeability_m_s and the §3.3.1
                soil moisture state

fn puddle_step(dt):                                               # dt = 0.1 s
    # ---- source terms ----
    for cell in parallel:
        precip = weather_query(cell.world_pos).precip_rate_mm_h / 3600 / 1000   # m/s
        sky    = sky_exposure[cell]                               # the baked sky-visibility term
        h[cell] += dt * precip * sky
        # infiltration (a Green-Ampt-derived capacity, §3.3.1)
        inf = min(h[cell]/dt, K[cell] * (1 + psi_dtheta_M[cell] / max(F[cell], 1e-6)))
        h[cell]  -= dt * inf
        F[cell]  += dt * inf
        # evaporation (a Penman-style term from T, U, RH, solar — all from §3.2)
        h[cell]  -= dt * evaporation_rate(cell)

    # ---- momentum (a depth-averaged shallow-water solve) ----
    for iter in 0..19:                                            # 20 Gauss-Seidel iterations
        for cell in parallel_checkerboard(iter):                  # a red-black ordering for
            grad_z = (z_b[e]+h[e]) - (z_b[w]+h[w]) / (2*dx)       # parallel Gauss-Seidel
            grad_z += ... (the y direction)
            tau_b = rho * g * n[cell]^2 * |u[cell]| * u[cell] / max(h[cell],1e-4)^(1/3)
            hu[cell] += dt * (-g*h[cell]*grad_z_x - tau_b.x)
            hv[cell] += dt * (-g*h[cell]*grad_z_y - tau_b.y)
            # a wet/dry front treatment: h < 1e-4 m is DRY, momentum is zeroed, and the
            # gradient uses a one-sided difference (the standard fix for shoreline instability)
            if h[cell] < 1e-4: hu[cell] = hv[cell] = 0

    # ---- mass update and upwinding ----
    for cell in parallel:
        flux_e = upwind_flux(hu, cell, cell.east)                 # a first-order upwind flux
        flux_n = upwind_flux(hv, cell, cell.north)
        h[cell] += dt/dx * (flux_w - flux_e) + dt/dy * (flux_s - flux_n)
        h[cell] = max(0, h[cell])
        u[cell] = hu[cell] / max(h[cell], 1e-4)

    # ---- outputs ----
    publish(puddle_depth_texture, h)          # 16-bit, 0–200 mm — read by the renderer,
                                              # the tyre model (B.5 step 3), pedestrian steering
                                              # (§3.8.4), Foley (§3.7.6), and the deform-layer
                                              # erosion (§3.4.6)
    publish(runoff_flux_to_hydrology, fluxes)  # §3.3.1's surface-runoff term
```

## B.7 Ballistic penetration resolve

Implements §3.6.4. Per projectile intersection, up to 12 layers.

```
fn resolve_penetration(proj, hit):
    stack   = query_layer_stack(hit.position, hit.normal)          # §3.4.2, mutable per cell
    v       = proj.velocity
    E_res   = 0.5 * proj.mass * |v|^2
    m_res   = proj.mass
    d       = proj.diameter
    depth_travelled = 0
    layers_penetrated = []

    for (i, layer) in enumerate(stack):                            # OUTERMOST FIRST
        if E_res <= 0: break

        theta   = angle_between(-normalize(v), layer.normal)       # the obliquity
        mat     = material_db[layer.material_id]                   # THE MATERIAL ONTOLOGY (§3.1)

        # ---- 1. ricochet test (§3.6.4) ----
        crit = mat.ballistic.critical_ricochet_deg[proj.construction]
               # fmj 62 | ap 71 | hp 55 | shotgun_pellet 48 (for concrete)
        if theta > crit and layer.thickness_mm > 2 * d:
            n = layer.normal
            v_new = reflect(v, n)
            # a randomised perturbation from the surface roughness — DETERMINISTIC SEED (§3.4.7)
            r = pcg32(hash(proj.id, i, 'ricochet'))
            v_new = rotate_by(v_new, gaussian_perturbation(r, sigma=mat.roughness_deg))
            speed_loss = uniform_from(r, 0.34, 0.71, by_material_class=mat.class)
            proj.velocity = v_new * |v| * speed_loss
            proj.yaw_deg  = uniform_from(r, 8, 40)                 # UNSTABLE — a ricochet tumbles
            proj.drag_multiplier = 2.4                             # §3.6.4
            spawn_deformation_event(mat, hit, E_res, 'ricochet_scar')
            emit_acoustic_event(hit, E_res, mat, 'ricochet')       # §3.7.6
            return PENETRATED_NONE

        # ---- 2. resistance (§3.6.4) ----
        # R(θ) = R0 / cos(θ)^n, scaled by thickness relative to the 200 mm reference
        R_layer = mat.ballistic.normal_0deg_mm_equiv
                * (layer.thickness_mm / 200)^0.5
                / (cos(theta) ** mat.ballistic.obliquity_exponent)
                * layer.state_factor()                             # 'cracked' 0.82,
                                                                   # 'breached' 0.44, etc.
        E_required = R_layer * projectile_effectiveness(proj, |v|, mat)
        # projectile_effectiveness: a function of construction (AP 1.8×, FMJ 1.0×, HP 0.62×
        # against hard targets / 1.34× against soft), velocity regime (a fragmentation threshold
        # for HP above 450 m/s), and mass/diameter

        # ---- 3. penetrate or stop ----
        if E_res < E_required:
            # STOPPED — embed, transfer momentum, deform
            transfer_momentum(hit.body, m_res, v)
            spawn_deformation_event(mat, hit, E_res, 'embedded')
            if proj.construction == 'hp': spawn_fragments(hit, count=0, energy=E_res*0.18)
            emit_acoustic_event(hit, E_res, mat, 'impact_thud')
            return STOPPED_AT(layer=i)

        # PENETRATED
        E_res -= E_required * (1 + 0.18 * behind_spall_energy_fraction(mat, E_res))
        layers_penetrated.push(i)

        # ---- 4. behind-face spall (§3.4.3 model 2) — brittle materials only ----
        if mat.deformation.model == 'fracture_voronoi':
            # the stress wave propagates at the bulk sound speed and reflects in tension
            # off the free far face
            t_wave = layer.thickness_mm / 1000 / mat.bulk_sound_speed_m_s
            sigma_tensile_far = compressive_stress(E_res, m_res, d) * reflection_coefficient(mat)
            if sigma_tensile_far > mat.tensile_strength_MPa:
                cone_half = mat.ballistic.spall_cone_half_angle_deg          # 32° for concrete
                mass = mat.ballistic.spall_mass_fraction * impacted_volume(layer, d)
                v_frag = proj.velocity * mat.ballistic.secondary_fragment_velocity_frac  # 0.28
                spawn_spall_cone(hit.position + layer.thickness*layer.normal,
                                 cone_half, mass, v_frag, seed=hash(proj.id, i, 'spall'))
                # ⇒ a 7.62 mm round hitting a 200 mm concrete wall from OUTSIDE ejects
                #   fragments INSIDE. Cover facing the wrong way is dangerous. (§3.4.3)

        # ---- 5. mass loss and fragmentation ----
        frac = {'hp': 0.42, 'fmj': 0.06, 'ap': 0.00, 'shotgun_pellet': 0.71}[proj.construction]
        if frac > 0 and E_required > E_res * 0.5:                  # a marginal penetration
            m_lost = m_res * frac
            spawn_fragments(hit, count=int_between(2, 9, mat.class), energy=0.5*m_lost*|v|^2*0.4)
            m_res -= m_lost
            E_res  = 0.5 * m_res * |v|^2 * (1 - 0.22*frac)         # a velocity loss too
            v = normalize(v) * sqrt(2*E_res/max(m_res, 1e-6))

        # ---- 6. mutate the layer stack (§3.4.2) ----
        layer.state = state_after(layer.state, mat, E_required)     # intact→cracked→breached
        journal_mutation(hit.cell_id, LAYER_STACK, layer)           # §5.8.4's delta journal
        spawn_deformation_event(mat, hit, E_required, mat.deformation.model)
        emit_acoustic_event(hit, E_required, mat, 'penetration')
        depth_travelled += layer.thickness_mm

    # ---- 7. terminal effects on a body (§3.6.5), if the last layer was tissue ----
    if hit.entity is_an_agent:
        apply_terminal_effects(hit.entity, hit.segment, E_res, m_res, v, proj, layers_penetrated)
    return PENETRATED(layers_penetrated, E_residual=E_res)
```

## B.8 Macro traffic — the Cell Transmission Model

Implements §3.8.1. 1 Hz over 12,400 links, 0.07 ms amortised.

```
STATE per link i: n[i] (vehicle count), plus static: L[i] (length), lanes[i],
                v_free[i], capacity[i], N_jam[i] (jam density × L)
CONSTANTS: w = 5.8 m/s (the backward wave speed), a = 1.6 (the Greenshields exponent)

fn ctm_step(dt = 1.0 s):
    # ---- 1. demand and supply ----
    for i in parallel_over(links) sorted_by id:                     # DETERMINISTIC
        v_i   = v_free[i] * (1 - n[i]/N_jam[i]) ** a                # the speed-density relation
        D[i]  = v_i * n[i]                                          # demand (veh/s)
        S[i]  = w * (N_jam[i] - n[i])                               # supply
        Q[i]  = capacity[i] * signal_state_factor(i, now())         # §2.8's simulated controllers

    # ---- 2. flows ----
    for i in parallel_over(links) sorted_by id:
        f[i] = min(D[i], S[i+1], Q[i])                              # the CTM flux
        # INCIDENTS: a crash, a cordon (§3.12.5), a flood (§3.3.1), a fire (§3.3.5), a plough
        # (§3.3.4), a collapse (§3.4.4) all set Q[i] = 0 or a reduced value. The queue then
        # propagates BACKWARDS at w = 5.8 m/s — visible from a helicopter as a growing line
        # of brake lights, emergent from the model (§3.8.1).

    # ---- 3. update ----
    for i in parallel_over(links) sorted_by id:
        n[i] = clamp(n[i] + (dt/L[i]) * (f[i-1] - f[i]), 0, N_jam[i])

    # ---- 4. the tier handoffs (§3.8.3) ----
    for i in links_within(macro_region):
        if not i.has_meso_packets and n[i] > MESO_THRESHOLD:
            spawn_mesoscopic_packets(i, count=round(n[i]),
                positions = stratified_sample_along(i),             # NEVER clustered
                speeds    = sample_from_speed_density(v_i, n[i]),
                destinations = sample_from_baked_OD_matrix(i))
        elif i.has_meso_packets and n[i] < MESO_RELEASE:
            retire_mesoscopic_packets(i)

    # ---- 5. THE CONSERVATION INVARIANT (§3.8.3, CI-asserted per commit) ----
    assert |sum(n) + sum(meso_packet_counts) + sum(micro_vehicle_counts)
            - total_region_vehicle_count| <= 1


fn dynamic_reassignment(dt = 1.0 s):                                # §3.8.3
    if not any_link_capacity_changed: return
    affected = subnetwork_around(changed_links, radius=900 links)
    # 4 Frank-Wolfe iterations toward a dynamic user optimum on the affected subnetwork only
    for k in 0..3:
        flow = frank_wolfe_step(affected, demand_prior=baked_assignment[slice_of_day()])
    publish(link_speeds, flow)                                      # consumed by §3.12.4's
                                                                    # street-ETA dispatch scoring,
                                                                    # §3.9's commute times, and
                                                                    # §5.8.5's corridor prediction
    request_signal_retiming(affected, for_units=dispatched_units_en_route())
        # the green wave (§2.8): reduces an emergency-vehicle ETA by 14–22% in dense urban
```

## B.9 Fire spread — Rothermel

Implements §3.3.5. 1/30 Hz on 64 m cells within 2 km of an ignition.

```
fn fire_step(dt):
    for cell in active_burning_cells sorted_by cell_id:             # DETERMINISTIC
        fuel   = cell.fuel_record                                   # a MaterialRecord with a
                                                                    # fire block (§3.1.2)
        if fuel.fuel_model == null: continue                        # non-combustible (§3.1.2)

        # ---- fuel moisture from the weather field (§3.2), not authored ----
        # fine fuels equilibrate in 1–10 h from T/RH via an EMC curve
        M_f = equilibrium_moisture(weather_query(cell).T, weather_query(cell).RH,
                                   timelag_h=fuel.fine_fuel_lag)
        M_live = live_fuel_moisture(cell, season(), soil_moisture(cell))    # §3.3.1

        # ---- Rothermel's spread rate (§3.3.5) ----
        beta   = fuel.bulk_density / fuel.particle_density
        beta_op= fuel.beta_optimal                                    # the fuel model's optimum
        Gamma_max = C_r * beta**(-A_f) / (beta * (1 + B_f))
        xi     = exp((0.792 + 0.681*sqrt(fuel.sigma_SA)) * (beta + 0.1))
                 / (192 + 0.2595*fuel.sigma_SA)
        I_R    = Gamma_max * fuel.net_fuel_load * epsilon_min * M_A(M_f) * S_T
        phi_w  = C_w * B_w * beta**(-E_w) * U_midflame**B_w           # U from §3.2 + M2/M4/M7
        phi_s  = 5.275 * beta**(-0.3) * tan(cell.slope)**2            # the terrain gradient
        epsilon= exp(-138 / fuel.sigma_SA)
        Q_ig   = 250 + 111600 * M_f
        R      = (I_R * xi * (1 + phi_w + phi_s)) / (fuel.bulk_density * epsilon * Q_ig)   # m/s

        # ---- derived outputs (§3.3.5) ----
        I      = fuel.heat_of_combustion * fuel.fuel_load * R         # fireline intensity, kW/m
        L_flame= 0.0775 * I**0.46                                     # Byram, m
        I_crit = (0.010 * cell.canopy_base_height
                  * (460 + 25.9*M_live))**1.5                         # Van Wagner
        crown  = (I > I_crit) and (U_midflame > U_critical(cell))

        # ---- spread to the 8 neighbours ----
        for nb in cell.neighbours_8 sorted_by index:                  # DETERMINISTIC ORDER
            if nb.state == BURNED or nb.fuel_load <= 0: continue
            if nb.fuel_moisture > nb.fuel.moisture_of_extinction: continue
            theta = angle_between(wind_vector, nb.direction)
            R_eff = R * (1 + phi_w * cos(theta) + phi_s * nb.upslope_factor)
            cell.spread_accumulator[nb] += R_eff * dt / cell.size_m
            if cell.spread_accumulator[nb] >= 1.0:
                ignite(nb, source=cell, seed=hash(cell.id, nb.id, tick()))

        # ---- spotting (§3.3.5): embers 0.1–2.4 km downwind ----
        if I > 800 and pcg32(hash(cell.id, tick(), 'spot')) < spotting_rate(I, U):
            d = transport_distance(I, U)
            target = cell_at(cell.pos + wind_direction * d)
            if target: ignite(target, source=cell, cause='spot')

        # ---- consumption and smoke (§3.3.5) ----
        cell.fuel_load -= burn_rate(I) * dt
        emit_smoke_particulate(cell, rate=f(I), into=weather_field)   # feeds §3.2's visibility
                                                                      # and solar irradiance —
                                                                      # a large fire darkens and
                                                                      # cools the region downwind
        if cell.fuel_load <= 0: cell.state = BURNED
```

## B.10 Delta journal compaction and the World Settle Pass

Implements §5.8.4, §6.4 #9, §6.10 C-13.

```
fn journal_compact(cell):                                           # every 600 ticks
    snap    = cell.snapshot
    journal = cell.journal
    k = largest_prefix(journal, such_that
            size(apply(snap, journal[0..k])) <= 18_KB
            and size(journal[k..]) <= 4_KB)
    cell.snapshot = apply(snap, journal[0..k])
    cell.snapshot_generation += 1
    cell.journal  = journal[k..]
    if cell.total_size > 64_KB:                                     # the hard cap
        overflow_ladder(cell)                                       # §6.4 #9:
            # (a) re-compact
            # (b) drop the oldest entries in settle_class order:
            #     ephemeral → short → medium → long
            # (c) if PROTECTED entries would be dropped: SPLIT the cell into 4 sub-slabs
            # PROTECTED ENTRIES ARE NEVER DROPPED. (§6.4 #9)


fn world_settle_pass(dt_sim):                                       # §5.8.4, at 1/60 Hz
    # Incidental mutations decay toward the authored baseline on a physically motivated
    # schedule. NARRATIVE AND STRUCTURAL MUTATIONS NEVER REVERT.
    for cell in resident_slabs sorted_by cell_id:                   # DETERMINISTIC
        for entry in cell.journal sorted_by settle_at_tick, then id:
            if entry.settle_class == 'protected': continue
            if now() < entry.settle_at_tick: continue

            match entry.mutation_type:
              'terrain_displace' ->                                  # §3.4.6
                  # washed out by cumulative rainfall, or compacted by traffic
                  if cumulative_rain_since(entry.tick) > entry.rain_erasure_threshold:
                      revert(entry)
                  elif traffic_passes_since(entry.tick) > 40:
                      entry.payload *= 0.7                            # a rut becomes a track
              'litter' ->
                  if municipal_refusal_service_due(cell.district): revert(entry)
                                                                    # §3.11.5's budget drives the
                                                                    # collection frequency
              'graffiti' ->
                  # §4.3.5: removed in 3–14 days in a high-land-value district,
                  # 90 days–never in a low one. The graffiti density map IS a readout of
                  # municipal spending inequality — emergent, not authored.
                  if now() - entry.tick > removal_delay(cell.land_value, city_budget):
                      revert(entry)
              'rubble_pile' ->
                  if work_order_exists(cell) and now() > entry.tick + clearing_duration:
                      revert(entry); remove_work_order(cell)          # a plough/clearance crew,
                                                                      # a visible ledger entity
              'structure_destroyed' ->
                  pass                                                # NEVER reverts
              'evidence' ->
                  if now() > entry.tick + evidence_decay(entry, weather_since(entry.tick)):
                      entry.integrity = 0                             # §3.12.6: inadmissible
              _:
                  if now() > entry.tick + entry.settle_duration: revert(entry)

            journal_remove(cell, entry)


fn reconcile_downtime(client_state, server_delta):                  # §5.13.4, §6.10 C-13
    # the settle pass MUST run as part of the time-advance, keyed on authoritative WorldTime,
    # never on real elapsed time — otherwise a save → advance → load produces a different
    # world than a continuous run. This is GP-11's hardest assertion (§6.10 C-13).
    t0 = client_state.world_time
    t1 = server_delta.world_time
    apply_authoritative_delta(client_state, server_delta)             # authoritative class only
    world_settle_pass_range(t0, t1)                                   # settle everything that
                                                                      # should have settled
    rederive_all(client_state)                                        # §5.10.2: never persisted
    assert state_hash(client_state) == state_hash(continuous_run(t0, t1))
```

## B.11 Interior layout grammar solve

Implements §4.3.1. Cook-time, 12 ms per floorplan, 91.4% first-pass yield.

```
fn solve_floorplan(input) -> Floorplan:
    for attempt in 0..5:                                            # the re-solve budget (§4.3.1)
        seed = hash(input.envelope_id, input.storey, attempt, GENERATOR_VERSION)
        rng  = pcg32(seed)                                          # DETERMINISTIC (§5.4.3 rule 6)

        # ---- Stage 1: the space program ----
        program = select_program_template(input.use_class, input.era, input.socio_tier,
                                          input.envelope_area, input.household_composition)
        # a list of (space_type, target_area, tolerance, count), scaled to the envelope

        # ---- Stage 2: the slicing-tree topology (§4.3.1) ----
        # a binary tree where each internal node is an H or V cut; leaves are rooms.
        # Enumerated by a stochastic grammar biased toward the family's typology.
        topo = sample_topology(input.style_family, len(program), input.envelope_aspect, rng)
        #   F01 Georgian   → double-pile, symmetrical
        #   F05 ranch      → single-pile, linear
        #   F18 shotgun    → single-pile, one room wide

        # ---- Stage 3: the constraint solve (simulated annealing, 4,000 iterations) ----
        state = initial_cut_positions(topo, program, input.envelope)
        cost  = evaluate(state)
        if cost == REJECTED: continue
        T = T_init
        for iter in 0..3999:
            cand = perturb(state, rng)                              # move a cut, swap two rooms,
                                                                    # flip an H/V, rotate a subtree
            c = evaluate(cand)
            if c == REJECTED: continue
            if c < cost or rng.float() < exp(-(c - cost)/T):
                state, cost = cand, c
            T *= 0.9986                                             # 4,000 iters → T_final/T_init
                                                                    # ≈ 0.0037

        plan = extract_rooms(state, topo)
        plan = insert_circulation(plan, program)                     # Stage 4
        plan = insert_services(plan)                                 # Stage 5
        plan = assign_finishes(plan, input)                          # Stage 6

        if validate(plan): return plan                              # Stage 7
    return FALLBACK_TEMPLATE[input.style_family][hash(seed) % 3]     # 54 authored safe templates


fn evaluate(state) -> f64 | REJECTED:
    rooms = extract_rooms(state, topo)

    # ---- HARD constraints (rejection) — §4.3.1 Stage 3 ----
    for r in rooms:
        if r.area <= 0 or r.outside_envelope: return REJECTED
        if r.is_habitable and not r.touches_exterior_wall: return REJECTED
    occ = sum(occupancy_load(r) for r in rooms)
    exits = count_exits(rooms)
    if occ > 49 and exits < 2: return REJECTED                      # an egress code requirement
    for r in rooms:
        if travel_distance_to_exit(r) > (sprinklered ? 60 : 45): return REJECTED
    if any(dead_end_corridor(r) > (sprinklered ? 15 : 6)): return REJECTED
    for s in spans(rooms):
        if s.clear_span > STRUCTURAL_LIMIT[input.structural_system]: return REJECTED
            # 6 m light timber | 9 m flat-plate | 14 m PT | 24 m steel portal
    for c in columns(rooms):
        if not column_aligns_with_storey_below(c): return REJECTED   # a column may not land in
                                                                     # a doorway downstairs
    if not accessible_route_exists(rooms, clear_width=0.915, turning_circle=1.5):
        return REJECTED                                              # §4.3.1 — BOTH a building
                                                                     # code requirement AND the
                                                                     # game's own accessibility
                                                                     # requirement (§6.6.6)

    # ---- SOFT constraints (cost) ----
    c  = sum((r.area - r.target)^2 / r.target^2 for r in rooms) * 1.0
    c += sum(-w for (a, b, w) in ADJACENCY if adjacent(rooms[a], rooms[b]))
        # kitchen↔dining +3.0, kitchen↔entry +1.2, bathroom↔bedroom +2.4,
        # utility↔kitchen +2.6, boiler↔exterior_wall +1.8, bathroom↔kitchen −4.0
    c += 3.8 * (1 - wet_wall_alignment_score(rooms))                 # §4.3.1 — the mechanism that
                                                                     # makes bathrooms stack
    for r in rooms where r.is_habitable:
        glazing = window_area(r) / r.area
        if glazing < 0.08: c += 2.2 * (0.08 - glazing) / 0.08        # a code daylight minimum
        if r.dual_aspect: c -= 1.4
        if r.depth_from_window > 3: c += 2.2
    c -= 2.0 * privacy_gradient_score(rooms)                         # public → private along
                                                                     # the circulation path
    c += sum(aspect_penalty(r) for r in rooms)                       # a 1:4 room is penalised
    circ = corridor_area(rooms) / total_area(rooms)
    c += 6.0 * max(0, abs(circ - 0.13) - 0.05)**2                    # an 8–18% target band
    return c


fn validate(plan) -> bool:                                           # §4.3.1 Stage 7
    if not every_window_maps_to_a_daylight_room(plan): return false  # CI gate INT-03
    if not every_facade_door_maps_to_circulation(plan): return false
    if not every_chimney_maps_to_a_service_element(plan): return false
    if not floor_levels_match_exterior(plan): return false
    if any_door_swing_conflict(plan): return false
    if not floodfill_from_entry_reaches_all_rooms(plan): return false # CI gate INT-05
    if any_nonmanifold_geometry(plan): return false
    if any_zero_area_room(plan): return false
    if wall_thickness_mismatch_with_layer_stack(plan): return false   # §3.4.2 coherence
    return true
```

## B.12 Cellular coverage bake — RF propagation

Implements §3.12.3.1. Cook-time, 2.4 core-hours region-wide, ~3 MB output.

```
fn bake_coverage():
    # ---- coarse grid: 32 m over 252 km² = 246,000 cells × 3 carriers ----
    for cell in coarse_grid:
        for carrier in [0,1,2]:
            best = NONE
            for site in carrier.sites:                              # 412 total: 148 macro,
                                                                    # 186 micro, 78 in-building DAS
                d = distance_2d(cell, site)
                if d > 12_000 m: continue                           # beyond a practical cell radius

                # free space + log-distance path loss
                PL = PL_at(d0=100 m, f=carrier.f_MHz)
                   + 10 * n_exponent(cell.environment_class) * log10(d / 100)
                   # n = 2.0 free | 2.9 suburban | 3.6 urban | 4.3 dense canyon | 1.8 in-building LOS

                # terrain diffraction — knife-edge, Lee approximation, up to 2 edges
                path = sample_terrain_profile(cell, site, resolution=2 m)
                edges = find_diffraction_edges(path)                 # local maxima on the profile
                L_terrain = 0
                for e in edges[0..1]:                                # a 2-edge recursive solve
                    nu = e.height * sqrt(2*(e.d1+e.d2) / (lambda * e.d1 * e.d2))
                    L_terrain += 6.9 + 20*log10(sqrt((nu-0.1)**2 + 1) + nu - 0.1)
                    # ⇒ THIS is what makes the Sable Range's leeward valleys and the Anza
                    #   Basin's badlands genuine dead zones without anyone painting one.

                # building entry + interior partitions (§3.7.2's transmission loss, REUSED —
                # one model, two consumers)
                L_building = entry_loss(cell.environment_class)      # 12 suburban | 18 urban |
                                                                     # 25 dense/low-E glazing
                if cell.is_interior:
                    L_building += sum(layer_stack.TL_at(carrier.f_MHz)
                                      for wall in walls_between(cell, nearest_exterior))

                # foliage — ITU-R P.833
                foliage_path = canopy_intersection_length(cell, site)
                L_foliage = 0.2 * carrier.f_MHz**0.3 * foliage_path**0.6
                # at 1,900 MHz: ≈ 1.86 dB per metre of old-growth canopy. 60 m → 22 dB;
                # 200 m → 45 dB ⇒ service lost. §2.5.5's forest coverage follows from physics.

                # lognormal shadowing (a deterministic per-cell hash, so the bake is reproducible)
                X_sigma = gaussian_from(pcg32(hash(cell.id, site.id, carrier)), sigma=cell.sigma)
                        # sigma = 6 dB suburban | 8 dB urban | 9 dB dense urban

                P_rx = site.P_tx + site.G_tx + mobile_G_rx - PL - L_terrain - L_building
                       - L_foliage + X_sigma

                if P_rx > best.P_rx: best = {site, P_rx}

            cell.class[carrier] = classify(best.P_rx)
                # > −104 dBm → VOICE_AND_DATA
                # > −110 dBm → VOICE
                # > −124 dBm → DATA_ONLY_WEAK / emergency-call-attempt
                # else       → NONE
    # ---- fine grid: 2 m within 400 m of any simulation source, streamed per G3 cell ----
    # same solver, a higher sampling density, ~1.8 MB resident (§3.12.3.1)

    # ---- radio (police/emergency): a SEPARATE model ----
    # 700/800 MHz ⇒ better diffraction and foliage penetration, worse building penetration.
    # 34 sites + 9 repeaters. Repeater coverage requires line of sight to the repeater.
    # ⇒ Radio dead zones exist where cellular does not (deep subsurface, a steel tank interior)
    #   and vice versa (Anza Basin). The asymmetry is a designed tactical space (§3.12.3.1).


fn query_coverage(pos, carrier) -> CoverageClass:
    if pos.z < terrain_height(pos.xy) - 2:                          # subsurface (§2.6.3)
        depth = terrain_height(pos.xy) - pos.z
        L = 20 * log10(depth) + layer_stack_TL_of_overhead_structure(pos)
        return classify(baked_P_rx_at(pos, carrier) - L)
    if pos.is_interior:
        return baked_fine_or_coarse(pos, carrier)                    # the interior bake accounts
    return interpolate_bilinear(baked_fine_or_coarse(pos, carrier))  # for the LayerStack
```

## B.13 Cordon perimeter — a minimum edge cut on the road graph

Implements §3.12.5 Tier 3. Event-driven, 0.9 ms on ~900 links.

```
fn compute_cordon(incident_cell, tier) -> Cordon:
    # A cordon is a MINIMUM CUT separating the incident from the road network, subject to:
    #   (a) the cut edges must be physically closable (a barricade, a unit, a spike strip)
    #   (b) AT LEAST ONE EDGE MUST REMAIN OPEN for emergency services (§6.4 #5 — NO PERIMETER
    #       IS UNBREACHABLE)
    #   (c) the inner perimeter is 40–120 m, the outer 150–400 m, by tier and incident class

    G = road_subgraph_around(incident_cell, radius = tier == 3 ? 400 m : 200 m)
    source = a virtual node connected to every link touching incident_cell, capacity INF
    sink   = a virtual node connected to every boundary link of G, capacity INF

    for e in G.edges:
        e.capacity = 1.0                       # unit capacity ⇒ a minimum EDGE cut
        if not physically_closable(e):         # a bridge span, a tunnel bore, a freeway mainline
            e.capacity = INF                   # cannot be closed with a barricade
        e.cost = closure_cost(e)               # prefer closing a local street to an arterial
                                               # (a real operational consideration, and it
                                               # minimises the traffic consequence)

    cut = min_cut_max_flow(G, source, sink, algorithm=Dinic)         # 0.9 ms on ~900 links

    # (b) the emergency-access exception
    emergency_edge = select(cut.edges, minimising response_ETA_from(nearest_station))
    cut.edges.remove(emergency_edge)
    emergency_edge.state = OPEN_FOR_AUTHORISED                       # the player can use it,
                                                                     # at risk (§6.4 #5)

    # apply the closures as REAL road-graph mutations (§3.12.5)
    for e in cut.edges:
        set_link_capacity(e, 0)                                      # the macro CTM (§B.8) sees
        assign_closure_asset(e, tier)                                # this IMMEDIATELY: civilian
            # a unit (if available) | a barricade vehicle | a portable barrier | a spike strip
        mark_pedestrian_obstacle(e)                                  # the §3.8.4 flow field sees it
    for e in cut.edges: journal_mutation(e.cell_id, ROAD_CLOSURE, e) # §5.8.4 — persists

    # command post and span of control (§3.12.5 Tier 3)
    cp = select_command_post_location(cut)                           # outside the inner ring,
    commander = assign_incident_commander(tier)                      # with egress visibility
    if direct_reports(commander) > 6:
        commander.degradation = span_of_control_penalty(direct_reports(commander))
        # a real ICS principle: exceeding a span of 6 degrades coordination

    # re-route civilian traffic (§B.8's dynamic reassignment)
    request_dynamic_reassignment(affected=cut.edges)
    request_signal_retiming(for_units=assigned_units, mode='green_wave')

    return Cordon(inner=cut.edges.where(d < 120 m),
                  outer=cut.edges.where(d >= 120 m),
                  emergency_access=emergency_edge,
                  command_post=cp, commander=commander,
                  max_hold_time = hold_time_for(tier, incident))      # 20 min – 8 h
```
