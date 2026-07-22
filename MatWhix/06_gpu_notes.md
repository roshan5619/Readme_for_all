# 06 — CPU performance, measured; GPU needs, documented

The whole pipeline runs CPU-only by requirement. This note records the
measured numbers, where the time actually goes, the knobs that trade
resolution for speed, and precisely what a GPU (or even just permitted
optimised libraries) would buy.

## Measured (this laptop, single-threaded pure NumPy, 640×960 canonical)

| Path | Wall clock |
|---|---|
| Light tier (gate + stabilize + envelope diff) | ~1.0–1.4 s/frame |
| Full stack (+ segment, classify, depth, volumes, Bayes) | ~2–3 s/frame |
| Full stack at 200–250 px/m canonical | roughly 2–3× faster |

Numbers vary ±40% with laptop thermals; treat them as order-of-magnitude.
Where the time goes (profiled): bilinear warps and LK window gathers
(`geometry/warp.py::bilinear_sample`), the per-channel envelope diff, HSV
conversion, and the per-blob watershed. All are linear in pixel count —
there is no super-linear stage anywhere in the pipeline.

Optimisations already banked during development (documented in code where
they live): per-blob cropped watershed/chamfer (5.7× full-stack win),
integral-image box blur (O(1) per pixel at any radius), batched LK across
all points (6×), single warp per frame (gray derived from warped RGB —
linear ops commute).

## Why we do NOT claim 30 FPS today, and why the architecture still holds

The 30 FPS requirement is met at the *architecture* level: the tier
scheduler runs cheap watchers per frame and expensive analytics at ~1 Hz or
on events, so the computational shape of the system matches the information
rate of the scene. The remaining gap is implementation substrate: pure
NumPy in one Python thread is roughly 10–50× off optimised native code for
exactly these workloads — a well-characterised, boring gap, not a research
risk. The rules permitted pre-built libraries as components; we chose to
hand-build everything to demonstrate the math, and we prefer reporting that
honestly over shipping an OpenCV pipeline and calling it from-scratch.

## The costed path to real time

1. **Free (config):** canonical at 250 px/m (4 mm/px — still fine for
   50 mm+ produce): ~2.5×. Active-class narrowing shrinks segment work.
2. **Cheap (still CPU):** swap the two proven-hot inner loops — warp
   sampling and LK gathers — for their `cv2.remap` / `cv2.calcOpticalFlowPyrLK`
   equivalents once the from-scratch versions have served their
   demonstrative purpose (the math is already validated against them).
   Expected 10–20× on the light tier → 30 FPS territory on a laptop CPU.
   Multi-threading tiers (stabilize on core 0, analytics on core 1) is
   another clean 2× because the tiers are already decoupled by design.
3. **GPU (what it actually buys):**
   - Warps, diffs, morphology, HSV: memory-bound, embarrassingly parallel
     — 50–100× on any integrated GPU. Overkill but available.
   - **The real GPU customer is the optional neural depth cue.** A
     relative-depth network (Depth Anything V2-small class) at 10–30 FPS
     needs ~2–4 GB VRAM on an entry discrete GPU (RTX 3050-class) or a
     Jetson Orin Nano at the edge. Scale-anchored by our bed homography
     (the survey's [1] recommended pattern) it would slot into the fusion
     as a fifth cue with per-pixel coverage — the single biggest accuracy
     upgrade available to this system, at zero architectural change.
   - Training anything (a produce segmenter fine-tune, the count
     regressor of survey [3]): single 8–16 GB GPU, hours not days, once a
     labelled dataset exists.

## Recommended deployment ladder

| Stage | Hardware | What runs |
|---|---|---|
| Demo / audit | any laptop, CPU | exactly this repo, as-is |
| Pilot on a truck | laptop CPU + step 2 above | 30 FPS light tier, 1 Hz analytics |
| Product | + entry GPU or Jetson | + neural depth 5th cue, densest accuracy |
