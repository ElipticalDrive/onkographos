# Volume Normalization


## Overview
Volume Normalization maps a volume tensor's scalar values to the range $[0, 1]$ while preserving constant positive masks as full foreground ($1.0$) [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A volume tensor $V$ containing scalar voxel values [G].
- Output: A normalized volume tensor $V_{norm}$ of identical shape with values in $[0, 1]$ [S1, unofficial].

**Data Structures:**
- The volume is represented as a dense array or sequence of floating-point numbers [G].

**Steps:**
1. Compute the global minimum `vol_min = volume.min()` and global maximum `vol_max = volume.max()` across all voxels [S1, unofficial].
2. Calculate the dynamic range `R = vol_max - vol_min` [S1, unofficial].
3. Check for degeneracy: If `R <= 1e-8`, treat the volume as constant [S1, unofficial].
   - If `vol_max > 0.0`, fill the output with `1.0` [S1, unofficial].
   - Otherwise, fill the output with `0.0` [S1, unofficial].
4. For non-constant volumes, apply the linear scaling formula to each voxel $v \in V$:
   $$v_{norm} = \frac{v - v_{min}}{v_{max} - v_{min}}$$
   [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_normalization(volume: list, epsilon: float = 1e-8) -> list:
    # Step 1: Compute global extrema
    vol_min = min(volume)
    vol_max = max(volume)
    
    # Step 2: Calculate range
    vol_range = vol_max - vol_min
    
    # Step 3: Handle degenerate/constant volumes
    if vol_range <= epsilon:
        if vol_max > 0.0:
            return [1.0] * len(volume)
        else:
            return [0.0] * len(volume)
    
    # Step 4: Apply linear normalization
    normalized = [(v - vol_min) / vol_range for v in volume]
    return normalized
```

**Parameters:**
- `epsilon`: Threshold for detecting constant volumes, typically $10^{-8}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Constant Volumes:** Division by zero is avoided by checking if the range is near zero; positive constant volumes map to $1.0$ and non-positive constant volumes map to $0.0$ [S1, unofficial].
- **Precision:** Floating-point arithmetic must be used to prevent integer division truncation [G].
- **Boundaries:** The method assumes valid scalar data; invalid values such as NaNs should be handled prior to normalization [G].

## Subtopics

### [Scalar Field Normalization](1842-scalar-field-normalization.md)
Scalar Field Normalization maps raw intensity values to a standardized numerical range, typically $[0, 1]$ or $[-1, 1]$, to resolve disparate data magnitudes and coordinate scales [G]. It sets itself apart from non-linear methods like [Power Transformation](2346-power-transformation.md) by preserving linear relationships within the mapped range, and from global statistical methods like [Percentile-based Normalization](5158-percentile-based-normalization.md) by relying on fixed or global extrema rather than distribution percentiles [G]. This linear scaling enables the use of universal thresholds for object distinction and ensures consistent behavior across different input resolutions and data types [G]. Choose this method when linear intensity relationships must be preserved for downstream processing [G].

## Operating characteristics
- Time complexity is $O(N)$ and memory complexity is $O(N)$, where $N$ is the number of voxels, as the method requires a single pass to find extrema and another to scale [G].
- Strengths, limitations, scaling behavior, and specific usage conditions are Not given in the sources.

## Background
The method originates from the `vgs` repository [S1, unofficial]. Its application in volume rendering, medical image analysis, neural network training, and segmentation pipelines is Not given in the sources.

## Sources
[S1] ValentinKraft/vgs, https://github.com/ValentinKraft/vgs, fetched 2026-10-27 (unofficial)
