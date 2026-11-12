# Bilinear Interpolation

Also called: Bilinear Sampling, Bi-linear Interpolation for Triangle Coloring

## Overview
Bilinear interpolation is a reconstruction scheme that estimates a scalar value at an arbitrary 2D coordinate by performing linear interpolation in both spatial dimensions using the four nearest grid points. It serves as the fundamental building block for higher-order methods like trilinear interpolation and is used in volume rendering to sample scalar fields on 2D planes or to approximate 3D values by restricting sampling to cell faces [S5, unofficial]. It sets itself apart from nearest neighbor interpolation by providing continuous, first-order smoothness, and from trilinear interpolation by operating in two dimensions rather than three, requiring fewer neighbor lookups (4 vs 8) [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 2D grid of scalar values $V$, defined on integer coordinates $(x, y)$ where $x \in [0, \text{resX}-1]$ and $y \in [0, \text{resY}-1]$. A query point $\mathbf{p} = (x, y)$ with continuous coordinates.
*   **Output:** The interpolated scalar value $V(\mathbf{p})$.

**Data Structures:**
The grid is stored as a 1D contiguous array `values` of size $\text{resX} \times \text{resY}$. The element at grid coordinates $(x, y)$ is stored at index $x \cdot \text{resY} + y$ [S6, unofficial].

**Formula:**
Let $x_1 = \lfloor x \rfloor$ and $y_1 = \lfloor y \rfloor$ be the integer coordinates of the lower-left corner of the cell containing $\mathbf{p}$. Let the fractional offsets be $d_x = x - x_1$ and $d_y = y - y_1$. The four neighboring voxel values are:
$$
\begin{aligned}
V_{00} &= V(x_1, y_1) \\
V_{10} &= V(x_1+1, y_1) \\
V_{01} &= V(x_1, y_1+1) \\
V_{11} &= V(x_1+1, y_1+1)
\end{aligned}
$$
The interpolated value is:
$$
V(x,y) = V_{00}(1-d_x)(1-d_y) + V_{10}d_x(1-d_y) + V_{01}(1-d_x)d_y + V_{11}d_x d_y
$$
[S6, unofficial]

**Parameters:**
*   **Boundary Handling:** Not explicitly defined in the core method. Common implementations use wrapping (modulo arithmetic) or clamping. Wrapping is implemented by handling negative indices: if $x < 0$, $x = \text{resX} - ((-x) \% \text{resX})$ [S6, unofficial]. Note: A bug in some implementations uses `resY` instead of `resX` for the x-axis modulo, causing incorrect wrapping [S6, unofficial].
*   **Coordinate Alignment:** In frameworks like PyTorch, `align_corners=True` maps the grid boundaries to $\pm 1$, affecting the calculation of $d_x$ and $d_y$ [S4, unofficial][S8, unofficial].

**Steps:**
1.  Compute integer bounds: $x_1 = \lfloor x \rfloor$, $y_1 = \lfloor y \rfloor$.
2.  Compute fractional offsets: $d_x = x - x_1$, $d_y = y - y_1$.
3.  Retrieve the four neighbor values $V_{00}, V_{10}, V_{01}, V_{11}$ from the grid, applying boundary handling if indices are out of bounds.
4.  Compute the weighted sum using the formula above.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bilinear_interpolate(values, resX, resY, x, y):
    # 1. Handle negative indices with wrapping [S6, unofficial]
    if x < 0:
        x = resX - ((-x) % resX)
    if y < 0:
        y = resY - ((-y) % resY)
    
    # 2. Compute integer bounds and fractional offsets
    x1 = int(x)
    y1 = int(y)
    dx = x - x1
    dy = y - y1
    
    # 3. Retrieve neighbors with boundary clamping for safety [G]
    # In a strict implementation, indices should be validated against resX/resY
    # or wrapped consistently.
    def get_val(ix, iy):
        # Wrap indices
        ix = ix % resX
        iy = iy % resY
        return values[ix * resY + iy]

    v00 = get_val(x1, y1)
    v10 = get_val(x1 + 1, y1)
    v01 = get_val(x1, y1 + 1)
    v11 = get_val(x1 + 1, y1 + 1)
    
    # 4. Compute interpolation
    # V(x,y) = V00(1-dx)(1-dy) + V10*dx(1-dy) + V01(1-dx)*dy + V11*dx*dy
    return v00 * (1 - dx) * (1 - dy) + \
           v10 * dx * (1 - dy) + \
           v01 * (1 - dx) * dy + \
           v11 * dx * dy
```

## Subtopics

### [Optimized Trilinear Reconstruction](2437-optimized-trilinear-reconstruction.md)
Optimized Trilinear Reconstruction accelerates trilinear interpolation (which builds on bilinear interpolation) by leveraging SIMD extensions (MMX, SSE, 3DNow!) to improve memory copying and interpolation speed compared to scalar implementations.

### [Multi-Dimensional Interpolation Extension](2784-multi-dimensional-interpolation-extension.md)
This method extends 1D reconstruction kernels to 3D for volume rendering. It constructs a 3D filter from a 1D base kernel to correctly reconstruct signals on a regular 3D mesh, avoiding sample frequency ripple artifacts that occur with rotationally-symmetric kernels on regular grids.

### [POCS Interpolation Hardware Implementation](6377-pocs-interpolation-hardware-implementation.md)
A hardware architecture for implementing POCS interpolation filters, designed to overcome the quality limitations of standard tri-linear interpolation while remaining feasible within VLSI constraints. It distinguishes itself by supporting larger kernel sizes ($3 \times 3 \times 3$ and $5 \times 5 \times 5$) using lookup tables and specialized memory banking to handle high data throughput required for multi-voxel sampling.

### [View Adapted Stepping Distances](13215-view-adapted-stepping-distances.md)
View Adapted Stepping Distances is a volume rendering acceleration technique that aligns sample points with voxel cube faces to enable bilinear interpolation instead of trilinear interpolation, while using adaptive termination to skip empty space. It differs by dynamically calculating the stepping distance based on the view direction to enforce this alignment, reducing neighbor lookups from 8 to 4.

### Linear Interpolation on Polyhedra
Uses linear interpolation to scan-convert the front and back faces of each polyhedron for unstructured meshes. It exhibits C1 discontinuities common to linear interpolation. Linear interpolation is used because reconstruction for non-uniform sampling on unstructured meshes is not well understood and is significantly more complex than uniform sampling [S9, unofficial].

### Temporal Interpolation
Linear interpolation between two sampled flow vectors of different time steps to handle dynamic vector fields. The interpolated vector is $M_R(v, t) = (1 - t)S_i(v) + tS_j(v)$. If the time index $k$ is not between $i$ and $j$, the equation yields extrapolation instead of interpolation [S10, unofficial].

### Hat-shaped Basis Functions
A method using hat-shaped basis functions to define the interpolant, involving actual leaf cells only on the boundary to remove ghost artifacts. It is unclear how to perform ray-isosurface intersections with this interpolant [S11, unofficial].

### SVGS Bilinear Interpolation
A spatially varying function that divides each Gaussian surfel into four quadrants using bilinear interpolation, assigning a learnable color and opacity value to each quadrant. The color is $F_c(p) = (1-u')(1-v')c_0 + (1-u')v'c_1 + u'(1-v')c_2 + u'v'c_3$, where $u', v'$ are sigmoid-scaled coordinates. This design may cause gradient vanishing issues [S12].

### Bilinear Interpolation with Sigmoid Scaling
A spatial color variation function that uses bilinear interpolation to model intra-primitive color changes, scaled by a sigmoid function to handle coordinates outside the standard (0,1) range. Coordinates are mapped via $u' = \sigma(u)$. This causes pixel gradients to approach zero near the center of a quadrant, leading to gradient vanishing during sharp color transitions [S12].

### Ray-Texture UV Mapping and Interpolation
Computes UV coordinates for ray-Gaussian intersections to sample spatially varying colors from texture maps using bilinear interpolation. UV coordinates are computed as $u_i(p_i) = ((t_u)_i \cdot p_i) / T_{size} + (U_i - 1)/2$. The texture is sampled at these UVs using bilinear interpolation to obtain the color [S13].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, involving a constant number of arithmetic operations and memory accesses (4 reads).
*   **Memory Complexity:** $O(1)$ auxiliary memory per sample.
*   **Strengths:** Computationally efficient; provides continuous gradients (unlike nearest neighbor); simple to implement and hardware-accelerated.
*   **Limitations:** Introduces blurring (low-pass filtering) compared to higher-order methods; suffers from C1 discontinuities in derivatives; not suitable for unstructured meshes without modification [S9, unofficial].
*   **When to use:** When performance is critical and high-frequency detail preservation is not the primary goal; as a component in higher-order schemes (e.g., trilinear); for 2D texture sampling in volume rendering contexts.
*   **When not to use:** When aliasing artifacts from nearest neighbor are unacceptable but blurring is also problematic (consider sinc or cubic); for unstructured meshes where non-uniform reconstruction is required [S9, unofficial].

## Background
Bilinear interpolation is a standard technique in computer graphics and image processing. It is used in volume rendering to sample scalar fields on 2D planes [S5, unofficial] and to determine colors within triangles when lighting is not applied [S2, unofficial]. It is also used in differentiable rendering pipelines for forward and adjoint operations on CPU and CUDA [S8, unofficial]. The method is foundational for more complex interpolation schemes and is widely supported in hardware and software libraries (e.g., PyTorch's `grid_sample`) [S4, unofficial][S8, unofficial].

## Sources
[S1] K-Planes: Explicit Radiance Fields in Space, Time, and Appearance, https://doi.org/10.1109/cvpr52729.2023.01201, fetched 2026-10-26
[S2] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S3] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S4] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S5] Volume Rendering for Computational Fluid Dynamics: Initial Results, https://www.nas.nasa.gov/assets/nas/pdf/techreports/1991/rnr-91-026.pdf, fetched 2026-10-26 (unofficial)
[S6] mkazhdan/IsoSurfaceExtraction, https://github.com/mkazhdan/IsoSurfaceExtraction, fetched 2026-10-27 (unofficial)
[S7] GPU-based Frequency Domain Volume Rendering - TU Wien, https://www.cg.tuwien.ac.at/research/publications/2004/Viola-2004-GPU/Viola-2004-GPU-Paper.pdf, fetched 2026-10-27 (unofficial)
[S8] itu-biai/torchtomo, https://github.com/itu-biai/torchtomo, fetched 2026-10-27 (unofficial)
[S9] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S10] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
[S11] CPU Isosurface Ray Tracing of Adaptive Mesh Refinement Da, https://www.sci.utah.edu/~feng/assets/papers/Fen-AMR-VIS18.pdf, fetched 2026-10-26 (unofficial)
[S12] SVGS: Enhancing Gaussian Splatting Using Primitives With Spatially Varying Colors., https://doi.org/10.1109/tvcg.2026.3690745, fetched 2026-10-26
[S13] TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3634643, fetched 2026-10-30
