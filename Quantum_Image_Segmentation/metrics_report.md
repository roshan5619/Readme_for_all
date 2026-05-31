# Quantum Image Segmentation -- Mandatory Metrics vs SOTA

## 1. NEQR encoding fidelity + quantum resources (spatial-preservation gate)

_Lossless representation: exact reconstruction => PSNR=inf, SSIM=1, MSE=0 (every boundary/texture pixel preserved). Resource columns substantiate the 'quantum' claim (per Ruan 2021)._

| image         | size  | qubits | gates | depth | PSNR | SSIM   | MSE    | EPI    | sim_time |
|---------------|-------|--------|-------|-------|------|--------|--------|--------|----------|
| 5.1.12.tiff   | 8x8   | 14     | 302   | 297   | inf  | 1.0000 | 0.0000 | 1.0000 | 0.14s    |
| 5.1.12.tiff   | 16x16 | 16     | 1246  | 1239  | inf  | 1.0000 | 0.0000 | 1.0000 | 0.14s    |
| 5.1.12.tiff   | 32x32 | 18     | 4991  | 4982  | inf  | 1.0000 | 0.0000 | 1.0000 | 0.46s    |
| 4.2.03.tiff   | 8x8   | 14     | 306   | 301   | inf  | 1.0000 | 0.0000 | 1.0000 | 0.05s    |
| 4.2.03.tiff   | 16x16 | 16     | 1106  | 1099  | inf  | 1.0000 | 0.0000 | 1.0000 | 0.19s    |
| 4.2.03.tiff   | 32x32 | 18     | 4339  | 4330  | inf  | 1.0000 | 0.0000 | 1.0000 | 0.50s    |
| boat.512.tiff | 8x8   | 14     | 337   | 332   | inf  | 1.0000 | 0.0000 | 1.0000 | 0.05s    |
| boat.512.tiff | 16x16 | 16     | 1286  | 1279  | inf  | 1.0000 | 0.0000 | 1.0000 | 0.15s    |
| boat.512.tiff | 32x32 | 18     | 4759  | 4750  | inf  | 1.0000 | 0.0000 | 1.0000 | 0.53s    |

## 2. QHED vs classical baselines on misc (no ground truth)

_misc has no masks -> tolerance-aware agreement with classical edges + edge density. Supervised SOTA numbers are in stage 3 (BSDS)._

| image         | size    | comparison        | QHED_edge_density | baseline_edge_density | tol1_agreement |
|---------------|---------|-------------------|-------------------|-----------------------|----------------|
| 5.1.12.tiff   | 128x128 | QHED-vs-sobel     | 0.1500            | 0.0559                | 0.9577         |
| 5.1.12.tiff   | 128x128 | QHED-vs-scharr    | 0.1500            | 0.0560                | 0.9590         |
| 5.1.12.tiff   | 128x128 | QHED-vs-laplacian | 0.1500            | 0.0084                | 0.9596         |
| 5.1.12.tiff   | 128x128 | QHED-vs-canny     | 0.1500            | 0.0975                | 0.4258         |
| boat.512.tiff | 128x128 | QHED-vs-sobel     | 0.1501            | 0.0113                | 0.9295         |
| boat.512.tiff | 128x128 | QHED-vs-scharr    | 0.1501            | 0.0114                | 0.9328         |
| boat.512.tiff | 128x128 | QHED-vs-laplacian | 0.1501            | 0.0041                | 0.8758         |
| boat.512.tiff | 128x128 | QHED-vs-canny     | 0.1501            | 0.1788                | 0.8204         |
| 4.2.03.tiff   | 128x128 | QHED-vs-sobel     | 0.1501            | 0.0287                | 0.8907         |
| 4.2.03.tiff   | 128x128 | QHED-vs-scharr    | 0.1501            | 0.0293                | 0.9032         |
| 4.2.03.tiff   | 128x128 | QHED-vs-laplacian | 0.1501            | 0.0123                | 0.9010         |
| 4.2.03.tiff   | 128x128 | QHED-vs-canny     | 0.1501            | 0.0842                | 0.5366         |
| house.tiff    | 128x128 | QHED-vs-sobel     | 0.1501            | 0.0784                | 0.9553         |
| house.tiff    | 128x128 | QHED-vs-scharr    | 0.1501            | 0.0763                | 0.9547         |
| house.tiff    | 128x128 | QHED-vs-laplacian | 0.1501            | 0.0078                | 0.9009         |
| house.tiff    | 128x128 | QHED-vs-canny     | 0.1501            | 0.1351                | 0.4845         |

## 3. BSDS500 supervised boundary SOTA (ODS / OIS / Boundary-F1 / Pratt / Hausdorff)

_Run with `--bsds` to download BSDS500 and compute supervised SOTA metrics._

## 4. QAOA Fg/Bg Segmentation on GrabCut-50

_Run with `--grabcut` to download GrabCut-50 and evaluate QAOA fg/bg segmentation._

## 5. QAOA Fg/Bg on Misc Images (auto-trimap, no human GT)

Evaluated on 2 misc images at 128×128, n_segments=12.

| method  | image       | IoU*   | Dice*  | PixelAcc* | n_qubits | sim_time |
|---------|-------------|--------|--------|-----------|----------|----------|
| QAOA    | house.tiff  | 0.0558 | 0.1057 | 0.5080    | 11       | 13.0s    |
| GrabCut | house.tiff  | 0.1008 | 0.1831 | 0.6944    | -        | -        |
| QAOA    | 4.2.03.tiff | 0.0793 | 0.1469 | 0.7965    | 11       | 8.1s     |
| GrabCut | 4.2.03.tiff | 0.1744 | 0.2970 | 0.8636    | -        | -        |
_\* metrics are vs the auto-generated center-ellipse trimap, NOT human GT. Use as a relative comparison between QAOA and GrabCut only._

Figures saved to `results/figures/misc_fgbg_*.png`

