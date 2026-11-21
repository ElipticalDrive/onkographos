# Split Conformal Prediction Calibration

Also called: Conformal Calibration, UA-NeRF Calibration

## Overview
Split Conformal Prediction Calibration is a post-hoc calibration method that yields provable, distribution-free per-ray coverage guarantees at any user-specified confidence level for uncertainty-aware volume rendering [S1, unofficial]. It addresses miscalibrated uncertainty estimates in neural radiance fields by using a held-out calibration set to adjust prediction intervals, ensuring empirical coverage matches the nominal confidence level [S1, unofficial]. Unlike standard uncertainty quantification which may produce over- or under-confident predictions, this method provides marginal coverage guarantees without assuming a specific data distribution [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   A trained uncertainty-aware NeRF model that outputs a predicted value $\hat{c}(r)$ and total uncertainty $U_{total}(r)$ for each ray $r$ [S1, unofficial].
    *   A calibration dataset $D_{cal}$ consisting of rays from held-out scenes (plants) entirely excluded from the training process [S1, unofficial].
    *   A significance level $\alpha$ (e.g., 0.10) [S1, unofficial].
*   **Outputs:**
    *   A conformal quantile $\hat{q}$ [S1, unofficial].
    *   Calibrated prediction intervals for test rays: $[\hat{c}(r) - \hat{q}\sqrt{U_{total}(r)}, \hat{c}(r) + \hat{q}\sqrt{U_{total}(r)}]$ [S1, unofficial].
    *   A boolean mask for density values to be zeroed prior to mesh extraction [S1, unofficial].

**Data Structures:**
*   **Calibration Set ($D_{cal}$):** A collection of rays $\{(r_j, c_j)\}_{j=1}^{n_{cal}}$ where $r_j$ is the ray geometry and $c_j$ is the ground truth observation [S1, unofficial].
*   **Non-Conformity Scores ($s_j$):** A scalar value for each calibration ray representing the normalized residual [S1, unofficial].
*   **Conformal Quantile ($\hat{q}$):** A single scalar value derived from the sorted scores [S1, unofficial].

**Parameters:**
*   **Significance Level ($\alpha$):** Typically 0.10 [S1, unofficial].
*   **Epsilon ($\varepsilon$):** $10^{-6}$, used to prevent division by zero in score calculation; results are stable for $\varepsilon \in [10^{-8}, 10^{-4}]$ [S1, unofficial].
*   **Density Zeroing Threshold:** $1.5 \hat{q}$, used to mask unreliable regions prior to mesh extraction [S1, unofficial].

**Steps:**
1.  **Construct Calibration Set:** Select a set of scenes (plants) that are completely disjoint from the training set to preserve exchangeability between calibration and test residuals [S1, unofficial]. Do not use held-out views of the same scenes, as view-correlated residuals violate exchangeability [S1, unofficial].
2.  **Compute Non-Conformity Scores:** For each ray $j$ in the calibration set, compute the score $s_j = \frac{\|\hat{c}(r_j) - c_j\|}{\sqrt{U_{total}(r_j) + \varepsilon}}$ [S1, unofficial].
3.  **Calculate Conformal Quantile:** Sort the scores $\{s_j\}_{j=1}^{n_{cal}}$ in ascending order and extract the empirical order statistic at index $k = \lceil (n_{cal} + 1)(1 - \alpha) \rceil$ to obtain $\hat{q}$ [S1, unofficial].
4.  **Apply Calibration:** For any test ray $r$, construct the prediction interval $\hat{c}(r) \pm \hat{q} \sqrt{U_{total}(r)}$, which provides marginal coverage $\ge 1 - \alpha$ [S1, unofficial].
5.  **Masking for Extraction:** Prior to mesh extraction, zero density values where $U_{total} > 1.5 \hat{q}$ to remove regions with excessive uncertainty [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def split_conformal_calibration(
    model, 
    calibration_rays, 
    calibration_ground_truth, 
    alpha=0.10, 
    epsilon=1e-6
):
    """
    Computes the conformal quantile for calibration.
    
    Args:
        model: Uncertainty-aware NeRF model with predict(r) -> (pred, uncertainty).
        calibration_rays: List of ray objects or tensors for calibration set.
        calibration_ground_truth: Array of ground truth values corresponding to rays.
        alpha: Significance level (e.g., 0.10).
        epsilon: Small constant for numerical stability.
        
    Returns:
        q_hat: The conformal quantile (empirical order statistic).
    """
    n_cal = len(calibration_rays)
    scores = np.zeros(n_cal)
    
    for j in range(n_cal):
        r_j = calibration_rays[j]
        c_j = calibration_ground_truth[j]
        c_hat, u_total = model.predict(r_j)
        residual_norm = np.linalg.norm(c_hat - c_j)
        scores[j] = residual_norm / np.sqrt(u_total + epsilon)
        
    sorted_scores = np.sort(scores)
    k = int(np.ceil((n_cal + 1) * (1 - alpha)))
    if k > n_cal:
        k = n_cal
    q_hat = sorted_scores[k - 1]
    return q_hat

def apply_calibration(model, test_rays, q_hat):
    """
    Applies the conformal quantile to generate prediction intervals.
    
    Args:
        model: Uncertainty-aware NeRF model.
        test_rays: List of test ray objects.
        q_hat: The conformal quantile computed from calibration.
        
    Returns:
        intervals: List of tuples (lower_bound, upper_bound) for each ray.
        masks: Boolean array indicating if density should be zeroed (U_total > 1.5 * q_hat).
    """
    intervals = []
    masks = []
    for r in test_rays:
        c_hat, u_total = model.predict(r)
        margin = q_hat * np.sqrt(u_total)
        intervals.append((c_hat - margin, c_hat + margin))
        masks.append(u_total > 1.5 * q_hat)
    return intervals, masks
```

**Edge Cases and Pitfalls:**
*   **Exchangeability Violation:** Using held-out views of the same scene for calibration violates exchangeability due to view-correlated residuals, leading to invalid coverage guarantees [S1, unofficial].
*   **Marginal vs. Conditional Coverage:** The guarantee is marginal (holds in expectation over the joint distribution of plants and rays), not conditional for specific anatomical regions or individual plants; empirical conditional coverage may deviate from the target [S1, unofficial].
*   **Small Calibration Sets:** Conditional conformal methods require larger per-region calibration sets than are available from current sparse-view benchmarks [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Evaluated on Pheno4D, ROSE-X, and UA-Field datasets [S1, unofficial]. At sparsity level $N=30$, UA-PlantNeRF achieved PSNR $28.7 \pm 0.6$ dB and SSIM $0.89 \pm 0.01$, outperforming baselines ($p < 0.05$) [S1, unofficial]. Conformal calibration achieved 89.7% empirical coverage at the 90% nominal target, reducing ECE from 0.143 to 0.041 [S1, unofficial].
*   **Strengths:** Provides distribution-free, provable coverage guarantees [S1, unofficial].
*   **Limitations:** Only guarantees marginal coverage; requires a sufficiently large, exchangeable calibration set; does not provide per-region conditional guarantees [S1, unofficial].

## Background
Split Conformal Prediction Calibration was introduced in the context of uncertainty-aware 3D plant reconstruction from sparse video frames using Neural Radiance Fields (UA-PlantNeRF) [S1, unofficial]. The method leverages conformal prediction principles to address uncalibrated uncertainty in deep learning-based volume rendering for high-throughput plant phenotyping applications [S1, unofficial].

## Sources
[S1] Uncertainty-Aware 3D Plant Reconstruction from Sparse Video Frames Using Neural Radiance Fields, https://doi.org/10.21203/rs.3.rs-9904108/v1, fetched 2026-10-26 (unofficial)
