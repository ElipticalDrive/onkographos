# Bilinear Interpolation

Also called: Bilinear Lerp, 2D Linear Interpolation, Linear Bi-interpolation, Bi-interpolation linéaire

## Overview
Bilinear interpolation estimates the value of a function at an arbitrary point within a two-dimensional rectangular grid by using the known values at the four surrounding corner points [S5]. It extends linear interpolation from one dimension to two by applying the process sequentially along each axis, resulting in a surface that is linear along any line parallel to the coordinate axes [S5]. In volume rendering, it is used for interpolating values on the faces of 3D grid cells [S6, S9, unofficial] and for sampling 2D texture maps or coordinates [S4, S7, unofficial]. It sets itself apart from nearest-neighbor interpolation by providing C⁰ continuity [S5].

## How it works
**Inputs and Outputs:**
*   **Input:** A 2D grid of values $V$ and a query coordinate $(x, y)$ in the continuous domain of the grid [S5].
*   **Output:** The interpolated scalar value $f(x, y)$ [S5].

**Data Structures:**
The grid is typically stored as a 1D contiguous array of size $resX \times resY$ [S8, unofficial]. The element at integer coordinates $(i, j)$ is stored at index $i \times resY + j$ [S8, unofficial].

**Steps:**
1.  **Coordinate Normalization:** Map the query coordinates $(x, y)$ to the grid's local coordinate system. The integer bounds are derived by flooring:
    $$
    x_0 = \lfloor x \rfloor, \quad y_0 = \lfloor y \rfloor
    $$
    The fractional parts (weights) are:
    $$
    w_x = x - x_0, \quad w_y = y - y_0
    $$
    where $w_x, w_y \in [0, 1)$ [S5][S7, S8, unofficial].

2.  **Boundary Handling:** If $x_0$ or $y_0$ fall outside the valid grid range, the implementation must handle boundaries. Common strategies include clamping to the nearest edge, wrapping (modulo arithmetic), or returning a default value (e.g., 0) [S8, S10, unofficial]. Note that negative index wrapping implementations may contain bugs if the modulo base is incorrect (e.g., using height instead of width for x-wrapping) [S8, unofficial].

3.  **Sample Retrieval:** Retrieve the four nearest neighbor values from the grid:
    *   $V_{00} = V(x_0, y_0)$
    *   $V_{10} = V(x_0 + 1, y_0)$
    *   $V_{01} = V(x_0, y_0 + 1)$
    *   $V_{11} = V(x_0 + 1, y_0 + 1)$

4.  **Interpolation:** Perform linear interpolation along the x-axis for both y-rows, then interpolate the results along the y-axis [S5].
    *   Interpolate along x at $y_0$:
        $$
        val_0 = (1 - w_x) V_{00} + w_x V_{10}
        $$
    *   Interpolate along x at $y_1$:
        $$
        val_1 = (1 - w_x) V_{01} + w_x V_{11}
        $$
    *   Interpolate along y between $val_0$ and $val_1$:
        $$
        f(x, y) = (1 - w_y) val_0 + w_y val_1
        $$

    This can be combined into the direct tensor product formula [S5][S8, unofficial]:
    $$
    f(x, y) = (1 - w_x)(1 - w_y)V_{00} + w_x(1 - w_y)V_{10} + (1 - w_x)w_y V_{01} + w_x w_y V_{11}
    $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bilinear_interpolate(grid, res_x, res_y, x, y):
    """
    grid: 1D list or array of values, size res_x * res_y.
    res_x: int, grid width.
    res_y: int, grid height.
    x, y: float, query coordinates.
    Returns: float, interpolated value.
    """
    if res_x <= 0 or res_y <= 0:
        return 0.0

    # 1. Determine integer bounds (floor)
    x0 = int(x)
    y0 = int(y)
    if x < 0:
        x0 = -int(-x) - 1
    if y < 0:
        y0 = -int(-y) - 1

    # 2. Calculate fractional weights
    wx = x - x0
    wy = y - y0

    # 3. Boundary handling (Clamping to nearest edge)
    x0_c = max(0, min(x0, res_x - 1))
    x1_c = max(0, min(x0 + 1, res_x - 1))
    y0_c = max(0, min(y0, res_y - 1))
    y1_c = max(0, min(y0 + 1, res_y - 1))

    # 4. Retrieve values (layout: index = x * res_y + y)
    v00 = grid[x0_c * res_y + y0_c]
    v10 = grid[x1_c * res_y + y0_c]
    v01 = grid[x0_c * res_y + y1_c]
    v11 = grid[x1_c * res_y + y1_c]

    # 5. Interpolate
    val0 = (1.0 - wx) * v00 + wx * v10
    val1 = (1.0 - wx) * v01 + wx * v11
    result = (1.0 - wy) * val0 + wy * val1

    return result
```

**Parameters:**
*   `align_corners`: In some libraries (e.g., PyTorch), this boolean determines if the corner pixels are centered at 0 and 1. If `True`, the grid coordinates are scaled differently [S10, unofficial].
*   `padding_mode`: Defines behavior for out-of-bounds coordinates (e.g., "zeros", "border", "reflection") [S10, unofficial].

**Edge Cases and Pitfalls:**
*   **Discontinuity:** Bilinear interpolation provides C⁰ continuity but the first derivatives (gradients) are discontinuous at the grid cell boundaries [S5][S12, unofficial].
*   **Negative Indices:** Implementations must carefully handle negative coordinates. A common bug is using the wrong dimension for modulo wrapping (e.g., using `resY` instead of `resX` for x-axis wrapping) [S8, unofficial].
*   **Extrapolation:** If the query point lies outside the valid grid bounds, the method performs extrapolation based on the boundary handling strategy, which can produce values outside the range of the input data [S10, unofficial].

## Subtopics

### [Optimized Trilinear Reconstruction](2437-optimized-trilinear-reconstruction.md)
Optimized Trilinear Reconstruction accelerates the trilinear interpolation of volume data during direct volume ray casting by leveraging SIMD (Single Instruction, Multiple Data) processor extensions such as MMX, SSE, or 3DNow!. It improves memory copying and interpolation speed compared to scalar implementations.

### [Multi-Dimensional Interpolation Extension](2784-multi-dimensional-interpolation-extension.md)
This method extends one-dimensional reconstruction kernels to three dimensions for volume rendering. It constructs a 3D filter from a 1D base kernel $h'$ to correctly reconstruct signals on a regular 3D mesh, setting itself apart from radial extension methods by avoiding sample frequency ripple artifacts that occur with rotationally-symmetric kernels on regular grids.

### [POCS Interpolation Hardware Implementation](6377-pocs-interpolation-hardware-implementation.md)
A hardware architecture for implementing POCS interpolation filters in volume rendering, designed to overcome the quality limitations of standard tri-linear interpolation while remaining feasible within VLSI constraints. It distinguishes itself from Bilinear Interpolation by supporting larger kernel sizes ($3 \times 3 \times 3$ and $5 \times 5 \times 5$) using lookup tables and specialized memory banking to handle the high data throughput required for multi-voxel sampling.

### [Seamless Interpolation](17470-seamless-interpolation.md)
Seamless Interpolation is a method for rendering smooth surfaces stored in hierarchical tile-based structures (TileTrees) that eliminates pixelated appearance and interpolation seams at tile boundaries. It solves the problem of discontinuous color transitions when sampling from partially covered faces or n-leaves by computing a weighted sum of contributions from up to three adjacent face tiles ($T_x, T_y, T_z$) based on the surface normal. It differs from standard bilinear interpolation by explicitly handling multi-face visibility via binary alpha flags and normal-based dampening to ensure continuity across adaptive resolution boundaries and partial coverage areas.

### Linear Interpolation on Polyhedra
Uses linear interpolation to scan-convert the front and back faces of each polyhedron [S12, unofficial]. Exhibits C1 discontinuities common to linear interpolation [S12, unofficial]. Linear interpolation is used because reconstruction for non-uniform sampling on unstructured meshes is not well understood and is significantly more complex than uniform sampling [S12, unofficial].

### Temporal Interpolation
Linear interpolation between two sampled flow vectors of different time steps to handle dynamic vector fields [S13, unofficial]. Formula: $M_R(v, t) = (1 - t)S_i(v) + tS_j(v)$ where $S_i(v), S_j(v)$ are vector field slices [S13, unofficial]. If $k$ is not in $[i, j]$, the equation yields extrapolation instead of interpolation [S13, unofficial].

### Hat-shaped Basis Functions
A method using hat-shaped basis functions to define the interpolant, involving actual leaf cells only on the boundary to remove ghost artifacts [S14, unofficial]. It is unclear how to perform ray-isosurface intersections with this interpolant [S14, unofficial].

### SVGS Bilinear Interpolation
A spatially varying function that divides each Gaussian surfel into four quadrants using bilinear interpolation, assigning a learnable color and opacity value to each quadrant [S15].
Formula:
$F_c(p) = (1-u')(1-v')c_0 + (1-u')v'c_1 + u'(1-v')c_2 + u'v'c_3$ [S15]
$F_\alpha(p) = (1-u')(1-v')\alpha_0 + (1-u')v'\alpha_1 + u'(1-v')\alpha_2 + u'v'\alpha_3$ [S15]
where $u', v'$ are normalized coordinates scaled by a sigmoid function: $u' = \frac{1}{1 + e^{-\lambda_s u}}$, $v' = \frac{1}{1 + e^{-\lambda_s v}}$ [S15].
Parameter: $\lambda_s = 5.0$ [S15].
Layout: Each Gaussian primitive stores 4 learnable colors ($c_0..c_3$) and 4 learnable opacities ($\alpha_0..\alpha_3$) [S15].
Pitfall: This design may cause gradient vanishing issues [S15].

### Bilinear Interpolation with Sigmoid Scaling
A spatial color variation function that uses bilinear interpolation to model intra-primitive color changes, scaled by a sigmoid function to handle coordinates outside the standard (0,1) range [S15]. Coordinates $(u, v)$ are passed through a sigmoid function before bilinear sampling: $u' = \sigma(u), v' = \sigma(v)$ [S15]. This maps arbitrary coordinates to (0,1) but compresses gradients near quadrant centers [S15]. Sigmoid scaling causes pixel gradients to approach zero near the center of a quadrant, leading to gradient vanishing during sharp color transitions and preventing perfect scene fitting [S15]. Bilinear interpolation is only effective when the spatial color pattern conforms to a four-quadrant distribution [S15].

### Ray-Texture UV Mapping and Interpolation
Computes UV coordinates for ray-Gaussian intersections to sample spatially varying colors from texture maps using bilinear interpolation [S16].
Steps:
1. Cast a ray from the camera origin to find intersection point $p_i$ with the i-th Gaussian [S16].
2. Compute UV coordinates $u_i$ and $v_i$ using the tangent/bitangent vectors, intersection point, texel size, and texture dimensions [S16].
3. Sample the texture map $T_i$ at $(u_i, v_i)$ using bilinear interpolation to obtain color $c_{tex}$ [S16].
Formula:
$u_i(p_i) = \frac{(t_u)_i \cdot p_i}{T_{size}} + \frac{U_i - 1}{2}$ [S16]
$v_i(p_i) = \frac{(t_v)_i \cdot p_i}{T_{size}} + \frac{V_i - 1}{2}$ [S16]
where $(t_u)_i$ is the tangent vector, $(t_v)_i$ is the bitangent vector, $T_{size}$ is the texel size, and $U_i, V_i$ are texture width/height [S16].

### Pixel Level Data Dependent Interpolation
A method that preserves edges better than bilinear interpolation [S17, unofficial].
Steps:
1. Scan the low-resolution source image X to initialize a 2D array $x_{mn}$, which records the edge direction of all four-pixel squares [S17, unofficial].
2. Scan the high-resolution target image Y to initialize a 2D array $y_{ij}$. Each $y_{ij}$ is inversely mapped to the sample image X [S17, unofficial].
3. Use the array $x_{mn}$ to identify the triangle in which the inversely mapped point falls in the source image X [S17, unofficial].
4. Calculate the value of $y_{ij}$ using barycentric interpolation [S17, unofficial].
5. Detect outliers by comparing the difference $|a - c|$ and $|b - d|$ of four pixel values (represented as heights) [S17, unofficial]. The pair with the smaller difference is treated as an edge [S17, unofficial].
Comparison: Compared to bilinear interpolation, pixel level data dependent interpolation keeps edges sharp and smooth areas smooth, whereas bilinear interpolation may blur edges [S17, unofficial]. In experiments with stamens (original photo 75x75 pixels, morphed texture 2048x2048 pixels), pixel level data dependent interpolation preserved details better than bilinear interpolation [S17, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per query, as it involves a fixed number of arithmetic operations and memory accesses [S5].
*   **Memory Complexity:** Not given in the sources.
*   **Accuracy:** Provides C⁰ continuity. The approximation error is of order $O(h^2)$, where $h$ is the grid spacing [S5]. It is suitable for smooth functions but less accurate for those with higher curvature.
*   **Strengths:** Provides smooth transitions without introducing artifacts [S5].
*   **Limitations:** Discontinuous gradients at cell boundaries can cause artifacts in high-frequency details or when used for gradient-based optimization [S5, S15]. It is not suitable for non-uniformly sampled data without significant modification [S12, unofficial].
*   **When to use:** When performance is critical and the data is relatively smooth, or as a component in higher-order methods (e.g., trilinear) [S5].
*   **When not to use:** When edge preservation is critical, methods like pixel level data dependent interpolation may be preferred [S17, unofficial]. For non-uniform sampling on unstructured meshes, linear interpolation is used due to complexity, but exhibits C1 discontinuities [S12, unofficial].

## Background
Bilinear interpolation is a fundamental technique in computer graphics and image processing. It is described in technical references for fire dynamics simulation visualization tools like Smokeview [S2, unofficial] and in general computer graphics literature [S5]. It is also used in Fourier space reconstruction for tomography, where it retrieves values from polar grids to Cartesian grids [S11, unofficial]. In modern differentiable rendering, it is implemented via hardware-accelerated operations like `grid_sample` in PyTorch [S4, S10, unofficial] and is a core component of spatially varying functions in Gaussian Splatting variants like SVGS [S15].

## Sources
[S1] K-Planes: Explicit Radiance Fields in Space, Time, and Appearance, https://doi.org/10.1109/cvpr52729.2023.01201, fetched 2026-10-26
[S2] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S3] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S4] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S5] trilinear interpolation, https://grokipedia.com/page/Trilinear_interpolation, fetched 2026-10-26
[S6] [PDF] Volume Rendering for Computational Fluid Dynamics: Initial Results, https://www.nas.nasa.gov/assets/nas/pdf/techreports/1991/rnr-91-026.pdf, fetched 2026-10-26 (unofficial)
[S7] DiffBMP: Differentiable Rendering with Bitmap Primitives, https://doi.org/10.48550/arXiv.2602.22625, fetched 2026-10-27 (unofficial)
[S8] mkazhdan/IsoSurfaceExtraction, https://github.com/mkazhdan/IsoSurfaceExtraction, fetched 2026-10-27 (unofficial)
[S9] PDF GPU-based Frequency Domain Volume Rendering - TU Wien, https://www.cg.tuwien.ac.at/research/publications/2004/Viola-2004-GPU/Viola-2004-GPU-Paper.pdf, fetched 2026-10-27 (unofficial)
[S10] itu-biai/torchtomo, https://github.com/itu-biai/torchtomo, fetched 2026-10-27 (unofficial)
[S11] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
[S12] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S13] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
[S14] PDF CPU Isosurface Ray Tracing of Adaptive Mesh Refinement Da, https://www.sci.utah.edu/~feng/assets/papers/Fen-AMR-VIS18.pdf, fetched 2026-10-26 (unofficial)
[S15] SVGS: Enhancing Gaussian Splatting Using Primitives With Spatially Varying Colors., https://doi.org/10.1109/tvcg.2026.3690745, fetched 2026-10-26
[S16] TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3634643, fetched 2026-10-30
[S17] Texture Mapping Volume Objects, https://doi.org/10.2312/vvg.20051006, fetched 2026-10-30 (unofficial)
[S18] astc-encoder/Docs/FormatOverview.md at main · ARM-software ... - GitHub, https://github.com/ARM-software/astc-encoder/blob/main/Docs/FormatOverview.md, fetched 2026-10-30
