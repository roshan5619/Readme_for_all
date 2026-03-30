# HFT_Solutions: Knowledge Guide

## Table of Contents
1. [Theoretical Foundations](#1-theoretical-foundations)
2. [Feature Selection Methods](#2-feature-selection-methods)
3. [Quantum Computing Components](#3-quantum-computing-components)
4. [VQC Architectures](#4-vqc-architectures)
5. [Classical ML Components](#5-classical-ml-components)
6. [Model Evaluation Framework](#6-model-evaluation-framework)
7. [API Reference](#7-api-reference)

---

## 1. Theoretical Foundations

### Why Quantum-Classical Hybrid?

Traditional credit/fraud risk models use purely classical approaches (logistic regression, gradient boosting). This framework introduces quantum computing at two levels:

1. **Feature Engineering** — Quantum Autoencoders compress selected features into latent quantum representations, potentially capturing non-linear relationships that classical autoencoders miss.

2. **Classification** — Variational Quantum Circuits (VQCs) serve as quantum neural networks. The data-reuploading architecture allows quantum feature maps to create decision boundaries in high-dimensional Hilbert space.

The hybrid approach hedges risk: an XGBoost model runs alongside the VQC, and a meta-learner ensembles both. If quantum provides no advantage, the ensemble gracefully degrades to classical performance.

### Data Flow Architecture

```
Raw Data (308 cols)
    |
    v
[Pre-screening: missing, variance, correlation, IV]
    |   ~60-100 features
    v
[Feature Selection Funnel: Boruta -> LASSO -> RFECV -> Permutation -> QUBO]
    |   10-12 features
    v
[Quantum Autoencoder] --> Latent features (augmentation)
    |
    v
[WoE Encoding] --> [Angular Scaling: 0 to pi] --> [ADASYN Oversampling]
    |
    v
[VQC Architecture A: Two-Pass QUBO ZZ]  ----+
[VQC Architecture B: Data-Reuploading]  ----+--> [Best VQC by val AUC]
    |                                              |
    v                                              v
[XGBoost on all pre-screened features]    [Winning VQC]
    |                                              |
    +----------> [Logistic Meta-Learner] <---------+
                         |
                         v
              [Isotonic Calibration]
                         |
                         v
              [Final Probabilities]
```

---

## 2. Feature Selection Methods

### 2.1 Pre-Screening (`prescreen_features`)

**Purpose:** Reduce 299 features to a manageable candidate set.

**Criteria (applied sequentially):**
1. **Missing values >50%** — Feature lacks sufficient data
2. **Sentinel replacement** — `inf`, `-1.0` values replaced with NaN
3. **Near-zero variance** — >95% same value (no discriminatory power)
4. **Median imputation** — Fill NaN with train set median
5. **Correlation |r| > 0.6** — Drop one from each correlated pair (keep higher univariate AUC). This is a HARD CONSTRAINT enforced here and again after QUBO.
6. **IV < 0.02** — Feature has negligible predictive power

### 2.2 Boruta SHAP (`boruta_shap_selection`)

**Purpose:** Identify features that are genuinely important vs. randomly useful.

**How it works:**
1. Train XGBoost on real features
2. Compute SHAP importance (not Gini — SHAP is model-agnostic and additive)
3. For 100 trials:
   - Create "shadow features" by permuting each original feature
   - Combine real + shadow features, retrain XGBoost
   - Feature gets a "hit" if its SHAP importance > max shadow importance
4. Features with >50% hit rate are accepted

**Why SHAP over Gini?** Gini importance is biased toward high-cardinality features. SHAP provides consistent, additive feature attributions based on Shapley values from cooperative game theory.

### 2.3 Stability Selection (`stability_selection_lasso`)

**Purpose:** Find features that are consistently selected across data perturbations.

**How it works:**
1. Run L1-penalized logistic regression on 100 bootstrap samples
2. For each bootstrap, record which features have non-zero coefficients
3. Features selected in >60% of bootstraps are "stable"
4. Intersect with Boruta-accepted features

**Why?** A feature might be selected by Boruta because of a specific data pattern. Stability selection tests whether the feature survives across different subsamples — a form of robustness validation.

### 2.4 RFECV (`rfecv_selection`)

**Purpose:** Find the optimal number of features by cross-validated elimination.

**How it works:**
1. Start with all stable features
2. Train XGBoost, evaluate AUC via 5-fold stratified CV
3. Remove the least important feature
4. Repeat until `min_features_to_select=5`
5. Select the feature set that maximizes mean CV AUC

**Produces:** An AUC-vs-number-of-features curve showing the optimal point.

### 2.5 Permutation Importance (`permutation_importance_filter`)

**Purpose:** Validate that each feature actually helps on unseen data.

**How it works:**
1. Train XGBoost on train set
2. For each feature on validation set: randomly shuffle that feature 10 times
3. Measure AUC drop — if AUC doesn't drop (importance <= 0), the feature isn't useful on validation data
4. Drop features with zero or negative importance

### 2.6 QUBO Feature Selection (`qubo_feature_selection`)

**Purpose:** Select the final 10-12 features using quantum-inspired optimization.

**QUBO (Quadratic Unconstrained Binary Optimization):**

The objective function combines three terms:

```
minimize: -lambda1 * sum(IV_i * x_i)           # Maximize IV
         + lambda2 * sum(|r_ij| * x_i * x_j)   # Penalize correlation
         + lambda3 * (sum(x_i) - target_k)^2    # Cardinality constraint
```

Where `x_i in {0,1}` indicates whether feature `i` is selected.

**Solved via:** Simulated annealing (`dwave-neal`), which approximates the solution a quantum annealer would find.

**Adaptive lambda3:** The algorithm adjusts `lambda3` to hit the 10-12 feature target range.

**Post-QUBO enforcement:** Even after QUBO, all pairs are checked for |r| < 0.6. Any violating pair has the lower-IV member dropped and replaced from the RFECV survivor pool.

---

## 3. Quantum Computing Components

### 3.1 PennyLane Framework

PennyLane is a cross-platform quantum ML library. In this project:

- **Devices:**
  - `default.qubit` — Pure Python simulator (local/CPU)
  - `lightning.qubit` — C++ optimized simulator (CPU)
  - `lightning.gpu` — CUDA GPU simulator (DGX)

- **Interface:** `torch` — All quantum circuits return torch tensors with autograd support

- **Diff methods:**
  - `backprop` — Classical backpropagation through the state vector (requires `default.qubit`)
  - `adjoint` — Efficient gradient computation via adjoint differentiation (requires `lightning.*`)

### 3.2 Quantum Autoencoder (`train_quantum_autoencoder`)

**Purpose:** Compress QUBO-selected features into a lower-dimensional quantum latent space.

**Architecture:**
```
Input state (n_features qubits)
    |
    v
[Encoder circuit: parameterized rotations + entangling layers]
    |
    v
[Measure latent qubits (n_latent < n_features)]
    |
    v
[Decoder circuit: reconstruct from latent]
    |
    v
Output state
```

**Loss function:** `1 - fidelity(input_state, reconstructed_state)`

Fidelity measures overlap between quantum states: `F = |<psi|phi>|^2`. Perfect reconstruction = fidelity of 1.

**Usage:** Latent representations are extracted and used alongside the original features (augmentation, not replacement).

### 3.3 Qubit Count

The number of qubits equals the number of QUBO-selected features (10-12). Each feature maps to one qubit via angular encoding:

```python
qml.AngleEmbedding(features, wires=range(n_qubits))
# Each feature value (scaled to [0, pi]) becomes a rotation angle on its qubit
```

---

## 4. VQC Architectures

### 4.1 Architecture A: Two-Pass QUBO ZZ + Feedback Gate (`VQCTwoPassQUBO`)

**Parameters:** ~222 for 10 qubits

**Structure:**
```
Pass 1 (5 layers):
    For each layer:
        AngleEmbedding(features)          # Data re-upload
        StronglyEntanglingLayers(params)  # Parameterized rotations + CNOT
        IsingZZ coupling                  # ZZ entanglement between adjacent qubits

Classical Feedback Gate:
    measurements_pass1 = [<Z_i> for i in range(n_qubits)]    # Expectation values
    feedback_input = concat(features, measurements_pass1)     # 2*n_qubits inputs
    feedback = Linear(2*n_qubits -> n_qubits) + tanh          # Classical neural layer

Pass 2 (2 layers):
    AngleEmbedding(feedback)              # Use feedback as new input
    StronglyEntanglingLayers(params)      # Lightweight second pass

Output:
    [<Z_0>, <Z_1>, ..., <Z_{n-1}>]       # Qubit expectation values
    Linear(n_qubits -> 2) + softmax       # Classification head
```

**Key insight:** The feedback gate allows the second pass to "see" what the first pass learned, creating a form of quantum-classical recurrence.

### 4.2 Architecture B: Data-Reuploading VQC (`VQCDataReuploading`)

**Parameters:** ~132 for 10 qubits

**Structure:**
```
4 layers, each:
    AngleEmbedding(features)              # Re-upload data every layer
    StronglyEntanglingLayers(params)      # Parameterized rotations + CNOT

Output:
    [<Z_0>, <Z_1>, ..., <Z_{n-1}>]
    Linear(n_qubits -> 2) + softmax
```

**Key insight:** Data re-uploading (encoding features in every layer, not just the first) has been shown to make VQCs universal function approximators (Perez-Salinas et al., 2020).

### 4.3 Architecture Selection

Both architectures are trained for 100 epochs. The one with higher **validation AUC** is selected as the "winner" and used in the ensemble.

```
Arch A val AUC vs Arch B val AUC --> Winner
```

The non-winning architecture's checkpoint is still saved for future analysis.

---

## 5. Classical ML Components

### 5.1 Weight of Evidence (WoE) Encoding

**Purpose:** Transform numeric features into a monotonic, informative encoding.

For each feature, bin values into 10 quantile-based bins, then compute:

```
WoE_bin = ln(Distribution of Bads in bin / Distribution of Goods in bin)
```

**Properties:**
- WoE is naturally monotonic with respect to the target
- Information Value (IV) = sum of (Dist_Bad - Dist_Good) * WoE across bins
- IV interpretation: <0.02 = useless, 0.02-0.1 = weak, 0.1-0.3 = medium, >0.3 = strong

### 5.2 Angular Scaling

After WoE encoding, features are scaled to [0, pi] for quantum circuit input:

```
RobustScaler -> MinMaxScaler -> multiply by pi
```

**Why pi?** Quantum rotation gates (RX, RY, RZ) use angles in [0, 2pi]. Scaling to [0, pi] ensures full coverage of one half-rotation, which is sufficient for binary classification.

### 5.3 ADASYN Oversampling

**Problem:** Imbalanced targets (credit default at 3.2%, fraud at 10.3%)
**Solution:** ADASYN (Adaptive Synthetic Sampling) generates synthetic minority samples, focusing on hard-to-classify regions.

```python
ADASYN(sampling_strategy=0.4)  # Target 40% minority ratio
```

Fallback to SMOTE if ADASYN fails (can happen with very few minority samples).

### 5.4 XGBoost Ensemble Component

XGBoost is trained on ALL pre-screened features (not just QUBO-selected), giving it access to a broader feature set than the VQC. This diversity strengthens the ensemble.

### 5.5 Logistic Meta-Learner

The ensemble combines VQC and XGBoost probabilities:

```python
meta_features = np.column_stack([vqc_proba, xgb_proba])
meta_learner = LogisticRegression()
meta_learner.fit(meta_features_train, y_train)
final_proba = meta_learner.predict_proba(meta_features)[:, 1]
```

### 5.6 Isotonic Calibration

**Purpose:** Ensure calibrated, monotonic decile ordering.

Isotonic regression fits a non-decreasing step function to the predicted probabilities, guaranteeing that higher predicted probabilities always correspond to higher observed bad rates.

---

## 6. Model Evaluation Framework

### 6.1 Metrics

| Metric | Formula | Interpretation |
|--------|---------|----------------|
| **AUC ROC** | Area under ROC curve | Overall discrimination (0.5 = random, 1.0 = perfect) |
| **Gini** | 2 * AUC - 1 | Normalized discrimination coefficient |
| **KS Statistic** | max\|TPR - FPR\| across thresholds | Maximum separation between good and bad distributions |
| **Top-2 Decile Catch Rate** | % of bads captured in top 20% of scores | Operational effectiveness (how many bads caught early) |

### 6.2 Decile Analysis

For each split (train/val/test), predictions are sorted into 10 equal-sized bins (deciles):

| Column | Meaning |
|--------|---------|
| `decile` | 1 (highest risk) to 10 (lowest risk) |
| `count` | Number of observations in decile |
| `n_bad` | Number of positive (bad) cases |
| `bad_rate` | n_bad / count |
| `catch_rate` | n_bad_in_decile / total_n_bad |
| `cumulative_catch` | Running sum of catch_rate |

**Monotonicity check:** Bad rates should decrease from decile 1 to 10. Non-monotonic ordering indicates poor calibration.

### 6.3 CSI (Characteristic Stability Index)

Also called PSI (Population Stability Index). Measures feature distribution shift between datasets:

```
CSI = sum( (actual_pct - expected_pct) * ln(actual_pct / expected_pct) )
```

| CSI Value | Interpretation |
|-----------|----------------|
| < 0.10 | No significant shift |
| 0.10 - 0.25 | Moderate shift, monitor |
| > 0.25 | Significant shift, investigate |

Computed for each QUBO-selected feature: train vs val and train vs test.

### 6.4 SHAP Explainability

SHAP (SHapley Additive exPlanations) provides model-agnostic feature attributions based on game-theoretic Shapley values.

**Plots generated:**
- **Beeswarm (summary):** Shows each feature's impact distribution across all predictions
- **Bar (importance):** Ranked mean absolute SHAP values
- **Waterfall:** Single-prediction breakdown showing each feature's contribution

Applied to the XGBoost ensemble component (TreeExplainer is exact for tree models, not approximate).

---

## 7. API Reference

### shared/quantum_utils.py — Complete Function Reference

#### Data Preprocessing

```python
compute_woe_iv(X, y, feature_names, n_bins=10, epsilon=1e-6)
```
Compute WoE encoding maps and IV scores. Returns `(woe_maps, iv_values)` where `woe_maps` is `{feature: [(lo, hi, woe), ...]}` and `iv_values` is `{feature: float}`.

```python
apply_woe(X, feature_names, woe_maps)
```
Transform features using pre-computed WoE maps. Returns transformed numpy array.

```python
prescreen_features(X_train_df, X_val_df, X_test_df, y_train, feature_cols,
                   missing_thresh=0.50, nzv_thresh=0.95, corr_thresh=0.60, iv_min=0.02)
```
Run all pre-screening criteria. Returns `(surviving_features, quick_iv_dict, updated_dataframes)`.

---

#### Feature Selection

```python
boruta_shap_selection(X_train, y_train, feature_names, n_trials=100, random_state=42)
```
Returns `(accepted_features, results_dataframe)`.

```python
stability_selection_lasso(X_train, y_train, feature_names,
                          n_bootstrap=100, threshold=0.60, random_state=42)
```
Returns `(stable_features, stability_scores_array)`.

```python
rfecv_selection(X_train, y_train, feature_names, random_state=42)
```
Returns `(selected_features, results_df, cv_results)`.

```python
permutation_importance_filter(X_val, y_val, feature_names, model, threshold=0.0)
```
Returns `(kept_features, importance_means)`.

```python
build_qubo(feature_names, iv_scores, corr_matrix,
           lambda1=0.10, lambda2=0.05, lambda3=0.0, target_k=11)
```
Returns QUBO matrix as `dict` for simulated annealing.

```python
qubo_feature_selection(feature_names, iv_scores, corr_matrix,
                       target_min=10, target_max=12, random_state=42)
```
Returns `list` of selected feature names.

```python
enforce_correlation_constraint(selected, corr_df, iv_scores, backup_pool,
                               corr_thresh=0.60, target_min=10)
```
Returns `list` of selected features with guaranteed |r| < 0.6.

---

#### Quantum Components

```python
train_quantum_autoencoder(X_train, n_latent, n_epochs=50, lr=0.01)
```
Trains fidelity-driven quantum autoencoder. Returns `(latent_representations, loss_history)`.

```python
class VQCDataReuploading(nn.Module):
    def __init__(self, n_qubits, n_layers=4)
    def forward(self, x) -> Tensor  # shape: (batch, 2)
```

```python
class VQCTwoPassQUBO(nn.Module):
    def __init__(self, n_qubits, n_layers_pass1=5, n_layers_pass2=2)
    def forward(self, x) -> Tensor  # shape: (batch, 2)
```

```python
train_vqc(model, train_loader, val_X, val_y, class_weights,
          epochs=100, lr=0.005, patience=12, arch_name="VQC")
```
Returns `(best_val_auc, train_history, val_history)`. Best model state auto-loaded.

```python
vqc_predict_proba(model, X_tensor)
```
Returns numpy array of positive-class probabilities.

---

#### Evaluation

```python
compute_importance_weights(X, y)
```
Returns weight array combining class weights and prediction difficulty.

```python
compute_decile_table(y_true, y_prob, split_name, n_bins=10)
```
Returns DataFrame with decile, count, n_bad, bad_rate, catch_rate, cumulative_catch.

```python
catch_rate_top20(decile_df, y_true)
```
Returns float: percentage of bads caught in top 2 deciles.

```python
check_monotonicity(decile_df, split_name)
```
Returns bool: True if bad rates are monotonically non-increasing.

```python
compute_csi(train_col, other_col, n_bins=10)
```
Returns float: CSI/PSI value.

```python
compute_metrics(y_true, y_prob, split_name)
```
Returns dict with keys: `split`, `auc`, `gini`, `ks`.

---

## Key Performance Constraints

| Constraint | Value | Enforced At |
|------------|-------|-------------|
| Correlation | \|r\| < 0.6 | Step 2 (pre-screen) AND Step 8 (post-QUBO) |
| VIF | < 5 | Step 3 (flagged) |
| Feature count | 10-12 | Step 8 (QUBO target) |
| Decile monotonicity | Non-increasing bad rates | Step 19 (checked) |
| Calibration | Isotonic (monotonic) | Step 16 (enforced) |
