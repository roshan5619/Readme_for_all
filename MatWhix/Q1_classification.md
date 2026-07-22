# Q1 — Classifying each vegetable from a single top-down (or any) view

## (A) The approach, in plain language

Two sub-problems hide inside this question, and we answer them separately:

**"…from any view."** A photo taken at an angle shows the produce in
perspective — far items look smaller, the crate looks like a trapezoid. We
remove that perspective *before* doing anything else. Because we know the
four bed/crate corners and the bed's true rectangle, there is exactly one
projective transform (a **homography**) that maps the slanted image onto a
clean, scaled, top-down view. We call this the **canonical view**. So
"top-down or any view" is a single answer: **any view is rectified to the
same top-down metric frame first.** (`geometry/homography.py`,
`calibration/bed_reference.py`, `geometry/warp.py`.)

**"…classify each vegetable."** In the canonical view we find each item
(segmentation, see Q2) and identify it with a **hybrid classifier**:

- **A fine-tuned CNN is the primary recogniser** — MobileNetV3-Small,
  pretrained on ImageNet, with its head retrained on a 15-class vegetable
  dataset (99.5% validation accuracy). It answers "what vegetable is this
  crop?" (`classification/cnn_classifier.py`, `train_classifier.py`).
- **A from-scratch prototype classifier is the fallback** — for the classes
  the CNN wasn't trained on (okra, ivy gourd, chilli, beetroot, onion, leafy
  greens) and whenever the CNN is unsure. It scores hand-built features
  against per-class prototypes (`classification/classifier.py`,
  `classification/features.py`).

The CNN wins when it is confident about a class it knows; otherwise the
geometry-aware prototype classifier decides. Each item records *which*
classifier named it, so the result is auditable.

## (B) The mathematics

### Rectification: the homography
A plane in the world and the image plane are related by a 3×3 homography
**H** (8 degrees of freedom). For a bed point (X, Y) and its image point
(u, v), in homogeneous coordinates:

```
[u]       [X]
[v] ~  H · [Y]        (~ means equal up to scale)
[1]       [1]
```

Each corner correspondence gives **two** linear equations (obtained by
cross-multiplying to eliminate the unknown scale). Four corners → eight
equations → solve **A h = 0** by taking the right singular vector of the
smallest singular value (the **Direct Linear Transform**, `estimate_homography`).
We apply **Hartley normalisation** first (translate the points' centroid to
the origin, scale so the mean distance is √2); without it, raw pixel
magnitudes (~10³) make the matrix badly conditioned and the SVD loses
precision. We warp by **inverse mapping** — for every output pixel we look
up where it came from via H⁻¹ and bilinearly sample — which leaves no holes
(`geometry/warp.py`).

**Why this gives metric size.** In the canonical view we fix a resolution
(e.g. 1200 pixels/metre), so an item's pixel area converts directly to a
real area, and its **equivalent diameter** is
`d = 2·√(area_px / π) / px_per_m` — in metres. That number is the single
most discriminative feature we have (see below).

### The prototype feature vector
For each segmented item we compute five physically-meaningful features
(`classification/features.py`):

- **Circular-mean hue** `H̄ = atan2(⟨sin H⟩, ⟨cos H⟩)`, plus the **resultant
  length** `R = |⟨e^{iH}⟩| ∈ [0,1]` which measures how *consistent* the
  colour is (used to down-weight the colour term when a patch is a mush of
  hues). Hue must be averaged on the circle, not linearly.
- **Saturation** and **Value** means (the potato/onion axis is saturation).
- **Texture** = mean gradient magnitude ÷ mean brightness (illumination-
  normalised busyness — a bitter gourd's ridges vs a capsicum's gloss).
- **Metric equivalent diameter** (above) — separates classes that share a
  colour but differ in size (a green chilli vs a green capsicum).
- **Eccentricity** from the mask's second-moment (inertia) tensor: with
  eigenvalues λ₁ ≥ λ₂, `ecc = √(1 − λ₂/λ₁)` — 0 for a ball, →1 for a rod.

### The prototype score is a diagonal Mahalanobis distance
Each class *k* is a Gaussian prototype with mean μₖ and per-feature spread
σₖ (stored as editable JSON — that *is* the "training set"). The log-score
of feature vector **f** is

```
log p(f | k)  =  Σᵢ  −½ ( (fᵢ − μₖ,ᵢ) / σₖ,ᵢ )²   +  const
```

i.e. a **diagonal Mahalanobis distance** — the "vector mapping with norms"
the brief asks for, with every feature's units normalised away by its own
σ. Hue uses wrapped angular distance and is weighted by the resultant length
R; size is compared in log-space (biological size scatter is multiplicative).
A softmax over classes, against an explicit "unknown" floor, gives calibrated
confidences. **Every decision decomposes into readable per-feature evidence.**

### The CNN component
The backbone is **frozen** (transfer learning): we run each training image
through it once to get a 576-dimensional embedding, cache the embeddings,
and train only a small linear head (multinomial logistic regression) on
them. This is why it fine-tunes in minutes on a CPU, and why it counts as a
*component* rather than a black-box end-to-end system — it only maps an
embedding to a label.

## (C) They might also ask…

**"If you have a CNN, why keep the hand-built classifier at all?"**
Three reasons. (1) Coverage: the public dataset has 15 classes; our fallback
covers the six Indian-market vegetables it lacks. (2) The prototype
classifier uses **metric size**, a feature the CNN structurally cannot see
(a CNN gets a resized crop with no scale) — so it disambiguates cases the CNN
can't, like a small vs large green item. (3) Graceful degradation: no torch,
no weights, no problem — the system still runs.

**"How can a single 2D image give you real metric size?"**
Through the bed homography. The bed is a known rectangle; once H maps bed
metres to pixels, the inverse fixes the pixels-per-metre of the canonical
view, so every projected area is metric. This is the same "known reference
object" trick used in metric monocular vision, with the bed as the
reference.

**"Why average hue on a circle?"**
Hue is an angle: red sits at both 0° and 360°. The naïve mean of 350° and
10° is 180° (cyan) — completely wrong. Averaging the unit vectors e^{iH} and
taking the resultant's angle gives 0° (red), correctly, and the resultant's
*length* is a free confidence measure.

**"What happens with no class hint at all (open-set)?"**
The CNN correctly names the dominant vegetable in every crate we tested
(tomato, potato, capsicum, carrot, brinjal). The honest caveat: for a
*cluster* blob, the prototype fallback can occasionally over-reach to a
class that isn't present (e.g. a phantom "watermelon" on a big green pile) —
because unconstrained open-set over 16 classes has inherent false positives.
Two mitigations, both supported: pass the stocked class list (`--classes`),
or extend the CNN's training folders to cover the missing classes so
recognition, not the fallback, decides them.

**"Is a homography valid when the produce is piled up, not flat?"**
The homography rectifies the **bed plane** exactly. Items sitting *above*
that plane are rendered slightly magnified — and we don't fight that; we
*use* it as a depth cue (apparent-size, see Q4). For classification the
residual perspective on a pile is small relative to the feature spreads, and
the metric-size feature already models item-size variation with a σ.
