# 3D Trilinear Interpolation

Also called: 3D Linear Interpolation, SampleGV, Trilinear Interpolant, Trilinear Sampling, Trilinear Volume Interpolation, Trilinear Voxel Interpolation, Voxel Trilinear Interpolation

## Overview
3D Trilinear Interpolation reconstructs the value of a scalar field at an arbitrary point within a voxel by linearly interpolating the values at the eight corners of the enclosing grid cell [S1, S2][S3, S4, S5, S6, S7, S8, S14, S16, S17, S18, S21, S22, unofficial][S23, book][S26]. It is the standard reconstruction filter for structured rectilinear volume data, providing $C^0$ continuity and smoother results than nearest-neighbor sampling at the cost of higher computational complexity [S3, S10, S21, S24, unofficial]. It differs from higher-order methods like [Cubic Volume Interpolation](4278-cubic-volume-interpolation.md) by using only the immediate 8 neighbors and linear weights, and from [VoxelGrid](2159-voxelgrid.md) by explicitly computing off-grid values rather than returning discrete cell contents.

## How it works
**Inputs:**
- A 3D grid of scalar values $V$ with dimensions $D_x \times D_y \times D_z$.
- A query point $P = (x, y, z)$ in continuous space.
- Grid origin $O = (x_{min}, y_{min}, z_{min})$ and voxel spacing $\Delta = (\Delta x, \Delta y, \Delta z)$ [S15, S21, unofficial]. If the grid is unit-spaced and aligned at the origin, $O=(0,0,0)$ and $\Delta=(1,1,1)$.

**Outputs:**
- A scalar value $V(P)$ representing the interpolated field value at $P$.

**Data Structures:**
- The grid $V$ is typically stored as a contiguous array or 3D array. For a regular grid, the value at integer indices $(i, j, k)$ is accessed as $V[i, j, k]$ [S7, S15, unofficial].
- The 8 corner values are indexed by binary offsets $(a, b, c) \in \{0, 1\}^3$, where $a$ corresponds to the x-dimension, $b$ to y, and $c$ to z.

**Steps:**
1. **Boundary Check:** If $P$ lies outside the grid bounds, return a default value (typically 0 or undefined) [S11, S15, unofficial]. Specifically, if $x < x_{min}$ or $x \ge x_{min} + D_x \Delta x$, etc., clamp or return 0 [S11, unofficial].
2. **Compute Integer Indices:** Determine the indices of the lower corner of the enclosing voxel:
   $$ i = \lfloor \frac{x - x_{min}}{\Delta x} \rfloor, \quad j = \lfloor \frac{y - y_{min}}{\Delta y} \rfloor, \quad k = \lfloor \frac{z - z_{min}}{\Delta z} \rfloor $$
   [S8, S11, S15, S17, S19, unofficial].
3. **Compute Fractional Offsets:** Calculate the relative position of $P$ within the voxel:
   $$ u = \frac{x - (x_{min} + i \Delta x)}{\Delta x}, \quad v = \frac{y - (y_{min} + j \Delta y)}{\Delta y}, \quad w = \frac{z - (z_{min} + k \Delta z)}{\Delta z} $$
   [S6, S8, S14, S15, S17, S22, unofficial]. Note: $u, v, w \in [0, 1)$.
4. **Retrieve Corner Values:** Fetch the 8 scalar values from the grid:
   $$
   \begin{aligned}
   V_{000} &= V[i, j, k], & V_{100} &= V[i+1, j, k], \\
   V_{010} &= V[i, j+1, k], & V_{110} &= V[i+1, j+1, k], \\
   V_{001} &= V[i, j, k+1], & V_{101} &= V[i+1, j, k+1], \\
   V_{011} &= V[i, j+1, k+1], & V_{111} &= V[i+1, j+1, k+1]
   \end{aligned}
   $$
   [S8, S11, S13, S15, S19, S22, unofficial].
5. **Interpolate:** Perform linear interpolation along the x-axis, then y, then z (or any separable order). The efficient 3-multiplication form is:
   $$
   \begin{aligned}
   V_{x00} &= V_{000} + u(V_{100} - V_{000}) \\
   V_{x10} &= V_{010} + u(V_{110} - V_{010}) \\
   V_{x01} &= V_{001} + u(V_{101} - V_{001}) \\
   V_{x11} &= V_{011} + u(V_{111} - V_{011}) \\
   V_{xy0} &= V_{x00} + v(V_{x10} - V_{x00}) \\
   V_{xy1} &= V_{x01} + v(V_{x11} - V_{x01}) \\
   V(P) &= V_{xy0} + w(V_{xy1} - V_{xy0})
   \end{aligned}
   $$
   [S15, S17, S19, S20, S22, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trilinear_interpolation(grid, origin, voxel_size, point):
    """
    grid: 3D array or function V(i, j, k)
    origin: tuple (x_min, y_min, z_min)
    voxel_size: tuple (dx, dy, dz)
    point: tuple (x, y, z)
    """
    x, y, z = point
    x_min, y_min, z_min = origin
    dx, dy, dz = voxel_size

    # 1. Boundary check
    if (x < x_min or x >= x_min + grid.shape[0] * dx or
        y < y_min or y >= y_min + grid.shape[1] * dy or
        z < z_min or z >= z_min + grid.shape[2] * dz):
        return 0.0  # [G] Default boundary condition; sources vary on handling [S11, unofficial]

    # 2. Compute integer indices
    i = int((x - x_min) // dx)
    j = int((y - y_min) // dy)
    k = int((z - z_min) // dz)

    # 3. Compute fractional offsets
    u = (x - (x_min + i * dx)) / dx
    v = (y - (y_min + j * dy)) / dy
    w = (z - (z_min + k * dz)) / dz

    # 4. Retrieve corner values
    # Assuming grid is accessible via indexing grid[i, j, k]
    v000 = grid[i, j, k]
    v100 = grid[i + 1, j, k]
    v010 = grid[i, j + 1, k]
    v110 = grid[i + 1, j + 1, k]
    v001 = grid[i, j, k + 1]
    v101 = grid[i + 1, j, k + 1]
    v011 = grid[i, j + 1, k + 1]
    v111 = grid[i + 1, j + 1, k + 1]

    # 5. Interpolate
    # Linear interpolation along X
    v_x00 = v000 + u * (v100 - v000)
    v_x10 = v010 + u * (v110 - v010)
    v_x01 = v001 + u * (v101 - v001)
    v_x11 = v011 + u * (v111 - v011)

    # Linear interpolation along Y
    v_xy0 = v_x00 + v * (v_x10 - v_x00)
    v_xy1 = v_x01 + v * (v_x11 - v_x01)

    # Linear interpolation along Z
    result = v_xy0 + w * (v_xy1 - v_xy0)

    return result
```

**Parameters:**
- `INTER_MODE`: An integer flag where 1 enables trilinear interpolation, falling back to nearest neighbor otherwise [S8, unofficial].
- `order`: For stencil-based variants, the order is 1 for trilinear (stencil size 3) [S29, unofficial].

**Edge Cases and Pitfalls:**
- **Non-uniform spacing:** If voxel sizes differ or cells are not unit-aligned, the fractional offsets must be normalized by the actual cell dimensions $(x_1 - x_0)$, not assumed to be 1 [S14, S21, S22, unofficial].
- **Curvilinear/Irregular Grids:** Trilinear interpolation assumes orthogonal, rectangular cells. For rotated or curvilinear cells, $C^0$ continuity is not guaranteed, and the interpolation function may vary depending on which adjacent cell the point is assigned to [S2].
- **Artifacts:** At large magnification factors, trilinear interpolation can produce "three-dimensional crosses" or diamond-shaped artifacts due to the shape of the linear kernel [S10, unofficial].
- **AMR Data:** In Adaptive Mesh Refinement, the distinction between node-centered and cell-centered data affects stitching and reconstruction, requiring careful handling of ghost cells or dual grids [S12, unofficial].

## Subtopics

### [Trilinear Interpolation (Compressed/Linked)](2629-trilinear-interpolation-compressed-linked.md)
Performs trilinear interpolation on a compressed sparse grid using a links array to map voxel indices to compressed value indices, handling boundary checks via masking or a zero-value hack. It builds on [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md) by adding a link lookup step that translates grid coordinates to compressed indices while retaining the regular 8-corner neighborhood structure.

### [Trilinear Gradient Interpolation](3470-trilinear-gradient-interpolation.md)
Computes the gradient of the trilinear interpolation operation with respect to the input scalar field values, enabling backpropagation through sparse voxel octree (SVO) or dense grid structures. It solves the problem of differentiating the spatial sampling process in neural rendering and physics simulations by distributing the loss gradients from the output samples back to the 8 corner voxels of the interpolated cell. Unlike forward interpolation which outputs a single value, this method outputs a gradient tensor of the same shape as the input payload, allowing end-to-end training of voxel-based representations. It differs from [Analytic Derivative of Tri-linear Interpolation](6919-analytic-derivative-of-tri-linear-interpolation.md) by explicitly handling the discrete lookup and accumulation logic required for sparse data structures, rather than assuming a continuous analytic derivative on a dense grid.

### [Cubic Volume Interpolation](4278-cubic-volume-interpolation.md)
Extends standard 3D trilinear interpolation by replacing linear mixture weights with cubic polynomial weights for each spatial axis. It differs from its parent [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md) by using cubic mixture weights to determine texture sampling coordinates and interpolation factors.

### [Quantile Interpolation for Uncertain Data](5008-quantile-interpolation-for-uncertain-data.md)
Interpolates nonparametric probability distributions (PDFs) represented as histograms or KDEs by breaking them into quantiles and interpolating the quantile widths linearly, resulting in a closed-form PDF at the interpolated point with linear time complexity relative to the number of quantiles. It solves the problem of reconstructing uncertain volume data where each grid vertex holds a distribution rather than a scalar, enabling Direct Volume Rendering (DVR) that preserves statistical properties like modality and variance. Unlike convolution-based nonparametric approaches which have exponential time complexity, this method is feasible for DVR reconstruction because its complexity scales linearly with the number of quantiles. It builds on [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md) by applying the trilinear logic to the widths of quantile intervals rather than scalar values.

### [Analytic Derivative of Tri-linear Interpolation](6919-analytic-derivative-of-tri-linear-interpolation.md)
Computes the exact partial derivatives of a tri-linearly interpolated grid function (such as color $\vec{c}$ or density $\sigma$) with respect to the sample location $p_i = (x_i, y_i, z_i)$. It is primarily used for gradient-based optimization in neural implicit representations, such as SLAM and NeRF variants, where analytical gradients are required for backpropagation or optimization steps. Unlike numerical differentiation or interpolated gradient methods, this approach provides the exact derivative of the underlying tri-linear basis function by solving for the polynomial coefficients of the interpolation within a single voxel.

### [Multivariate Functional Approximation](8173-multivariate-functional-approximation.md)
Models 3D scalar fields using polynomial functions (specifically B-splines) defined over micro-blocks of the data grid. It solves the problem of accurate off-grid sample evaluation and smooth shading in volume rendering by providing higher-order continuity than trilinear interpolation without requiring ghost voxels or boundary padding. Unlike its sibling [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md), which is limited to C0 continuity and linear accuracy, MFA approximates C1 continuity across partition boundaries, reducing Mach ringing artifacts and polygon counts in isosurface extraction while maintaining a compact data footprint.

### [Ghost Cell Stitching for Node-Centered AMR](8949-ghost-cell-stitching-for-node-centered-amr.md)
Handles data continuity between different Adaptive Mesh Refinement (AMR) levels in node-centered grids by introducing ghost cells at level boundaries. It solves the problem of rendering artifacts (such as cracks or "ghosting") that occur when isosurface extraction or volume rendering requires access to neighboring cells that reside on different processing elements or different refinement levels. Unlike methods that redistribute the entire dataset, this approach uses simulation-provided abutment information to selectively exchange only the necessary boundary data, ensuring that stitch cells can be generated consistently across coarse-fine interfaces.

### [Smoothstep Trilinear Interpolation](9344-smoothstep-trilinear-interpolation.md)
Used in multiresolution hash encodings to achieve $C^1$-smooth interpolation (continuous first derivatives) in the spatial domain. It solves the problem of derivative discontinuities at grid boundaries inherent in standard trilinear interpolation, which can cause artifacts in neural rendering and gradient-based optimization. It sets itself apart from higher-order polynomial interpolations (like bicubic) by achieving similar smoothness properties at effectively zero computational cost, as the overhead is hidden by memory bottlenecks. It builds on [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md) by modifying the interpolation weights rather than the data structure or lookup count.

### [Barycentric Interpolation](9603-barycentric-interpolation.md)
Interpolates learnable feature vectors at the $n+1$ vertices of a simplex cell using barycentric coordinates derived from sorted, skewed coordinates. It serves as an efficient alternative to computationally expensive $n$-linear interpolation (such as trilinear interpolation) by maintaining similarity to standard interpolation schemes while avoiding the kernel summation overhead found in simplex noise algorithms. It is specifically designed for simplex-based structures where points are mapped to vertices via coordinate skewing and simplicial subdivision.

### [Semi-Lagrangian Fluid Advection](12285-semi-lagrangian-fluid-advection.md)
Updates physical quantities (volume and velocity) on a fixed Eulerian grid by back-tracing from the current grid center to the previous time step using the velocity field, then interpolating the value. It differs from Lagrangian methods by keeping particles fixed to the grid, and from pure Eulerian finite-difference advection by using characteristic tracing which allows larger time steps without instability. It builds on [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md) for the sampling step.

### Grid Querying with Interpolation
Queries volume density $\sigma$ and color $c$ from feature grids along camera rays using interpolation operators [S28]. It reduces computational burden by replacing trilinear interpolation with linear/bilinear interpolation of vector/matrix factors [S28].

### Majorant Fix for Interpolation
Adjusts the stored maximum density value in each leaf node to account for interpolation artifacts. When interpolating near leaf boundaries, the stencil may sample neighboring leaves with higher values, causing the leaf's stored max to be an underestimate. This function ensures the majorant is a valid upper bound for any interpolated value within the leaf [S29, unofficial]. It iterates over all leaves, computes the interpolation Area of Effect (AoE) bounding box, checks 3x3x3 neighbor offsets, and updates the leaf's max density [S29, unofficial]. The `order` parameter is 1 for trilinear interpolation [S29, unofficial]. A pitfall is that the function only updates leaf nodes, not non-leaf nodes [S29, unofficial].

### ID-Based Interpolation
Uses ID values at each vertex to perform interpolation, compared against standard tri-linear interpolation of scalar values [S30, unofficial]. The average relative error $A$ is calculated as $A_{avg} = (\sum E_i / n) \times 100$, where $n=8$ for a cubical voxel [S30, unofficial].

### 1D Trilinear Interpolation
Interpolates the voxel value at a non-integer z-position by linearly interpolating between the two nearest integer z-slices [S31, unofficial]. It calculates $z_1 = \lfloor z \rfloor$ and $z_2 = \lceil z \rceil$, fetches $V_1$ and $V_2$, and returns $V_1 + (V_2 - V_1) \times \frac{z - z_1}{z_2 - z_1}$ [S31, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, requiring 7 multiplications and 15 additions/subtractions in the standard separable implementation [S20, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space, accessing 8 neighboring voxels [S5, S10, unofficial].
- **Performance:** On an SGI Iris 4D50GT, rendering rectilinear grids was significantly faster than curvilinear ones, with precalculation of trilinear coefficients for curvilinear volumes taking ~110 seconds [S2]. For regular grids, no precalculation is needed [S2].
- **Strengths:** Provides smooth, continuous variation between sample points, superior to nearest-neighbor interpolation which creates uniform regions of constant value [S21, unofficial]. It is the standard filter for ray casting in direct volume rendering [S3, S12, unofficial].
- **Limitations:** Computationally more expensive than nearest-neighbor [S24, unofficial]. Can produce artifacts (diamonds/crosses) at high magnification [S10, unofficial]. Continuity is not guaranteed for irregular or rotated cells [S2].
- **When to use:** Standard reconstruction for structured rectilinear volume data in direct volume rendering [S3, S12, unofficial].
- **When not to use:** When $C^1$ continuity is required (use [Smoothstep Trilinear Interpolation](9344-smoothstep-trilinear-interpolation.md) or [Cubic Volume Interpolation](4278-cubic-volume-interpolation.md)), or when dealing with irregular/unstructured grids where piecewise linear or higher-order methods on simplices are more appropriate [S1, S2].

## Background
Trilinear interpolation was introduced as a standard method for reconstructing values in regular, rectilinear, and curvilinear datasets for volume rendering [S1, S2]. It is widely used in Direct Volume Rendering (DVR) pipelines, particularly in ray-casting techniques where sample points fall between voxel centers [S3, S4, S6, S14, unofficial]. The method assumes the underlying function varies linearly along axes parallel to the cartesian axes [S2]. It is often cited in the context of interactive volume rendering and medical image visualization [S1][S14, unofficial].

## Sources
[S1] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S2] Direct volume rendering of curvilinear volumes, https://doi.org/10.1145/99308.99318, fetched 2026-10-28
[S3] Direct Volume Rendering Techniques, https://www.scribd.com/presentation/380929057/10-raycasting, fetched 2026-10-25 (unofficial)
[S4] Direct Volume Rendering (DVR): Ray-casting | PPT - SlideShare, https://www.slideshare.net/slideshow/direct-volume-rendering-dvr-raycasting/265104120, fetched 2026-10-26 (unofficial)
[S5] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
[S6] Antara: An Interactive 3D Volume Rendering and Visualization Framework, https://arxiv.org/abs/1812.04233v1, fetched 2026-10-26 (unofficial)
[S7] kkhese/Volume-Rendering, https://github.com/kkhese/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S8] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
[S9] Efficient Encoding of Graphics Primitives with Simplex-based Structures, https://doi.org/10.48550/arxiv.2311.15439, fetched 2026-10-26 (unofficial)
[S10] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S11] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S12] CPU volume rendering of adaptive mesh refinement data, https://www.researchgate.net/publication/321321949_CPU_volume_rendering_of_adaptive_mesh_refinement_data, fetched 2026-10-26 (unofficial)
[S13] arigo/CascatedVoxelConeTracing, https://github.com/arigo/CascatedVoxelConeTracing, fetched 2026-10-26 (unofficial)
[S14] [PDF] Ray Casting Based Volume Rendering of Medical Images, https://ansne.du.ac.ir/article_198_0bab262270f7a40e3ac17e01ea3d912b.pdf, fetched 2026-10-26 (unofficial)
[S15] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
[S16] PDF Production Volume Rendering - Clemson University, https://jtessen.people.clemson.edu/reports/papers_files/ProdVolRender.pdf, fetched 2026-10-27 (unofficial)
[S17] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
[S18] Volume Rendering Volume Rendering, https://docslib.org/doc/1400821/volume-rendering, fetched 2026-10-28 (unofficial)
[S19] HiroSakuraba/destructible-voxel-engine, https://github.com/HiroSakuraba/destructible-voxel-engine, fetched 2026-10-28 (unofficial)
[S20] PDF Direct Volume Rendering, https://www.cs.unh.edu/~cs880/lectureNotes/DVR.pdf, fetched 2026-10-28 (unofficial)
[S21] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
[S22] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
[S23] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S24] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
[S25] VOLUME RENDERING Application by Thomas Lidy, https://www.cg.tuwien.ac.at/courses/Visualisierung/2001-2002/Ergebnisse/Beispiel1/LidyT/description.html, fetched 2026-10-29
[S26] New metric for evaluating the accuracy of marching isosurfacing algorithms, https://doi.org/10.1145/2638404.2638484, fetched 2026-11-05
[S27] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
[S28] Balanced Spherical Grid for Egocentric View Synthesis, https://doi.org/10.1109/cvpr52729.2023.01592, fetched 2026-10-29
[S29] Shibodd/volume_path_tracer, https://github.com/Shibodd/volume_path_tracer, fetched 2026-10-26 (unofficial)
[S30] Segmentation Assisted Object Distinction for Direct Volume Rendering, https://doi.org/10.22044/jadm.2019.7207.1854, fetched 2026-10-29 (unofficial)
[S31] joshcodd/volume-rendering, https://github.com/joshcodd/volume-rendering, fetched 2026-10-26 (unofficial)
