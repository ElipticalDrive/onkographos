# Trilinear Interpolation

Also called: 3D Linear Interpolation, Interpolação trilinear, Resampling and Interpolation, Trilinear Interpolant, Voxel Trilinear Interpolation

## Overview
Trilinear interpolation estimates the scalar value at an arbitrary point within a voxel grid by computing a weighted average of the values of the 8 surrounding voxels, based on their distances to the sample point [S1, S8, S11, unofficial]. It solves the problem of discretization artifacts (blockiness) inherent in nearest-neighbor sampling by providing a continuous, first-order approximation of the underlying scalar field [S1, S17, unofficial]. It produces softer, higher-quality images than nearest-neighbor interpolation but requires approximately 5 times more computation time [S1, S6, S15, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field defined on a regular grid with dimensions $D_x \times D_y \times D_z$, and a query position $\mathbf{p} = (x, y, z)$ in voxel space.
*   **Output:** A single scalar value $V(\mathbf{p})$.

**Data Structures:**
The volume data is stored in a 3D array (or flattened 1D array with row-major indexing). For a query point, the algorithm accesses the 8 corner values of the containing voxel:
$$
\begin{aligned}
v_{000} &= \text{Grid}(i_0, j_0, k_0) \\
v_{100} &= \text{Grid}(i_1, j_0, k_0) \\
v_{010} &= \text{Grid}(i_0, j_1, k_0) \\
v_{110} &= \text{Grid}(i_1, j_1, k_0) \\
v_{001} &= \text{Grid}(i_0, j_0, k_1) \\
v_{101} &= \text{Grid}(i_1, j_0, k_1) \\
v_{011} &= \text{Grid}(i_0, j_1, k_1) \\
v_{111} &= \text{Grid}(i_1, j_1, k_1)
\end{aligned}
$$
where $i_1 = i_0 + 1$, $j_1 = j_0 + 1$, $k_1 = k_0 + 1$.

**Algorithm Steps:**

1.  **Coordinate Offset:** Shift the query position by $-0.5$ in each dimension to align voxel centers with integer grid coordinates. This ensures that a sample point exactly at the center of a voxel returns that voxel's value rather than an average of its neighbors [S1, S8, unofficial].
    $$ \mathbf{p}_{\text{lattice}} = \mathbf{p} - 0.5 $$

2.  **Index Calculation:** Compute the integer indices of the lower-left corner of the containing voxel using the floor function.
    $$ i_0 = \lfloor p_{\text{lattice}, x} \rfloor, \quad j_0 = \lfloor p_{\text{lattice}, y} \rfloor, \quad k_0 = \lfloor p_{\text{lattice}, z} \rfloor $$

3.  **Boundary Clamping:** Clamp the upper indices ($i_1, j_1, k_1$) to prevent out-of-bounds access if the query point lies on the grid boundary. If $i_0 = D_x - 1$, set $i_1 = i_0$ (and similarly for $y, z$) [S9, S11, unofficial].

4.  **Fractional Weights:** Calculate the fractional offset (weights) within the voxel.
    $$ w_x = p_{\text{lattice}, x} - i_0, \quad w_y = p_{\text{lattice}, y} - j_0, \quad w_z = p_{\text{lattice}, z} - k_0 $$
    These weights lie in the range $[0, 1]$.

5.  **Interpolation:** Perform linear interpolation sequentially along the x, y, and z axes.
    *   **X-axis:** Interpolate pairs along the x-dimension for the bottom ($k_0$) and top ($k_1$) faces.
        $$
        \begin{aligned}
        c_{00} &= v_{000}(1 - w_x) + v_{100}w_x \\
        c_{10} &= v_{010}(1 - w_x) + v_{110}w_x \\
        c_{01} &= v_{001}(1 - w_x) + v_{101}w_x \\
        c_{11} &= v_{011}(1 - w_x) + v_{111}w_x
        \end{aligned}
        $$
    *   **Y-axis:** Interpolate the results from the X-step along the y-dimension.
        $$
        \begin{aligned}
        c_0 &= c_{00}(1 - w_y) + c_{10}w_y \\
        c_1 &= c_{01}(1 - w_y) + c_{11}w_y
        \end{aligned}
        $$
    *   **Z-axis:** Interpolate the results from the Y-step along the z-dimension to get the final value.
        $$ V(\mathbf{p}) = c_0(1 - w_z) + c_1 w_z $$

**Expanded Formula:**
The result can also be expressed as a weighted sum of the 8 corners:
$$
\begin{aligned}
V(\mathbf{p}) = & \ v_{000}(1-w_x)(1-w_y)(1-w_z) + v_{100}w_x(1-w_y)(1-w_z) + \\
& \ v_{010}(1-w_x)w_y(1-w_z) + v_{110}w_x w_y (1-w_z) + \\
& \ v_{001}(1-w_x)(1-w_y)w_z + v_{101}w_x(1-w_y)w_z + \\
& \ v_{011}(1-w_x)w_y w_z + v_{111}w_x w_y w_z
\end{aligned}
$$
[S1, S11, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trilinear_interpolation(grid, p, dims):
    """
    grid: 3D array of shape (Dz, Dy, Dx)
    p: tuple (x, y, z) in voxel space
    dims: tuple (Dx, Dy, Dz)
    """
    # 1. Offset by -0.5
    px, py, pz = p[0] - 0.5, p[1] - 0.5, p[2] - 0.5
    
    # 2. Floor indices
    i0 = int(px)
    j0 = int(py)
    k0 = int(pz)
    
    # 3. Upper indices and clamping
    i1 = min(i0 + 1, dims[0] - 1)
    j1 = min(j0 + 1, dims[1] - 1)
    k1 = min(k0 + 1, dims[2] - 1)
    
    # 4. Fractional weights
    wx = px - i0
    wy = py - j0
    wz = pz - k0
    
    # 5. Fetch 8 corner values
    # Note: grid is indexed as [z, y, x]
    v000 = grid[k0, j0, i0]
    v100 = grid[k0, j0, i1]
    v010 = grid[k0, j1, i0]
    v110 = grid[k0, j1, i1]
    v001 = grid[k1, j0, i0]
    v101 = grid[k1, j0, i1]
    v011 = grid[k1, j1, i0]
    v111 = grid[k1, j1, i1]
    
    # 6. Interpolate X
    c00 = v000 * (1 - wx) + v100 * wx
    c10 = v010 * (1 - wx) + v110 * wx
    c01 = v001 * (1 - wx) + v101 * wx
    c11 = v011 * (1 - wx) + v111 * wx
    
    # 7. Interpolate Y
    c0 = c00 * (1 - wy) + c10 * wy
    c1 = c01 * (1 - wy) + c11 * wy
    
    # 8. Interpolate Z
    result = c0 * (1 - wz) + c1 * wz
    
    return result
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** If the query point lies exactly on the upper boundary of the grid, the upper neighbor index would exceed the array bounds. Clamping the index to the last valid index is required [S9, S11, unofficial].
*   **Zero-Width Voxels:** If the grid has degenerate voxels (width 0), division by zero may occur if using explicit division for weights. The fractional weight calculation $w = p - i_0$ avoids this, but boundary clamping is still necessary [S11, unofficial].
*   **Offset Requirement:** Failing to apply the $-0.5$ offset results in incorrect interpolation weights, causing the center of a voxel to return an average of its 8 neighbors instead of its own value [S1, S8, unofficial].

## Subtopics

### [Plenoxels Interpolation Methods](935-plenoxels-interpolation-methods.md)
Uses trilinear interpolation on a sparse grid of spherical harmonic coefficients to enable gradient-based optimization, differing from standard scalar trilinear interpolation by storing directional radiance data.

### [Multi-Texture Trilinear Estimation](1164-multi-texture-trilinear-estimation.md)
Implements trilinear interpolation via a stack of 2D textures and dependent texturing to approximate 3D sampling on hardware that lacks native 3D texture support, relying on the Piecewise Regular Interval (PRI) data structure.

### [Semi-Analytical Volume Rendering with Trilinear Interpolation](1312-semi-analytical-volume-rendering-with-trilinear-interpolatio.md)
Uses trilinear interpolation to define the scalar field within hexahedral cells but replaces uniform ray marching with analytical integration of the piecewise-linear function to avoid isosurface artifacts.

### [Trilinear Interpolation for X-ray Attenuation Approximation](10000-trilinear-interpolation-for-x-ray-attenuation-approximation.md)
Uses trilinear interpolation to approximate the line integral of the Linear Attenuation Coefficient for differentiable X-ray rendering, trading geometric precision for speed and regularization.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, requiring exactly 8 memory fetches and 7 linear interpolations (14 multiplications, 7 additions) [S2, S15, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space.
*   **Performance:** Produces significantly higher quality images than nearest-neighbor interpolation but takes approximately 5 times longer to compute [S1, S6, unofficial]. It is faster than higher-order methods like tri-cubic interpolation [S15, unofficial].
*   **Strengths:** Simple to implement, hardware-supported (via 3D texture sampling units in GPUs), and provides smooth results suitable for many volume rendering applications [S13, S15, unofficial].
*   **Limitations:** Introduces blurring and high-frequency artifacts (false edges or "star patterns") due to the infinite extent of the triangular pulse kernel's Fourier transform [S7, unofficial]. It is not suitable for applications requiring sharp feature preservation or exact geometric reconstruction [S7, S15, unofficial].
*   **When to use:** General-purpose volume rendering where visual smoothness is prioritized over sharpness, and computational resources are limited compared to higher-order methods [S15, unofficial].
*   **When not to use:** When sharp edges must be preserved, or when exact geometric intersection is required (e.g., precise medical diagnostics where aliasing artifacts could be misleading) [S7, S15, unofficial].

## Background
Trilinear interpolation is the direct 3D extension of bilinear interpolation, defined as the linear interpolation of two bilinear interpolations [S15, unofficial]. It was widely adopted in early volume rendering techniques to improve upon the blocky artifacts of nearest-neighbor sampling. The method is fundamental to many volume rendering pipelines, including those implemented in OpenVDB and standard GPU texture sampling hardware [S1, S13, unofficial]. It is also used in related fields such as fluid simulation and computer graphics for texture mapping [S15, unofficial].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S3] 医疗影像开发实战:如何用光线投射算法快速生成DRR图像(附Python代码), https://un.csdn.net/98nhkcg5t04h, fetched 2026-10-26 (unofficial)
[S4] (No) More Marching Cubes, https://doi.org/10.2312/vg/vg07/081-090, fetched 2026-10-26
[S5] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S6] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
[S7] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S8] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S9] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S10] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S11] ank1001/DualMarchingCubes, https://github.com/ank1001/DualMarchingCubes, fetched 2026-10-27 (unofficial)
[S12] ncotta/VolumeRendering, https://github.com/ncotta/VolumeRendering, fetched 2026-10-27 (unofficial)
[S13] PDF Volume Rendering - Isenberg, https://tobias.isenberg.cc/graphics/uploads/Class/07-volume-rendering.pdf, fetched 2026-10-28 (unofficial)
[S14] PDF Direct Volume Rendering, https://www.cs.unh.edu/~cs880/lectureNotes/DVR.pdf, fetched 2026-10-28 (unofficial)
[S15] Trilinear Interpolation, https://www.scratchapixel.com/lessons/mathematics-physics-for-computer-graphics/interpolation//trilinear-interpolation.html, fetched 2026-10-29 (unofficial)
[S16] A Practical Evaluation of Popular Volume Rendering ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/PracticalEvaluationofVolumeRenderingAlgorithms.pdf, fetched 2026-10-29
[S17] Total Internal Reflection, https://avishek.net/2022/12/05/pytorch-guide-plenoxels-nerf-part-2.html, fetched 2026-10-29 (unofficial)
[S18] renderers – diffdrr, https://vivekg.dev/DiffDRR/api/renderers.html, fetched 2026-10-29
