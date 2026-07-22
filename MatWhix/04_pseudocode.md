# 04 — Pseudocode for every major module

Deliberately close to the real code's structure (same names, same order) so
each block can be checked against its implementation file directly.

---

## Calibration (once per rig)

```
CALIBRATE(bed_corners_px, bed_LWH, camera_hfov):
    K ← intrinsics from hfov (f = W/2 / tan(hfov/2)), centre principal point
    world_corners ← [(0,0), (L,0), (L,W), (0,W)]           # bed meters
    H_world→img ← NormalisedDLT(world_corners, bed_corners_px)
    H_img→canon ← diag(s,s,1) · H_world→img⁻¹              # s = px per meter
    [r1 r2 t] ← K⁻¹ H_world→img / ||K⁻¹H·e1||  (sign: bed in front)
    R ← Procrustes([r1, r2, r1×r2]);  camera_height ← |(−Rᵀt)_z|
```

## Normalised DLT + RANSAC

```
NORMALISED_DLT(src, dst):                      # ≥4 correspondences
    Ts, Td ← similarity transforms: centroid→0, mean radius→√2
    for each pair: emit 2 rows of A from the cross-multiplied projection eqs
    h ← right singular vector of A with smallest σ;  H ← Td⁻¹ ĥ Ts;  H /= h33

RANSAC_H(src, dst, iters=500, tol=3px):
    best ← ∅
    repeat iters (early-exit at 90% consensus):
        sample 4 non-collinear pairs → H_cand
        inliers ← {i : ||H_cand·srcᵢ − dstᵢ|| < tol}
        keep largest inlier set
    return NORMALISED_DLT(all inliers)         # final polish
```

## Per-frame pipeline (tiered)

```
PROCESS_FRAME(rgb, t):
    gray ← luma(rgb)
    ## tier 0: quality gate (every frame)
    if Var(Laplacian(gray)) < blur_min or clipped_fraction > max:
        KALMAN.coast();  return "rejected, coasting"

    ## tier 1: stabilize (every frame)
    track prev→cur corners by pyramidal LK
    H_step ← RANSAC_H(cur_pts, prev_pts)
    if credible: H_to_calib ← H_to_calib · H_step  else: coast on old warp
    canon_rgb ← inverse-warp(rgb, H_img→canon · H_to_calib)

    ## tier 1: change watch (every frame)
    change_regions ← ENVELOPE_DIFF(prev_canon_rgb, canon_rgb)

    run_segment  ← change_regions ≠ ∅ or timer(6 Hz) or forced
    run_inventory← timer(1 Hz) or removal detected or forced

    if run_segment:   instances ← SEGMENT_AND_CLASSIFY(canon_rgb)
    events ← CLASSIFY_CHANGES(change_regions, prev/cur foreground, flow)
    for ev in events:
        changelog.append(ev)
        if ev.kind ∈ {removed, added} and ev.class known:
            KALMAN.apply_event(ev.class, ±ev.count)
        if ev.revealed ≠ ∅:  BAYES.observe_reveal(ev.revealed)

    if run_segment and run_inventory:
        height_map ← DEPTH_FUSION(instances)
        volumes, masses ← VOLUMES(instances, height_map)
        hidden ← BAYES.estimate(sub_surface_volume(height_map, instances))
        KALMAN.measure(per class: visible + hidden, with sigmas)
        density_grid ← DENSITY(instances, masses, height_map, hidden)
    return dashboard state
```

## Envelope differencing

```
ENVELOPE_DIFF(prev_rgb, cur_rgb):
    valid ← warp-covered in both frames, eroded 12, border band cleared
    for each RGB channel:
        p, c ← channel − boxblur₁₅(channel)        # kill lighting drift
        d_ch ← max(c − dilate₃ₓ₃(p), erode₃ₓ₃(p) − c, 0)   # envelope escape
    d ← max over channels;  d[¬valid] ← 0
    σ ← max(MAD(d[valid])·1.4826, shot_noise_floor)
    mask ← open/close(d > 5σ)
    return connected components of mask with area ≥ half smallest item
```

## Segmentation + classification

```
SEGMENT_AND_CLASSIFY(canon_rgb):
    hsv ← RGB→HSV;  fg ← foreground(hsv)      # GMM (sequence) or S-threshold (still)
    for each connected blob of fg (on its crop):
        d ← chamfer distance;  peaks ← local maxima (sep = 0.3·d_min·s, depth gate)
        instances ← peaks ≤ 1 ? blob : watershed(−d, peaks)
    for each instance:
        f ← features: circular-mean hue (+concentration), sat, val,
                       texture energy, metric eq. diameter, eccentricity
        class ← argmax over classes, max over modes of
                 Σ −½((f−μ)/σ)²   (hue wrapped & concentration-weighted,
                                    size in log space) → softmax w/ unknown floor
    # cluster handling
    for instances larger than 1.5·(class size + 2σ): mark cluster
    cluster.count ← area / median(area of same-class singletons)  [prior fallback]
```

## Depth fusion → height map

```
DEPTH_FUSION(instances):
    layers ← BFS on contact graph seeded by floor-touching instances
    for each instance:
        E ← {(z, σ) from: topology layer·0.816·extent,
                          apparent size h(1−1/m) with σ = h·σ_rel/m²,
                          parallax median-z of its 3D points (if truck moved)}
        z ← Σz/σ² / Σ1/σ²;   σ ← (Σ1/σ²)^−½          # inverse-variance fusion
    height_map ← max over instances of dome(z_top, shading relief); floor = 0
    fill unknown slivers by Jacobi relaxation (avg of neighbours, 8 sweeps)
```

## Temporal parallax (when the truck moves)

```
PARALLAX(frame_a, frame_b, K, bed):
    matches ← LK-track corners a→b;  abstain if median flow < 0.7 px
    F ← normalised 8-point;  rank-2 project
    E ← KᵀFK, singular values → (1,1,0)
    (R,t) ← the decomposition placing triangulated points before both cameras
    X ← DLT triangulation of all matches           # up to global scale
    scale ← camera_height / median(floor-point distances along bed normal)
    return bed-frame points X·scale, σ_z ∝ 1/median disparity
```

## Hidden-layer Bayes

```
BAYES:
    init: α_c ← α₀ · stratification_prior(c | hidden depth range)
    observe_reveal(counts):  α_c += counts_c            # conjugate update
    estimate(V_sub ± σ_V):
        w ← α/Σα;  Var(w) ← w(1−w)/(Σα+1)
        for each class c:
            V_c ← V_sub·w_c·φ_c;  N_c ← V_c/E[v_c];  M_c ← V_c·ρ_c
            rel² ← (σ_V/V_sub)² + (σ_w/w)² + (σ_φ/φ)² + σ_size²
            report N_c with 90% CI = N_c(1 ± 1.645·rel), M_c ± ...
```

## Kalman bank

```
per class, per quantity q ∈ {count, mass, volume}:
    predict(u): x += u;  P += Q          # u = event delta (sale/restock)
    update(z, R): K ← P/(P+R);  x += K(z−x);  P ← (1−K)P
    coast(): predict(0)                  # rejected frame: belief holds, P grows
```
