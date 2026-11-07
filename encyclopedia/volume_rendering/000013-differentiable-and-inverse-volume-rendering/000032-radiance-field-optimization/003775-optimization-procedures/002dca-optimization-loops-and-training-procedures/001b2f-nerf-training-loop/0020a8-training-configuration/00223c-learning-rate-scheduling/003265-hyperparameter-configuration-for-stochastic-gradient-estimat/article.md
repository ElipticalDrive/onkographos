# Hyperparameter Configuration for Stochastic Gradient Estimation


## Overview
This article defines the hyperparameter configuration required to optimize differentiable rendering methods using first-order (OurG, OurH) and second-order (OurHVP, OurHVPA) stochastic gradient estimators [S1]. It solves the problem of selecting optimal sampling, learning rate, and trust region parameters for inverse rendering tasks such as caustic optimization, shape reconstruction, and texture mapping [S1]. What sets this configuration apart from standard NeRF training is the explicit use of **Sigma annealing** for distribution bandwidth control and **antithetic sampling** to reduce variance, alongside distinct parameter sets for first-order versus second-order optimization loops [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Task identifier (e.g., QUAD, BOX2, BOX10, MUG, SHAD, BUNNY, TEXTURE, CNN, CNN5) [S1].
*   Method identifier (OurG, OurH, OurHVP, OurHVPA, or FR22) [S1].
*   Current optimization iteration $t$ [S1].

**Outputs:**
*   Current learning rate $\eta_t$ (for first-order methods) [S1].
*   Current trust region radius $\Delta_t$ (for second-order methods) [S1].
*   Current distribution bandwidth $\sigma_t$ [S1].
*   Sample count configuration (doubled via antithetic sampling) [S1].

### Data Structures
The hyperparameters are stored in a lookup table indexed by Method and Task. The structure contains the following fields:
*   `samples`: Integer or float, base number of samples per iteration.
*   `sigma_start`: Float, initial bandwidth $\sigma$ for Sigma annealing.
*   `sigma_end`: Float, final bandwidth $\sigma$ for Sigma annealing.
*   `learning_rate`: Float, initial learning rate (first-order only).
*   `trust_region`: Float, initial search bound (second-order only).
*   `line_search_iter`: Integer, number of line search iterations (second-order only).
*   `line_search_tol`: Float, convergence tolerance for line search (second-order only).
*   `recompute`: Integer, iterations between line search re-estimations (second-order only).

### Numbered Steps

1.  **Retrieve Base Parameters**: Look up the hyperparameter set for the specified Method and Task from the configuration table [S1].
2.  **Calculate Current Bandwidth ($\sigma_t$)**: Perform linear Sigma annealing from `sigma_start` to `sigma_end` over the total optimization duration $T$.
    $$ \sigma_t = \sigma_{start} + (\sigma_{end} - \sigma_{start}) \cdot \frac{t}{T} $$
    Where $t$ is the current iteration and $T$ is the total number of iterations [S1].
3.  **Determine Sample Count**:
    *   Let $N_{base}$ be the `samples` value from the configuration.
    *   Apply antithetic sampling: The effective sample count is $N = 2 \cdot N_{base}$ [S1].
    *   Note: The source states "The sample size, which is always antithetic and doubles the sample size, applies to our methods and FR22" [S1].
4.  **Configure Gradient Estimator**:
    *   If Method is **OurG** or **OurH** (First-Order):
        *   Set learning rate $\eta = $ `learning_rate` [S1].
        *   Update parameters using $\eta$ and the stochastic gradient estimate [S1].
    *   If Method is **OurH**, **OurHVP**, or **OurHVPA** (Second-Order):
        *   Set trust region radius $\Delta = $ `trust_region` [S1].
        *   Perform line search with `line_search_iter` iterations and tolerance `line_search_tol` [S1].
        *   Check if re-estimation is needed: If current iteration $t$ is a multiple of `recompute`, re-estimate the line search parameters [S1].

### Formulas and Definitions
*   **Sigma Annealing**: Linear schedule for bandwidth $\sigma$. Controls the distribution used for stochastic estimation [S1].
*   **Antithetic Sampling**: For every sample $x$, an antithetic sample $-x$ (or symmetric counterpart) is generated, effectively doubling the sample count for variance reduction [S1].
*   **Trust Region**: Initial search bound for second-order methods. Defines the region within which the quadratic model is trusted [S1].
*   **Recompute**: Frequency (in iterations) at which the line search is re-estimated for second-order methods [S1].

### Parameters and Typical Values
The following table lists the exact hyperparameters for each method and task as provided in the source [S1].

| Method | Task | Samples | Sigma (start) | Sigma (end) | Learning Rate | Trust Region | Line Search Iter | Line Search Tol | Recompute |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **FR22** | QUAD | 4 | 0.5 | 0.5 | - | - | - | - | - |
| | BOX2 | 1 | 0.01 | 0.01 | - | - | - | - | - |
| | BOX10 | 6 | 0.3 | 0.3 | - | - | - | - | - |
| | MUG | 1.5 | 0.01 | 0.01 | - | - | - | - | - |
| | SHAD | 6 | 0.05 | 0.05 | - | - | - | - | - |
| | BUNNY | 0.6 | 0.1 | 0.1 | - | - | - | - | - |
| | TEXTURE | 1 | 0.1 | 0.1 | - | - | - | - | - |
| | CNN | 3 | 0.01 | 0.01 | - | - | - | - | - |
| | CNN5 | 1 | 0.02 | 0.02 | - | - | - | - | - |
| **OurG** | QUAD | 4 | 0.5 | 0.5 | 1e-4 | - | - | - | - |
| | BOX2 | 1 | 0.01 | 0.01 | 0.01 | - | - | - | - |
| | BOX10 | 6 | 0.3 | 0.3 | 1e-4 | - | - | - | - |
| | MUG | 1.5 | 0.01 | 0.01 | 0.01 | - | - | - | - |
| | SHAD | 6 | 0.05 | 0.05 | 1e-4 | - | - | - | - |
| | BUNNY | 0.6 | 0.1 | 0.1 | 0.01 | - | - | - | - |
| | TEXTURE | 1 | 0.1 | 0.1 | 1e-4 | - | - | - | - |
| | CNN | 3 | 0.01 | 0.01 | 0.01 | - | - | - | - |
| | CNN5 | 1 | 0.02 | 0.02 | 1e-4 | - | - | - | - |
| **OurH** | QUAD | 4 | 0.5 | 0.5 | - | 50 | 5 | 1e-3 | 5 |
| | BOX2 | 1 | 0.01 | 0.01 | - | 5 | 5 | 1e-3 | 2 |
| | BOX10 | 6 | 0.3 | 0.3 | - | 10 | 10 | 1e-3 | 10 |
| | MUG | 1.5 | 0.01 | 0.01 | - | 30 | 3 | 1e-3 | 3 |
| | SHAD | 6 | 0.05 | 0.05 | - | 10 | 1 | 1e-3 | 1 |
| | BUNNY | 0.6 | 0.1 | 0.1 | - | 10 | 1 | 1e-3 | 1 |
| | TEXTURE | 1 | 0.1 | 0.1 | - | 5 | 10 | 1e-3 | 10 |
| | CNN | 3 | 0.01 | 0.01 | - | 30 | 5 | 1e-3 | 5 |
| | CNN5 | 1 | 0.02 | 0.02 | - | 20 | 2 | 1e-3 | 2 |
| **OurHVP** | QUAD | 4 | 0.5 | 0.5 | - | 50 | 5 | 1e-3 | 5 |
| | BOX2 | 1 | 0.01 | 0.01 | - | 2 | 3 | 1e-3 | 10 |
| | BOX10 | 6 | 0.3 | 0.3 | - | 10 | 10 | 1e-3 | 10 |
| | MUG | 1.5 | 0.01 | 0.01 | - | 30 | 3 | 1e-3 | 3 |
| | SHAD | 6 | 0.05 | 0.05 | - | 10 | 1 | 1e-3 | 1 |
| | BUNNY | 0.6 | 0.1 | 0.1 | - | 10 | 1 | 1e-3 | 1 |
| | TEXTURE | 1 | 0.1 | 0.1 | - | 5 | 10 | 1e-3 | 10 |
| | CNN | 3 | 0.01 | 0.01 | - | 30 | 5 | 1e-3 | 5 |
| | CNN5 | 1 | 0.02 | 0.02 | - | 20 | 2 | 1e-3 | 2 |
| **OurHVPA** | QUAD | 4 | 0.5 | 0.5 | - | 50 | 5 | 1e-3 | 5 |
| | BOX2 | 1 | 0.01 | 0.01 | - | 2 | 10 | 1e-3 | 10 |
| | BOX10 | 6 | 0.3 | 0.3 | - | 10 | 10 | 1e-3 | 10 |
| | MUG | 1.5 | 0.01 | 0.01 | - | 30 | 3 | 1e-3 | 3 |
| | SHAD | 6 | 0.05 | 0.05 | - | 10 | 1 | 1e-3 | 1 |
| | BUNNY | 0.6 | 0.1 | 0.1 | - | 10 | 1 | 1e-3 | 1 |
| | TEXTURE | 1 | 0.1 | 0.1 | - | 5 | 10 | 1e-3 | 10 |
| | CNN | 3 | 0.01 | 0.01 | - | 30 | 5 | 1e-3 | 5 |
| | CNN5 | 1 | 0.02 | 0.02 | - | 20 | 2 | 1e-3 | 2 |

*Note: The source table [S1] lists "Sigma (end)" for FR22 and OurG as identical to "Sigma (start)" in many cases (e.g., QUAD: 0.5/0.5), implying no annealing or constant bandwidth for those specific entries. For OurH/OurHVP/OurHVPA, "Sigma (end)" values are provided in the source table (e.g., QUAD: 0.05 for OurHVP/HVPA).*

### Edge Cases and Pitfalls
*   **Non-Integer Samples**: Tasks like MUG (1.5) and BUNNY (0.6) use fractional sample counts. The implementation must handle this, likely by rounding or probabilistic sampling, though the source does not specify the exact mechanism [S1].
*   **Constant Sigma**: For FR22 and OurG, `sigma_start` often equals `sigma_end`. The annealing formula still applies but results in a constant $\sigma$ [S1].
*   **Second-Order Recompute**: If `recompute` is 1, the line search is re-estimated every iteration, which may be computationally expensive [S1].
*   **Antithetic Doubling**: The source explicitly states antithetic sampling doubles the sample size. If the base sample count is 0 or negative, this leads to errors. All listed values are positive [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_hyperparameters(method, task):
    """
    Returns the hyperparameter configuration for a given method and task.
    Based on Table 3 from [S1].
    """
    config = {
        "FR22": {
            "QUAD": {"samples": 4, "sigma_start": 0.5, "sigma_end": 0.5},
            "BOX2": {"samples": 1, "sigma_start": 0.01, "sigma_end": 0.01},
            "BOX10": {"samples": 6, "sigma_start": 0.3, "sigma_end": 0.3},
            "MUG": {"samples": 1.5, "sigma_start": 0.01, "sigma_end": 0.01},
            "SHAD": {"samples": 6, "sigma_start": 0.05, "sigma_end": 0.05},
            "BUNNY": {"samples": 0.6, "sigma_start": 0.1, "sigma_end": 0.1},
            "TEXTURE": {"samples": 1, "sigma_start": 0.1, "sigma_end": 0.1},
            "CNN": {"samples": 3, "sigma_start": 0.01, "sigma_end": 0.01},
            "CNN5": {"samples": 1, "sigma_start": 0.02, "sigma_end": 0.02}
        },
        "OurG": {
            "QUAD": {"samples": 4, "sigma_start": 0.5, "sigma_end": 0.5, "learning_rate": 1e-4},
            "BOX2": {"samples": 1, "sigma_start": 0.01, "sigma_end": 0.01, "learning_rate": 0.01},
            "BOX10": {"samples": 6, "sigma_start": 0.3, "sigma_end": 0.3, "learning_rate": 1e-4},
            "MUG": {"samples": 1.5, "sigma_start": 0.01, "sigma_end": 0.01, "learning_rate": 0.01},
            "SHAD": {"samples": 6, "sigma_start": 0.05, "sigma_end": 0.05, "learning_rate": 1e-4},
            "BUNNY": {"samples": 0.6, "sigma_start": 0.1, "sigma_end": 0.1, "learning_rate": 0.01},
            "TEXTURE": {"samples": 1, "sigma_start": 0.1, "sigma_end": 0.1, "learning_rate": 1e-4},
            "CNN": {"samples": 3, "sigma_start": 0.01, "sigma_end": 0.01, "learning_rate": 0.01},
            "CNN5": {"samples": 1, "sigma_start": 0.02, "sigma_end": 0.02, "learning_rate": 1e-4}
        },
        "OurH": {
            "QUAD": {"samples": 4, "sigma_start": 0.5, "sigma_end": 0.05, "trust_region": 50, "line_search_iter": 5, "line_search_tol": 1e-3, "recompute": 5},
            "BOX2": {"samples": 1, "sigma_start": 0.01, "sigma_end": 0.01, "trust_region": 5, "line_search_iter": 5, "line_search_tol": 1e-3, "recompute": 2},
            "BOX10": {"samples": 6, "sigma_start": 0.3, "sigma_end": 0.1, "trust_region": 10, "line_search_iter": 10, "line_search_tol": 1e-3, "recompute": 10},
            "MUG": {"samples": 1.5, "sigma_start": 0.01, "sigma_end": 0.01, "trust_region": 30, "line_search_iter": 3, "line_search_tol": 1e-3, "recompute": 3},
            "SHAD": {"samples": 6, "sigma_start": 0.05, "sigma_end": 0.05, "trust_region": 10, "line_search_iter": 1, "line_search_tol": 1e-3, "recompute": 1},
            "BUNNY": {"samples": 0.6, "sigma_start": 0.1, "sigma_end": 0.1, "trust_region": 10, "line_search_iter": 1, "line_search_tol": 1e-3, "recompute": 1},
            "TEXTURE": {"samples": 1, "sigma_start": 0.1, "sigma_end": 0.1, "trust_region": 5, "line_search_iter": 10, "line_search_tol": 1e-3, "recompute": 10},
            "CNN": {"samples": 3, "sigma_start": 0.01, "sigma_end": 0.01, "trust_region": 30, "line_search_iter": 5, "line_search_tol": 1e-3, "recompute": 5},
            "CNN5": {"samples": 1, "sigma_start": 0.02, "sigma_end": 0.02, "trust_region": 20, "line_search_iter": 2, "line_search_tol": 1e-3, "recompute": 2}
        },
        "OurHVP": {
            "QUAD": {"samples": 4, "sigma_start": 0.5, "sigma_end": 0.05, "trust_region": 50, "line_search_iter": 5, "line_search_tol": 1e-3, "recompute": 5},
            "BOX2": {"samples": 1, "sigma_start": 0.01, "sigma_end": 0.01, "trust_region": 2, "line_search_iter": 3, "line_search_tol": 1e-3, "recompute": 10},
            "BOX10": {"samples": 6, "sigma_start": 0.3, "sigma_end": 0.1, "trust_region": 10, "line_search_iter": 10, "line_search_tol": 1e-3, "recompute": 10},
            "MUG": {"samples": 1.5, "sigma_start": 0.01, "sigma_end": 0.01, "trust_region": 30, "line_search_iter": 3, "line_search_tol": 1e-3, "recompute": 3},
            "SHAD": {"samples": 6, "sigma_start": 0.05, "sigma_end": 0.05, "trust_region": 10, "line_search_iter": 1, "line_search_tol": 1e-3, "recompute": 1},
            "BUNNY": {"samples": 0.6, "sigma_start": 0.1, "sigma_end": 0.1, "trust_region": 10, "line_search_iter": 1, "line_search_tol": 1e-3, "recompute": 1},
            "TEXTURE": {"samples": 1, "sigma_start": 0.1, "sigma_end": 0.1, "trust_region": 5, "line_search_iter": 10, "line_search_tol": 1e-3, "recompute": 10},
            "CNN": {"samples": 3, "sigma_start": 0.01, "sigma_end": 0.01, "trust_region": 30, "line_search_iter": 5, "line_search_tol": 1e-3, "recompute": 5},
            "CNN5": {"samples": 1, "sigma_start": 0.02, "sigma_end": 0.02, "trust_region": 20, "line_search_iter": 2, "line_search_tol": 1e-3, "recompute": 2}
        },
        "OurHVPA": {
            "QUAD": {"samples": 4, "sigma_start": 0.5, "sigma_end": 0.05, "trust_region": 50, "line_search_iter": 5, "line_search_tol": 1e-3, "recompute": 5},
            "BOX2": {"samples": 1, "sigma_start": 0.01, "sigma_end": 0.01, "trust_region": 2, "line_search_iter": 10, "line_search_tol": 1e-3, "recompute": 10},
            "BOX10": {"samples": 6, "sigma_start": 0.3, "sigma_end": 0.1, "trust_region": 10, "line_search_iter": 10, "line_search_tol": 1e-3, "recompute": 10},
            "MUG": {"samples": 1.5, "sigma_start": 0.01, "sigma_end": 0.01, "trust_region": 30, "line_search_iter": 3, "line_search_tol": 1e-3, "recompute": 3},
            "SHAD": {"samples": 6, "sigma_start": 0.05, "sigma_end": 0.05, "trust_region": 10, "line_search_iter": 1, "line_search_tol": 1e-3, "recompute": 1},
            "BUNNY": {"samples": 0.6, "sigma_start": 0.1, "sigma_end": 0.1, "trust_region": 10, "line_search_iter": 1, "line_search_tol": 1e-3, "recompute": 1},
            "TEXTURE": {"samples": 1, "sigma_start": 0.1, "sigma_end": 0.1, "trust_region": 5, "line_search_iter": 10, "line_search_tol": 1e-3, "recompute": 10},
            "CNN": {"samples": 3, "sigma_start": 0.01, "sigma_end": 0.01, "trust_region": 30, "line_search_iter": 5, "line_search_tol": 1e-3, "recompute": 5},
            "CNN5": {"samples": 1, "sigma_start": 0.02, "sigma_end": 0.02, "trust_region": 20, "line_search_iter": 2, "line_search_tol": 1e-3, "recompute": 2}
        }
    }
    return config[method][task]

def update_hyperparameters(method, task, t, T):
    """
    Computes current hyperparameter values for iteration t.
    """
    params = get_hyperparameters(method, task)
    
    # 1. Sigma Annealing
    sigma_t = params["sigma_start"] + (params["sigma_end"] - params["sigma_start"]) * (t / T)
    
    # 2. Sample Count (Antithetic)
    base_samples = params["samples"]
    effective_samples = 2 * base_samples
    
    # 3. Method-Specific Parameters
    if method in ["OurG", "OurH"]:
        # First-order
        learning_rate = params["learning_rate"]
        trust_region = None
        line_search_iter = None
        line_search_tol = None
        recompute = None
    elif method in ["OurH", "OurHVP", "OurHVPA"]:
        # Second-order
        learning_rate = None
        trust_region = params["trust_region"]
        line_search_iter = params["line_search_iter"]
        line_search_tol = params["line_search_tol"]
        recompute = params["recompute"]
    else:
        # FR22 or others
        learning_rate = None
        trust_region = None
        line_search_iter = None
        line_search_tol = None
        recompute = None
        
    return {
        "sigma": sigma_t,
        "samples": effective_samples,
        "learning_rate": learning_rate,
        "trust_region": trust_region,
        "line_search_iter": line_search_iter,
        "line_search_tol": line_search_tol,
        "recompute": recompute
    }
```

## Operating characteristics
*   **Time Complexity**: Not given in the sources. The source mentions that CMA-ES does not converge on the QUAD task, implying the stochastic methods are more efficient or stable for certain inverse rendering problems [S1].
*   **Memory Complexity**: Not given in the sources.
*   **Measured Performance**: The methods (OurG, OurH, OurHVP, OurHVPA) are applied to tasks including QUAD, BOX2, BOX10, MUG, SHAD, BUNNY, TEXTURE, CNN, and CNN5 [S1]. The source notes that for the two-bounce experiment (Stanford bunny observed through two mirrors), OurHVPA successfully optimizes rotation and translation [S1].
*   **Strengths**: The use of antithetic sampling and Sigma annealing provides controlled variance reduction and distribution adaptation [S1]. Second-order methods (OurH, OurHVP, OurHVPA) use trust regions and line searches for robust convergence [S1].
*   **Limitations**: The hyperparameters are task-specific and must be "optimally chosen" for each task, as shown in Table 3 [S1]. This implies a lack of universal hyperparameters.
*   **When to use**: Use when optimizing differentiable rendering methods with first-order or second-order stochastic gradient estimators, particularly for inverse rendering tasks like caustic optimization or shape reconstruction [S1].

## Background
*   **Origin**: The method is described in "Stochastic Gradient Estimation for Higher-Order Differentiable Rendering" [S1].
*   **History**: The paper introduces first-order (OurG, OurH) and second-order (OurHVP, OurHVPA) estimators for differentiable rendering [S1].
*   **Usage**: Applied to inverse rendering tasks such as caustic optimization (QUAD), shape reconstruction (BOX2, BOX10, BUNNY), and texture mapping (TEXTURE, CNN, CNN5) [S1].

## Sources
[S1] Stochastic Gradient Estimation for Higher-Order Differentiable Rendering, https://doi.org/10.1109/iccv51701.2025.02618, fetched 2026-10-29
