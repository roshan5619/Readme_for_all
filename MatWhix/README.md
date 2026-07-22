# MatWhiz — Monocular Truck-Bed Vegetable Inventory

One camera looking down at a truck bed full of mixed vegetables. From that single
view the system works out: what vegetables are there, how many, how much they
weigh, how much volume they occupy (and what's left free), how the mass is
spread across the bed, and — the hard part — a physics-grounded estimate of
what's hiding *underneath* the visible top layer.

Built for a challenge that explicitly rewards mathematics over black boxes, so
the rule I held myself to everywhere in `src/`:

> **OpenCV is used strictly as an I/O and display layer** (reading images,
> writing video, drawing the dashboard overlay). Every algorithm that actually
> computes something — filtering, segmentation, homography, optical flow,
> triangulation, Kalman filtering, Bayesian inference — is implemented from
> first principles in NumPy in this repo. If you want to see the math, it's in
> the code, not behind an `import`.

The **one** exception, added deliberately and kept quarantined, is the
optional CNN vegetable classifier (`docs/07`): the contest rules allow a
pretrained model *as a component*, so a fine-tuned backbone names the
vegetable while every downstream number stays from-scratch math. It lives
behind a graceful-fallback boundary — no torch, no weights, no problem.

## Quickstart

```
pip install -r requirements.txt

# single image -> full inventory report (HTML + annotated overlays)
python demo.py --input path\to\photo.jpg --out runs\demo1

# a folder of images = a frame sequence (sorted by name) -> report + change log
python demo.py --input path\to\frames\ --out runs\demo2

# video works through the same door
python demo.py --input path\to\clip.mp4 --out runs\demo3
```

You do **not** have to tell it which vegetables are present — omit
`--classes` and it recognises them itself. Recognition is a **hybrid**: a
fine-tuned CNN (MobileNetV3-Small) names the vegetable when it's confident,
and the from-scratch prototype classifier covers anything outside the CNN's
training set. The CNN is optional — with no trained weights the system falls
back to the prototype classifier and runs exactly as before. To enable it:

```
pip install -r requirements-cnn.txt
# download the dataset + train (a few minutes on CPU) -- see docs/07
python train_classifier.py --data "data/veg_dataset/Vegetable Images" --out data/models/veg_cnn.pt
# from then on demo.py auto-detects the weights and uses the CNN
```

If you know the truck bed geometry, tell the system — it's the metric anchor
that turns pixels into meters:

```
python demo.py --input photo.jpg --bed-corners "120,80 1800,95 1750,1020 90,990" --bed-size 2.4x1.6x0.5
```

`--bed-corners` are the four bed corners in image pixels (clockwise from
top-left), `--bed-size` is length x width x wall-height in meters. Without
them the whole frame is treated as the bed with a default scale and the report
carries a fat warning that absolute sizes are uncalibrated.

## Layout

```
docs/          the approach document, literature survey, derivations, pseudocode
data/priors/   physical constants: densities, size distributions, packing fractions
src/veggievision/
  imops/          from-scratch image processing (conv, morphology, watershed, ...)
  geometry/       homography (normalized DLT + RANSAC), warping, epipolar geometry
  calibration/    camera intrinsics + the truck-bed metric reference
  stabilization/  frame quality gate, Lucas-Kanade tracking, canonical-view warp
  segmentation/   GMM background model, instance splitting
  classification/ hand-built features + prototype classifier
  depth/          parallax pseudo-stereo, apparent-size, shading, cue fusion
  volume/         height-map integration, ellipsoid fits, density mapping
  occlusion/      Bayesian latent-inventory model (the hidden layers)
  events/         frame differencing, event classification, change log
  tracking/       Kalman filters keeping estimates sane through bumps
  dashboard/      overlay rendering + HTML report
tests/         fixtures with analytically known answers
```

## Verifying it works

```
set PYTHONPATH=src;.        (PowerShell: $env:PYTHONPATH = "src;.")
python -m unittest discover tests
```

33 tests, all against analytically known ground truth: geometry estimators
must recover known transforms to numerical precision, the priors must
reproduce published bulk densities, and the full pipeline runs on a
rendered scene with known composition — counts, masses, the removal event,
and the blur-rejection path are all asserted. `runs/example_single_image/`
and `runs/example_sequence/` hold pre-generated reports from those same
fixtures.

Measured accuracy on the ground-truth fixture: 15/15 instances found,
14/15 classified correctly (the miss is the potato/onion pair — genuinely
confusable), pumpkin mass within 2% of its physical prior, zero false
removal events across jittered no-change frames, and the one real removal
detected, attributed, and logged. Performance on a laptop CPU (pure
single-threaded NumPy, by design): ~1-1.4 s light tier, ~2-3 s full
analytics — see `docs/06_gpu_notes.md` for the measured path to 30 FPS.

## Real market photos

`images/` holds 8 real photos (crates of tomatoes, chillies, okra, ivy
gourd, capsicum, bottle gourd, a potato pile, and a mixed display); their
processed inventories live in `runs/real_*/` — every one closes against
real-world physics (the full tomato crate comes out at 26.2 kg; real roma
crates weigh 25–28 kg). The per-photo scorecard with reality checks is at
the end of `docs/00_knowledge_guide.md`.

## Docs worth reading first

- `docs/00_knowledge_guide.md` — **start here**: every mechanism step by
  step, with worked examples using the real-photo numbers
- `docs/07_cnn_classifier.md` — the hybrid CNN recognition component: why,
  how it's trained, and how it hands off to the prototype classifier
- `docs/papers/` — local PDFs of the open-access research used + a link
  index for the paywalled classics, each annotated with what we took
- `docs/01_problem_analysis.md` — the framings considered and why this one won
- `docs/02_literature_survey.md` — 15 sources, what each contributed, where it landed in code
- `docs/03_approach.md` — the full mathematical derivation of every stage
- `docs/04_pseudocode.md` — per-module pseudocode (contest deliverable)
- `docs/05_innovation_justification.md` — why the packing-physics Bayesian
  model for occluded layers is the novel piece
- `docs/06_gpu_notes.md` — everything here runs on CPU; this note covers what
  a GPU would buy and where a neural depth cue would slot in
