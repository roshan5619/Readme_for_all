# Quantum Fraud Risk Implementation Guide

## Table of Contents
1. [Project Structure](#1-project-structure)
2. [Environment Setup](#2-environment-setup)
3. [Running the Pipeline](#3-running-the-pipeline)
4. [Understanding the Code](#4-understanding-the-code)
5. [Configuration Options](#5-configuration-options)
6. [Output Files](#6-output-files)
7. [Current Results](#7-current-results)
8. [Troubleshooting](#8-troubleshooting)
9. [Extending the Model](#9-extending-the-model)
10. [Quick Reference](#10-quick-reference)

---

## 1. Project Structure

```
Quantum_Fraud_Risk/
├── quantum_fraud_model.py              # Main training & evaluation pipeline
├── 13981_rec_with_perf_sep_16_Fraud.csv # Dataset (7,674 records × 17 columns)
├── Custom_details_fr.pdf               # Benchmark reference (§7.2 targets)
├── docs/
│   ├── IMPLEMENTATION_GUIDE.md         # This file
│   └── KNOWLEDGE_GUIDE.md             # Theory & math
└── outputs/                            # Generated after training
    ├── final_summary.csv               # Consolidated metrics vs. benchmarks
    ├── model_performance.csv           # AUC / Gini / KS per split
    ├── iv_results.csv                  # Information Value (14 features × 3 splits)
    ├── vif_results.csv                 # Variance Inflation Factors
    ├── csi_stability.csv               # Characteristic Stability Index
    ├── decile_analysis_train.csv       # Decile table (Train)
    ├── decile_analysis_val.csv         # Decile table (VAL)
    ├── decile_analysis_test.csv        # Decile table (Test)
    ├── correlation_heatmap.png         # 14×14 feature correlation matrix
    ├── vif_analysis.png                # VIF bar chart with thresholds
    ├── vqc_training_curves.png         # Loss & validation AUC over epochs
    ├── calibration_curve.png           # Raw vs. isotonic-calibrated (Test)
    ├── roc_curve_comparison.png        # ROC curves (Train / VAL / Test)
    ├── confusion_matrix_train.png      # Confusion matrix (Train)
    ├── confusion_matrix_val.png        # Confusion matrix (VAL)
    ├── confusion_matrix_test.png       # Confusion matrix (Test)
    ├── rank_order_chart.png            # % Risky by decile (line, all splits)
    └── decile_bad_rate_all.png         # % Risky by decile (bar, all splits)
```

---

## 2. Environment Setup

### Step 1: Navigate to Project

```powershell
cd "C:\Users\rosha\Documents\HotFoot_Technologies\Quantum_Fraud_Risk"
```

### Step 2: Create Virtual Environment

```powershell
python -m venv .venv
```

### Step 3: Activate Environment

**Windows PowerShell:**
```powershell
.\.venv\Scripts\Activate.ps1
```

**Windows CMD:**
```cmd
.\.venv\Scripts\activate.bat
```

**Linux/Mac:**
```bash
source .venv/bin/activate
```

### Step 4: Install Dependencies

```powershell
pip install --upgrade pip
pip install numpy pandas matplotlib seaborn scikit-learn statsmodels
pip install pennylane pennylane-lightning torch
pip install xgboost imbalanced-learn
pip install dwave-neal dimod
```

### Step 5: Verify Installation

```powershell
python -c "import pennylane, torch, xgboost, imblearn, neal, dimod; print('All OK')"
```

Expected output:
```
All OK
```

### Package Versions (tested)

| Package | Minimum Version |
|---------|----------------|
| pennylane | 0.37+ |
| pennylane-lightning | 0.37+ |
| torch | 2.0+ |
| xgboost | 2.0+ |
| imbalanced-learn | 0.12+ |
| dwave-neal | 0.6+ |
| dimod | 0.12+ |
| scikit-learn | 1.3+ |
| statsmodels | 0.14+ |

---

## 3. Running the Pipeline

### Training

```powershell
python quantum_fraud_model.py
```

**Expected runtime:** 30–90 minutes (VQC dominates; QKSVM is skipped)

| Step | Duration | What Happens |
|------|----------|--------------|
| Steps 1–6 | ~10 sec | Data load, correlation, VIF, WoE, scaling, QUBO/SA |
| Step 7 (VQC) | 30–90 min | 100 epochs, data-reuploading VQC training |
| Step 8 (QKSVM) | Skipped | Was taking hours; 2-arm ensemble used instead |
| Steps 9–16 | ~2 min | XGBoost, ensemble, calibration, deciles, charts |

### Console Output (expected)

```
======================================================================
STEP 1: Data Loading & Splits
======================================================================
Dataset shape: (7674, 17)
Fraud rate (overall): 0.1064
Train: 4,331  |  VAL: 1,083  |  Test: 2,260
Fraud rate — Train: 0.1062  VAL: 0.1063  Test: 0.1071

STEP 2: Correlation Heatmap
  Saved: correlation_heatmap.png

STEP 3: VIF Calculation
  ...
  Saved: vif_results.csv, vif_analysis.png

STEP 4: WoE Encoding + IV
  ...
  Saved: iv_results.csv

STEP 5: Scaling for Quantum Encoding
  Feature matrix shape: (4331, 14)
  Angular range: [0.000, 3.142]

STEP 6: QUBO Feature Selection (Simulated Annealing)
  Selected N features: [...]
  N_QUBITS = N

STEP 7: Arm A — Data-Reuploading VQC (PennyLane)
  Applying ADASYN oversampling for VQC training...
  Training VQC (NQ × 3L, 100 epochs)...
    Epoch  10/100  loss=0.XXXX  val_AUC=0.XXXX  (best=0.XXXX)
    Epoch  20/100  loss=0.XXXX  val_AUC=0.XXXX  (best=0.XXXX)
    ...
    Epoch 100/100  loss=0.XXXX  val_AUC=0.XXXX  (best=0.XXXX)
  Best VQC val AUC: 0.XXXX

STEP 8: Arm B — QKSVM SKIPPED (running 2-arm ensemble: VQC + XGBoost)

STEP 9: Arm C — XGBoost + ADASYN
  ...

STEP 10: Weighted Ensemble
  ...

STEP 11: Isotonic Regression Calibration
  ...

STEP 12: Performance Metrics
  ...

STEP 13: Decile Analysis
  ...

STEP 14: Rank Order Charts
  ...

STEP 15: CSI (Variable Stability)
  ...

======================================================================
  FINAL RESULTS SUMMARY
======================================================================
  ...

Done! All outputs saved to: outputs/
```

---

## 4. Understanding the Code

### 4.1 Step 1: Data Loading & Splits (Lines 63–98)

```python
df = pd.read_csv(DATA_FILE)

# Separate existing train / test splits
df_train_full = df[df["Split"] == "train"].reset_index(drop=True)
df_test       = df[df["Split"] == "test"].reset_index(drop=True)

# Create VAL from train (stratified 80/20)
df_train, df_val = train_test_split(
    df_train_full, test_size=0.20,
    random_state=42, stratify=df_train_full["bad_f"],
)
```

**What it does:**
- Loads CSV with 7,674 records (14 features + `bad_f` target + `Split` column)
- Uses pre-defined train/test split from CSV
- Creates VAL by carving 20% from train (stratified on fraud label)
- Final: Train (4,331) | VAL (1,083) | Test (2,260)

### 4.2 Step 2: Correlation Heatmap (Lines 100–114)

Computes Pearson correlation on raw features and saves a 14×14 heatmap. Used to identify redundant feature pairs (|r| > 0.6).

### 4.3 Step 3: VIF Calculation (Lines 116–143)

```python
_X_scaled = StandardScaler().fit_transform(X_train_raw)
vif_data = pd.DataFrame({
    "Feature": FEATURE_COLS,
    "VIF": [variance_inflation_factor(_X_scaled, i) for i in range(len(FEATURE_COLS))]
})
```

**What it does:**
- Standardizes features, then computes VIF per feature
- VIF > 5 = multicollinearity concern; all features here are < 2.6

### 4.4 Step 4: WoE Encoding + IV (Lines 145–227)

```python
def compute_woe_iv(X, y, feature_names, n_bins=10, epsilon=1e-6):
    # Quantile-based binning → WoE per bin → IV per feature
    ...

woe_maps, iv_train = compute_woe_iv(X_train_raw, y_train, FEATURE_COLS, n_bins=10)
X_train_woe = apply_woe(X_train_raw, FEATURE_COLS, woe_maps)
```

**What it does:**
1. Bins each feature into 10 quantile-based bins (on training data)
2. Computes WoE = ln(P(X|bad) / P(X|good)) per bin
3. Computes IV = sum of (dist_bad - dist_good) × WoE per feature
4. Applies WoE transformation to all splits (using train-fit bins)

### 4.5 Step 5: Scaling Pipeline (Lines 229–251)

```python
robust_scaler = RobustScaler()
X_train_r = robust_scaler.fit_transform(X_train_woe)

minmax_scaler = MinMaxScaler()
X_train_mm = minmax_scaler.fit_transform(X_train_r)

# Angular encoding [0, π]
X_train_ang = X_train_mm * np.pi
```

**What it does:**
1. **RobustScaler** — centers on median, scales by IQR (outlier-resistant)
2. **MinMaxScaler** — scales to [0, 1]
3. **Angular** — multiplies by π → [0, π] for quantum gate rotations

### 4.6 Step 6: QUBO Feature Selection (Lines 253–310)

```python
def build_qubo(feature_names, iv_scores, corr_matrix, lambda1=0.15, lambda2=0.08):
    # Q[i,i] = λ1 - iv_norm[i]       (sparsity vs. relevance)
    # Q[i,j] = λ2 * |corr[i,j]|      (redundancy penalty)
    ...

bqm = dimod.BinaryQuadraticModel.from_qubo(Q)
sampler = neal.SimulatedAnnealingSampler()
response = sampler.sample(bqm, num_reads=1000, num_sweeps=2000, seed=42)
```

**What it does:**
- Formulates a QUBO (Quadratic Unconstrained Binary Optimization) problem
- Diagonal: sparsity penalty minus IV reward → prefer high-IV features
- Off-diagonal: correlation penalty → avoid selecting redundant features
- Solved via Simulated Annealing (1000 reads, 2000 sweeps)
- Falls back to top-8 by IV if fewer than 6 features selected

### 4.7 Step 7: Arm A — Data-Reuploading VQC (Lines 312–481)

```python
@qml.qnode(dev, interface="torch", diff_method="adjoint")
def vqc_circuit(inputs, weights):
    for layer in range(n_layers):
        qml.AngleEmbedding(inputs, wires=range(n_qubits), rotation="Y")
        qml.StronglyEntanglingLayers(
            weights[layer].unsqueeze(0), wires=range(n_qubits)
        )
    return [qml.expval(qml.PauliZ(i)) for i in range(n_qubits)]
```

**What it does:**
- **Data reuploading**: features are re-encoded at every layer (not just once)
- **3 StronglyEntanglingLayers**: parameterized rotations + CNOT entanglement
- **AngleEmbedding (Y-rotation)**: each feature → RY(θ) on its qubit
- **Measurement**: PauliZ expectation on each qubit → classical FC layer → 2-class output
- **Training**: 100 epochs, Adam (lr=0.005), ReduceLROnPlateau, CrossEntropyLoss with class weights
- **ADASYN**: oversamples minority class (fraud) to 40% ratio before VQC training
- **Best model**: tracked by validation AUC, restored after training

### 4.8 Step 8: Arm B — QKSVM (Skipped)

```python
print("STEP 8: Arm B — QKSVM SKIPPED (running 2-arm ensemble: VQC + XGBoost)")
qksvm_train = None
qksvm_val   = None
qksvm_test  = None
```

The Projected Quantum Kernel SVM is skipped for speed. It would build a 600×600 kernel matrix using a ZZFeatureMap circuit — see [Section 9.1](#91-re-enabling-qksvm) to re-enable.

### 4.9 Step 9: Arm C — XGBoost + ADASYN (Lines 492–536)

```python
xgb_model = XGBClassifier(
    n_estimators=500, max_depth=6, learning_rate=0.05,
    subsample=0.8, colsample_bytree=0.8,
    scale_pos_weight=spw, eval_metric="auc",
)
```

**What it does:**
- Classical gradient-boosted trees on full WoE-scaled features (not just QUBO-selected)
- ADASYN oversampling (40% fraud ratio)
- `scale_pos_weight` = n_neg / n_pos (~8.4) for imbalance handling
- Early stopping via eval_set on validation

### 4.10 Step 10: Weighted Ensemble (Lines 538–564)

```python
# 2-arm: VQC + XGBoost (QKSVM skipped)
val_stack = np.column_stack([vqc_val, xgb_val])
meta_lr = LogisticRegression(C=100, random_state=42, max_iter=500)
meta_lr.fit(val_stack, y_val)

def ensemble_proba(vqc_p, xgb_p):
    stack = np.column_stack([vqc_p, xgb_p])
    return meta_lr.predict_proba(stack)[:, 1]
```

**What it does:**
- Stacks VQC and XGBoost probability predictions
- Logistic Regression meta-learner learns optimal combination weights on validation set
- Produces calibrated ensemble probabilities for all splits

### 4.11 Step 11: Isotonic Regression Calibration (Lines 566–598)

```python
iso_reg = IsotonicRegression(out_of_bounds="clip", increasing=True)
iso_reg.fit(ens_train, y_train)

cal_train = iso_reg.predict(ens_train)
cal_val   = iso_reg.predict(ens_val)
cal_test  = iso_reg.predict(ens_test)
```

**What it does:**
- Fits isotonic regression (PAV algorithm) on training ensemble scores
- `increasing=True` ensures higher raw scores → higher calibrated probabilities
- `out_of_bounds="clip"` handles unseen score ranges
- Designed to enforce monotonic decile ordering

### 4.12 Steps 12–16: Metrics, Deciles, Charts, CSI, Summary

- **Step 12**: Computes AUC, Gini (2×AUC−1), KS (max|TPR−FPR|) for all splits
- **Step 13**: Builds decile tables (10 equal bins by predicted score, descending), computes catch rate (top 20%) and monotonicity check
- **Step 14**: Rank order line charts + decile bar charts
- **Step 15**: CSI (Characteristic Stability Index) for all 14 features (train vs. test/val)
- **Step 16**: Final summary table with benchmark comparison

---

## 5. Configuration Options

### Hyperparameters (in quantum_fraud_model.py)

| Parameter | Location | Default | Effect |
|-----------|----------|---------|--------|
| `RANDOM_STATE` | Line 60 | 42 | Reproducibility seed |
| `n_bins` (WoE) | Line 208 | 10 | Number of quantile bins |
| `lambda1` (QUBO) | Line 261 | 0.15 | Sparsity penalty |
| `lambda2` (QUBO) | Line 261 | 0.08 | Correlation penalty |
| `num_reads` (SA) | Line 286 | 1000 | SA samples |
| `num_sweeps` (SA) | Line 286 | 2000 | SA iterations per read |
| `n_layers` (VQC) | Line 324 | 3 | Depth of VQC circuit |
| `EPOCHS` (VQC) | Line 400 | 100 | Training epochs |
| `batch_size` (VQC) | Line 397 | 64 | Mini-batch size |
| `lr` (VQC) | Line 402 | 0.005 | Adam learning rate |
| `patience` (VQC) | Line 404 | 12 | LR scheduler patience |
| `n_estimators` (XGB) | Line 588 | 500 | Number of trees |
| `max_depth` (XGB) | Line 589 | 6 | Max tree depth |
| `learning_rate` (XGB) | Line 590 | 0.05 | XGB learning rate |
| `sampling_strategy` (ADASYN) | Line 388 | 0.4 | Minority oversampling ratio |

### Recommended Settings by Use Case

| Use Case | EPOCHS | n_layers | lr | batch_size | n_estimators |
|----------|--------|----------|------|------------|-------------|
| Quick test | 20 | 2 | 0.01 | 128 | 100 |
| Standard (current) | 100 | 3 | 0.005 | 64 | 500 |
| High accuracy | 200 | 4 | 0.003 | 32 | 800 |
| Fast iteration | 50 | 2 | 0.01 | 128 | 300 |

---

## 6. Output Files

### 6.1 CSV Files (8 total)

| File | Contents |
|------|----------|
| `final_summary.csv` | Consolidated metrics (AUC, Gini, Catch Rate, Monotonicity) with benchmarks |
| `model_performance.csv` | AUC, Gini, KS per split |
| `iv_results.csv` | Information Value per feature (Train / VAL / Test) |
| `vif_results.csv` | VIF per feature |
| `csi_stability.csv` | CSI per feature (Test vs. Train, VAL vs. Train) |
| `decile_analysis_train.csv` | Train decile table (Total, Bad, Good, % Risky) |
| `decile_analysis_val.csv` | VAL decile table |
| `decile_analysis_test.csv` | Test decile table |

### 6.2 PNG Files (10 total)

| File | Description |
|------|-------------|
| `correlation_heatmap.png` | 14×14 Pearson correlation matrix |
| `vif_analysis.png` | VIF bar chart (red >5, orange >2.5, green <2.5) |
| `vqc_training_curves.png` | VQC loss (left) and VAL AUC (right) over epochs |
| `calibration_curve.png` | Raw ensemble vs. isotonic-calibrated (Test set) |
| `roc_curve_comparison.png` | ROC curves for Train / VAL / Test |
| `confusion_matrix_train.png` | 2×2 confusion matrix (Train) |
| `confusion_matrix_val.png` | 2×2 confusion matrix (VAL) |
| `confusion_matrix_test.png` | 2×2 confusion matrix (Test) |
| `rank_order_chart.png` | % Risky by decile (line plot, all 3 splits) |
| `decile_bad_rate_all.png` | % Risky by decile (bar chart, all 3 splits) |

---

## 7. Current Results

### 7.1 Performance Summary

| Split | AUC (%) | Gini | KS | Catch Rate (20%) | Monotonic |
|-------|---------|------|----|-------------------|-----------|
| Train | 99.7 | 99.5 | 95.8 | 99.8% | Yes |
| VAL | 80.9 | 61.8 | 56.1 | 66.1% | No |
| Test | 82.1 | 64.3 | 57.3 | 68.6% | No |

### 7.2 Benchmark Comparison (vs. §7.2 Custom_details_fr.pdf)

| Metric | Split | Model | Benchmark | Delta |
|--------|-------|-------|-----------|-------|
| AUC (%) | Train | 99.7 | 89.0 | +10.7 |
| AUC (%) | VAL | 80.9 | 91.1 | -10.2 |
| AUC (%) | Test | 82.1 | 88.5 | -6.4 |
| Gini | Train | 99.5 | 78.0 | +21.5 |
| Gini | VAL | 61.8 | 82.2 | -20.4 |
| Gini | Test | 64.3 | 76.9 | -12.6 |
| Catch Rate | Train | 99.8% | 75.3% | +24.5% |
| Catch Rate | VAL | 66.1% | 73.0% | -6.9% |
| Catch Rate | Test | 68.6% | 75.2% | -6.6% |

**Observation:** Train performance is exceptional but VAL/Test generalization gaps suggest overfitting. The 2-arm ensemble (without QKSVM) and current hyperparameters need tuning to close the gap.

### 7.3 Information Value (IV) — Feature Ranking

| Feature | IV_Train | IV_VAL | IV_Test |
|---------|----------|--------|---------|
| salary_income_3m | 1.1458 | 2.4256 | 0.8825 |
| max_debit_amount | 0.8205 | 1.9758 | 0.7299 |
| salary_round_fig_pc_3m | 0.6686 | 1.1984 | 0.6371 |
| avg_amt_cash_out_3m | 0.6429 | 0.7104 | 0.8110 |
| var_salary_maxmin | 0.6281 | 0.9373 | 0.6095 |
| max_balance_3m | 0.4625 | 0.3648 | 0.3582 |
| amt_cash_in_cash_out_ratio_3m | 0.3757 | 0.3795 | 0.4643 |
| max_debit_amt_2days_m2 | 0.3050 | 0.1846 | 0.3270 |
| avg_end_month_bal_salary_ratio | 0.2992 | 0.5186 | 0.3544 |
| num_cash_out_3m | 0.2313 | 0.2804 | 0.2377 |
| max_min_endmonth_bal_ratio | 0.1734 | 0.2839 | 0.1629 |
| end_month_balance_m1_m2m3 | 0.1728 | 0.1873 | 0.3207 |
| num_cash_out_1m_3m_ratio | 0.0928 | 0.1347 | 0.0618 |
| sal_prc_yn | 0.0000 | 0.0000 | 0.0000 |

All top features have IV > 0.3 (strong predictive power). `sal_prc_yn` has zero IV across all splits.

### 7.4 VIF Analysis

| Feature | VIF |
|---------|-----|
| max_balance_3m | 2.5926 |
| max_debit_amount | 2.3025 |
| avg_end_month_bal_salary_ratio | 2.0262 |
| salary_income_3m | 1.9625 |
| salary_round_fig_pc_3m | 1.7629 |
| avg_amt_cash_out_3m | 1.6053 |
| var_salary_maxmin | 1.3768 |
| max_debit_amt_2days_m2 | 1.2862 |
| num_cash_out_3m | 1.1548 |
| sal_prc_yn | 1.0918 |
| num_cash_out_1m_3m_ratio | 1.0408 |
| amt_cash_in_cash_out_ratio_3m | 1.0122 |
| max_min_endmonth_bal_ratio | 1.0016 |
| end_month_balance_m1_m2m3 | 1.0015 |

All VIF < 5 — no multicollinearity concerns. Highest = 2.59 (max_balance_3m).

### 7.5 CSI Stability

| Feature | CSI_Test | CSI_VAL |
|---------|----------|---------|
| max_balance_3m | 0.0154 | 0.0123 |
| avg_end_month_bal_salary_ratio | 0.0140 | 0.0124 |
| max_debit_amount | 0.0120 | 0.0104 |
| num_cash_out_1m_3m_ratio | 0.0063 | 0.0340 |
| salary_income_3m | 0.0021 | 0.0003 |
| sal_prc_yn | 0.0000 | 0.0000 |

All CSI < 0.035 — excellent stability. No feature drift between splits.

### 7.6 Decile Tables

**Train:**

| Decile | Total | Bad | % Risky |
|--------|-------|-----|---------|
| 1 | 433 | 408 | 94.23 |
| 2 | 433 | 51 | 11.78 |
| 3 | 433 | 1 | 0.23 |
| 4–10 | 433 each | 0 | 0.00 |

Train catch rate (top 20%): 99.8% — Monotonic: Yes

**VAL:**

| Decile | Total | Bad | % Risky |
|--------|-------|-----|---------|
| 1 | 109 | 54 | 49.54 |
| 2 | 108 | 22 | 20.37 |
| 3 | 108 | 15 | 13.89 |
| 4 | 108 | 0 | 0.00 |
| 5 | 109 | 6 | 5.50 |
| 6 | 108 | 5 | 4.63 |
| 7 | 108 | 5 | 4.63 |
| 8 | 108 | 1 | 0.93 |
| 9 | 108 | 3 | 2.78 |
| 10 | 109 | 4 | 3.67 |

VAL catch rate (top 20%): 66.1% — Monotonic: **No** (decile 5 > decile 4)

**Test:**

| Decile | Total | Bad | % Risky |
|--------|-------|-----|---------|
| 1 | 227 | 118 | 51.98 |
| 2 | 226 | 48 | 21.24 |
| 3 | 226 | 19 | 8.41 |
| 4 | 226 | 2 | 0.88 |
| 5 | 226 | 34 | 15.04 |
| 6 | 226 | 5 | 2.21 |
| 7 | 226 | 5 | 2.21 |
| 8 | 226 | 2 | 0.88 |
| 9 | 226 | 2 | 0.88 |
| 10 | 226 | 7 | 3.10 |

Test catch rate (top 20%): 68.6% — Monotonic: **No** (decile 5 > decile 4)

---

## 8. Troubleshooting

### Issue: VQC Training is Extremely Slow

**Cause:** Quantum circuit simulation is CPU-intensive; each sample runs through the circuit individually.

**Solutions:**
1. Reduce epochs: `EPOCHS = 50`
2. Reduce circuit depth: `n_layers = 2`
3. Increase batch size: `batch_size = 128` (fewer gradient updates)
4. Ensure `lightning.qubit` backend is installed (faster than `default.qubit`)

### Issue: ADASYN Fails

**Error:**
```
ValueError: No samples will be generated with the provided ratio settings
```

**Cause:** Fraud rate already close to the target `sampling_strategy=0.4`.

**Solution:** Lower the target ratio:
```python
adasyn = ADASYN(sampling_strategy=0.3, random_state=42, n_neighbors=3)
```

### Issue: ModuleNotFoundError for pennylane-lightning

**Error:**
```
ModuleNotFoundError: No module named 'pennylane_lightning'
```

**Solution:**
```powershell
pip install pennylane-lightning
```

### Issue: Non-Monotonic Deciles on VAL/Test

**Cause:** Isotonic regression is fit on train scores; out-of-distribution val/test scores may not preserve monotonicity.

**Possible Fixes:**
1. Increase training data variety (reduce ADASYN ratio)
2. Reduce VQC overfitting (fewer layers, more regularization)
3. Add QKSVM arm back for better ensemble diversity (see Section 9.1)
4. Post-hoc decile smoothing (average neighboring deciles)

### Issue: NaN in Loss During VQC Training

**Cause:** Learning rate too high or exploding gradients.

**Solution:**
```python
optimizer = torch.optim.Adam(vqc_model.parameters(), lr=0.001)
```

### Issue: All VQC Predictions are the Same Class

**Cause:** Class imbalance overwhelming the model.

**Solution:** Check that class weights are applied:
```python
print(f"Class weights: {class_weights}")
# Should show: tensor([1.0, ~8.4])
```

---

## 9. Extending the Model

### 9.1 Re-enabling QKSVM (Arm B)

To restore the 3-arm ensemble, replace the QKSVM skip block (Step 8) with the full implementation. Key changes:

1. **Uncomment** the full Step 8 code block (ZZFeatureMap circuit, kernel matrix builder, SVC fitting)
2. **Update Step 10 ensemble** to use 3 arms:
   ```python
   val_stack = np.column_stack([vqc_val, qksvm_val, xgb_val])

   def ensemble_proba(vqc_p, qk_p, xgb_p):
       stack = np.column_stack([vqc_p, qk_p, xgb_p])
       return meta_lr.predict_proba(stack)[:, 1]
   ```
3. **Reduce kernel subset** for speed: change `train_size=600` to `train_size=300`

**Expected additional runtime:** 15–45 minutes for the 600-sample kernel matrix.

### 9.2 Tuning for Better Generalization

If train AUC >> test AUC (overfitting):

```python
# Reduce VQC capacity
n_layers = 2            # was 3
EPOCHS = 50             # was 100

# Increase XGBoost regularization
xgb_model = XGBClassifier(
    max_depth=4,         # was 6
    reg_alpha=0.1,       # L1 regularization
    reg_lambda=1.0,      # L2 regularization
    min_child_weight=5,  # higher = more conservative
)

# Reduce ADASYN ratio
adasyn = ADASYN(sampling_strategy=0.25)  # was 0.4
```

### 9.3 Adding New Features

1. Add feature name to `FEATURE_COLS` list (line 43)
2. QUBO will automatically include it in the selection process
3. If more than ~14 features: consider PCA reduction before VQC (qubits are expensive)

### 9.4 Different Quantum Backends

```python
# Faster: lightning.qubit (C++ statevector)
dev = qml.device("lightning.qubit", wires=n_qubits)

# More accurate: default.qubit (Python statevector)
dev = qml.device("default.qubit", wires=n_qubits)

# Cloud: Amazon Braket (requires AWS account)
dev = qml.device("braket.aws.qubit", device_arn="arn:aws:braket:::device/quantum-simulator/amazon/sv1", wires=n_qubits)
```

---

## 10. Quick Reference

### Commands

| Task | Command |
|------|---------|
| Run pipeline | `python quantum_fraud_model.py` |
| Install deps | `pip install pennylane torch xgboost imbalanced-learn dwave-neal dimod` |
| Activate env | `.\.venv\Scripts\Activate.ps1` |
| Check installs | `python -c "import pennylane, torch, xgboost, imblearn, neal; print('OK')"` |

### Key Files

| File | Purpose |
|------|---------|
| `quantum_fraud_model.py` | Full training & evaluation pipeline |
| `13981_rec_with_perf_sep_16_Fraud.csv` | Dataset (7,674 records) |
| `Custom_details_fr.pdf` | §7.2 benchmark targets |
| `outputs/final_summary.csv` | Consolidated results |
| `outputs/model_performance.csv` | AUC / Gini / KS |

### Important Parameters

| Parameter | Default | Effect |
|-----------|---------|--------|
| `N_QUBITS` | Auto (QUBO) | Must match selected features |
| `n_layers` | 3 | More = more capacity, slower |
| `EPOCHS` | 100 | More = better fit, risk overfitting |
| `lr` | 0.005 | Lower = stable but slow |
| `n_estimators` | 500 | More trees = better fit |
| `sampling_strategy` | 0.4 | ADASYN minority ratio target |

### Architecture Summary

```
Raw Features (14)
    ↓
WoE Encoding (10 bins per feature)
    ↓
RobustScaler → MinMaxScaler → [0, π]
    ↓
QUBO Feature Selection (SA) → N selected features
    ↓
┌─────────────────┐    ┌─────────────────┐
│  Arm A: VQC     │    │  Arm C: XGBoost │
│  (N qubits,     │    │  (500 trees,    │
│   3 layers,     │    │   ADASYN,       │
│   data reupload)│    │   full features)│
└────────┬────────┘    └────────┬────────┘
         │                      │
         └──────────┬───────────┘
                    ↓
      Logistic Meta-Learner (2-arm)
                    ↓
      Isotonic Regression Calibration
                    ↓
      Calibrated Fraud Probabilities
                    ↓
      Decile Analysis + Reporting
```
