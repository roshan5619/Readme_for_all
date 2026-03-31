# Quantum-Classical Hybrid Feature Selection Pipeline
## Complete Knowledge Guide, Implementation Guide & Numerical Walkthrough

---

## Part 1: Detailed Explanation

### What Does This Pipeline Do?

This pipeline solves a **fraud detection** problem with ~300 raw features. The core challenge: too many features cause overfitting, slow inference, and poor generalization. The pipeline selects exactly **15 optimal features** using a two-stage funnel:

```
300 features ──▶ [Classical GPU SHAP] ──▶ 18 features ──▶ [Quantum QAOA] ──▶ 15 features ──▶ [XGBoost + Calibration] ──▶ Fraud Probabilities
```

### Why Two Stages?

A quantum circuit with 300 qubits is impossible on any current simulator or hardware. The classical pre-screen (SHAP) is a **dimensionality bridge** — it cheaply eliminates the obvious noise so the expensive quantum optimizer only handles the hard, subtle selection among the top 18 candidates.

---

### Stage-by-Stage Breakdown

#### Stage 0: Data Ingestion

```
Raw CSV ──▶ Sentinel replacement ──▶ One-hot encoding ──▶ Train/Test split
```

- **Sentinel values** (9999, -2) are replaced with `NaN`. These are common placeholders in credit data meaning "not applicable" or "missing."
- **No imputation is performed** for the model itself — XGBoost handles `NaN` natively by learning optimal split directions for missing values.
- **One-hot encoding** converts categorical columns (e.g., `state = "TX"`) into binary columns (`state_TX = 1`).
- Data is split using a pre-existing `split` column (not random), preserving temporal or business logic separation.

#### Stage 1: Classical GPU Pre-Screening (SHAP)

**Goal:** Reduce 300 features → 18 using SHAP importance from a quick XGBoost model.

**How SHAP works here:**
1. A lightweight XGBoost is trained on all 300 features (100 trees, depth 4).
2. SHAP (SHapley Additive exPlanations) computes, for every prediction, how much each feature contributed to that prediction.
3. The mean absolute SHAP value across all samples gives each feature an importance score.
4. The top 18 features by SHAP importance survive.

**Why SHAP over simple feature importance?**
XGBoost's built-in `feature_importance` counts splits or gains, which is biased toward high-cardinality features. SHAP is theoretically grounded in cooperative game theory (Shapley values) and gives consistent, fair attributions.

**Why 18?**
18 qubits is manageable for a state-vector simulator. The state vector has 2¹⁸ = 262,144 amplitudes — fits comfortably in GPU/CPU RAM. Going to 25+ qubits would require 2²⁵ = 33M+ amplitudes and become very slow.

#### Stage 2: Quantum QAOA Feature Selection (18 → 15)

This is the core innovation. It frames "which 15 of 18 features to keep" as a **combinatorial optimization problem**, then solves it with a quantum algorithm.

##### Step 2a: Build the QUBO Matrix

**QUBO** = Quadratic Unconstrained Binary Optimization. We define a binary variable xᵢ ∈ {0, 1} for each feature — 1 means "selected," 0 means "rejected."

The objective function to **minimize** is:

```
minimize:  Σᵢ Σⱼ Qᵢⱼ · xᵢ · xⱼ
```

The Q matrix encodes three competing goals:

| Component | Formula | Effect |
|-----------|---------|--------|
| **Relevance** (diagonal) | −α · MI(featureᵢ, target) | Reward features that predict the target |
| **Redundancy** (off-diagonal) | +β · \|corr(featureᵢ, featureⱼ)\| | Penalize selecting two correlated features |
| **Cardinality constraint** | penalty · (Σ xᵢ − k)² expanded | Force exactly k=15 features to be selected |

The cardinality penalty expands as:
```
penalty · (Σ xᵢ − k)² = penalty · [Σᵢ xᵢ² − 2k·Σᵢ xᵢ + k² + 2·Σᵢ<ⱼ xᵢxⱼ]
```

Since xᵢ ∈ {0,1}, xᵢ² = xᵢ. This gives:
- Diagonal contribution: `penalty · (1 − 2k)` per feature (since xᵢ² = xᵢ)
- Off-diagonal contribution: `+2 · penalty` per pair

The code combines all three into Q:
```python
Q[i,i] = -α·relevance[i] + penalty·(1 - 2k)    # simplified: -α·rel - 2·penalty·k + penalty
Q[i,j] = β·redundancy[i,j] + 2·penalty           # for i < j
```

##### Step 2b: Convert QUBO → Ising Hamiltonian

Quantum computers work with spin variables Zᵢ ∈ {+1, −1}, not binary xᵢ ∈ {0, 1}. The mapping is:

```
xᵢ = (1 − Zᵢ) / 2
```

Substituting into the QUBO objective and simplifying gives the Ising form:

```
H = Σᵢ hᵢ·Zᵢ  +  Σᵢ<ⱼ Jᵢⱼ·Zᵢ·Zⱼ  +  constant
```

Where:
```
hᵢ = −Qᵢᵢ/2  −  Σⱼ≠ᵢ Qᵢⱼ/4
Jᵢⱼ = Qᵢⱼ/4
```

The constant offset is dropped because it doesn't affect which state minimizes H.

##### Step 2c: QAOA Circuit

**QAOA** (Quantum Approximate Optimization Algorithm) is a hybrid quantum-classical algorithm:

1. **Initialize:** Apply Hadamard gates to all qubits → creates equal superposition of all 2¹⁸ possible feature subsets simultaneously.

2. **Cost layer:** Apply `e^{−iγH_cost}` — this rotates the quantum state so that low-energy (good) solutions get higher amplitude. The cost Hamiltonian is the Ising H from Step 2b.

3. **Mixer layer:** Apply `e^{−iα·H_mixer}` where H_mixer = Σ Xᵢ. This "mixes" amplitudes between solutions, allowing the optimizer to explore different subsets.

4. **Repeat** steps 2-3 for `depth=2` layers (called "p" in QAOA literature).

5. **Measure:** The circuit outputs a probability distribution over all 2¹⁸ bitstrings. The bitstring with the highest probability is the quantum algorithm's answer for the best feature subset.

##### Step 2d: Training the Circuit

The angles γ and α (2 per layer × 2 layers = 4 parameters total) are optimized classically using the Adam optimizer:

```
Loop 60 times:
    1. Run quantum circuit with current (γ, α)
    2. Measure expected energy ⟨H_cost⟩
    3. Compute gradients via parameter-shift rule
    4. Update (γ, α) using Adam
```

This is the "variational" or "hybrid" aspect — the quantum circuit proposes solutions, classical optimization tunes the circuit parameters.

##### Step 2e: Failsafe

Due to finite circuit depth (p=2) and optimization noise, the quantum state may not perfectly satisfy the constraint Σxᵢ = 15. The code classically trims (if >15) or pads (if <15) using the leftover SHAP ordering. This guarantees exactly 15 features.

#### Stage 3: Final Model Training

With 15 features locked in:

1. **5-Fold Stratified CV** generates out-of-fold (OOF) predictions — these serve as the "VAL" segment for unbiased evaluation.
2. **Isotonic Calibration** wraps XGBoost via `CalibratedClassifierCV`. This ensures predicted probabilities are well-calibrated (a prediction of 0.10 means ~10% of those cases are actually fraud).
3. **Final model** is trained on all training data with calibration and scored on Train, Test, and VAL.

#### Stage 4: Evaluation

Three metrics are reported:

| Metric | Formula | Interpretation |
|--------|---------|----------------|
| **AUC** | Area under ROC curve | Overall discrimination (0.5 = random, 1.0 = perfect) |
| **Gini** | (2 × AUC − 1) × 100 | Normalized AUC, common in credit risk (0 = random, 100 = perfect) |
| **Catch Rate @20%** | % of frauds in top 20% of scores | Operational metric: if you investigate your riskiest 20%, how many frauds do you catch? |

**Rank Order Charts** show fraud rate by decile — a well-performing model shows a steep downward slope from decile 1 (highest risk) to decile 10 (lowest risk).

---

## Part 2: Knowledge Guide — Core Concepts

### 2.1 QUBO (Quadratic Unconstrained Binary Optimization)

**What:** A mathematical framework for expressing combinatorial optimization over binary variables.

**General form:**
```
minimize f(x) = xᵀ Q x = Σᵢ Σⱼ Qᵢⱼ xᵢ xⱼ     where xᵢ ∈ {0, 1}
```

**Why it matters:** Many real-world problems (knapsack, graph coloring, portfolio optimization, feature selection) can be cast as QUBO. Once in QUBO form, they map directly to quantum hardware.

**Key property:** Since xᵢ ∈ {0,1}, we have xᵢ² = xᵢ, so the diagonal of Q acts as "linear" terms.

### 2.2 Ising Model

**What:** A physics model from statistical mechanics describing interacting spins.

**Hamiltonian:**
```
H = Σᵢ hᵢ σᵢ + Σᵢ<ⱼ Jᵢⱼ σᵢ σⱼ
```

where σᵢ ∈ {+1, −1} are spin variables.

**Connection to QUBO:** Every QUBO has an equivalent Ising formulation via x = (1−σ)/2. This is essential because quantum gates naturally operate on spin-like (qubit) degrees of freedom.

### 2.3 QAOA (Quantum Approximate Optimization Algorithm)

**Invented by:** Farhi, Goldstone, and Gutmann (2014).

**Core idea:**
- Encode the optimization objective as a quantum Hamiltonian H_C.
- Start in equal superposition |+⟩^n (all solutions equally likely).
- Alternate between "cost evolution" (amplifies good solutions) and "mixer evolution" (explores the solution space).
- The depth parameter p controls the circuit's expressiveness. As p → ∞, QAOA converges to the exact optimal solution.

**Circuit structure for depth p:**
```
|ψ(γ,α)⟩ = [e^{-iα_p H_M} · e^{-iγ_p H_C}] · ... · [e^{-iα_1 H_M} · e^{-iγ_1 H_C}] · |+⟩^n
```

**Parameters:** 2p real numbers (γ₁...γ_p, α₁...α_p), optimized classically.

### 2.4 Mutual Information (MI)

**What:** Measures how much knowing feature X reduces uncertainty about target Y.

```
MI(X; Y) = Σ_{x,y} p(x,y) · log[ p(x,y) / (p(x)·p(y)) ]
```

- MI = 0 → X and Y are independent (useless feature).
- Higher MI → X is more informative about Y.
- Unlike correlation, MI captures **nonlinear** relationships.

### 2.5 SHAP (SHapley Additive exPlanations)

Based on Shapley values from game theory. For a prediction f(x):

```
f(x) = φ₀ + Σᵢ φᵢ
```

where φᵢ is feature i's contribution. The mean |φᵢ| across all samples gives global importance.

**Key properties:**
- **Consistency:** If a feature's true contribution increases, its SHAP value never decreases.
- **Local accuracy:** SHAP values sum exactly to the model output.
- **Efficiency:** TreeSHAP runs in O(TLD²) for tree models, making it practical for 300 features.

### 2.6 Isotonic Calibration

Raw XGBoost outputs are **not** calibrated probabilities. Isotonic regression fits a non-decreasing step function mapping raw scores → true event rates. It's more flexible than Platt scaling (sigmoid) and is preferred when you have enough data (>1000 samples per fold).

### 2.7 PennyLane Specifics

- **`lightning.qubit`**: A C++-accelerated state-vector simulator. Much faster than the default `default.qubit` for 15-20 qubits. Does NOT require a GPU.
- **`pnp.array(..., requires_grad=True)`**: PennyLane wraps NumPy so that autograd can track gradients through quantum circuits. Regular `np.array` would break gradient computation.
- **Parameter-shift rule**: PennyLane computes quantum gradients by evaluating the circuit at shifted parameter values: ∂f/∂θ = [f(θ+π/2) − f(θ−π/2)] / 2. This requires 2 circuit evaluations per parameter per gradient step.

---

## Part 3: Implementation Guide

### 3.1 Environment Setup

```bash
# Core
pip install numpy pandas scikit-learn xgboost shap matplotlib seaborn

# Quantum
pip install pennylane pennylane-lightning

# Optional: GPU XGBoost (requires CUDA)
pip install xgboost --upgrade  # Ensure CUDA-capable build
```

**Hardware requirements:**
- Classical pre-screen: Any machine. GPU optional (speeds up XGBoost SHAP from ~2min to ~20s).
- QAOA (18 qubits): ~4 GB RAM, runs on CPU. `lightning.qubit` uses efficient C++ backends.
- Final XGBoost: GPU recommended for production; CPU works fine for <100K rows.

### 3.2 Configuration Parameters

| Parameter | Value | Why | How to Tune |
|-----------|-------|-----|-------------|
| `top_k` (SHAP screen) | 18 | Max qubits for fast simulation | Increase if you have >16 GB RAM (up to ~24) |
| `target_features` | 15 | Business requirement | Set based on model complexity budget |
| `alpha` (relevance) | 1.0 | Weight for MI reward | Increase to favor predictive power |
| `beta` (redundancy) | 0.5 | Weight for correlation penalty | Increase to favor diversity |
| `penalty` | 10.0 | Constraint enforcement strength | Must be >> alpha, beta to enforce cardinality |
| `depth` (p) | 2 | QAOA layers | More layers = better solution but slower; 2-4 is typical |
| `optimizer steps` | 60 | Training iterations | 50-100 is usually sufficient for 18 qubits |
| `n_estimators` | 100 | XGBoost trees | Tune via early stopping in production |

### 3.3 Step-by-Step Implementation Checklist

```
□ 1. Load CSV, replace sentinels with NaN
□ 2. One-hot encode categoricals
□ 3. Split by pre-existing split column
□ 4. Train quick XGBoost on all features
□ 5. Compute SHAP values, select top 18
□ 6. Compute MI (relevance) and |correlation| (redundancy) on top 18
□ 7. Build QUBO matrix Q (18×18)
□ 8. Convert Q → Ising (h vector, J matrix)
□ 9. Build PennyLane Hamiltonian from h, J
□ 10. Define QAOA circuit with depth=2
□ 11. Optimize 4 parameters over 60 steps
□ 12. Sample highest-probability bitstring
□ 13. Trim/pad to exactly 15 features
□ 14. Train calibrated XGBoost on 15 features
□ 15. Evaluate AUC, Gini, Catch Rate
```

### 3.4 Common Pitfalls & Fixes

**Pitfall 1: Using `np.array` instead of `pnp.array` for QAOA params.**
Symptom: Gradients are all zero; cost never changes.
Fix: Always use `from pennylane import numpy as pnp` and set `requires_grad=True`.

**Pitfall 2: Penalty too small.**
Symptom: Quantum solution selects 3 or 25 features instead of 15.
Fix: Set penalty ≥ 5× max(alpha × max_MI, beta × 1.0). A value of 10.0 works for normalized data.

**Pitfall 3: SHAP computed on imputed data.**
Symptom: Features that only "exist" after imputation rank high.
Fix: The code correctly uses XGBoost's native NaN handling for SHAP. Do NOT impute before SHAP.

**Pitfall 4: Calibration with too few folds.**
Symptom: Isotonic calibration overfits.
Fix: Use cv≥3 inside CalibratedClassifierCV, and ensure each fold has ≥500 positive cases.

### 3.5 Adapting for Your Data

**Different number of raw features:**
- <50 features: Skip SHAP pre-screen entirely. Run QAOA directly on all features.
- 50-100 features: Set `top_k=20-25`. Requires ~8 GB RAM for simulation.
- 100+ features: Use the two-stage pipeline as-is.

**Different target feature count:**
- Change `target_features` in the QAOA call.
- The penalty term automatically adjusts since it uses `target_features` (k) in the formula.

**Non-fraud use cases:**
The pipeline is domain-agnostic. Replace `bad_d` with any binary target: churn, conversion, default, etc.

---

## Part 4: Numerical Example (4 Features, Select 2)

Let's walk through the entire QAOA pipeline with a tiny example you can verify by hand.

### Setup

We have 4 features (A, B, C, D) and want to select exactly **k = 2**.

**Mutual Information (relevance) with target:**
```
MI(A) = 0.8    (very predictive)
MI(B) = 0.6    (moderately predictive)
MI(C) = 0.3    (weakly predictive)
MI(D) = 0.1    (barely predictive)
```

**Absolute correlation matrix (redundancy):**
```
        A     B     C     D
  A  [ 1.0   0.9   0.2   0.1 ]
  B  [ 0.9   1.0   0.3   0.2 ]
  C  [ 0.2   0.3   1.0   0.1 ]
  D  [ 0.1   0.2   0.1   1.0 ]
```

Note: A and B are highly correlated (0.9) — selecting both is wasteful.

**Hyperparameters:** α = 1.0, β = 0.5, penalty = 2.0, k = 2.

### Step 1: Build Q Matrix (4×4)

**Diagonal entries:** `Q[i,i] = −α·MI(i) + penalty·(1 − 2k)`

Since penalty = 2.0 and k = 2: the constraint term = 2.0 × (1 − 4) = −6.0

```
Q[0,0] = −1.0×0.8 + (−6.0) = −6.8     (feature A)
Q[1,1] = −1.0×0.6 + (−6.0) = −6.6     (feature B)
Q[2,2] = −1.0×0.3 + (−6.0) = −6.3     (feature C)
Q[3,3] = −1.0×0.1 + (−6.0) = −6.1     (feature D)
```

More negative diagonal → more incentive to select that feature.

**Off-diagonal entries (i < j):** `Q[i,j] = β·|corr(i,j)| + 2·penalty`

```
Q[0,1] = 0.5×0.9 + 4.0 = 4.45    (A-B pair: high redundancy penalty!)
Q[0,2] = 0.5×0.2 + 4.0 = 4.10
Q[0,3] = 0.5×0.1 + 4.0 = 4.05
Q[1,2] = 0.5×0.3 + 4.0 = 4.15
Q[1,3] = 0.5×0.2 + 4.0 = 4.10
Q[2,3] = 0.5×0.1 + 4.0 = 4.05
```

**Full Q matrix:**
```
Q = [ −6.80   4.45   4.10   4.05 ]
    [  0.00  −6.60   4.15   4.10 ]
    [  0.00   0.00  −6.30   4.05 ]
    [  0.00   0.00   0.00  −6.10 ]
```
(Upper triangular; lower triangle is zero by convention since QUBO uses i ≤ j.)

### Step 2: Evaluate All 2⁴ = 16 Possible Subsets

The QUBO cost for a bitstring x = (x_A, x_B, x_C, x_D) is:

```
f(x) = Σᵢ Q[i,i]·xᵢ + Σᵢ<ⱼ Q[i,j]·xᵢ·xⱼ
```

Let's compute f(x) for all subsets of size 2 (the interesting ones):

**x = (1,1,0,0) → Select A, B:**
```
f = Q[0,0]·1 + Q[1,1]·1 + Q[0,1]·1·1
  = −6.80 + (−6.60) + 4.45
  = −8.95
```

**x = (1,0,1,0) → Select A, C:**
```
f = −6.80 + (−6.30) + 4.10
  = −9.00  ← LOWEST (BEST)!
```

**x = (1,0,0,1) → Select A, D:**
```
f = −6.80 + (−6.10) + 4.05
  = −8.85
```

**x = (0,1,1,0) → Select B, C:**
```
f = −6.60 + (−6.30) + 4.15
  = −8.75
```

**x = (0,1,0,1) → Select B, D:**
```
f = −6.60 + (−6.10) + 4.10
  = −8.60
```

**x = (0,0,1,1) → Select C, D:**
```
f = −6.30 + (−6.10) + 4.05
  = −8.35
```

**Ranking (lower is better):**
```
1. {A, C} → −9.00  ✅ OPTIMAL
2. {A, B} → −8.95
3. {A, D} → −8.85
4. {B, C} → −8.75
5. {B, D} → −8.60
6. {C, D} → −8.35
```

**Interpretation:** Even though A and B are the two most predictive features individually, the QUBO correctly identifies {A, C} as optimal because A and B are 90% correlated — selecting both adds redundancy. Feature C (MI=0.3) provides complementary information to A.

### Step 3: QUBO → Ising Conversion

**h vector (single-qubit biases):**
```
h[i] = −Q[i,i]/2 − Σⱼ Q[i,j]/4   (summing over all j ≠ i in upper triangle)
```

For feature A (i=0):
```
h[0] = −(−6.80)/2 − (4.45 + 4.10 + 4.05)/4
     = 3.40 − 3.15
     = 0.25
```

For feature B (i=1):
```
h[1] = −(−6.60)/2 − (4.45/4 + 4.15/4 + 4.10/4)
     = 3.30 − (1.1125 + 1.0375 + 1.025)
     = 3.30 − 3.175
     = 0.125
```

For feature C (i=2):
```
h[2] = −(−6.30)/2 − (4.10/4 + 4.15/4 + 4.05/4)
     = 3.15 − (1.025 + 1.0375 + 1.0125)
     = 3.15 − 3.075
     = 0.075
```

For feature D (i=3):
```
h[3] = −(−6.10)/2 − (4.05/4 + 4.10/4 + 4.05/4)
     = 3.05 − (1.0125 + 1.025 + 1.0125)
     = 3.05 − 3.05
     = 0.00
```

**J matrix (two-qubit couplings):**
```
J[i,j] = Q[i,j] / 4
```
```
J[0,1] = 4.45/4 = 1.1125
J[0,2] = 4.10/4 = 1.025
J[0,3] = 4.05/4 = 1.0125
J[1,2] = 4.15/4 = 1.0375
J[1,3] = 4.10/4 = 1.025
J[2,3] = 4.05/4 = 1.0125
```

### Step 4: PennyLane Hamiltonian

The Ising Hamiltonian becomes:
```
H = 0.25·Z₀ + 0.125·Z₁ + 0.075·Z₂ + 0.00·Z₃
  + 1.1125·Z₀Z₁ + 1.025·Z₀Z₂ + 1.0125·Z₀Z₃
  + 1.0375·Z₁Z₂ + 1.025·Z₁Z₃ + 1.0125·Z₂Z₃
```

In PennyLane code:
```python
coeffs = [0.25, 0.125, 0.075,         # Single-qubit
          1.1125, 1.025, 1.0125,       # Two-qubit (row 0)
          1.0375, 1.025, 1.0125]       # Two-qubit (rows 1,2)

obs = [qml.PauliZ(0), qml.PauliZ(1), qml.PauliZ(2),
       qml.PauliZ(0) @ qml.PauliZ(1), qml.PauliZ(0) @ qml.PauliZ(2), qml.PauliZ(0) @ qml.PauliZ(3),
       qml.PauliZ(1) @ qml.PauliZ(2), qml.PauliZ(1) @ qml.PauliZ(3), qml.PauliZ(2) @ qml.PauliZ(3)]

H_cost = qml.Hamiltonian(coeffs, obs)
```

### Step 5: QAOA Execution (Conceptual)

**Initialization:** 4 Hadamard gates create:
```
|ψ₀⟩ = |++++⟩ = (1/4) Σ_{x∈{0,1}⁴} |x⟩
```
All 16 bitstrings have equal amplitude 1/4 (probability 1/16 each).

**After optimization (60 Adam steps, depth p=2):**
The circuit learns angles γ₁, γ₂, α₁, α₂ such that the bitstring `1010` (features A and C selected) gets the highest probability amplitude.

**Measurement outcome:**
```
P(1010) ≈ 0.28   ← Highest! This is the answer.
P(1100) ≈ 0.22   ← Second best (A, B)
P(1001) ≈ 0.15   ← Third (A, D)
...other states share remaining probability
```

The algorithm reads bitstring `1010` → selects features A and C → matches our brute-force optimal from Step 2.

### Step 6: Why Quantum?

For 4 features, brute force checks 2⁴ = 16 states — trivial. But for 18 features, brute force means 2¹⁸ = 262,144 evaluations. For 30 features, it's over 1 billion. QAOA explores this exponential space in superposition, converging to near-optimal solutions with polynomial-time classical optimization of only 2p parameters.

---

## Appendix: Quick-Reference Formula Sheet

```
QUBO diagonal:     Q[i,i] = −α·MI(i) − 2·penalty·k + penalty
QUBO off-diagonal: Q[i,j] = β·|corr(i,j)| + 2·penalty        (i < j)

Ising bias:        h[i] = −Q[i,i]/2 − Σ_{j≠i} Q[i,j]/4
Ising coupling:    J[i,j] = Q[i,j]/4                           (i < j)

QUBO → spin:       xᵢ = (1 − Zᵢ)/2

QAOA state:        |ψ⟩ = Πₚ [e^{−iα_p H_M} · e^{−iγ_p H_C}] |+⟩ⁿ

Gini coefficient:  Gini = (2·AUC − 1) × 100
Catch rate @k%:    (frauds in top k%) / (total frauds) × 100
```
