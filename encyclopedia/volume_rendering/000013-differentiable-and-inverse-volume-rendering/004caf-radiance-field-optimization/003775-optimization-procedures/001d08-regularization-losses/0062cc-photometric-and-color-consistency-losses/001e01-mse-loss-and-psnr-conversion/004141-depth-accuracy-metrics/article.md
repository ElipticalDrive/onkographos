# Depth Accuracy Metrics

Also called: AbsRel, RMSE

## Overview
Depth Accuracy Metrics evaluate the geometric fidelity of predicted depth maps against ground truth data in radiance field optimization pipelines [S1, S2, S3]. They quantify scale- and translation-invariant errors on valid depth pixels [S1]. They are used to evaluate depth prior quality and monocular depth estimation outputs in NeRF pipelines [S2, S3].

## How it works
**Inputs and Outputs:**
- **Inputs:** Ground truth depth map $d$, predicted depth map $\hat{d}$, and a binary validity mask identifying pixels $k=1,...,K$ with valid ground truth depth [S2].
- **Outputs:** Scalar error values for RMSE, RMSE log, AbsRel, and SqRel [S2].

**Data Structures:**
- Depth maps are represented as 2D arrays of floating-point values [G].
- The validity mask is a boolean array of the same dimensions, where `True` indicates a pixel with valid ground truth [S2].

**Steps:**
1. **Preprocessing:** Normalize depth to a consistent range if required by the dataset (e.g., [0, 1] for H3DS) [S1].
2. **Masking:** Filter to pixels $k=1,...,K$ with valid ground truth depth [S2]. For specific datasets, mask both output and ground truth to the non-empty region predicted by the baseline [S1].
3. **Computation:** Calculate the selected metric using the formulas below for all $K$ valid pixels [S2].

**Formulas:**
Let $K$ be the number of pixels with valid ground truth depth [S2].

1. **Root Mean Squared Error (RMSE):**
   $$RMSE(d, \hat{d}) = \sqrt{\frac{1}{K} \sum_{k=1,...,K} (d_k - \hat{d}_k)^2}$$
   [S2]

2. **Log RMSE (RMSE log):**
   $$RMSE_{log}(d, \hat{d}) = \sqrt{\frac{1}{K} \sum_{k=1,...,K} (\log d_k - \log \hat{d}_k)^2}$$
   [S2]

3. **Absolute Relative Error (AbsRel):**
   $$AbsRel(d, \hat{d}) = \frac{1}{K} \sum_{k=1,...,K} \frac{|d_k - \hat{d}_k|}{d_k}$$
   [S2]

4. **Squared Relative Error (SqRel):**
   $$SqRel(d, \hat{d}) = \frac{1}{K} \sum_{k=1,...,K} \frac{(d_k - \hat{d}_k)^2}{d_k}$$
   [S2]

**Parameters:**
- $K$: Number of pixels with valid ground truth depth [S2].

**Edge Cases and Pitfalls:**
- **Geometry Collapse:** In methods like HeadNeRF, geometry may collapse to a plane in front of the subject, requiring careful mask handling to avoid misleading error scores [S1].
- **Invalid Pixels:** Errors must only be computed on valid depth pixels to avoid skewing results [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_depth_metrics(gt_depth: np.ndarray, pred_depth: np.ndarray, valid_mask: np.ndarray) -> dict:
    """
    Compute depth accuracy metrics for valid pixels.
    
    Args:
        gt_depth: Ground truth depth map (H, W).
        pred_depth: Predicted depth map (H, W).
        valid_mask: Boolean mask of valid pixels (H, W).
        
    Returns:
        Dictionary containing RMSE, RMSE_log, AbsRel, SqRel.
    """
    # Extract valid pixels
    gt_valid = gt_depth[valid_mask]
    pred_valid = pred_depth[valid_mask]
    
    K = len(gt_valid)
    if K == 0:
        return {"RMSE": np.nan, "RMSE_log": np.nan, "AbsRel": np.nan, "SqRel": np.nan}
    
    # RMSE
    rmse = np.sqrt(np.mean((gt_valid - pred_valid) ** 2))
    
    # RMSE log
    rmse_log = np.sqrt(np.mean((np.log(gt_valid) - np.log(pred_valid)) ** 2))
    
    # AbsRel
    abs_rel = np.mean(np.abs(gt_valid - pred_valid) / gt_valid)
    
    # SqRel
    sq_rel = np.mean((gt_valid - pred_valid) ** 2 / gt_valid)
    
    return {
        "RMSE": rmse,
        "RMSE_log": rmse_log,
        "AbsRel": abs_rel,
        "SqRel": sq_rel
    }
```

## Subtopics

### [MDE evaluation metrics](12301-mde-evaluation-metrics.md)
A set of quantitative metrics used to evaluate the accuracy of predicted depth maps against ground truth, plus a consistency metric for multi-view alignment. These metrics assess the fidelity of monocular depth estimation (MDE) modules within radiance field optimization pipelines, distinguishing between per-image accuracy and global scene consistency.

## Operating characteristics
- **Time Complexity:** $O(K)$, where $K$ is the number of valid pixels [G].
- **Memory Complexity:** $O(K)$ to store valid pixel values during computation [G].
- **Strengths:** RMSE provides a direct measure of geometric error magnitude [S2]. SqRel emphasizes errors relative to ground truth depth [S2].
- **Limitations:** Metrics are sensitive to outliers in depth predictions [S2]. SqRel heavily penalizes errors in regions with small ground truth depth values [S2].

## Background
Originating from evaluations in neural radiance field (NeRF) research, these metrics were adopted to assess the geometric quality of depth priors and monocular depth estimation modules [S1, S3]. The H3DS dataset introduced specific protocols for scale- and translation-invariant evaluation [S1]. The metrics are used in works such as DäRF and studies on outdoor NeRFs [S2, S3].

## Sources
[S1] Real-Time Radiance Fields for Single-Image Portrait View Synthesis, https://doi.org/10.1145/3592460, fetched 2026-10-27
[S2] Depth-Guided NeRF Training via Earth Mover’s Distance, https://link.springer.com/chapter/10.1007/978-3-031-73039-9_1, fetched 2026-10-27
[S3] Digging into Depth Priors for Outdoor Neural Radiance Fields, https://doi.org/10.1145/3581783.3612306, fetched 2026-10-30
