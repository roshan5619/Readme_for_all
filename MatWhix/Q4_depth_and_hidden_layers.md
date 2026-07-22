# Q4 — Inferring depth and hidden layers from a 2D image alone

This is the hardest question and the one the evaluators will push on. Answer
it in two halves: **surface depth** (the height of what we can see) and
**hidden layers** (what lies beneath). The second half is the project's
novel contribution.

## (A) The approach, in plain language

**Start by naming the wall.** A single image has no absolute depth — the same
picture is consistent with a small scene up close or a large scene far away.
So we do two honest things: recover depth *relative* structure from monocular
cues, and fix the *absolute scale* from the known bed. We never pretend the
camera measured depth directly.

**Surface depth = fusion of four independent monocular cues** (`depth/`),
each weak alone, each with an honest error bar:

| Cue | Idea | Strong where | Blind where |
|---|---|---|---|
| Apparent size | items above the bed render **magnified** | high piles | near the floor |
| Stacking topology | produce doesn't levitate → layers from a contact graph | low layers | fully packed beds |
| Temporal parallax | two frames of a moving truck = a stereo pair | truck moving | parked |
| Shape-from-shading | brightness across one item ≈ its surface slope | dome *shape* | absolute scale |

We combine them by **inverse-variance fusion** (`depth/fusion.py`): each cue
reports (estimate, σ), and the fused value weights each by 1/σ² — so a cue
that is honest about being useless in a situation automatically gets almost
no vote there. The fused per-item heights are rasterised into a metric
**height map** h(x,y).

**Hidden layers = a Bayesian latent-inventory model** (`occlusion/bayes.py`,
`occlusion/packing.py`). No sensor can see buried produce, so we *infer* it,
but not by guessing — by combining three sources of real information:
(1) the sub-surface **bulk volume** from the height map, (2) **granular
packing physics** that bounds how much solid fits in that volume, and (3) the
**loading prior** (heavy items sink) plus **every removal event**, which
exposes a lower layer and is therefore a literal sample of the hidden mix.
The result is a *posterior with credible intervals*, and its uncertainty
**shrinks over the working day** as sales reveal more of the load.

## (B) The mathematics

### Cue 1 — apparent-size depth
The canonical view is calibrated to the bed floor. An item whose centre
floats at height z is closer to the camera than the calibration assumes, so
it is magnified by m = h_cam / (h_cam − z), where m = observed diameter ÷
expected diameter. Invert:

```
z = h_cam · (1 − 1/m) ,        σ_z = h_cam · σ_rel / m²
```

The σ_z formula is the important part: at floor level (m ≈ 1) with a 15%
size prior at h_cam = 2.5 m, σ_z ≈ ±0.37 m per item — **almost useless
alone**, and the cue *says so*, which is why fusion down-weights it there.
Averaged over a layer of N items it tightens as 1/√N. (`depth/apparent_size.py`)

### Cue 2 — stacking topology
Build the item **contact graph**: an item touching visible floor is layer 0;
breadth-first search assigns every other item `1 + min(neighbour layers)`.
A layer index converts to height using the **hexagonal-close-packing
inter-layer rise**: a sphere layer settles into the pockets of the one below
it, rising by √(2/3) ≈ 0.816 of its diameter, so
`z ≈ layer × 0.816 × item_extent`. Precise exactly where apparent-size is
blind (distinguishing layer 0 from layer 1 — a few cm the magnification
can't see). (`depth/topology.py`)

### Cue 3 — temporal parallax (pseudo-stereo)
When the truck moves, two frames are a stereo pair with an unknown baseline.
Estimate the **fundamental matrix** F from ≥8 tracked points (normalised
8-point algorithm, rank-2 enforced); form the **essential matrix** E = KᵀFK
(snap singular values to (1,1,0)); decompose to relative rotation and
translation (R, t) with the cheirality test (points must lie in front of both
cameras); **triangulate** the matches to 3D — up to one global scale. That
scale is the piece parallax can't supply, and the **bed floor supplies it**:
visible-floor points must sit at the known camera height, and their median
fixes the scale. (`geometry/epipolar.py`, `depth/parallax.py`)

### Cue 4 — shape-from-shading (shape only, never scale)
Within a single item (one albedo, one light) the normalised smoothed
intensity is a serviceable proxy for the *relative* surface height — it
shapes each item's dome in the height map but never sets its absolute peak.
Full SfS needs known lighting/albedo (lab conditions); we deliberately use
only the weak, robust version. (`depth/shading.py`)

### Fusion
Independent Gaussian estimates combine by inverse-variance (the
minimum-variance unbiased estimator, and algebraically a one-shot Kalman
update):

```
ẑ = ( Σ zᵢ/σᵢ² ) / ( Σ 1/σᵢ² ) ,      1/σ̂² = Σ 1/σᵢ²
```

No cue is ever switched off by an `if`; bad cues lose the argument
arithmetically. (`depth/fusion.py`)

### The hidden layers — the Bayesian latent-inventory model
This is the novel piece. Let V_sub be the sub-surface bulk volume (occupied
volume from the height map minus the visible items' claimed volume). The
hidden region is a **mixture** of classes with unknown volume-fractions
**w** = (w₁…w_C). Given w, physics fixes everything:

```
hidden solid volume of class c :  V_c = V_sub · w_c · φ_c
hidden count of class c        :  N_c = V_c / E[v_c]
hidden mass of class c         :  M_c = V_c · ρ_c
```

so the *only* genuine unknown is **w**, which gets a **Dirichlet** posterior
(the conjugate distribution for proportions):

```
prior:     w ~ Dir(α₀ · w₀)        w₀ = stratification prior (heavy sinks, light floats)
evidence:  each reveal of k items of class c →  α_c ← α_c + k     (conjugate: one addition)
posterior mean:  E[w_c] = α_c / Σα ,   Var[w_c] = w_c(1−w_c)/(Σα + 1)
```

φ (the packing fraction) is anchored on **Donev et al., *Science* 303:990
(2004)**: random close packing of spheres jams at φ ≈ 0.64, and ellipsoids —
i.e. vegetables — pack *denser*, φ ≈ 0.68–0.71, because rotational freedom
fills the void pockets. The hidden count's credible interval combines, in
quadrature, the Dirichlet variance of w, the packing σ_φ, the sub-surface
volume σ, and the item-size prior — so the report prints N_c with a 90%
interval, never a bare number (`occlusion/bayes.py`, lines implementing the
`rel` propagation and the `count_lo/count_hi` = N ± 1.645·σ bounds).

**The operational punchline:** because every sale reveals a lower layer and
adds pseudo-counts, the posterior *strength* Σα grows through the day and the
intervals **narrow** — the system gets *more* certain about the hidden load
the more the vendor sells. No published occlusion method has this property.

## (C) They might also ask…

**"You can't get depth from one image — how is this not fake?"**
Correct that absolute depth is unrecoverable from one image *without a
reference* — which is exactly why we (a) anchor scale on the known bed, and
(b) recover *relative* structure from multiple monocular cues, each reported
with an honest σ. We never claim a measured depth; we claim a *fused
estimate with error bars*, and the bar widens honestly when the cues are
weak (e.g. a single still image with no parallax).

**"Isn't inferring hidden produce just guessing with extra steps?"**
The difference between a guess and a posterior is (1) it is bounded by a
measured physical constant (packing fraction φ, Donev 2004), (2) it is
conditioned on real evidence (the loading prior *and* every reveal event),
and (3) it comes with a credible interval that quantifies the residual
uncertainty. A guess gives a number; this gives a distribution whose
variance we can state and watch shrink.

**"Justify the packing fraction — why 0.64–0.71?"**
0.64 is the classical Bernal random-close-packing density of equal spheres.
Donev et al. showed experimentally and in simulation that ellipsoids jam
denser (up to ~0.74) because non-spherical particles have rotational degrees
of freedom that let them fill each other's gaps — measured ~10 contacts per
particle vs 6 for spheres. Near-spherical produce gets 0.64; elongated
produce gets the ellipsoid bump; thin rods (chillies) go the *other* way
(they tangle like pick-up sticks, ~0.5). Every value carries a σ.

**"What if no floor is visible at all — a completely packed bed?"**
Then the stacking cue is *unanchored* (it can order the pile but not locate
its base), and "no floor visible" is itself evidence that the bed is full.
We apply a **rim-fill prior**: the load reaches roughly the surface we see,
bounded by the wall height, with a wide (~30% of wall height) σ so the
volume and hidden-layer outputs wear the assumption honestly. On the real
tomato crate this took the estimate from an absurd 5 L "occupied" to a
sensible 41 L of a 54 L crate (76% full).

**"How do you separate the visible layer's volume from the hidden bulk?"**
The height map integral gives total occupied bulk. Each visible item claims
its own solid volume divided by its class packing fraction (its share of the
pile). Subtract that claimed bulk from the total → the sub-surface bulk
V_sub that feeds the Bayesian model. It's bookkeeping, but it keeps the
visible measurement and the hidden inference from double-counting.
