# Pre-filtered Cubic B-spline Interpolation
Also called: Pre-filtered B-spline Interpolation
## Overview
Pre-filtered Cubic B-spline Interpolation is a high-quality sampling technique for direct volume ray casting that improves reconstruction accuracy by compensating for the low-pass filtering effect of the cubic B-spline kernel [S1, unofficial]. Unlike standard cubic B-spline interpolation, which can cause excessive blurring, this method pre-processes the voxel data to invert the kernel's frequency response, allowing for sharper results without increasing memory footprint [S1, unofficial]. It is chosen when visual fidelity is prioritized over the raw performance of hardware-accelerated tri-linear interpolation [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: Discrete volume data $V_{raw}$ on a 3D grid.
- Output: Interpolated scalar value $V(\mathbf{p})$ at query position $\mathbf{p}$.

**Data Structures:**
- Pre-filtered volume $V_{pf}$: Same dimensions and layout as $V_{raw}$. Each voxel stores a modified scalar value.
- Standard cubic B-spline kernel weights: Determined by the relative distance of $\mathbf{p}$ to the 8 surrounding grid points.

**Steps:**
1. **Pre-filtering (Offline/Pre-processing):**
   Compute the pre-filtered volume $V_{pf}$ from the raw volume $V_{raw}$. The cubic B-spline kernel acts as a convolution filter. To compensate, $V_{pf}$ is computed by applying an inverse filter or a specific correction to $V_{raw}$. The source notes that "all voxel values are pre-processed" [S1, unofficial]. The exact mathematical inversion of the cubic B-spline kernel is not detailed in the provided sources [G: Typically involves deconvolution or a specific 3-tap filter in 1D applied separably, but exact coefficients are Not given in the sources].

2. **Ray Casting and Sampling:**
   For each ray, determine the query position $\mathbf{p}$ in the volume's local coordinate system .

3. **Interpolation:**
   Identify the 8 neighboring voxels in $V_{pf}$ that surround $\mathbf{p}$. Compute the cubic B-spline weights $w_i$ based on the distance from $\mathbf{p}$ to each neighbor's center. Calculate the interpolated value:
   $$
   V(\mathbf{p}) = \sum_{i=1}^{8} w_i V_{pf}(\mathbf{p}_i)
   $$
   This step uses "local lookups" during ray evaluation [S1, unofficial].

**Parameters:**
- Kernel: Cubic B-spline.
- Pre-filtering: Applied once per volume.

**Edge Cases and Pitfalls:**
- Performance: The method incurs a "performance penalty" compared to tri-linear interpolation [S1, unofficial].
- Implementation: Requires custom shader code for the B-spline weights and pre-filtered data access, bypassing standard hardware texture filtering which defaults to tri-linear [S1, unofficial].
- Precision: Standard floating-point precision is assumed; no specific bit-packing is mentioned.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-processing step (executed once)
def prefilter_volume(V_raw):
    # Exact inverse filter coefficients for cubic B-spline are not given in sources
    # This is a placeholder for the deconvolution step
    V_pf = apply_inverse_b_spline_filter(V_raw)
    return V_pf

# Runtime step (executed per ray sample)
def sample_prefiltered_b_spline(V_pf, p):
    # 1. Map p to grid coordinates
    grid_p = world_to_volume_grid(p)
    
    # 2. Find 8 neighbors
    # floor coordinates
    x0, y0, z0 = floor(grid_p)
    # fractional offsets
    fx, fy, fz = grid_p - [x0, y0, z0]
    
    # 3. Compute cubic B-spline weights
    # w(t) = (1 - |t|^3 + 3|t|^2 - 3|t| + 1)/2 for |t|<1? 
    # Standard cubic B-spline basis functions are needed. 
    # Not given in sources, so we assume a standard basis function B(t)
    
    # Weights for the 8 corners
    w000 = B(fx) * B(fy) * B(fz)
    w100 = B(1-fx) * B(fy) * B(fz)
    w010 = B(fx) * B(1-fy) * B(fz)
    w001 = B(fx) * B(fy) * B(1-fz)
    w110 = B(1-fx) * B(1-fy) * B(fz)
    w101 = B(1-fx) * B(fy) * B(1-fz)
    w011 = B(fx) * B(1-fy) * B(1-fz)
    w111 = B(1-fx) * B(1-fy) * B(1-fz)
    
    # 4. Lookup and sum
    val = (w000 * V_pf[x0, y0, z0] +
           w100 * V_pf[x0+1, y0, z0] +
           w010 * V_pf[x0, y0+1, z0] +
           w001 * V_pf[x0, y0, z0+1] +
           w110 * V_pf[x0+1, y0+1, z0] +
           w101 * V_pf[x0+1, y0, z0+1] +
           w011 * V_pf[x0, y0+1, z0+1] +
           w111 * V_pf[x0+1, y0+1, z0+1])
           
    return val
```
## Subtopics

### [Isogeometric Volume Rendering](14205-isogeometric-volume-rendering.md)
Extends spline-based representation to geometry and scalar fields for pixel-accurate surfaces. Chosen when geometric exactness is critical.

## Operating characteristics
- Time Complexity: Higher than tri-linear interpolation due to custom weight calculation and lack of hardware acceleration for B-splines [S1, unofficial].
- Memory Complexity: Same as raw volume data; no larger storage footprint is required [S1, unofficial].
- Strengths: Reduces blurring artifacts inherent in standard B-spline interpolation; maintains high visual quality [S1, unofficial].
- Limitations: Performance penalty; requires pre-processing step; more complex implementation than standard texture sampling [S1, unofficial].
- When to use: High-quality offline or interactive rendering where blurring is unacceptable and hardware tri-linear is insufficient.
- When not to use: Real-time applications with strict performance constraints where tri-linear interpolation is acceptable.
## Background
The method is cited as an example of advanced on-the-fly interpolation combined with pre-filtering to address artifacts in volume rendering [S1, unofficial]. It is discussed in the context of common artifacts, specifically as an alternative to tri-linear interpolation which remains popular due to hardware support [S1, unofficial].
## Sources
[S1] Common Artifacts in Volume Rendering, https://doi.org/10.48550/arxiv.2109.13704, fetched 2026-10-26 (unofficial)
