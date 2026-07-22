# Q2 — Counting remaining items per class in real time, informed by depth

## (A) The approach, in plain language

Counting a crate of vegetables has three difficulties, each with a dedicated
mechanism:

1. **Touching items look like one blob.** A crate of tomatoes is a single
   connected region of "red". We split it into individuals with a
   **distance-transform + watershed** — the classical way to cut touching
   round objects apart at the necks where they meet
   (`imops/distance.py`, `imops/watershed.py`, `segmentation/segmenter.py`).

2. **Dense piles can't be split by geometry.** When items overlap so heavily
   that the blob has no separable cores, we stop pretending to resolve
   individuals and **count by area regression**: number ≈ blob area ÷ the
   area of one item, where "one item" is calibrated from the *resolved
   singletons of the same class in the same photo*
   (`classification/classifier.py::cluster_count`).

3. **Depth tells us what counting-in-2D cannot.** The **stacking-layer graph**
   (`depth/topology.py`) tells us which blobs rest on the floor vs on top of
   others, and the height map bounds how many layers deep the pile goes.
   This is how counting becomes *inventory* rather than *surface count*: the
   visible layer is counted directly, and the hidden layers are inferred
   (Q4) and added — the total the vendor actually cares about.

**"…in real time."** The camera runs at 30 fps, but the *inventory* changes
at human speed (someone grabs a tomato every few seconds). So we run a
**tiered scheduler** (`config.py::TierRates`, `pipeline.py`): cheap change-
detection every frame, expensive counting only at ~1 Hz or the moment a
change is detected. Finally, a **Kalman filter** per class
(`tracking/kalman.py`) smooths the count and applies instant decrements when
a removal event fires.

## (B) The mathematics

### Splitting touching items — distance transform + watershed
Compute the **distance transform** of the foreground mask: every pixel gets
its distance to the nearest background pixel (`chamfer_distance`, an O(N)
parallel relaxation with Borgefors weights 1 / 1.351 approximating Euclidean
distance). The **centres** of roundish items are the local maxima of this
field — the points deepest inside the blob. Those maxima seed a
**marker-controlled watershed** run on the *negative* distance field: basins
flood outward from each centre and collide exactly at the necks where two
items touch — which is the true boundary
(`imops/watershed.py`, Vincent–Soille immersion).

The one critical parameter — the minimum separation between peaks — is set
from **physics, not pixels**: `0.3 × (smallest item's metric diameter) ×
px_per_m`. Because the priors live in metres, the same code splits cherry
tomatoes and pumpkins correctly from any camera height.

### Counting the unsplittable — self-calibrated regression
For a cluster region of area `A_cluster`, the count is

```
N  =  A_cluster / A_item
```

where `A_item` is **the median mask area of resolved single items of the
same class in this same frame**. Using the frame's own singletons is the key
trick: it cancels two biases at once — this truck's tomatoes aren't the
catalogue mean size, and the segmentation morphology inflates every region
by the same few boundary pixels. The ratio doesn't care about either. If no
singleton exists to calibrate against, we fall back to the prior item size.
Cluster counts carry a wide (~30%) uncertainty, reported honestly.

### Real time — the tiering argument
Let the frame rate be 30 Hz. The per-frame work is only: quality gate
(variance of Laplacian), stabilisation (track + homography), and colour
envelope differencing — all O(N) single passes. Segmentation +
classification run at ~6 Hz; full depth + volume + hidden-layer Bayes at
~1 Hz; **and any detected change pulls the full stack forward immediately.**
The justification: an image changes at 30 Hz because of *vibration*, but the
*inventory* only changes when a hand removes something — so the expensive
mathematics runs two orders of magnitude less often than the frame clock and
nothing that matters is missed. The tier *architecture* is the real-time
claim; the current pure-NumPy prototype is ~1 s/frame on the heavy tier, and
`docs/06_gpu_notes.md` gives the measured path to 30 fps.

### Kalman smoothing with event decrements
Per class, per quantity (count, mass, volume) a scalar Kalman filter
(`tracking/kalman.py`):

```
predict:  x ← x + u ,      P ← P + Q       (u = event delta, e.g. −2 tomatoes)
update:   K = P / (P + R);  x ← x + K(z − x);  P ← (1 − K)P
```

Two design choices carry the weight: a detected removal enters as the
**control input u**, so the count drops the instant a sale is seen (no lag);
and the measurement noise **R is each measurement's own reported σ**, so a
low-confidence frame nudges the belief instead of yanking it. A rejected
frame simply skips the update — the belief coasts, its uncertainty P grows —
which is exactly how a vibrating truck's dropped frames should be handled.

## (C) They might also ask…

**"How is this real-time on a CPU with no GPU?"**
By matching computation to information rate, not frame rate. The 30 fps
watchers are all linear-time single passes; the expensive per-item geometry
runs at ~1 Hz or on-event. Nothing in the pipeline is super-linear in pixel
count. See the tiering argument above and `docs/06_gpu_notes.md`.

**"Concretely, how does *depth* help you *count*?"**
Two ways. (1) **Occlusion accounting**: a surface count is not an inventory —
the height map + packing physics tell us how much produce sits *below* the
visible layer, which is added as the hidden count (Q4). (2) **Layer
disambiguation**: the stacking graph distinguishes "two items side by side"
from "one item with a highlight that segmentation split in two", because
genuine neighbours share a contact edge while a spurious split doesn't
correspond to a real layer boundary.

**"How do you count items that are identical and touching?"**
The watershed doesn't need them to differ in appearance — it cuts on
*geometry* (the distance-field necks between their centres), so twenty
identical touching tomatoes still separate into twenty basins as long as
each has a distinct distance-field maximum. When they overlap so much that
maxima merge, we switch to area regression rather than guess.

**"What about counting while the truck is moving / shaking?"**
Counting happens in the **stabilised canonical view**: we first cancel the
camera motion by tracking features and re-estimating the homography every
frame (Lucas–Kanade + RANSAC), so the counter sees a still bed. Blurred
frames are rejected by the quality gate and the Kalman filter coasts through
them. This is why the count doesn't flicker on a bumpy road.

**"Won't the count double-count or drop items between frames?"**
The Kalman filter maintains one belief per class across frames rather than
recounting from scratch, and inventory changes only enter through
*attributed events* (a classified removal/addition). Ambiguous change defers
to the next measurement pass instead of inventing a count change — a change
log that occasionally misses a beat is recoverable; one that invents sales
is not.
