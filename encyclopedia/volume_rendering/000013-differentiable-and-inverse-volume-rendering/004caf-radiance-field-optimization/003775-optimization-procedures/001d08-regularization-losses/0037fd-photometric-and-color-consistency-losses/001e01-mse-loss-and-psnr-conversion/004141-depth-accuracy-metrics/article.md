# Depth Accuracy Metrics

Also called: AbsRel, Depth Evaluation Metrics, Depth error metrics, RMSE, RMSE log, SqRel

## Overview
Depth Accuracy Metrics quantify the fidelity of predicted depth maps against ground truth data in radiance field optimization pipelines, distinguishing between per-image accuracy and global scene consistency [S1]. Unlike RGB-focused metrics such as MSE or PSNR, these metrics evaluate geometric correctness by computing error statistics over pixels with valid ground truth depth [S1, S2].

## How it works
**Inputs and Outputs:**
- **Inputs:**
  - $d$: Ground truth depth map.
  - $\hat{d}$: Predicted depth map.
  - $K$: Number of pixels $k$ where ground truth depth $d_k$ is valid (non-zero and within bounds) [S1].
- **Outputs:** Scalar values for RMSE, RMSE log, AbsRel, and SqRel.

**Data Structures:**
- Depth maps are treated as 2D arrays of floating-point values.
- Validity mask is implicit: only pixels where $d_k > 0$ and $d_k$ is not NaN are included in the summation [S1].

**Formulas:**
Let $k$ index pixels $1, \dots, K$ with valid ground truth.

1. **Root Mean Squared Error (RMSE):**
   $$RMSE(d, \hat{d}) = \sqrt{\frac{1}{K} \sum_{k=1}^{K} (d_k - \hat{d}_k)^2}$$ [S1]

2. **Root Mean Squared Error on Log Scale (RMSE log):**
   $$RMSE_{log}(d, \hat{d}) = \sqrt{\frac{1}{K} \sum_{k=1}^{K} (\log d_k - \log \hat{d}_k)^2}$$ [S1]

3. **Absolute Relative Error (AbsRel):**
   $$AbsRel(d, \hat{d}) = \frac{1}{K} \sum_{k=1}^{K} \frac{|d_k - \hat{d}_k|}{d_k}$$ [S1]

4. **Squared Relative Error (SqRel):**
   $$SqRel(d, \hat{d}) = \frac{1}{K} \sum_{k=1}^{K} \frac{(d_k - \hat{d}_k)^2}{d_k}$$ [S1]

**Parameters:**
- $K$: Count of valid pixels. If $K=0$, metrics are undefined [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_depth_metrics(d, d_hat):
    """
    Compute depth accuracy metrics.
    d: 2D array of ground truth depth.
    d_hat: 2D array of predicted depth.
    Returns: dict with RMSE, RMSE_log, AbsRel, SqRel.
    """
    # Identify valid pixels: ground truth must be positive and finite
    valid_mask = (d > 0) & (d == d) & (d_hat == d_hat)
    
    # Extract valid values
    d_valid = d[valid_mask]
    d_hat_valid = d_hat[valid_mask]
    
    K = len(d_valid)
    if K == 0:
        return {
            "RMSE": float('nan'),
            "RMSE_log": float('nan'),
            "AbsRel": float('nan'),
            "SqRel": float('nan')
        }
    
    # RMSE
    diff_sq = (d_valid - d_hat_valid) ** 2
    rmse = math.sqrt(sum(diff_sq) / K)
    
    # RMSE log
    # Handle potential log(0) or negative values if d_hat is invalid, 
    # though valid_mask should ensure d > 0. d_hat should ideally be > 0 for log.
    # If d_hat <= 0, log is undefined. We assume d_hat is positive for valid pixels.
    log_diff_sq = (math.log(d_valid) - math.log(d_hat_valid)) ** 2
    rmse_log = math.sqrt(sum(log_diff_sq) / K)
    
    # AbsRel
    abs_rel_sum = sum(abs(d_valid - d_hat_valid) / d_valid)
    abs_rel = abs_rel_sum / K
    
    # SqRel
    sq_rel_sum = sum((d_valid - d_hat_valid) ** 2 / d_valid)
    sq_rel = sq_rel_sum / K
    
    return {
        "RMSE": rmse,
        "RMSE_log": rmse_log,
        "AbsRel": abs_rel,
        "SqRel": sq_rel
    }
```

**Edge Cases and Pitfalls:**
- **Invalid Depth:** Pixels with $d_k \le 0$ or NaN are excluded from $K$ [S1].
- **Log Scale:** $RMSE_{log}$ requires $d_k > 0$ and $\hat{d}_k > 0$. If $\hat{d}_k \le 0$, the metric is undefined [S1].
- **Division by Zero:** $AbsRel$ and $SqRel$ divide by $d_k$. Since $d_k > 0$ for valid pixels, this is safe [S1].

## Subtopics

### [MDE evaluation metrics](12301-mde-evaluation-metrics.md)
Extends these metrics to assess multi-view consistency and monocular depth estimation module fidelity within radiance field pipelines.

## Operating characteristics
- **Time Complexity:** $O(K)$, where $K$ is the number of valid pixels. Linear scan of the depth map.
- **Memory Complexity:** $O(K)$ to store valid depth values during computation.
- **Strengths:** 
  - RMSE provides a standard geometric error measure.
  - RMSE log is robust to scale differences and emphasizes relative errors in distant objects.
  - AbsRel and SqRel are scale-invariant and penalize large relative errors more heavily [S1, S2].
- **Limitations:** 
  - Sensitive to outliers in RMSE.
  - Requires valid ground truth, which may be sparse in real-world scenes [S1].
- **When to Use:** Evaluating depth prior quality in NeRF variants and monocular depth estimation [S2].

## Background
These metrics are standard in computer vision for depth estimation evaluation [S2]. They are used in recent works like DäRF and Depth-Guided NeRF to compare depth prior quality and geometric accuracy [S1, S2]. The specific set (RMSE, RMSE log, AbsRel, SqRel) is commonly reported together to provide a comprehensive view of depth accuracy [S1].

## Sources
[S1] Depth-Guided NeRF Training via Earth Mover’s Distance, https://link.springer.com/chapter/10.1007/978-3-031-73039-9_1, fetched 2026-10-27
[S2] Digging into Depth Priors for Outdoor Neural Radiance Fields, https://doi.org/10.1145/3581783.3612306, fetched 2026-10-30
