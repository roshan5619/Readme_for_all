# Quantum Fraud Risk — Knowledge Guide

## Table of Contents
1. [Overview](#1-overview)
2. [Target Variable Definition](#2-target-variable-definition)
3. [The 14 Features](#3-the-14-features)
4. [Weight of Evidence (WoE) Encoding](#4-weight-of-evidence-woe-encoding)
5. [Information Value (IV)](#5-information-value-iv)
6. [Scaling Pipeline](#6-scaling-pipeline)
7. [QUBO Feature Selection](#7-qubo-feature-selection)
8. [The Quantum Circuit (VQC)](#8-the-quantum-circuit-vqc)
9. [VQC Architecture & Training](#9-vqc-architecture--training)
10. [XGBoost Classical Anchor](#10-xgboost-classical-anchor)
11. [Ensemble & Meta-Learner](#11-ensemble--meta-learner)
12. [Isotonic Regression Calibration](#12-isotonic-regression-calibration)
13. [Performance Metrics](#13-performance-metrics)
14. [Stability Analysis](#14-stability-analysis)
15. [Numerical Examples](#15-numerical-examples)
16. [References](#16-references)

---

## 1. Overview

This project uses a **Quantum-Classical Hybrid Model** to detect fraud from bank statement features. The pipeline is:

```
Raw Features (14)
    ↓
WoE Encoding (quantile bins → ln(P(bad)/P(good)))
    ↓
RobustScaler → MinMaxScaler → ×π → [0, π]
    ↓
QUBO Feature Selection (Simulated Annealing)
    ↓
┌──────────────────────────┐    ┌───────────────────────┐
│  Arm A: VQC              │    │  Arm C: XGBoost       │
│  Data-Reuploading        │    │  500 trees + ADASYN   │
│  PennyLane (N qubits,    │    │  Full WoE features    │
│  3 layers, ADASYN)       │    │  scale_pos_weight     │
└────────────┬─────────────┘    └───────────┬───────────┘
             │                              │
             └──────────────┬───────────────┘
                            ↓
              Logistic Meta-Learner (on VAL)
                            ↓
              Isotonic Regression Calibration
                            ↓
              Calibrated P(fraud) → Decile Rank
```

### Why Quantum for Fraud Detection?

1. **Non-linear feature interactions** — Quantum circuits create entangled feature representations that capture interactions classical logistic regression cannot
2. **Small data advantage** — With ~7K records, quantum kernels can outperform deep learning (which needs 100K+ samples) [arXiv:2312.00260]
3. **Exponentially large feature spaces** — N qubits encode 2^N dimensional Hilbert space with only O(N) parameters
4. **Class imbalance resilience** — QML advantage grows with class imbalance [Scientific Reports 2025]

---

## 2. Target Variable Definition

### Target Column: `bad_f`

| bad_f Value | Interpretation | Class Label |
|-------------|----------------|-------------|
| **1** | Fraud detected | **Fraud** |
| **0** | No fraud observed | **Good** |

### Data Profile

| Metric | Value |
|--------|-------|
| Total records | 7,674 |
| Total features | 14 |
| Overall fraud rate | 10.64% |
| Imbalance ratio | ~1:8.4 (fraud : good) |

### Split Strategy

The CSV contains a `Split` column with values `train` (5,413 rows) and `test` (2,261 rows). Since there is no validation split in the CSV, we create one:

```
CSV "train" (5,413)  ──→  stratified 80/20 ──→  Train (4,331)  +  VAL (1,083)
CSV "test"  (2,261)  ──→  kept as-is       ──→  Test  (2,260)
```

| Split | Records | Fraud Count | Fraud Rate |
|-------|---------|-------------|------------|
| Train | 4,331 | 460 | 10.62% |
| VAL | 1,083 | 115 | 10.63% |
| Test | 2,260 | 242 | 10.71% |

---

## 3. The 14 Features

| Feature | Description | Risk Interpretation | IV (Train) |
|---------|-------------|---------------------|------------|
| `salary_income_3m` | Total salary income over 3 months | Low income = higher risk | 1.1458 |
| `max_debit_amount` | Largest single debit transaction | Large debits = potential stress | 0.8205 |
| `salary_round_fig_pc_3m` | % of salary in round figures (3m) | Round figures may indicate synthetic income | 0.6686 |
| `avg_amt_cash_out_3m` | Average cash-out amount (3 months) | High cash-out = risky behavior | 0.6429 |
| `var_salary_maxmin` | Max-min salary variance | Unstable income = higher risk | 0.6281 |
| `max_balance_3m` | Maximum account balance (3 months) | Low max balance = limited buffer | 0.4625 |
| `amt_cash_in_cash_out_ratio_3m` | Cash in ÷ cash out ratio (3 months) | < 1.0 = spending > income | 0.3757 |
| `max_debit_amt_2days_m2` | Max debit in 2-day window (month 2) | Burst spending pattern | 0.3050 |
| `avg_end_month_bal_salary_ratio` | End-of-month balance ÷ salary | Low ratio = living paycheck to paycheck | 0.2992 |
| `num_cash_out_3m` | Number of cash-out transactions (3m) | High frequency = risky | 0.2313 |
| `max_min_endmonth_bal_ratio` | Max ÷ min end-of-month balance | High volatility = unstable | 0.1734 |
| `end_month_balance_m1_m2m3` | End-month balance trend (M1 vs M2+M3) | Declining balance = risk | 0.1728 |
| `num_cash_out_1m_3m_ratio` | Cash-out count (1m) ÷ (3m) | Recent spike in transactions | 0.0928 |
| `sal_prc_yn` | Salary present (binary flag) | No IV — constant across classes | 0.0000 |

---

## 4. Weight of Evidence (WoE) Encoding

### What is WoE?

WoE transforms each feature into a measure of its predictive power for the target variable. It replaces raw feature values with the **log-odds ratio** of being "bad" (fraud) vs. "good" within each bin.

### Mathematical Definition

For a feature bin $b$:

$$\text{WoE}_b = \ln\left(\frac{P(X \in b \mid \text{fraud})}{P(X \in b \mid \text{good})}\right) = \ln\left(\frac{n_{\text{bad},b} / N_{\text{bad}}}{n_{\text{good},b} / N_{\text{good}}}\right)$$

Where:
- $n_{\text{bad},b}$ = number of frauds in bin $b$
- $n_{\text{good},b}$ = number of goods in bin $b$
- $N_{\text{bad}}$ = total frauds in training data
- $N_{\text{good}}$ = total goods in training data

### Interpretation

| WoE Value | Meaning |
|-----------|---------|
| Large positive | Bin has disproportionately many frauds |
| Near zero | Bin has similar fraud/good distribution |
| Large negative | Bin has disproportionately many goods |

### Binning Strategy

- **10 quantile-based bins** per feature (equal-count)
- Bins are fit on **training data only** (prevents data leakage)
- Applied to VAL and Test using the same bin edges
- Edge cases: values below/above bin range get the nearest bin's WoE
- Epsilon smoothing (1e-6) prevents division by zero

### Why WoE for Quantum Circuits?

1. **Monotonic transformation** — preserves rank ordering, maps to smooth values
2. **Handles outliers** — extreme values are absorbed into end bins
3. **Normalizes scale** — all features become comparable WoE values
4. **Captures non-linear relationships** — each bin can have different risk levels

---

## 5. Information Value (IV)

### Definition

IV measures the overall predictive power of a feature:

$$\text{IV} = \sum_{b=1}^{B} \left(\frac{n_{\text{bad},b}}{N_{\text{bad}}} - \frac{n_{\text{good},b}}{N_{\text{good}}}\right) \times \text{WoE}_b$$

### Interpretation Scale

| IV Range | Predictive Power |
|----------|-----------------|
| < 0.02 | Not useful |
| 0.02 – 0.10 | Weak |
| 0.10 – 0.30 | Medium |
| 0.30 – 0.50 | Strong |
| > 0.50 | Very strong / suspicious (check for leakage) |

### Feature Ranking by IV

In our model:
- **5 features** with IV > 0.50 (very strong): salary_income_3m, max_debit_amount, salary_round_fig_pc_3m, avg_amt_cash_out_3m, var_salary_maxmin
- **5 features** with IV 0.17–0.46 (strong/medium)
- **3 features** with IV 0.09–0.17 (medium/weak)
- **1 feature** with IV = 0 (sal_prc_yn — no predictive value)

IV is used as the **reward signal** in the QUBO feature selection (higher IV = more desirable to select).

---

## 6. Scaling Pipeline

### Three-Stage Scaling: WoE → RobustScaler → MinMaxScaler → Angular

#### Stage 1: RobustScaler

$$X_{\text{robust}} = \frac{X_{\text{WoE}} - \text{median}(X_{\text{WoE}})}{\text{IQR}(X_{\text{WoE}})}$$

Where IQR = Q3 - Q1 (interquartile range).

**Why:** Resistant to outliers. Unlike StandardScaler (which uses mean/std), RobustScaler uses median/IQR, so a single extreme WoE value won't distort the scale.

#### Stage 2: MinMaxScaler

$$X_{\text{mm}} = \frac{X_{\text{robust}} - \min(X_{\text{robust}})}{\max(X_{\text{robust}}) - \min(X_{\text{robust}})}$$

Maps to [0, 1] range.

#### Stage 3: Angular Encoding

$$\theta = X_{\text{mm}} \times \pi$$

Maps to [0, π] for quantum gate rotations.

### Why [0, π]?

| Angle | Qubit State | Bloch Sphere Position |
|-------|-------------|----------------------|
| θ = 0 | \|0⟩ (ground state) | North pole |
| θ = π/2 | Equal superposition | Equator |
| θ = π | \|1⟩ (excited state) | South pole |

Using [0, π] gives **maximum discrimination power** — features span the full range from |0⟩ to |1⟩.

---

## 7. QUBO Feature Selection

### What is QUBO?

QUBO (Quadratic Unconstrained Binary Optimization) is a combinatorial optimization problem native to quantum annealers. It selects the optimal subset of features by balancing:

1. **Relevance** (IV score) — prefer informative features
2. **Sparsity** — prefer fewer features (simpler model)
3. **Redundancy** — avoid selecting correlated features

### QUBO Formulation

Decision variables: $x_i \in \{0, 1\}$ (1 = select feature $i$)

$$E(\mathbf{x}) = \sum_i (\lambda_1 - s_i) x_i + \sum_{i<j} \lambda_2 |\rho_{ij}| x_i x_j$$

Where:
- $s_i$ = normalized IV score of feature $i$ (in [0, 1])
- $\rho_{ij}$ = Pearson correlation between features $i$ and $j$
- $\lambda_1 = 0.15$ = sparsity penalty
- $\lambda_2 = 0.08$ = redundancy penalty

### QUBO Matrix

- **Diagonal:** $Q_{ii} = \lambda_1 - s_i$ (selects features where IV exceeds sparsity cost)
- **Off-diagonal:** $Q_{ij} = \lambda_2 |\rho_{ij}|$ (penalizes selecting correlated pairs)

### Solving with Simulated Annealing

Since we don't have access to a D-Wave quantum annealer, we use **Simulated Annealing (SA)** — a classical heuristic that mimics quantum annealing:

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `num_reads` | 1000 | Number of independent SA runs |
| `num_sweeps` | 2000 | Iterations per run |
| `seed` | 42 | Reproducibility |

The best solution across all reads is selected. If fewer than 6 features are chosen, we fall back to the top 8 by IV.

---

## 8. The Quantum Circuit (VQC)

### Circuit Architecture: Data-Reuploading VQC

```
Layer 1                    Layer 2                    Layer 3
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│ RY(θ₀) ─── SEL  │      │ RY(θ₀) ─── SEL  │      │ RY(θ₀) ─── SEL  │
│ RY(θ₁) ─── SEL  │  →   │ RY(θ₁) ─── SEL  │  →   │ RY(θ₁) ─── SEL  │  →  ⟨Z⟩
│ RY(θ₂) ─── SEL  │      │ RY(θ₂) ─── SEL  │      │ RY(θ₂) ─── SEL  │
│  ...      ...    │      │  ...      ...    │      │  ...      ...    │
│ RY(θₙ) ─── SEL  │      │ RY(θₙ) ─── SEL  │      │ RY(θₙ) ─── SEL  │
└──────────────────┘      └──────────────────┘      └──────────────────┘
↑ Data re-upload           ↑ Data re-upload           ↑ Data re-upload
  + trainable                + trainable                + trainable

SEL = StronglyEntanglingLayer (Rot gates + CNOT entanglement)
```

### Key Design Principle: Data Reuploading

In a standard VQC, features are encoded **once** at the beginning. In a **data-reuploading** VQC, features are re-encoded at **every layer**:

```python
for layer in range(n_layers):
    qml.AngleEmbedding(inputs, wires=range(n_qubits), rotation="Y")
    qml.StronglyEntanglingLayers(weights[layer].unsqueeze(0), wires=range(n_qubits))
```

**Why data reuploading?**
- Proven +10–15% AUC gain over single-encoding [arXiv:2509.25245]
- Acts like a deep neural network: each layer "re-reads" the data in a new basis
- Breaks the expressibility bottleneck of shallow quantum circuits

### AngleEmbedding

Each feature θ_i is encoded as a rotation around the Y-axis:

$$R_Y(\theta) = \begin{pmatrix} \cos(\theta/2) & -\sin(\theta/2) \\ \sin(\theta/2) & \cos(\theta/2) \end{pmatrix}$$

After applying $R_Y(\theta)|0\rangle$:
- P(measure 0) = cos²(θ/2)
- P(measure 1) = sin²(θ/2)

### StronglyEntanglingLayers

Each layer applies:
1. **Rot(φ, θ, ω)** gate on each qubit — the most general single-qubit rotation:

$$\text{Rot}(\phi, \theta, \omega) = R_Z(\omega) R_Y(\theta) R_Z(\phi)$$

2. **CNOT entanglement** between neighboring qubits — creates quantum correlations:

$$\text{CNOT}|a, b\rangle = |a, a \oplus b\rangle$$

Parameters per layer: N_qubits × 3 = 3N trainable angles.

### Measurement

PauliZ expectation value on each qubit:

$$\langle Z_i \rangle = P_i(|0\rangle) - P_i(|1\rangle) \in [-1, +1]$$

These N values are fed to a classical fully-connected layer.

---

## 9. VQC Architecture & Training

### Full Model

```
Input (N features, angular-encoded)
    ↓
┌─────────────────────────────────────────┐
│  QUANTUM CIRCUIT                        │
│  3 × [AngleEmbedding + SEL]             │
│  N qubits, 3N×3 = 9N trainable params  │
│  Measure ⟨Z₀⟩, ⟨Z₁⟩, ..., ⟨Zₙ⟩        │
└─────────────────────────────────────────┘
    ↓
[⟨Z₀⟩, ..., ⟨Zₙ⟩] ∈ [-1, +1]ᴺ
    ↓
┌─────────────────────────────────────────┐
│  CLASSICAL LAYER                        │
│  Linear(N → 2)                          │
│  2N + 2 parameters                      │
└─────────────────────────────────────────┘
    ↓
Softmax → [P(Good), P(Fraud)]
```

### Parameter Count

| Component | Formula | Example (N=10) |
|-----------|---------|----------------|
| Quantum weights | 3 layers × N × 3 | 90 |
| Classical FC | N × 2 + 2 | 22 |
| **Total** | 9N + 2N + 2 = 11N + 2 | **112** |

Compare: a classical MLP with similar capacity would need 500+ parameters.

### Training Configuration

| Setting | Value | Purpose |
|---------|-------|---------|
| Optimizer | Adam | Adaptive learning rate |
| Learning rate | 0.005 | Moderate for quantum circuits |
| Scheduler | ReduceLROnPlateau | Halves LR after 12 epochs without improvement |
| Min LR | 1e-5 | Lower bound |
| Loss | CrossEntropyLoss | With class weights [1.0, ~8.4] |
| Batch size | 64 | Trade-off: speed vs. gradient quality |
| Epochs | 100 | Best model saved by val AUC |
| Backend | lightning.qubit | C++ statevector simulator (fast) |
| Diff method | adjoint | Efficient gradient computation |

### ADASYN Oversampling

Before training, the minority class (fraud) is oversampled using **ADASYN** (Adaptive Synthetic Sampling):

- Target ratio: 0.4 (40% fraud after oversampling)
- Neighbors: 5 (for density estimation)
- Unlike SMOTE, ADASYN generates **more** synthetic samples near the decision boundary (harder cases)

### Class Weights

$$w_{\text{fraud}} = \frac{n_{\text{good}}}{n_{\text{fraud}}} \approx 8.4$$

This makes misclassifying a fraud ~8.4× more costly than misclassifying a good customer.

### Gradient Computation

PennyLane uses the **adjoint differentiation** method:

$$\frac{\partial \langle Z \rangle}{\partial \theta_k} = \text{computed via reverse-mode through the circuit}$$

This is more efficient than the parameter-shift rule for statevector simulators (one backward pass vs. 2 circuit evaluations per parameter).

---

## 10. XGBoost Classical Anchor

### Why Include a Classical Model?

1. **Ensemble diversity** — XGBoost captures different patterns than VQC
2. **Stability** — classical models are well-understood and debuggable
3. **Full feature set** — XGBoost uses all 14 WoE features (not just QUBO-selected)
4. **Baseline guarantee** — ensures the ensemble is at least as good as classical alone

### Configuration

| Parameter | Value | Meaning |
|-----------|-------|---------|
| n_estimators | 500 | Number of boosted trees |
| max_depth | 6 | Maximum tree depth |
| learning_rate | 0.05 | Shrinkage factor |
| subsample | 0.8 | Row sampling per tree |
| colsample_bytree | 0.8 | Column sampling per tree |
| scale_pos_weight | ~8.4 | Handles class imbalance |
| eval_metric | AUC | Optimization target |

### ADASYN for XGBoost

Same oversampling applied:
- Target ratio: 0.4
- Applied to the full 14-feature WoE space (not QUBO-selected)
- Separate ADASYN instance from VQC (each arm gets its own oversampled data)

---

## 11. Ensemble & Meta-Learner

### Stacking Architecture

```
                   VQC probabilities ────┐
                                         ├──→ LogisticRegression ──→ Ensemble P(fraud)
                   XGB probabilities ────┘
                   (fit on VAL set)
```

### How It Works

1. **Each arm** produces P(fraud) for every sample in all splits
2. **Stack** the arm probabilities as columns: `[P_vqc, P_xgb]`
3. **Fit** a LogisticRegression meta-learner on VAL labels
4. **Predict** ensemble P(fraud) for Train, VAL, and Test

### Why Logistic Regression Meta-Learner?

- Simple, interpretable (learned coefficients = arm weights)
- C=100 (near-unregularized) — trusts the arm predictions
- Calibrated output (logistic function produces true probabilities)
- No risk of overfitting with just 2 input features

### Weight Interpretation

The meta-learner's coefficients reveal how much each arm contributes:

```python
print(meta_lr.coef_)
# Example: [[0.8, 1.5]] → XGBoost contributes ~1.9× more than VQC
```

---

## 12. Isotonic Regression Calibration

### What is Isotonic Regression?

Isotonic regression fits a **non-decreasing step function** to the data using the **Pool Adjacent Violators (PAV)** algorithm.

### PAV Algorithm

Given pairs (score_i, label_i) sorted by score:

1. Start with each point as its own block
2. If block i has a higher average than block i+1 (violation):
   - **Pool** them: merge into one block, average their labels
3. Repeat until no violations remain

Result: a monotonically non-decreasing mapping from raw scores to calibrated probabilities.

### Why Isotonic Calibration?

1. **Monotonicity guarantee** — higher raw score always maps to higher P(fraud)
2. **Non-parametric** — no assumptions about the score distribution
3. **Decile-friendly** — monotonic scores → monotonic decile bad rates (in theory)

### Configuration

```python
iso_reg = IsotonicRegression(out_of_bounds="clip", increasing=True)
iso_reg.fit(ens_train, y_train)
```

- `increasing=True` — higher ensemble score = higher fraud probability
- `out_of_bounds="clip"` — val/test scores outside train range are clipped to [min, max]
- Fit on **training** scores, applied to all splits

### Limitation

Isotonic regression guarantees monotonicity on the **training** data. On unseen data (VAL/Test), out-of-distribution scores may create non-monotonic deciles — this is observed in the current results.

---

## 13. Performance Metrics

### 13.1 AUC-ROC (Area Under ROC Curve)

The ROC curve plots **True Positive Rate** (sensitivity) vs. **False Positive Rate** (1 - specificity) at all classification thresholds.

$$\text{TPR} = \frac{TP}{TP + FN}, \quad \text{FPR} = \frac{FP}{FP + TN}$$

| AUC | Interpretation |
|-----|----------------|
| 0.50 | Random guessing |
| 0.60–0.70 | Poor |
| 0.70–0.80 | Acceptable |
| 0.80–0.90 | Good |
| 0.90+ | Excellent |

### 13.2 Gini Coefficient

$$\text{Gini} = 2 \times \text{AUC} - 1$$

| AUC | Gini | Meaning |
|-----|------|---------|
| 0.50 | 0.0 | No discrimination |
| 0.75 | 0.5 | Good |
| 0.90 | 0.8 | Excellent |
| 1.00 | 1.0 | Perfect |

### 13.3 KS Statistic (Kolmogorov-Smirnov)

$$\text{KS} = \max_t | \text{TPR}(t) - \text{FPR}(t) |$$

KS measures the **maximum separation** between the cumulative distributions of frauds and goods. Higher = better separation.

| KS | Interpretation |
|----|----------------|
| < 20% | Poor |
| 20–40% | Acceptable |
| 40–60% | Good |
| > 60% | Excellent |

### 13.4 Decile Analysis

**Process:**
1. Sort all samples by P(fraud) **descending**
2. Divide into 10 equal-sized groups (deciles)
3. Decile 1 = highest predicted risk, Decile 10 = lowest

**Key metrics per decile:**
- **% Risky**: percentage of actual frauds in that decile
- **Catch Rate**: percentage of all frauds captured up to that decile

### 13.5 Catch Rate (Top 20%)

The percentage of all actual frauds captured in the top 2 deciles (riskiest 20% of population).

$$\text{Catch Rate}_{20\%} = \frac{\text{frauds in decile 1 + decile 2}}{\text{total frauds}} \times 100\%$$

**Business meaning:** "By flagging the riskiest 20% of applications, we catch X% of all frauds."

### 13.6 Rank Ordering (Monotonicity)

A model has **good rank ordering** if % Risky decreases monotonically from Decile 1 → 10:

```
Good:  15% → 12% → 9% → 7% → 5% → 4% → 3% → 2% → 1% → 0.5%  ✓

Bad:   15% → 12% → 14% → 7% → 5% → 6% → 3% → 2% → 1% → 0.5%  ✗
                    ↑              ↑
              Inversions (decile 3 > decile 2, decile 6 > decile 5)
```

Monotonicity is a **hard requirement** for production risk models — it ensures the scoring system is consistent.

---

## 14. Stability Analysis

### 14.1 VIF (Variance Inflation Factor)

Measures multicollinearity:

$$\text{VIF}_j = \frac{1}{1 - R_j^2}$$

Where $R_j^2$ is the R-squared from regressing feature $j$ on all other features.

| VIF | Interpretation |
|-----|----------------|
| 1.0 | No correlation |
| 1–5 | Acceptable |
| 5–10 | Moderate concern |
| > 10 | Severe (remove feature) |

All features in this model have VIF < 2.6 — no multicollinearity.

### 14.2 CSI (Characteristic Stability Index)

CSI measures how much a feature's distribution has shifted between two datasets:

$$\text{CSI} = \sum_{b=1}^{B} (p_b^{\text{other}} - p_b^{\text{train}}) \times \ln\left(\frac{p_b^{\text{other}}}{p_b^{\text{train}}}\right)$$

Where $p_b$ is the proportion of samples in bin $b$.

| CSI | Interpretation |
|-----|----------------|
| < 0.10 | Stable — no significant shift |
| 0.10–0.25 | Minor shift — monitor |
| > 0.25 | Major shift — investigate |

All features here have CSI < 0.035 — **excellent stability** across train/test/val.

### 14.3 Pearson Correlation

$$r_{xy} = \frac{\sum(x_i - \bar{x})(y_i - \bar{y})}{\sqrt{\sum(x_i - \bar{x})^2 \sum(y_i - \bar{y})^2}}$$

**Threshold:** |r| ≥ 0.6 indicates high correlation (potential redundancy). The QUBO formulation penalizes selecting highly correlated feature pairs.

---

## 15. Numerical Examples

### Example 1: Full Pipeline for One Customer

**Raw feature values:**
```
salary_income_3m           = 25,000
max_debit_amount           = 8,000
avg_amt_cash_out_3m        = 3,500
max_balance_3m             = 45,000
num_cash_out_3m            = 12
```

**Step 1: WoE Encoding**

Assume training bin edges and WoE values:
```
salary_income_3m:  25,000 falls in bin [20K, 30K] → WoE = -0.45
max_debit_amount:  8,000 falls in bin [5K, 10K]   → WoE = +0.32
avg_amt_cash_out:  3,500 falls in bin [3K, 4K]    → WoE = -0.18
max_balance_3m:    45,000 falls in bin [40K, 50K]  → WoE = -0.62
num_cash_out_3m:   12 falls in bin [10, 15]        → WoE = +0.15
```

**Step 2: RobustScaler** (centers on median, scales by IQR)
```
salary_income_3m:  (-0.45 - (-0.10)) / 0.80 = -0.4375
max_debit_amount:  (+0.32 - (+0.05)) / 0.60 = +0.4500
(and so on for each feature)
```

**Step 3: MinMaxScaler → Angular**
```
After MinMax: [-0.4375 maps to ~0.30]  → angle = 0.30 × π = 0.942 rad
              [+0.4500 maps to ~0.72]  → angle = 0.72 × π = 2.262 rad
```

**Step 4: Quantum Circuit**

After 3 layers of AngleEmbedding + StronglyEntanglingLayers (with trained weights):
```
⟨Z₀⟩ = -0.35,  ⟨Z₁⟩ = +0.22,  ⟨Z₂⟩ = +0.68,  ⟨Z₃⟩ = -0.15,  ⟨Z₄⟩ = +0.41
```

**Step 5: Classical Layer**
```
logits = Linear(5 → 2):  [−0.25, +0.18]
Softmax:  P(Good) = 0.395,  P(Fraud) = 0.605
```

**Step 6: Ensemble**

XGBoost independently predicts P(fraud) = 0.42.
Meta-learner combines: ensemble_P = LogisticRegression([0.605, 0.42]) → 0.53

**Step 7: Isotonic Calibration**

Raw 0.53 → calibrated 0.48 (based on training isotonic fit)

**Final:** This customer has a 48% calibrated fraud probability → falls in Decile 2 (high risk).

---

### Example 2: Decile Table Calculation

**Test set:** 2,260 customers, 242 actual frauds

**Sorted by calibrated P(fraud) descending, divided into 10 groups:**

| Decile | Count | Frauds | % Risky | Cum Frauds | Catch Rate |
|--------|-------|--------|---------|------------|------------|
| 1 | 226 | 118 | 52.2% | 118 | 48.8% |
| 2 | 226 | 48 | 21.2% | 166 | 68.6% |
| 3 | 226 | 19 | 8.4% | 185 | 76.4% |
| 4 | 226 | 2 | 0.9% | 187 | 77.3% |
| ... | ... | ... | ... | ... | ... |

**Top-2 Catch Rate:** 166 / 242 = **68.6%** (by flagging riskiest 20%, we catch 68.6% of all frauds)

---

### Example 3: QUBO Feature Selection

**Simplified 4-feature example:**

Features: F1 (IV=0.8), F2 (IV=0.3), F3 (IV=0.6), F4 (IV=0.1)
Correlations: |corr(F1,F3)| = 0.7, all others < 0.2

**QUBO matrix (λ1=0.15, λ2=0.08):**
```
         F1      F2      F3      F4
F1    [0.15-1.0  0.02    0.056   0.02 ]     (Q11 = λ1 - IV_norm)
F2    [         0.15-0.29  0.02   0.02 ]
F3    [                  0.15-0.71  0.02 ]
F4    [                           0.15-0.0]
```

Diagonal: F1 = -0.85, F2 = -0.14, F3 = -0.56, F4 = +0.15
Off-diagonal: Q(F1,F3) = 0.08 × 0.7 = 0.056

**Optimal solution:** x = [1, 1, 0, 0]
- F1 selected (highest IV, strong negative diagonal)
- F2 selected (modest IV, no correlation penalty)
- F3 **rejected** (high IV but high correlation with F1 adds penalty)
- F4 rejected (positive diagonal = sparsity cost exceeds IV benefit)

---

## 16. References

### Core Papers

1. **arXiv:2312.00260 (2023) — QMKL**
   Quantum Multiple Kernel Learning for fraud detection. All quantum models outperformed best classical on HSBC Digital Payments data.

2. **arXiv:2509.25245 (2025) — VQC for Financial Fraud**
   ZZFeatureMap + circular entanglement VQC achieves 94.3% accuracy. Demonstrates data reuploading advantage.

3. **Nature Communications 2024 — Thanasilp et al.**
   Exponential concentration in quantum kernels. Projected kernels avoid this pitfall — basis for our QKSVM design.

4. **arXiv:2309.01127 (2023) — QGNN**
   Quantum Graph Neural Network AUC 0.85 vs GraphSAGE 0.77 on fraud graph data.

5. **AWS Blog July 2024 — Deloitte Italy**
   PennyLane + Amazon Braket production deployment for financial fraud.

### Techniques

6. **Scientific Reports (2025) — QML and Class Imbalance**
   QML advantage grows with class imbalance — directly relevant to our ~10% fraud rate.

7. **arXiv:2508.18514 (2025) — Wells Fargo**
   RL initialization to avoid barren plateaus in financial quantum circuits.

8. **arXiv:2509.02863 (2025) — QI-SMOTE**
   Quantum-inspired synthetic minority oversampling technique.

### Textbooks & Frameworks

9. **PennyLane Documentation** — pennylane.ai
   Quantum computing framework used for VQC implementation.

10. **XGBoost Documentation** — xgboost.readthedocs.io
    Gradient boosting framework for the classical ensemble arm.

11. **dwave-neal** — docs.ocean.dwavesys.com
    Simulated annealing sampler for QUBO optimization.

---

## Summary

| Concept | Key Formula / Value |
|---------|---------------------|
| WoE | ln(P(X\|bad) / P(X\|good)) per bin |
| IV | Σ (dist_bad - dist_good) × WoE |
| Angular encoding | θ = MinMax(Robust(WoE)) × π |
| QUBO energy | E = λ₁\|x\| + λ₂Σ\|ρᵢⱼ\|xᵢxⱼ − Σsᵢxᵢ |
| VQC params | 3 layers × N × 3 (quantum) + N × 2 + 2 (classical) |
| Gini | 2 × AUC − 1 |
| KS | max\|TPR − FPR\| |
| Catch Rate (20%) | frauds in top-2 deciles / total frauds |
| VIF threshold | < 5 acceptable, > 10 severe |
| CSI threshold | < 0.10 stable, > 0.25 major shift |
