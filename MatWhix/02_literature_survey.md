# 02 — Literature Survey

What was read, what was taken from it, and where it landed in the code.
Fifteen sources across the six pillars of the problem. Each entry ends with
**→ used in**, pointing at the module that embodies the finding.

---

## Pillar 1 — Monocular depth estimation

**[1] "Survey on Monocular Metric Depth Estimation" (arXiv:2501.11841, 2025).**
Maps the field's arc: relative-depth networks (MiDaS, DPT) → zero-shot
*metric* models (ZoeDepth's adaptive binning over MiDaS; Metric3D v2; Depth
Anything; UniDepth; Depth Pro). Key extractions: (a) metric monocular depth
from networks alone remains scale-brittle across cameras — models need
intrinsics awareness or in-domain anchoring; (b) the robust pattern in
applications is *relative depth from the network, scale from geometry*.
That pattern is exactly our optional-NN-cue design: any relative depth map
can be dropped into the fusion after scale-anchoring against the bed
homography, and the classical cues continue to police it.
**→ used in:** design of `depth/fusion.py`'s cue interface; docs/06 GPU path.

**[2] ZoeDepth (Bhat et al., 2023) and Metric3D v2 (Yin et al., 2024)** —
representative zero-shot metric models, noted as the concrete candidates
for the optional cue slot; both are far over our CPU budget at frame rate,
which is measured and stated rather than assumed.
**→ used in:** docs/06 recommendations.

## Pillar 2 — Counting under occlusion (agriculture)

**[3] "Deep Count: Fruit Counting Based on Deep Simulated Learning"
(Rahnemoonfar & Sheppard, Sensors 2017, PMC5426829).** Counts tomatoes at
91% accuracy under occlusion *without detecting individuals* — a
count-by-regression architecture trained on simulated data. Two lessons
taken: (a) counting does not require resolving every instance — regression
over aggregate evidence is legitimate and accurate; (b) simulation is a
valid training/validation substrate when real labels don't exist. Our
cluster counting is the classical analogue: area regression against a
per-frame self-calibrated item area, with the same philosophy.
**→ used in:** `classification/classifier.py::cluster_count`.

**[4] "Intelligent Fruit Yield Estimation" review (Front. Plant Sci. 2021).**
Surveys orchard counting; documents that occlusion is THE dominant error
source and that multi-view imaging is the standard mitigation. Confirms
that a single fixed view must model occlusion explicitly rather than hope —
supporting the Bayesian latent-layer design over a "count what you see"
approach.
**→ used in:** justification for `occlusion/` existing at all.

## Pillar 3 — Volume/weight of food from one image

**[5] "Single Image-Based Food Volume Estimation Using Monocular
Depth-Prediction Networks" (Springer 2020) and [6] MFP3D (arXiv:2411.10492,
2024).** Both validate the chain *depth map → integrate → volume* on food,
and both identify the same two error dominators: metric scale and the
unseen underside. Our answers: scale from the bed (stronger than their
plate-sized references), underside from the class ellipsoid prior. Their
reported single-item volume errors (10-20%) calibrate our per-instance
sigma of ~17-18%.
**→ used in:** `volume/ellipsoid.py` error budget; `volume/heightmap.py`.

## Pillar 4 — Pile volumetrics

**[7] DIVESPOT (arXiv:2407.05415, 2024).** Depth-camera pile volume via
ground calibration + voxel-column integration; ~2% error on corn piles,
robust to 90% voxel compression — but 86-433 s per cloud on edge hardware.
Three extractions: (a) height-map integration is the right volumetric
backbone (we use it verbatim, from monocular height instead of a depth
camera); (b) their 2% is the sensor-equipped ceiling our 5-15% honestly
trails; (c) their runtime is the cautionary tale that motivated tiered
scheduling from day one.
**→ used in:** `volume/heightmap.py`; `config.py::TierRates` rationale.

**[8] "Stockpile Volume Estimation in Open and Confined Environments: A
Review" (2023).** Industrial context: photogrammetry workflows reach 2-5%
volume accuracy with survey-grade rigs and GCP markers; accuracy collapses
to ~8% without ground control. Reinforces that our bed-corner calibration
IS our ground control, and that reporting volume without stating the
calibration story is meaningless.
**→ used in:** the uncalibrated-run warning in `dashboard/report.py`.

## Pillar 5 — The hidden layers (packing physics + occlusion reasoning)

**[9] Donev et al., "Improving the Density of Jammed Disordered Packings
Using Ellipsoids", Science 303:990 (2004).** The load-bearing citation.
Random close packing of spheres jams at φ≈0.64 (the classic Bernal/Scott
result); ellipsoids — i.e., vegetables — jam *denser*, φ≈0.68-0.71 at
M&M-like aspect ratios, approaching 0.74, because rotational degrees of
freedom fill pocket voids (measured ~10 contacts/particle vs 6 for
spheres). This converts an unmeasurable quantity (hidden count) into a
bounded one: hidden solid volume = bulk sub-surface volume × φ, with φ known
to a few percent per shape class. Without this the occlusion answer is
folklore; with it, it's physics.
**→ used in:** `data/priors/packing.json`; `occlusion/packing.py`.

**[10] Amodal instance segmentation line: Li & Malik (ECCV 2016,
arXiv:1604.08202) through "Amodal Ground Truth and Completion in the Wild"
(arXiv:2312.17247, 2024).** Deep models that predict full extents of
partially visible objects. Extractions: (a) the task decomposition —
visible mask, occlusion relation, completed mask — which our classical
analogue mirrors (watershed instance → stacking topology → ellipsoid
completion); (b) the honest observation that even SOTA amodal models only
complete *partially visible* objects; fully buried ones are beyond any
segmentation method and require exactly the kind of volumetric prior
reasoning we built.
**→ used in:** `depth/topology.py` design; scope boundary of `occlusion/`.

**[11] Bidisperse/mixture packing (e.g., "Maximum packing densities of
basic 3D objects", Sci. Bulletin 2010).** Size mixtures pack denser than
either component (small grains fill large-grain voids). We stay linear in
the mix (conservative) and note this as the refinement that would raise
hidden-mass estimates a few percent in pumpkin+onion loads.
**→ used in:** `occlusion/packing.py` comment + future-work note.

## Pillar 6 — Change detection, tracking, and the classical backbone

**[12] Stauffer & Grimson, "Adaptive background mixture models for
real-time tracking" (CVPR 1999); Zivkovic's adaptive-K variant (2004);
GMM-background review (Springer AIR 2018).** The per-pixel mixture idea —
appearance components earning "background" status through persistence, so
lighting changes get learned instead of alarmed on. Our sequence-mode
foreground model is a faithful vectorised implementation; the review's
catalogued failure modes (sudden global illumination, camouflage) informed
the differ's local-normalisation and the color-not-gray decision.
**→ used in:** `segmentation/background.py::GMMBackground`.

**[13] Ground-plane scale recovery for monocular VO (arXiv:1903.00912 and
the plane-geometry follow-up 2101.05995).** Absolute scale from a known
plane + camera height — the same trick our bed anchor performs for
triangulated parallax points; their robustified plane-point selection
inspired the floor-vote median in `depth/parallax.py`.
**→ used in:** `depth/parallax.py` metric anchoring.

**[14] The classical canon, cited at point of use in code:** Zhang's
calibration (TPAMI 2000) — docs/03 §2; Hartley's normalised 8-point (TPAMI
1997) and Hartley & Zisserman's *Multiple View Geometry* — homography/
epipolar modules; Lucas-Kanade (IJCAI 1981) + Shi-Tomasi (CVPR 1994) —
tracker; Otsu (1979) — thresholding; Vincent & Soille (TPAMI 1991) —
watershed; Borgefors (1986) — chamfer weights; Hu (1962) moments; Dalal &
Triggs HOG (CVPR 2005); Kalman (1960). These aren't decorative citations —
each names the exact algorithm implemented by hand in `imops/`, `geometry/`,
`stabilization/`, `tracking/`.

**[15] Bulk density references: FAO/INFOODS Density Database v2 (2012);
Engineering Toolbox food bulk densities; Int. J. Chem. Studies 6(5):2018
potato study (bulk 704-738 kg/m³ across size grades).** Populated
`data/priors/densities.json`; the potato study's bulk figure cross-checks
our true-density × φ derivation (1085 × 0.66 ≈ 716 — inside their band),
which is the kind of two-source consistency check every prior in the DB
should eventually get.
**→ used in:** `data/priors/densities.json`, `packing.json` validation.

---

## What the survey changed about the design (summary)

1. Height-map integration adopted as the volumetric backbone [7], with
   monocular error budget calibrated against food-domain results [5][6].
2. Packing physics adopted as the occlusion keystone [9], linear-mix
   conservative variant [11].
3. Count-by-regression legitimised for unresolvable clusters [3].
4. Scale-from-known-geometry preferred over network metric depth [1][13];
   NN depth demoted to optional fused cue.
5. Per-pixel GMM chosen for sequence background modelling, with its known
   failure modes designed around [12].
6. Tiered scheduling motivated by the runtime gap between pile-volumetrics
   research code and frame-rate requirements [7].
