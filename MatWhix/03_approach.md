# 03 — The Approach: mathematical foundations, stage by stage

This is the derivation document. Every formula the pipeline computes is
developed here from first principles, in pipeline order. Code references
point at the implementation of exactly the math shown.

---

## 0. Notation and coordinate frames

- **Image frame**: pixels (u, v) in the raw camera image.
- **Bed frame**: meters (X, Y, Z); X along bed length, Y across width,
  Z up out of the bed floor. The bed floor is the plane Z = 0.
- **Canonical frame**: pixels of the rectified top-down view; a pure
  scaling of bed-floor coordinates, s = px_per_m.
- Homogeneous coordinates carry a tilde implicitly; "~" means equality up
  to scale.

## 1. The camera model

Pinhole projection with intrinsics K:

    x ~ K [R | t] X,     K = [[fx, 0, cx], [0, fy, cy], [0, 0, 1]]

We obtain K from one spec number (horizontal FOV): fx = (W/2)/tan(hfov/2),
square pixels, centred principal point (`calibration/intrinsics.py`). Full
Zhang calibration (homographies of a checkerboard at several poses give
constraints on the image of the absolute conic ω = K⁻ᵀK⁻¹; decompose by
Cholesky) is documented for rigs that can afford the one-time procedure —
but note §2: K errors do NOT touch on-plane measurements, only the parallax
cue, which is one voice among four.

## 2. The bed as metric anchor

### 2.1 Homography by normalised DLT
For points on the plane Z=0 the projection collapses to a plane-to-plane
homography H (3×3, 8 dof). Each correspondence (X,Y)→(u,v) yields two
linear equations (cross-multiplying away the scale):

    u(h31 X + h32 Y + h33) = h11 X + h12 Y + h13
    v(h31 X + h32 Y + h33) = h21 X + h22 Y + h23

Stack ≥4 points: A h = 0, solve as the right singular vector of the
smallest singular value — the algebraic least-squares optimum. **Hartley
normalisation** (translate centroids to origin, scale mean radius to √2) is
applied first and undone after: raw pixel magnitudes (~10³) would make A's
entries span six orders of magnitude and burn the SVD's precision.
(`geometry/homography.py`)

### 2.2 Camera pose from H
For the Z=0 plane, H = λK[r₁ r₂ t]. Then:

    [r₁ r₂ t] = λ⁻¹ K⁻¹ H,   λ = ||K⁻¹H·e₁||,  sign from cheirality
    r₃ = r₁ × r₂,  then Procrustes-project [r₁ r₂ r₃] to the nearest
    rotation (SVD, UVᵀ) because noise never yields a perfect one.

Camera centre C = −Rᵀt; **camera height above the floor = |C_z|** — needed
by the apparent-size cue. (`calibration/bed_reference.py`)

### 2.3 The canonical view
H_img→canon = S · H⁻¹ with S = diag(s, s, 1). Every frame is warped by
**inverse mapping** (for each output pixel, sample the source at H⁻¹x) with
bilinear interpolation — forward mapping would leave holes. Consequence
that pays for everything: in this view, 1 px² = (1/s)² m², areas are
metric, and the truck's vibration reduces to re-estimating one matrix per
frame. (`geometry/warp.py`)

## 3. Robustness under motion

### 3.1 Frame quality gate
Blur score = Var(∇²I) (variance of the 4-neighbour Laplacian). Motion blur
is a low-pass; second derivatives die first, so this collapses steeply
(measured: 2601 sharp → 6.7 blurred on the test scene). Corruption score =
fraction of pixels at the sensor rails. Frames failing either are NOT
processed — the Kalman layer coasts (§10), so refusing bad data costs a
little variance and buys immunity to garbage. (`stabilization/quality.py`)

### 3.2 Stabilization
Shi-Tomasi corners (structure tensor G; score λ_min — the conditioning of
the tracking equations themselves) are tracked frame-to-frame by pyramidal
Lucas-Kanade. LK derivation: linearise brightness constancy,

    I(x+d, t+1) ≈ I(x,t) + ∇I·d + I_t  = 0
    ⇒ aggregate over a window:  G d = −b,
      G = Σ[∇I ∇Iᵀ],  b = Σ[∇I · I_t]

iterated Newton-style, coarse-to-fine over an anti-aliased image pyramid so
the linearisation's small-motion assumption holds at every level.
RANSAC (500 minimal 4-point samples; inlier = reprojection < 2.5 px;
all-inlier refit) fits the frame-to-frame homography, rejecting tracks on
moving produce/hands — they are simply outliers to the rigid-scene model.
Correction chains: H(cur→calib) = H(prev→calib)·H(cur→prev). If tracking
fails credibility (<12 inliers), we coast on the last correction and lower
downstream confidence — a wrong-but-plausible homography poisons everything
after it, so refusing is the optimal move.
(`stabilization/tracker.py`, `stabilizer.py`)

## 4. Segmentation

### 4.1 Foreground
Sequences: per-pixel adaptive Gaussian mixture (Stauffer-Grimson form).
Pixel matches its nearest component within 2.5σ; matched components update
with rate α (means, isotropic variance, weights), unmatched decay; no match
replaces the lightest component. Foreground = pixel whose matched component
lacks weight — persistence is what earns background status, so lighting is
learned, novelty is flagged. Single stills: saturation thresholding with
TWO candidate cuts — Otsu's between-class-variance optimum σ²_b(t) =
w₀w₁(μ₀−μ₁)², and a noise-floor cut (histogram mode + 4·1.4826·MAD);
min() of the two is taken because Otsu's failure mode (cutting between
produce modes, e.g. deleting all dull potatoes while keeping tomatoes) was
observed directly in testing. (`segmentation/background.py`)

### 4.2 Instances: distance transform + watershed
Chamfer distance (Borgefors weights 1, 1.351) computed by parallel
relaxation d ← min(d, shift(d)+w); sweep count is bounded by the largest
inscribed radius — object-scale, not image-scale. Roundish item centres are
maxima of d; maxima seed a **Vincent-Soille immersion watershed** on −d,
quantised to 64 levels with vectorised label growth per level. Basins
collide precisely at inter-item necks. Peak suppression radius is set from
PHYSICS: 0.3× the smallest active class's metric diameter through px_per_m
— the same code splits cherry tomatoes and pumpkins from any camera height
because the priors live in meters.
**Known limit, by design:** items overlapping so heavily that the union's
distance field is unimodal cannot be split by geometry — detected by a
class-aware size test and routed to count-by-regression (§6).
(`imops/distance.py`, `imops/watershed.py`, `segmentation/segmenter.py`)

## 5. Classification

Features per instance (`classification/features.py`):
- **Circular hue statistics**: mean = arg of the resultant of unit vectors;
  the resultant's length is a free confidence measure (hue concentration).
- Saturation, value means.
- **Texture**: mean gradient magnitude / mean brightness (illumination-
  normalised busyness).
- **Metric size**: equivalent diameter in meters — the feature only a
  calibrated system has, and the one that separates same-colored classes
  four sizes apart.
- Eccentricity from the mask's second-moment eigenvalues; HOG descriptor
  available for texture-structured cases (implemented per Dalal-Triggs:
  cell-wise magnitude-weighted orientation histograms, unsigned, L2).

Classifier: per-class Gaussian prototype modes; score = Σ −½((f−μ)/σ)² per
feature — a diagonal **Mahalanobis distance**, i.e. the brief's "vector
mapping with norms" with units normalised away. Hue term uses wrapped
distance and is weighted by measured hue concentration. Size compared in
log space (biological scatter is multiplicative). Softmax over class scores
plus an explicit "unknown" floor yields calibrated-ish posteriors; winner's
posterior is the reported confidence. Every decision decomposes into
readable per-feature evidence. (`classification/classifier.py`)

## 6. Counting

Resolved instances count 1. Class-aware oversize test (diameter > 1.5×
(μ_size + 2σ_size)) flags merged blobs; their count = blob area / median
area of same-class resolved singletons in the SAME frame — self-calibration
that cancels both prior-size bias and morphology bias (verified: a merged
triple of tangent tomatoes regresses to 3.4 → with its two loose siblings
totals 5 for a ground truth of 5). Fallback denominator is the prior size.
Cluster counts carry 30% sigma into fusion. (`pipeline.py` pass 2)

## 7. Depth: four cues and their error budgets

**(a) Apparent size** (`depth/apparent_size.py`). The canonical warp
assumes points lie on Z=0; an object at height z is closer to the camera,
so it renders magnified by m = h_cam/(h_cam − z):

    z = h_cam (1 − 1/m),    m = d_observed / d_expected

Error propagation: σ_z = h_cam·σ_rel/m², σ_rel ≈ 0.15 (biological size
scatter — irreducible per item). At h_cam = 2.5 m: σ_z ≈ ±0.35 m per item
(nearly useless alone), tightening as 1/√N over a layer's N items. The cue
REPORTS this honestly and lets fusion weight it accordingly.

**(b) Stacking topology** (`depth/topology.py`). Produce doesn't levitate:
items adjacent to visible floor are layer 0; BFS over the contact graph
assigns layers upward. Layer→meters via the hexagonal close packing
inter-layer rise √(2/3)·(item extent) ≈ 0.816·2c. Precise exactly where
apparent-size is blind (low layers); unanchored (+8 cm sigma) when no floor
is visible anywhere.

**(c) Temporal parallax** (`depth/parallax.py`, `geometry/epipolar.py`).
Two frames from the moving truck = a stereo pair with unknown baseline.
Normalised 8-point estimate of F (rank-2 enforced by zeroing σ₃); E = KᵀFK
snapped to singular values (1,1,0); four-fold (R,t) ambiguity resolved by
cheirality (triangulated points must sit in front of both cameras); DLT
triangulation per match. Scale: triangulation gives shape, the bed gives
meters — visible-floor points' distances along the bed-plane normal must
equal the known camera height, and their median vote fixes the global
scale (median = robustness to votes landing on produce). Self-assessed
sigma from median disparity; the cue abstains below ~0.7 px of flow.

**(d) Shape-from-shading** (`depth/shading.py`) — deliberately confined:
within one instance (single albedo, one light), normalised smoothed
intensity is a serviceable *relative* relief proxy for convex produce.
It shapes each instance's dome; it never sets scale. Full SfS needs known
lighting/albedo — lab conditions this problem doesn't have.

**Fusion** (`depth/fusion.py`). Independent Gaussian estimates combine by
inverse variance:

    ẑ = (Σ zᵢ/σᵢ²)/(Σ 1/σᵢ²),   1/σ̂² = Σ 1/σᵢ²

— the minimum-variance unbiased combination, and algebraically a one-shot
Kalman update, which is why it composes cleanly with §10. No cue is ever
switched off by flags; weak cues lose the argument arithmetically. The
fused per-instance heights rasterise into a height map h(x,y): domes shaped
by the shading relief, pinned floor zeros, unknown slivers closed by Jacobi
relaxation of Laplace's equation (surface continuity as a boundary-value
problem).

## 8. Volume, weight, density

**Occupied volume** = Σ h(x,y)·Δa over the canonical grid (Δa = 1/s²) —
the stockpile-standard voxel-column integral. Vacant = bed box − occupied.
Volume sigma treats per-pixel height errors as correlated within instances
(they share one fused estimate): σ_V scales with covered area, not its
square root — the conservative, correct choice. (`volume/heightmap.py`)

**Per-item volume**: for a filled ellipse, central second moments are
μ₂₀ = A²/4, μ₀₂ = B²/4 in the principal frame — so the pixel-coordinate
covariance eigenvalues give semi-axes A = 2√λ₁, B = 2√λ₂ in closed form,
robust to boundary raggedness (moments integrate; a noisy edge pixel moves
them O(1/area)). The unseen vertical semi-axis C comes from the class
prior scaled isotropically by the observed size ratio (clipped to [0.5, 2]).
V = 4/3·π·A·B·C; **mass = V × true flesh density** (priors DB, FAO-sourced).
Error budget per item: unseen axis 15% ⊕ fit 8% ⊕ density prior — carried,
not hidden. Clusters: N × E[V_class] with the 30% count sigma.
(`volume/ellipsoid.py`)

**Density map**: instance masses spread over their footprints, binned into
20 cm cells; hidden mass (§9) spread ∝ per-cell sub-surface volume. Doubles
as an axle-load safety readout. (`volume/density_map.py`)

## 9. The hidden layers: packing physics + Bayesian inference

The generative model (full derivation in docs/05):

    V_sub = occupied volume − Σ_visible (Vᵢ/φ_class(i))     [bulk below the shell]
    hidden solid of class c = V_sub · w_c · φ_c
    hidden count = ·/E[v_c],   hidden mass = ·× ρ_c

with w ~ Dirichlet(α), the only genuinely unknown object. Priors: α₀·w₀
where w₀ integrates the stratification table (heavy-bottom/light-top from
the brief) over the hidden depth range; α₀ = 8 effective observations.
**Evidence:** every removal that exposes previously hidden produce is a
literal sample from w — conjugacy makes the update α_c += k_revealed, one
addition per event. Posterior credible intervals combine Dirichlet variance
(shrinks with every reveal — uncertainty decreases through the day),
packing σ_φ, sub-surface volume σ, and item-size σ in quadrature via
first-order propagation. φ values are anchored on Donev et al. (Science
2004): spheres 0.64, produce-like ellipsoids 0.68-0.71 (survey [9]).
(`occlusion/bayes.py`, `occlusion/packing.py`)

## 10. Change detection and events

**Differencing** runs in the stabilized canonical view, in COLOR (a red
tomato and a gray bed are iso-luminant — grayscale differencing is blind to
the most common event on the truck; found empirically). Per channel, after
local normalisation (subtract 15 px box mean — kills lighting drift):
a pixel is changed only if its new value escapes the [min, max] **envelope**
of its 3×3 neighbourhood in the previous frame. Any sub-pixel resample is a
convex blend of neighbours and cannot leave that envelope; a real removal
leaves it by the full contrast — so stabilizer-residual edge shimmer is
*mathematically* zero while true change is untouched. Threshold: 5× a
MAD-robust noise sigma floored at camera shot noise; regions smaller than
half the smallest item are ignored; warp-fill borders are masked out.
Result on the jittered test sequence: 0 false events on 8 no-change frames,
1 correctly attributed removal on the removal frame.
(`events/differ.py`)

**Event classification**: mask logic decides removed/added when foreground
disagrees before/after; ambiguous produce→produce regions are judged by
optical-flow **coherence** (resultant length of flow direction vectors —
the same circular statistic as hue): coherent motion = shift/settle
(inventory untouched), incoherent appearance change = removal-with-reveal.
A global coherent-flow check catches whole-load settling after bumps and
defers per-region stories during the chaos. Removals expose new instances
→ those are the Bayes model's evidence feed. Only classified, attributed
events touch inventory. (`events/flow.py`)

## 11. Temporal state: Kalman filtering

Per class, per quantity (count/mass/volume), a random-walk-with-jumps
model:

    predict: x ← x + u (event delta),  P ← P + Q
    update:  K = P/(P+R);  x ← x + K(z − x);  P ← (1−K)P

Event deltas enter as control input u — a detected sale moves the belief
instantly, no measurement lag. R comes from each measurement's own reported
sigma, so low-confidence frames move beliefs less; rejected frames simply
skip the update and P grows by Q (principled coasting). The three filters
per class are driven coherently (removing 2 tomatoes decrements count,
mass, and volume consistently). (`tracking/kalman.py`)

## 12. The real-time argument

The image changes at 30 Hz (vibration); the inventory changes at human
speed. So: stabilize + quality gate + envelope differencing every frame
(cheap watchers); segmentation ~6 Hz; depth ~2 Hz; full volumetrics +
Bayes ~1 Hz; and ANY detected event pulls the full stack in immediately.
Nothing that matters is missed, and the expensive math runs two orders of
magnitude less often than the frame clock. Measured wall-clock (pure
single-threaded NumPy, this laptop): light tier ~1.0-1.4 s, full stack
~2-3 s at 400 px/m — and docs/06 gives the measured scaling knobs and the
standard optimisation path (the algorithms are linear in pixels; the two
hot inner loops are textbook SIMD/GPU targets). The tier ARCHITECTURE, not
micro-optimisation, is the load-bearing real-time claim.
