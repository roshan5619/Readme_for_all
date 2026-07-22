# 01 — Problem Analysis: variations explored, framing chosen, and why

Before writing a line of code I mapped the space of ways this problem can be
framed, because the framing decides everything downstream. This document
records the variations considered and the reasoning that picked the winner.

## The problem in one sentence

From a single monocular camera over a truck bed of mixed, stratified,
occluding vegetables, continuously estimate: per-class identity, count,
weight, volume, spatial density, and the composition of layers the camera
cannot see — while the truck moves, at or near frame rate.

## What makes it fundamentally hard (the three walls)

1. **The scale wall.** A single camera cannot measure absolute size. Double
   every distance and every object size and the image is pixel-identical.
   Any monocular system that reports kilograms is getting metric scale from
   *somewhere* outside the image — the only design freedom is choosing where
   from, honestly.
2. **The occlusion wall.** No optical sensor measures what light doesn't
   reach. The bottom layers are *unmeasurable*, period. Every system must
   guess them; the only design freedom is whether the guess is principled
   (priors + evidence + stated uncertainty) or hidden inside a hand-wave.
3. **The motion wall.** The camera platform vibrates, so naive frame
   differencing sees "change" everywhere, and blur destroys measurements
   randomly. The system needs an explicit notion of frame trust.

Naming these walls early mattered: each got a dedicated architectural
answer (bed-as-ruler, Bayesian latent inventory, stabilize-then-gate), and
nothing downstream pretends a wall isn't there.

## Framings considered

### A. End-to-end deep learning ("YOLO + depth net + count head")
Train/fine-tune detectors, a monocular depth network, maybe a counting CNN.
- **For:** best raw detection accuracy on messy real imagery; fastest to a
  flashy demo.
- **Against:** the brief explicitly disallows pre-built end-to-end pipelines
  as final solutions and scores mathematical rigor and transparency; no
  labelled truck-vegetable dataset exists on day one; CPU-only real-time is
  hostile to deep stacks; and when it's wrong, nobody can say why.
- **Verdict:** rejected as the *spine*, retained as an optional *component*
  slot (a depth network can serve as one fused cue — see docs/06).

### B. Pure classical geometry ("SfM reconstructs everything")
Full structure-from-motion of the load, dense multi-view stereo, mesh it,
slice it.
- **For:** maximal geometry credentials.
- **Against:** produce piles are near-textureless in depth-relevant ways,
  the baseline is tiny at market stops, dense MVS on CPU is nowhere near
  frame rate, and SfM alone still answers nothing about identity, weight,
  or hidden layers. Reconstruction is a MEANS here, not the deliverable.
- **Verdict:** rejected as the spine; its useful core (epipolar geometry on
  frame pairs, triangulated sparse points, metric anchoring) became one of
  the four depth cues.

### C. "Weigh the pile with physics" (chosen)
Treat the problem as *measurement + inference*: calibrated plane geometry
turns the camera into a ruler for everything visible; granular-packing
physics plus Bayesian inference handles everything invisible; temporal
filtering handles everything shaky. Vision techniques are cue generators
feeding one explicit probabilistic state.
- **For:** every number the dashboard shows has a derivation and an error
  bar; each cue is independently testable; degradation is graceful (cues
  drop out, uncertainty widens, nothing lies); it directly maximises the
  scoring rubric (rigor, completeness, innovation, robustness); CPU
  real-time is achievable by tiering because the expensive stages run at
  inventory speed, not vibration speed.
- **Against:** raw per-instance detection accuracy on adversarial imagery
  will trail a trained detector — accepted and stated, with the component
  slot as the escape hatch.

## Sub-problem framing decisions worth recording

- **Where does metric scale come from?** Candidates: known camera height
  (fragile — the pile itself changes the geometry), a placed marker (extra
  hardware), known object sizes (circular — sizes are what we estimate), or
  the **truck bed dimensions** (measured once with a tape, rigid, always in
  frame, defines both the scale AND the working coordinate frame). The bed
  won on every axis. It also gives camera pose for free via homography
  decomposition.
- **Work in image space or bed space?** All measurement happens in a
  rectified top-down **canonical view** (the bed frame). This one decision
  collapses three problems: vibration becomes a homography update,
  differencing becomes meaningful, and pixel areas become square meters.
- **One depth cue or several?** Every monocular cue fails somewhere:
  apparent-size is blind near the floor, parallax dies when parked, shading
  can't do absolute scale, stacking topology unanchors on packed beds. But
  their failures are *uncorrelated*, so inverse-variance fusion of all four
  beats any one — and the fusion arithmetic IS the honesty mechanism (a
  cue's sigma is its confession).
- **Counts as integers or beliefs?** Beliefs (Kalman-filtered reals with
  sigma, Dirichlet posterior for hidden mix). Rounding happens at the
  dashboard, never inside the pipeline — premature integerisation throws
  away exactly the information the fusion needs.
- **What is an "event"?** Removal, addition, shift/settle, or global settle
  — distinguished by mask logic + optical-flow coherence. Only classified,
  attributed events touch inventory; ambiguous change defers to the next
  measurement pass. A change log that occasionally misses one beat is
  recoverable; one that invents sales is not.

## Requirement-to-module map

| Brief requirement | Answered by |
|---|---|
| Classification | canonical-view segmentation + prototype classifier (metric size as a feature) |
| Counting | watershed instances + self-calibrated cluster regression + Kalman |
| Weight | ellipsoid fits x true density; hidden mass via packing model |
| Volume | height-map integration (occupied/vacant); per-item ellipsoids |
| Density mapping | per-cell mass grid, visible + model-based hidden |
| Depth / layering | four-cue fusion; stacking topology; Bayesian latent layers |
| 30 FPS operation | tiered scheduling: cheap watchers every frame, expensive analytics on change or timer |
| Occlusion | Dirichlet latent-inventory model fed by reveal events |
| Irregular shapes | moment-based ellipsoid fits, no rigid templates; superquadrics documented as the upgrade |
| Motion robustness | quality gate -> stabilizer -> envelope differencing -> Kalman coasting |
| Partial frame loss | quality gate rejects, Kalman coasts, report says so |
