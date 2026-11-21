# Percentile-based Normalization

Also called: DRR Image Normalization, Grayscale Mapping

## Overview
Percentile-based Normalization scales scalar field data to a target range (typically [0, 1] or [0, 255]) by using robust statistical bounds (percentiles) rather than absolute global extrema [S1, S2, unofficial]. This method solves the problem of outlier-driven contrast loss, where a few extreme values compress the dynamic range of the majority of the data [S2, unofficial]. It differs from [Scaling](2287-scaling.md) and [Scalar Field Normalization](1842-scalar-field-normalization.md) by ignoring the absolute minimum and maximum in favor of percentile thresholds, thereby enhancing visibility of low-intensity features and reducing the impact of noise or artifacts [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A scalar field array (e.g., CT projection or volume data).
- Output: A normalized array in the range [0, 1] (float) or [0, 255] (uint8).

**Data Structures:**
- Input array: $N$-dimensional numeric array (float32/float64).
- Parameters: Lower percentile $p_{lo}$, upper percentile $p_{hi}$.

**Steps:**
1. **Filter Non-Finite Values:** Identify and exclude non-finite values (NaN, Inf) from the dataset [S2, unofficial]. If the resulting valid set is empty, return default bounds (e.g., 0.0, 1.0) [S2, unofficial].
2. **Compute Percentile Bounds:** Calculate the lower bound $min\_val$ as the $p_{lo}$-th percentile and the upper bound $max\_val$ as the $p_{hi}$-th percentile of the valid data [S1, S2, unofficial].
3. **Handle Degenerate Bounds:**
   - If $min\_val$ or $max\_val$ are non-finite, or if $max\_val \le min\_val$, fallback to the global min/max of the valid data [S2, unofficial].
   - If the global min equals the global max (constant data), set $max\_val = min\_val + 1.0$ to avoid division by zero [S2, unofficial].
4. **Clip Data:** Clip the input array values to the range $[min\_val, max\_val]$ [S1, S2, unofficial].
5. **Normalize:** Apply linear scaling:
   $$
   image\_normalized = \frac{image\_clipped - min\_val}{max\_val - min\_val + \epsilon}
   $$
   where $\epsilon$ is a small constant (e.g., $1e-8$) to prevent division by zero [S1, unofficial].
6. **Scale to Output Range (Optional):** For 8-bit visualization, multiply by 255 and cast to uint8:
   $$
   image\_8bit = \lfloor image\_normalized \times 255 \rfloor
   $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def percentile_normalize(image, p_lo=5.0, p_hi=95.0, output_range=(0, 255), epsilon=1e-8):
    """
    Normalize image using percentile-based bounds.
    
    Args:
        image: Input scalar field array.
        p_lo: Lower percentile (default 5.0).
        p_hi: Upper percentile (default 95.0).
        output_range: Tuple (min_out, max_out) for scaling (default (0, 255)).
        epsilon: Small constant for numerical stability.
        
    Returns:
        Normalized array in output_range.
    """
    # Step 1: Filter non-finite values
    valid_mask = np.isfinite(image)
    valid_data = image[valid_mask]
    
    if valid_data.size == 0:
        # Return default range if no valid data
        return np.full_like(image, output_range[0], dtype=np.float32)
    
    # Step 2: Compute percentile bounds
    min_val = np.percentile(valid_data, p_lo)
    max_val = np.percentile(valid_data, p_hi)
    
    # Step 3: Handle degenerate cases
    if not np.isfinite(min_val) or not np.isfinite(max_val) or max_val <= min_val:
        min_val = np.min(valid_data)
        max_val = np.max(valid_data)
        
    if max_val == min_val:
        max_val = min_val + 1.0
        
    # Step 4: Clip data
    image_clipped = np.clip(image, min_val, max_val)
    
    # Step 5: Normalize
    normalized = (image_clipped - min_val) / (max_val - min_val + epsilon)
    
    # Step 6: Scale to output range
    out_min, out_max = output_range
    result = normalized * (out_max - out_min) + out_min
    
    return result
```

**Parameters:**
- $p_{lo}$: Lower percentile threshold. Typical value: 1.0–5.0 [S1, S2, unofficial].
- $p_{hi}$: Upper percentile threshold. Typical value: 95.0–99.8 [S1, S2, unofficial].
- $\epsilon$: Numerical stability constant. Typical value: $1e-8$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Single Outlier:** A single extreme voxel can dominate the absolute min/max range; percentile bounds mitigate this [S2, unofficial].
- **Constant Data:** If all valid values are identical, division by zero occurs; handled by adding 1.0 to the max bound [S2, unofficial].
- **Non-Finite Values:** NaN or Inf values must be excluded before percentile calculation to avoid incorrect bounds [S2, unofficial].

## Subtopics

### Robust Percentile Contrast Limit Calculation
Computes robust lower and upper contrast limits for volume rendering by ignoring non-finite values and using percentiles [S2, unofficial]. It differs from standard normalization by explicitly handling degenerate cases (empty data, constant data) and providing a fallback to min/max if percentiles are invalid [S2, unofficial]. It is used to determine value bounds for visualization, ensuring that outliers do not compress the contrast [S2, unofficial]. Parameter: `prc` (tuple of percentiles, default (1.0, 99.8)) [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ due to sorting required for percentile calculation, where $N$ is the number of valid voxels [S1, S2, unofficial].
- **Memory Complexity:** $O(N)$ to store valid data and intermediate arrays [S1, S2, unofficial].
- **Strengths:** Robust to outliers; enhances contrast in low-intensity regions; simple to implement [S1, S2, unofficial].
- **Limitations:** Computationally more expensive than min/max normalization; sensitive to the choice of percentiles [S2, unofficial].
- **When to Use:** When data contains significant outliers or noise that would distort min/max-based normalization [S1, S2, unofficial].
- **When Not to Use:** When real-time performance is critical and data is clean (use [Scaling](2287-scaling.md)) [S1, unofficial].

## Background
Originated in medical imaging for generating Digital Radiography (DRR) images and volume visualization, where robust contrast adjustment is essential for visualizing anatomical structures [S1, S2, unofficial]. It is widely used in CT projection processing and volume rendering pipelines to map high-precision floating-point data to 8-bit display ranges [S1, S2, unofficial].

## Sources
[S1] 医疗影像开发实战:如何用光线投射算法快速生成DRR图像(附Python代码), https://un.csdn.net/98nhkcg5t04h, fetched 2026-10-26 (unofficial)
[S2] stacy-zhang/voxel, https://github.com/stacy-zhang/voxel, fetched 2026-10-28 (unofficial)
