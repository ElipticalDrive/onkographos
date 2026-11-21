# Scalar Field Normalization

Also called: Min-Max Normalization, Global Min-Max Scaling

## Overview
Scalar Field Normalization maps the raw intensity values of a volume tensor to a standardized numerical range, typically $[0, 1]$ or $[-1, 1]$, to resolve disparate data magnitudes and coordinate scales [S2, S4, S5, S6, unofficial]. It sets itself apart from non-linear methods like [Power Transformation](2346-power-transformation.md) by preserving linear relationships within the mapped range, and from global statistical methods like [Percentile-based Normalization](5158-percentile-based-normalization.md) by relying on fixed or global extrema rather than distribution percentiles [G]. This linear scaling enables the use of universal thresholds (e.g., 0.5) for object distinction [S6, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A scalar field volume $V$ containing raw intensity values (integer or float).
- Output: A normalized volume $V'$ with values in the target range $[a, b]$ (typically $[0, 1]$), stored as `float32` [S2, S6, unofficial].

**Data Structures:**
- The volume is treated as a contiguous array of scalar values. Boundary checks may require spatial coordinates $(x, y, z)$ [S4, unofficial].

**Steps:**
1. Cast input volume data to `float32` to ensure precision consistency [S1, S2, unofficial].
2. Replace any `NaN` values with 0 using `np.nan_to_num(f)` [S3, unofficial].
3. If coordinates are outside the dataset bounds (e.g., $x < 0$ or $x \geq \text{width}$), return 0 [S4, unofficial].
4. Iterate over all data elements to find the global minimum (`field_min`) and global maximum (`field_max`) of the scalar field [S5, unofficial].
5. If `f.max() == 0` or `field_max <= field_min`, return a zero-filled array of the same shape [S2, S3, unofficial].
6. Apply the linear scaling formula to each voxel value $x$:
   $$ x' = \frac{(b - a)(x - \min)}{\max - \min} + a $$
   where $a$ is the lowest normalized value (typically 0) and $b$ is the highest (typically 1) [S6, unofficial].
7. Clamp the result to the range $[0.0, 1.0]$ to handle floating-point inaccuracies [S1, unofficial].
8. Store the normalized float back into the output buffer [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def scalar_field_normalization(volume: np.ndarray, target_min: float = 0.0, target_max: float = 1.0) -> np.ndarray:
    # Step 1: Cast to float32
    f = volume.astype(np.float32)
    
    # Step 2: Handle NaNs
    f = np.nan_to_num(f, nan=0.0)
    
    # Step 3: Check for empty or zero-only volume
    if f.max() == 0:
        return np.zeros_like(f)
    
    # Step 4: Calculate global min and max
    field_min = f.min()
    field_max = f.max()
    
    # Step 5: Avoid division by zero
    if field_max <= field_min:
        return np.zeros_like(f)
    
    # Step 6: Apply linear scaling formula
    normalized = ((target_max - target_min) * (f - field_min) / (field_max - field_min)) + target_min
    
    # Step 7: Clamp to [0.0, 1.0]
    normalized = np.clip(normalized, 0.0, 1.0)
    
    return normalized
```

**Parameters:**
- `target_min` ($a$): The lowest normalized value. Typically 0 [S6, unofficial].
- `target_max` ($b$): The highest normalized value. Typically 1 [S6, unofficial].
- `dtype`: Output data type. Always `float32` [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** Occurs if $\max = \min$ (constant volume) or if the volume is empty. The algorithm must check `max > min` or `std > 0` before normalization and return zeros in such cases [S2, unofficial].
- **NaN Values:** Raw data may contain `NaN`. These must be replaced (e.g., with 0) before calculating min/max to prevent propagation [S3, unofficial].
- **Outliers:** Global min/max are sensitive to extreme outliers, which can compress the visible range of the majority of the data [G].

## Subtopics

### [Scaling](2287-scaling.md)
Linear transformation of values or coordinates to a normalized range. Preserves linear relationships but is sensitive to outliers.

### [Percentile-based Normalization](5158-percentile-based-normalization.md)
Uses robust statistical bounds (percentiles) to ignore outliers, preventing contrast loss in data with extreme values.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the total number of voxels, as it requires a single pass to find min/max and a second pass to scale values [G].
- **Memory Complexity:** $O(N)$ for the output array [G].
- **Strengths:** Enables universal thresholding [S6, unofficial].
- **Limitations:** Highly sensitive to outliers; a single extreme value can render the majority of the data indistinguishable in the normalized range [G].
- **When to Use:** When data magnitudes are consistent, or when a simple linear mapping is required for downstream algorithms that expect $[0, 1]$ inputs [G].
- **When Not to Use:** When the data contains significant outliers that distort the global range, or when preserving local contrast is more important than global intensity consistency [G].

## Background
Scalar field normalization is a fundamental preprocessing step in volume rendering and image processing [G]. It originated from the need to standardize data for visualization and algorithmic processing [G]. The application of a universal threshold (0.5) after normalization for object distinction was highlighted in segmentation-assisted volume rendering contexts [S6, unofficial].

## Sources
[S1] mversiotech/volume-rendering-demo, https://github.com/mversiotech/volume-rendering-demo, fetched 2026-10-26 (unofficial)
[S2] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
[S3] imanfs/inverse-problems-CT-scans - GitHub, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
[S4] Torbenhjl/Marching-Tetrahedra-on-CT-headscan, https://github.com/Torbenhjl/Marching-Tetrahedra-on-CT-headscan, fetched 2026-10-27 (unofficial)
[S5] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S6] Segmentation Assisted Object Distinction for Direct Volume Rendering, https://doi.org/10.22044/jadm.2019.7207.1854, fetched 2026-10-29 (unofficial)
