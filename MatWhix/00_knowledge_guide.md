# 00 — The Knowledge Guide

A step-by-step walk through every mechanism in the system: what it does,
why it exists, how the math works, and a worked example with real numbers —
most taken straight from the actual runs on the market photos in `images/`.
Read this first; the formal derivations live in `03_approach.md`.

---

## The big picture in 60 seconds

```
photo/frame ──► [0 quality gate] ──► [1 stabilize to top-down view]
                                            │
                              ┌─────────────┴──────────────┐
                              ▼                            ▼
                    [2 what changed?]            [3 find & name produce]
                    (every frame, cheap)         (segmentation+classification)
                              │                            │
                              ▼                            ▼
                    [7 change log,               [4 how high is everything?]
                     sale events]                (4 depth cues, fused)
                              │                            │
                              │                            ▼
                              │                  [5 liters and kilograms]
                              │                            │
                              │                            ▼
                              └────────────► [6 what's hidden underneath?]
                                             (packing physics + Bayes)
                                                           │
                                                           ▼
                                             [8 Kalman-smoothed dashboard]
```

One idea underlies everything: **the truck bed (or crate) is the only thing
of known size in view, so it becomes the ruler, the coordinate system, and
the anchor for every metric claim.**

---

## Step 0 — Turning a camera into a measuring device

**The problem.** A single camera cannot measure size. A 6 cm tomato at 1 m
and a 12 cm tomato at 2 m produce *identical* pixels. Every monocular
system that outputs kilograms smuggles in a known length somewhere; ours is
declared openly: the bed/crate dimensions, measured once with a tape.

**The mechanism.** Four bed corners in the image + the bed's real
dimensions define a 3×3 *homography* H mapping bed-meters to image-pixels.
We estimate it with the normalised DLT (each corner gives 2 linear
equations; SVD solves the stack; Hartley normalisation keeps the arithmetic
stable). Inverting H and scaling gives the **canonical view**: a synthetic
top-down image where 1 pixel = a fixed number of millimeters, everywhere.

**Worked example (tomato crate photo).** We told the system the crate is
0.54 × 0.36 m and asked for 1200 px/m. The canonical view is then
648 × 432 px, and *any* blob's area in it converts directly:
a tomato mask of 3,385 px → equivalent diameter 2·√(3385/π)/1200 =
**0.055 m** — a 5.5 cm tomato, read off a photo with no depth sensor.

**Bonus.** Decomposing H = K[r₁ r₂ t] also yields the camera's height above
the bed — needed later by the apparent-size depth cue. One rectangle, four
gifts: scale, coordinates, warp target, camera pose.

---

## Step 1 — The quality gate (refusing bad frames is a feature)

**Why.** On a moving truck some frames are motion-blurred mush. A blurred
frame doesn't carry *less* information — it carries *wrong* information
(blur widens objects → inflates area → inflates volume → inflates weight).

**How.** Variance of the Laplacian (a second-derivative filter). Blur
destroys fine detail first, so this number collapses early.

**Worked example.** Our test scene scores **2601** sharp; after a σ=3 blur
it scores **6.7** — a 400× drop for a blur a human might squint past.
Frames failing the gate are simply not measured; the Kalman layer (step 8)
carries beliefs through the gap with growing uncertainty. Nothing lies.

---

## Step 2 — Stabilization (making the camera stand still, in software)

**Why.** Frame differencing is the cheap watchdog for "did someone take
something?" — but a 5 px camera jiggle makes *every edge in the image*
light up as fake change.

**How.** Shi–Tomasi corners (points whose local structure tensor has two
strong eigenvalues — provably the points that track well) are followed
frame-to-frame with pyramidal Lucas–Kanade (solve G·d = −b over a window,
coarse-to-fine). RANSAC then fits a single homography to the tracked
motion, automatically discarding points that moved *with produce or hands*
rather than with the rigid scene. The correction chains frame after frame.

**Worked example.** A synthetic shake of (4.3, −2.7) px + 0.57° rotation:
150/150 corners tracked, median error **0.04 px**, in 60 ms. Stabilized
canonical frames differ by 0.35 gray-levels on average; unstabilized, 45.6
— a **130× reduction**, which is the difference between a usable change
detector and noise.

---

## Step 3 — Finding the produce (foreground extraction)

**Why it's hard.** No training data. The separator has to come from
physics: produce is pigmented, bed materials are not.

**How.** Threshold the HSV *saturation* channel. The threshold self-tunes
two ways and takes the more inclusive: Otsu's optimum (perfect for clean
two-population histograms) and a noise-floor cut (histogram mode + 4 robust
sigmas — immune to Otsu's multi-modal failure). Very dark pixels (shadow
pits) are excluded. Morphological open/close — with *metric* radii, ~4 mm —
clean specks and heal specular highlights.

**The war story (worth telling your boss).** The real potato pile broke
this: dusty potatoes measure median saturation **0.093** — *less colorful
than many bed materials* — so the mask claimed only 26% of a full pile,
and the rest impersonated visible floor. The fix is a second physics
argument: look at the *rejected* region — genuine floor is optically
smooth, unsaturated produce is texture everywhere. Rejected-region gradient
(after a 1 px blur to kill sensor noise): bare beds ≈ 1–2, produce ≈ 4+.
When more than half the frame is "background" but that background is busy,
the background is produce. After the fix: potato coverage 26% → ~100%, and
the pile's inventory went from nonsense to 88 kg (see step 6).

---

## Step 4 — One blob into individual vegetables (the watershed split)

**Why.** Touching tomatoes form ONE connected blob. Counting needs
individuals.

**How — the two-step classic:**
1. **Distance transform**: each blob pixel gets its distance to the blob
   edge. Item centres = deep points = local maxima.
2. **Watershed**: flood the *negative* distance field from those maxima.
   Basins grow outward and collide exactly at the necks where items touch
   — which is where the true boundary is.

The peak-separation parameter comes from the priors *in meters* (0.3 × the
smallest active item's diameter), so the same code splits cherry tomatoes
and pumpkins from any camera height.

**Worked example.** Two overlapping 35 px-radius discs: chamfer max = 33.8
(true answer 35, chamfer is a 4%-accurate approximation), watershed splits
the union into 3,622 + 3,622 px — exactly even, as symmetry demands.

**The honest limit.** Three tomatoes pressed deep into each other produce a
distance field with ONE peak (measured: 17.8 px deep at the *centroid* of
the triple — deeper than any single tomato's 13!). Geometry cannot split
that. Which is why the next two steps exist.

---

## Step 5 — Naming things (classification without a neural network)

**How.** Five features per instance, each carrying real physics:
- **hue** — circular mean (average the unit vectors, not the angles — the
  naive mean of 350° and 10° is 180°, i.e. calling a tomato cyan);
  the resultant's length doubles as "how much to trust color here"
- **saturation, value** — pigment strength, brightness
- **texture** — gradient energy ÷ brightness (a bitter gourd's ridges vs a
  capsicum's gloss)
- **metric size** — the calibration dividend: five of our market classes
  are *all green*; size and shape tell them apart, color never could
- **eccentricity** — 0 = ball, 1 = line (a chilli is 0.96, a capsicum 0.35)

Each class is a prototype (editable JSON — that's the whole "training
set"); the score is a sum of per-feature Gaussian log-penalties (a diagonal
Mahalanobis distance), softmaxed against an explicit "unknown" floor.

**Worked example (real crate).** A dusty potato instance measured
sat 0.32, hue 34.5°, diameter 0.075 m. Against the potato prototype
(sat 0.30±0.13, hue 38±14, size 0.068): all terms small → potato at 0.80
confidence. The known confusable pair is potato↔onion (similar size AND
color) — on the synthetic benchmark it costs one item in fifteen, and we
report that rather than hide it.

---

## Step 6 — Counting the uncountable (cluster regression)

**Why.** Step 4's honest limit: merged piles that geometry can't split.

**How — two mechanisms discovered the hard way:**
1. **The cluster hypothesis.** A merged mass of 40 tomatoes has *perfect
   tomato color* and an *impossible tomato size*. Under the full scorer the
   size term buried the color evidence → "unknown" → counted zero (this
   deleted three-quarters of the real crate on first contact). Oversized
   unknowns are therefore re-scored on color+texture only; a confident
   winner is accepted as a CLUSTER of that class.
2. **Self-calibrated count regression.** Cluster count = cluster area ÷
   median area of *resolved singletons of the same class in the same
   frame*. Using the frame's own singletons cancels both biases at once:
   this truck's tomatoes aren't the catalogue mean, and the mask morphology
   inflates every region equally — the ratio doesn't care.

**Worked example (real tomato crate).** Largest cluster: 69,766 px.
Median singleton tomato: ~2,000 px. The regression + the resolved singles
give **122 visible tomatoes (9.2 kg)** — eyeball count of the photo says
90–110 on the surface, so we're in band, and the ±18 sigma admits the
slack honestly.

---

## Step 7 — Height from a flat photograph (four weak cues, one strong fusion)

No single monocular cue survives contact with reality, but their failure
modes don't overlap — so each reports (estimate, sigma) and an
inverse-variance average lets arithmetic pick the winner per situation:

| Cue | Physics | Where it's good | Where it's blind |
|---|---|---|---|
| Apparent size | items above the bed render magnified: z = h(1−1/m) | high piles | near the floor (σ≈±0.35 m per item!) |
| Stacking topology | produce doesn't levitate: BFS layers over the contact graph | low layers, parked truck | fully covered beds (no anchor) |
| Temporal parallax | two frames from a moving truck = a stereo pair (8-point → triangulate → bed-anchored scale) | any time the truck moves | parked, or motion < 0.7 px |
| Shape-from-shading | within one item, brightness tracks surface slope | dome *shape* | absolute scale (never asked to provide it) |

**Worked example of fusion.** A floor-level tomato: apparent-size says
z = 0.02 ± 0.35 m (useless, admits it), topology says z = 0.028 ± 0.014 m
(confident). Inverse-variance blend: weights 1/0.35² vs 1/0.014² — topology
wins 600:1, fused z = 0.028 ± 0.014. No if-statements, no thresholds; bad
cues lose the argument numerically.

**The rim-fill prior (packed beds).** When *no* floor is visible anywhere,
"anchorless" is itself evidence — beds fill from the bottom, so the load
reaches roughly the surface we see, bounded by the wall height. The scene
gets a base offset of (wall − one item) with a fat ±30%-of-wall sigma.
Real tomato crate before this prior: 5 L occupied (absurd). After: 41 L of
a 54 L crate — 76% full, which is what the photo shows.

---

## Step 8 — Liters and kilograms

**Occupied volume** = sum of the height map × pixel area (the stockpile
industry's voxel-column integral). **Vacant** = bed box − occupied — the
"how much more can I load" number.

**Per-item mass**: closed-form ellipse fit from image moments (for a filled
ellipse, covariance eigenvalues λ give semi-axes 2√λ — no iterative
fitting, robust to ragged edges), third axis from the class prior scaled by
the item's own size, then × true flesh density from the FAO-sourced priors.

**Worked example.** Real capsicum crate: 30 visible capsicums fitted →
3.72 kg → **124 g each**. Market capsicums run 100–150 g. No scale touched
a vegetable.

**A pattern worth knowing:** *mass* is the robust output (volume × density
× packing — errors partially cancel), *count* is the fragile one (it
divides by per-item volume, so a 20% size-prior error becomes a 70% count
error for small items). The chilli crate shows it: mass 11.5 kg (solid —
real chilli crates hold 10–12 kg), count CI enormous (±950) because
chillies are 3 ml each. The report prints both with intervals; the vendor
trusts accordingly.

---

## Step 9 — The layer no camera can see (the packing-physics Bayes model)

**The chain, with the real tomato-crate numbers:**

1. Occupied bulk volume (step 8): **41.3 L**
2. Visible items claim their share: 122 tomatoes × 78 ml ÷ φ → ~15 L
3. Sub-surface bulk: 41.3 − 15 ≈ **26 L** of pile we cannot see into
4. Physics bounds what's in it: random close packing of near-spheres jams
   at **φ ≈ 0.64** (Donev et al., Science 2004 — a measured constant, not
   a tuning knob). Solid volume hidden: 26 × 0.64 ≈ 16.6 L
5. Class mixture w over that volume is the only true unknown → Dirichlet
   posterior. Prior: the stratification table (heavy classes sink) +
   what's visible on top. Here: single-class crate, w = 1.
6. Hidden count = 16.6 L ÷ 78 ml ≈ **218 tomatoes**; hidden mass ≈ 17 kg.
7. Total: 122 + 218 = **334 ± 132 tomatoes, 26.2 kg** — and a real 54 L
   crate of romas weighs 25–28 kg. The physics closes.

**The operational trick (the novel piece):** every sale is a core sample.
A removal exposes previously hidden items; those are literal draws from w,
and Dirichlet conjugacy folds them in with one addition (α_c += k). The
hidden-layer uncertainty *shrinks all day* — the dashboard shows the
posterior strength growing as the vendor works.

---

## Step 10 — "Someone took three tomatoes" (change events)

Three hard-won mechanisms make the change log trustworthy:

1. **Color differencing, not grayscale.** A red tomato (luma ≈ 92) on a
   gray bed (≈ 95) is *iso-luminant*: removing it changes the gray image by
   3 levels — invisible. Differencing per RGB channel and taking the max
   restores the ~120-level red-channel contrast the event actually has.
   (Found when the test removal produced literally zero signal.)
2. **The envelope test.** Sub-pixel stabilizer residual × steep edge beats
   any threshold. But a sub-pixel resample is a convex blend of neighbours,
   so it *cannot* leave the neighbourhood's [min,max] range — while a real
   removal leaves it by full contrast. Edge shimmer: identically zero.
3. **Flow coherence for the ambiguous case.** Produce→produce change is
   either the load *settling* (coherent flow — everything slides together)
   or a *removal that exposed deeper produce* (incoherent). Coherence =
   resultant length of flow direction vectors — the same circular statistic
   as hue, reused.

**Measured result:** 8 jittered no-change frames → **0 events**. The frame
with a genuine removal → exactly one event, `removed: tomato`, logged with
location and ~mass. Only classified, attributed events touch inventory —
an ambiguous blob defers to the next measurement pass rather than
inventing a sale.

## Step 11 — Steady numbers (Kalman filtering)

Per class, per quantity: `predict x←x+u, P←P+Q; update x←x+K(z−x)` with
K = P/(P+R). Three design choices do the work: **events enter as control
input u** (a detected sale moves the belief *instantly*, no lag);
**R comes from each measurement's own reported sigma** (a shaky frame moves
beliefs less); **rejected frames just skip the update** (belief coasts, P
grows — principled, not special-cased).

**Worked example.** Belief: 10 tomatoes ± 0.5. A poor-quality frame
measures 12 ± 2. K = 0.25/(0.25+4) = 0.06 → new belief 10.12. The noisy
measurement nudged, not yanked. Then a removal event u = −2 fires: belief
drops to 8.12 immediately. That asymmetry — skeptical of noise, instant on
events — is what makes the dashboard readable.

---

## The real-photo scorecard (your images, this pipeline)

| Photo | Output (visible + hidden, Kalman-tracked) | Reality check |
|---|---|---|
| Tomato crate | 334 ± 132 tomatoes, **26.2 kg**, 76% full | full 54 L roma crate ≈ 25–28 kg ✔ |
| Potato pile | 509 ± 213 potatoes, **88.3 kg** → 173 g each | full 122 L bin ≈ 91 kg by packing physics ✔ |
| Capsicum crate | 116 ± 46, **13.2 kg** → 124 g each | market capsicum 100–150 g ✔ |
| Green chilli crate | **11.5 kg** (count CI huge — see step 8's pattern) | chilli crates hold 10–12 kg ✔ |
| Okra crate | 910 ± 464, **17.4 kg** | full okra crate 15–18 kg ✔ |
| Ivy gourd crate | **27.4 kg** ± wide | plausible-to-high; single photo, no reveals yet |
| Bottle gourd crate | 10.5 visible, **17.9 kg** → 1.7 kg each | bottle gourds 1.2–2 kg ✔ (undercounts touching pale gourds) |
| Mixed display | per-class table in `runs/real_mixed_display/` | hardest scene; wide CIs, honest |

Known limits, stated: potato↔onion confusion; very dark produce (beetroot)
can hide below the color threshold in mixed scenes; counts of tiny items
carry wide intervals by arithmetic necessity; single photos can't use
parallax or reveal-events (both wake up on sequences).

## Where to read the code

Follow the pipeline order: `config.py` (all constants + priors) →
`imops/` (the hand-built image math) → `geometry/homography.py` (step 0)
→ `stabilization/` (steps 1–2) → `segmentation/` (steps 3–4) →
`classification/` (steps 5–6) → `depth/` (step 7) → `volume/` (step 8) →
`occlusion/` (step 9) → `events/` (step 10) → `tracking/kalman.py`
(step 11) → `pipeline.py` (the conductor). Every file's comments tell you
why it exists and what broke before it looked this way.
