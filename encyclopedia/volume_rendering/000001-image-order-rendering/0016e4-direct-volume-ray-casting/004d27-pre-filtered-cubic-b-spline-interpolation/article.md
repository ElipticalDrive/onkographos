# Pre-filtered Cubic B-spline Interpolation
Also called: Pre-filtered B-spline Interpolation
## Overview
Pre-filtered Cubic B-spline Interpolation is an on-the-fly interpolation approach combined with a pre-filtering step to perform more accurate interpolations without larger data storage [S1, unofficial]. It is chosen when reconstruction accuracy is prioritized over the raw performance of hardware-accelerated tri-linear interpolation [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: Discrete volume data $V_{raw}$ on a 3D grid.
- Output: Interpolated scalar value $V(\mathbf{p})$ at query position $\mathbf{p}$.

**Data Structures:**
- Pre-filtered volume $V_{pf}$: Same dimensions and layout as $V_{raw}$, storing pre-processed voxel values [S1, unofficial].

**Steps:**
1. **Pre-filtering (Offline/Pre-processing):** Pre-process all voxel values in $V_{raw}$ to produce $V_{pf}$ [S1, unofficial]. The exact mathematical inversion or filter coefficients are not detailed in the sources [G: Typically involves separable deconvolution or inverse filtering in 1D applied across dimensions, but exact coefficients are Not given in the sources].
2. **Ray Casting and Sampling:** For each ray sample position $\mathbf{p}$, perform local lookups in $V_{pf}$ [S1, unofficial].
3. **Interpolation:** Combine the surrounding voxel values using cubic B-spline basis functions to compute $V(\mathbf{p})$ [S1, unofficial].

**Parameters:**
- Kernel: Cubic B-spline.
- Pre-filtering: Applied once per volume [S1, unofficial].

**Edge Cases and Pitfalls:**
- Performance: The method incurs a "performance penalty" compared to tri-linear interpolation [S1, unofficial].
- Implementation: Requires custom shader code for the B-spline weights and pre-filtered data access, bypassing standard hardware texture filtering which defaults to tri-linear [S1, unofficial].
- Precision: Standard floating-point precision is assumed; no specific bit-packing is mentioned.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cubic_b_spline_basis(u):
    """Standard 1D cubic B-spline basis function for u in [0, 1]."""
    u2 = u * u
    u3 = u2 * u
    w0 = (1.0 - u)**3 / 6.0
    w1 = (3.0 * u3 - 6.0 * u2 + 4.0) / 6.0
    w2 = (-3.0 * u3 + 3.0 * u2 + 3.0 * u + 1.0) / 6.0
    w3 = u3 / 6.0
    return w0, w1, w2, w3

def world_to_volume_grid(p, origin, spacing):
    """Transform world coordinates to volume grid coordinates."""
    return [(p[i] - origin[i]) / spacing[i] for i in range(3)]

def apply_inverse_b_spline_filter(V_raw):
    # Exact inverse filter coefficients for cubic B-spline are not given in sources
    # This function represents the pre-processing step described in [S1, unofficial]
    return V_raw  # Placeholder for deconvolution step

def prefilter_volume(V_raw, origin, spacing):
    """Pre-process all voxel values to produce V_pf."""
    V_pf = apply_inverse_b_spline_filter(V_raw)
    return V_pf

def sample_prefiltered_b_spline(V_pf, p, origin, spacing):
    """Interpolate value at query position p using pre-filtered volume."""
    grid_p = world_to_volume_grid(p, origin, spacing)
    x0, y0, z0 = math.floor(grid_p[0]), math.floor(grid_p[1]), math.floor(grid_p[2])
    fx, fy, fz = grid_p[0] - x0, grid_p[1] - y0, grid_p[2] - z0

    w0x, w1x, w2x, w3x = cubic_b_spline_basis(fx)
    w0y, w1y, w2y, w3y = cubic_b_spline_basis(fy)
    w0z, w1z, w2z, w3z = cubic_b_spline_basis(fz)

    weights_x = [w0x, w1x, w2x, w3x]
    weights_y = [w0y, w1y, w2y, w3y]
    weights_z = [w0z, w1z, w2z, w3z]

    val = 0.0
    for i in range(4):
        for j in range(4):
            for k in range(4):
                val += weights_x[i] * weights_y[j] * weights_z[k] * V_pf[x0 + i, y0 + j, z0 + k]
    return val
```
## Subtopics

### [Isogeometric Volume Rendering](14205-isogeometric-volume-rendering.md)
Extends spline-based representation to geometry and scalar fields for pixel-accurate surfaces. Chosen when geometric exactness is critical.

## Operating characteristics
- Time Complexity: Higher than tri-linear interpolation due to a performance penalty [S1, unofficial].
- Memory Complexity: Same as raw volume data; no larger storage footprint is required [S1, unofficial].
- Strengths: Provides more accurate interpolations than tri-linear methods [S1, unofficial].
- Limitations: Performance penalty; requires pre-processing step; more complex implementation than standard texture sampling [S1, unofficial].
- When to use: High-quality rendering where reconstruction accuracy is prioritized over raw performance [S1, unofficial].
- When not to use: Real-time applications with strict performance constraints where tri-linear interpolation is acceptable [S1, unofficial].
## Background
The method is cited as an example of advanced on-the-fly interpolation combined with pre-filtering to address artifacts in volume rendering [S1, unofficial]. It is discussed in the context of common artifacts, specifically as an alternative to tri-linear interpolation which remains popular due to hardware support [S1, unofficial].
## Sources
[S1] Common Artifacts in Volume Rendering, https://doi.org/10.48550/arxiv.2109.13704, fetched 2026-10-26 (unofficial)
