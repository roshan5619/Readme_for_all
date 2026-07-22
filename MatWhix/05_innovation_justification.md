# 05 — Innovation Justification

What in this system is genuinely new, what is a novel combination, and what
is deliberately classical. Claims are scoped carefully — overclaiming
novelty is the fastest way to lose a technical audience.

---

## Claim 1 (the centerpiece): a physics-grounded Bayesian posterior over
## the layers no camera can see — whose uncertainty SHRINKS during operation

Every occluded region in every vision system is a guess. The innovation is
not guessing better pixels — it's changing what kind of object the guess
is. We model the hidden region's class mixture as a Dirichlet random
variable and connect it to observables through *granular packing physics*:

    hidden solid volume = (bulk sub-surface volume) × φ
    φ = random-close-packing fraction ≈ 0.64 (spheres) … 0.71 (ellipsoids)

φ is not a tuning knob — it's a measured physical constant per shape class
(Donev et al., Science 2004), with a few percent spread. That single fact
converts "how many onions are buried?" from unanswerable to a bounded
inference problem.

Then the operational insight: **every sale is a core sample.** When a
customer's removal exposes previously hidden produce, the newly visible
items are literal draws from the hidden mixture, and Dirichlet conjugacy
folds them in with one addition per event (α_c += k). Consequences:

- The system's hidden-layer confidence *increases monotonically through
  the working day* — the opposite of drift. The dashboard displays the
  posterior strength so the vendor watches certainty grow.
- The change log stops being just a receipt trail and becomes a sensor.
- The math is O(1) per event — real-time by construction.

We found no prior published system doing produce-inventory occlusion
inference this way (nearest neighbours: amodal segmentation, which
completes *partially visible* objects only, and stockpile volumetrics,
which measures bulk but never composition). Scoped claim: the novelty is
the *combination and operationalisation* — Dirichlet latent mixture +
RCP physics + reveal-events-as-evidence — not any single ingredient.

## Claim 2: the truck bed as a full metric reference frame

Using a known object for scale is standard. Using the bed as the *entire
working coordinate system* is the design move: one measured rectangle
simultaneously provides (a) absolute scale for every measurement, (b) the
canonical top-down frame where vibration reduces to one homography update,
(c) camera pose (height!) via plane-pose decomposition, feeding the
apparent-size depth cue, and (d) the metric anchor that converts parallax
triangulation from shape-only to meters. Four birds, one tape measure, no
added hardware — on a problem whose constraint list bans added hardware.

## Claim 3: stacking topology as a depth cue

"Produce doesn't levitate" — items not adjacent to visible floor must rest
on other items, so BFS over the instance contact graph yields discrete
layer indices, converted to meters through the hexagonal-close-packing
inter-layer rise (√(2/3) ≈ 0.816 of item extent). It is precise exactly
where the classical monocular cues are blind (low layers, stationary
truck), and it costs microseconds. We have not seen contact-graph layering
used as a quantitative monocular depth cue in the produce/pile literature.

## Claim 4: honest-sigma cue fusion as the robustness mechanism

Not new mathematics (inverse-variance fusion is textbook) — the design
discipline is the contribution: every cue must publish a variance derived
from its actual physics (apparent-size: σ_z = h·σ_rel/m², which honestly
says "±0.35 m near the floor"; parallax: ∝1/disparity, which honestly says
"useless when parked"). No cue is ever gated by an if-statement; bad cues
lose arithmetically. Robustness emerges from bookkeeping rather than
heuristics, and every dashboard number carries a defensible interval.

## Claim 5: the envelope differencer for vibrating platforms

Sub-pixel stabilizer residuals × steep edges defeat thresholded
differencing (and integer-shift-tolerant differencing — we measured both
failing). The min-max neighbourhood envelope test exploits a small theorem:
a sub-pixel resample is a convex combination of neighbours and therefore
cannot exit the neighbourhood's [min, max] range — while true change exits
by full contrast. False edge-shimmer response is *identically zero*, not
merely suppressed. Combined with per-channel color differencing (a red
tomato and a gray bed are iso-luminant — grayscale differencing is provably
blind to the commonest event on this truck; we hit this in testing) and a
shot-noise floor, the measured result on jittered sequences: 0 false events
in 8 no-change frames, 1/1 true removal detected and attributed.

## Deliberately NOT innovated

Homography estimation, LK tracking, watershed, Kalman filtering, Otsu,
moments-based ellipse fitting: all implemented faithfully from the
literature, by hand, because the brief rewards *understanding* — and
because boring, proven components are what you want under the interesting
ones. Innovation budget was spent only where the problem genuinely lacked
an off-the-shelf answer: the occlusion inference, the cue set, and the
vibration-proof change detection.
