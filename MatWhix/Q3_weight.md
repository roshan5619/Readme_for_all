# Q3 — Estimating weight via volume × density inference

## (A) The approach, in plain language

Weight is never measured — there is no scale. It is **inferred**:

```
mass  =  volume  ×  density
```

For each visible item we estimate its **volume** by fitting a 3D ellipsoid
to it, and multiply by the **true flesh density** of that vegetable class
(a physical constant we look up). For the produce hidden below the surface,
we estimate a bulk volume and convert it with packing physics (that chain is
Q4). Both live in `volume/ellipsoid.py` with densities in
`data/priors/densities.json`.

The elegant part is that we never need a template or a 3D scan: the ellipse
we can *see* in the top-down view already encodes two of the three axes, and
the third (the height we can't see) comes from the class's known proportions.

## (B) The mathematics

### Volume from image moments — no iterative fitting
Vegetables are irregular, so we fit an ellipsoid rather than assume a sphere.
The fit is **closed-form from second moments**, which is both fast and robust
to a ragged mask boundary. For a filled ellipse, the central second moments
in its principal frame are μ₂₀ = A²/4 and μ₀₂ = B²/4. So if we take the
covariance matrix of the item's pixel coordinates and find its eigenvalues
λ₁ ≥ λ₂, the two visible semi-axes are

```
A = 2√λ₁ ,   B = 2√λ₂      (in pixels → divide by px_per_m for metres)
```

This is robust because moments integrate over the *whole* region — a few
noisy edge pixels shift them by only O(1/area), unlike contour fitting which
chases every wobble.

The **third semi-axis C** (vertical, unseen) comes from the class prior,
scaled isotropically by how big *this* item is relative to its class mean:

```
C = C_prior × √( (A·B) / (A_prior·B_prior) )     (clamped to [0.5, 2]×)
```

— the same "a 1.2× wider tomato is 1.2× taller" assumption that underlies
lognormal size priors. Then

```
V = (4/3) π · A · B · C ,      mass = V × ρ_true
```

### Why *true* density × packing fraction, not "bulk density"
A pile's *bulk* density (mass ÷ pile volume, air gaps included) is a
different, coarser quantity. We deliberately store the **true flesh density**
ρ_true (e.g. potato ≈ 1085 kg/m³) and, separately, a **packing fraction** φ
(fraction of a pile that is actually vegetable). This keeps two independent
error sources separate and auditable: a per-item volume error and a
packing-geometry error. For a *single* visible item, φ doesn't enter at all
(we measured its own ellipsoid); φ only appears for the *hidden bulk* (Q4).
As a sanity cross-check, ρ_true × φ reproduces published bulk densities —
1085 × 0.66 ≈ 716 kg/m³ for potatoes, inside the measured 704–738 band.

### Error propagation → the reported ± kg
The mass of one item is a product of independent factors, so **relative
variances add**:

```
(σ_m / m)²  =  (σ_C/C)²  +  (σ_fit)²  +  (σ_ρ/ρ)²
            ≈  0.15²     +  0.08²     +  (density spread)²
```

giving ~18–20% per item; aggregated over many items the independent parts
partially average down. Every mass in the report is printed with this ±, and
`data/priors/densities.json` carries a per-class density spread that flows
straight into σ_ρ.

## (C) They might also ask…

**"Why an ellipsoid and not a superquadric or a real 3D reconstruction?"**
Superquadrics (the EMS/CVPR-2022 line) generalise the ellipsoid with two
extra shape exponents and would fit a slightly irregular potato better — but
against our *other* error sources (the unseen third axis, the density
spread) that refinement buys almost nothing, while costing an iterative
optimisation per item. The ellipsoid gives a closed-form, moment-based fit
that is fast and robust; we cite superquadrics as the natural upgrade if
per-item volume ever becomes the dominant error term.

**"Where do the density numbers come from — did you make them up?"**
No — they are from the **FAO/INFOODS density database** and the Engineering
Toolbox food tables, with a potato-specific cross-check against a published
physical-properties study (bulk 704–738 kg/m³). Each class carries a spread
(σ) reflecting ripeness/variety variation, and that spread is propagated
into the reported mass uncertainty rather than hidden.

**"You can only see the top of the item — how do you know its height?"**
We don't measure it; we infer it from the class's typical proportions,
scaled by the item's own visible size, and we *charge for the uncertainty*
(the σ_C/C ≈ 15% term above). This is the honest monocular limit: the unseen
dimension is a prior, and its error is in the ± we report.

**"Which is more trustworthy, your counts or your weights?"**
Weights. This is counter-intuitive but it's arithmetic: count = volume ÷
per-item-volume, so a 20% size-prior error becomes a large count error for
tiny items (a chilli is ~3 ml — dividing by it amplifies error). Mass is
volume × density × packing, where the errors partially cancel and never get
divided by a tiny number. The chilli crate shows it: mass ≈ 11.5 kg (solid,
matching a real 10–12 kg crate) while the *count* interval is huge. We
report both with intervals so a user trusts each appropriately.

**"How would a real scale change this?"**
It wouldn't be allowed (single-camera constraint), but it would collapse the
density and packing uncertainty to zero and turn our inference into a
measurement. Everything upstream (segmentation, volume) would still be
needed to attribute the total mass to classes.
