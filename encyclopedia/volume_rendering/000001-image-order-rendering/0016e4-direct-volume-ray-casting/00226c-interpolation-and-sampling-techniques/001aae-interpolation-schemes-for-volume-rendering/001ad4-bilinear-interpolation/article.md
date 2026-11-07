# Bilinear Interpolation

Also called: Bilinear Sampling, Bi-linear Interpolation for Triangle Coloring

## Overview
Bilinear interpolation estimates values at arbitrary 2D coordinates within a regular grid by performing linear interpolation first in one dimension and then in the other [S2, S5, S6, unofficial]. It is used for 2D texture sampling [S4, unofficial], determining colors within triangles when lighting is not applied [S2, unofficial], calculating scalar values at ray-cell face intersections [S5, unofficial], and retrieving feature values in multi-scale plane representations [S1]. It differs from [Trilinear Interpolation](1531-trilinear-interpolation.md) by operating only on two dimensions, requiring extension to the third axis for full 3D volume sampling [G].

## How it works
**Inputs and Outputs:**
- Input: A regular 2D grid of values $V_{grid}$ with dimensions $resX \times resY$, and a query point $(x, y)$ in continuous coordinate space [S6, unofficial].
- Output: The interpolated scalar value $V(x, y)$ [S6, unofficial].

**Data Structures:**
The grid is stored as a 1D contiguous array `values` of size $resX \times resY$ [S6, unofficial]. The element at integer coordinates $(x, y)$ is stored at index $x \cdot resY + y$ [S6, unofficial]. The grid metadata includes:
- `resX`: Integer width of the grid.
- `resY`: Integer height of the grid.
- `values`: Pointer to the data array [S6, unofficial].

**Algorithm Steps:**
1. **Coordinate Normalization and Indexing:**
   Compute integer bounds and fractional distances: $x_1 = \lfloor x \rfloor$, $d_x = x - x_1$ [S6, unofficial]. Handle negative $x$ with $x_1 = -\lfloor -x \rfloor - 1$ if $x < 0$ [S6, unofficial]. Apply the same logic for $y$ to obtain $y_1$ and $d_y$ [S6, unofficial].
2. **Neighbor Fetching:**
   Retrieve the four nearest neighbor values from the grid:
   - $V_{00} = Grid(x_1, y_1)$
   - $V_{10} = Grid(x_1 + 1, y_1)$
   - $V_{01} = Grid(x_1, y_1 + 1)$
   - $V_{11} = Grid(x_1 + 1, y_1 + 1)$ [S6, unofficial].
3. **Interpolation:**
   Perform linear interpolation along the x-axis for the bottom and top edges, then interpolate the result along the y-axis:
   $$ V(x, y) = V_{00}(1 - d_x)(1 - d_y) + V_{10}d_x(1 - d_y) + V_{01}(1 - d_x)d_y + V_{11}d_x d_y $$
   This expands to two sequential linear interpolations:
   $$ V_{bottom} = V_{00}(1 - d_x) + V_{10}d_x $$
   $$ V_{top} = V_{01}(1 - d_x) + V_{11}d_x $$
   $$ V(x, y) = V_{bottom}(1 - d_y) + V_{top}d_y $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bilinear_interpolate(values, resX, resY, x, y):
    """
    Perform bilinear interpolation on a 2D grid.
    
    Args:
        values: 1D array of size resX * resY.
        resX: Width of the grid.
        resY: Height of the grid.
        x: Query x-coordinate.
        y: Query y-coordinate.
        
    Returns:
        Interpolated value.
    """
    # 1. Compute integer bounds and fractional parts (matches [S6, unofficial] logic)
    x1 = int(x)
    dx = x - x1
    if x < 0:
        x1 = -int(-x) - 1
        dx = x - x1

    y1 = int(y)
    dy = y - y1
    if y < 0:
        y1 = -int(-y) - 1
        dy = y - y1

    # 2. Clamp to valid range if outside boundaries [G]
    x1 = max(0, min(x1, resX - 1))
    y1 = max(0, min(y1, resY - 1))

    # 3. Fetch neighbors using 1D layout: index = x * resY + y [S6, unofficial]
    idx_00 = x1 * resY + y1
    idx_10 = (x1 + 1) * resY + y1 if x1 + 1 < resX else idx_00
    idx_01 = x1 * resY + (y1 + 1) if y1 + 1 < resY else idx_00
    idx_11 = (x1 + 1) * resY + (y1 + 1) if (x1 + 1 < resX and y1 + 1 < resY) else idx_00

    V_00 = values[idx_00]
    V_10 = values[idx_10]
    V_01 = values[idx_01]
    V_11 = values[idx_11]

    # 4. Interpolate
    V_bottom = V_00 * (1 - dx) + V_10 * dx
    V_top = V_01 * (1 - dx) + V_11 * dx
    return V_bottom * (1 - dy) + V_top * dy
```

**Parameters:**
- `align_corners`: Boolean determining coordinate alignment. If `True`, input and output tensor corners are aligned to pixel centers; if `False`, they are aligned to pixel edges [S4, S8, unofficial].
- `padding_mode`: Boundary handling strategy. Value: `"zeros"` (or other supported modes) [S8, unofficial].

**Edge Cases and Pitfalls:**
- **Negative Indices:** Incorrect handling of negative coordinates can lead to out-of-bounds access or incorrect wrapping. A documented bug in some implementations uses the wrong dimension for modulo operations when wrapping negative x-coordinates (e.g., applying `resY` instead of `resX`) [S6, unofficial].
- **Boundary Conditions:** Values at the grid edges require explicit handling (clamping, wrapping, or padding) to avoid undefined behavior [S6, unofficial].
- **Precision:** Floating-point precision errors in coordinate calculation can cause sampling of adjacent pixels if flooring/ceiling logic is mishandled [S6, unofficial].

## Subtopics

### [Optimized Trilinear Reconstruction](2437-optimized-trilinear-reconstruction.md)
Accelerates the trilinear interpolation of color and opacity values during direct volume ray casting by utilizing SIMD processor extensions such as MMX, SSE, or 3DNow!. It builds on bilinear interpolation by extending the operation to three dimensions while optimizing arithmetic and memory operations for specific hardware capabilities.

### [Multi-Dimensional Interpolation Extension](2784-multi-dimensional-interpolation-extension.md)
Extends one-dimensional reconstruction kernels to three dimensions for volume rendering. It constructs a 3D filter from a 1D base kernel $h'$ to correctly reconstruct signals on a regular 3D mesh, setting itself apart from radial extension methods by avoiding sample frequency ripple artifacts that occur with rotationally-symmetric kernels on regular grids.

### [POCS Interpolation Hardware Implementation](6377-pocs-interpolation-hardware-implementation.md)
A hardware architecture for implementing Piecewise Constant (POCS) interpolation filters in volume rendering, designed to overcome the quality limitations of standard tri-linear interpolation while remaining feasible within VLSI constraints. It distinguishes itself from bilinear interpolation by supporting larger kernel sizes (up to $5 \times 5 \times 5$) using lookup tables and specialized memory banking to handle high data throughput.

### Linear Interpolation on Polyhedra
Uses linear interpolation to scan-convert the front and back faces of each polyhedron for unstructured meshes [S9, unofficial]. It exhibits C1 discontinuities common to linear interpolation [S9, unofficial]. Linear interpolation is used because reconstruction for non-uniform sampling on unstructured meshes is not well understood and is significantly more complex than uniform sampling [S9, unofficial].

### Temporal Interpolation
Linear interpolation between two sampled flow vectors of different time steps to handle dynamic vector fields [S10, unofficial]. The interpolated vector is calculated as $M_R(v, t) = (1 - t)S_i(v) + tS_j(v)$ where $S_i(v), S_j(v)$ are vector field slices [S10, unofficial]. If the target time index $k$ is not in $[i, j]$, the equation yields extrapolation instead of interpolation [S10, unofficial].

### Hat-shaped Basis Functions
A method using hat-shaped basis functions to define the interpolant, involving actual leaf cells only on the boundary to remove ghost artifacts [S11, unofficial]. It is unclear how to perform ray-isosurface intersections with this interpolant [S11, unofficial].

### SVGS Bilinear Interpolation
A spatially varying function that divides each Gaussian surfel into four quadrants using bilinear interpolation, assigning a learnable color and opacity value to each quadrant [S12]. The color and opacity are computed as $F_c(p) = (1-u')(1-v')c_0 + (1-u')v'c_1 + u'(1-v')c_2 + u'v'c_3$ and $F_\alpha(p) = (1-u')(1-v')\alpha_0 + (1-u')v'\alpha_1 + u'(1-v')\alpha_2 + u'v'\alpha_3$, where $c_i$ and $\alpha_i$ are learnable parameters and $u', v'$ are normalized coordinates [S12]. Each Gaussian primitive stores 4 learnable colors and 4 learnable opacities [S12]. This design may cause gradient vanishing issues [S12].

### Bilinear Interpolation with Sigmoid Scaling
A spatial color variation function that uses bilinear interpolation to model intra-primitive color changes, scaled by a sigmoid function to handle coordinates outside the standard $(0,1)$ range [S12]. Coordinates $(u, v)$ are passed through $u' = \frac{1}{1 + e^{-\lambda_s u}}$, $v' = \frac{1}{1 + e^{-\lambda_s v}}$ where $\lambda_s = 5.0$ controls the changing rate [S12]. Sigmoid scaling causes pixel gradients to approach zero near the center of a quadrant, leading to gradient vanishing during sharp color transitions and preventing perfect scene fitting [S12]. Bilinear interpolation is only effective when the spatial color pattern conforms to a four-quadrant distribution [S12].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, as it involves a fixed number of arithmetic operations (4 multiplications, 3 additions) and memory accesses (4 reads) regardless of grid size [G].
- **Memory Complexity:** $O(1)$ auxiliary space. The grid storage is $O(resX \cdot resY)$ [G].
- **Strengths:** Low computational cost, smooth results compared to nearest-neighbor, and broad hardware/library support [G].
- **Limitations:** Introduces blurring (low-pass filtering) which attenuates high-frequency details [G]. It is not suitable for unstructured meshes where non-uniform sampling requires more complex reconstruction [S9, unofficial].
- **When to use:** Regular 2D grids, texture sampling, and as a foundational component of 3D trilinear interpolation [G].
- **When not to use:** Unstructured meshes (use Linear Interpolation on Polyhedra), high-frequency detail preservation (use higher-order interpolation like Tri-cubic), or when extrapolation is required (use Temporal Interpolation with care) [G].

## Background
Bilinear interpolation is a fundamental technique in computer graphics and image processing [G]. It was historically applied in visualization for determining colors within triangles when lighting is not applied [S2, unofficial]. In modern pipelines, it is used for 2D texture sampling [S4, unofficial], calculating scalar values at ray-cell face intersections in volume rendering [S5, unofficial], retrieving multi-scale features in k-planes representations [S1], and performing differentiable forward and adjoint operations in computational tomography [S8, unofficial].

## Sources
[S1] K-Planes: Explicit Radiance Fields in Space, Time, and Appearance, https://doi.org/10.1109/cvpr52729.2023.01201, fetched 2026-10-26
[S2] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S3] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S4] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S5] [PDF] Volume Rendering for Computational Fluid Dynamics: Initial Results, https://www.nas.nasa.gov/assets/nas/pdf/techreports/1991/rnr-91-026.pdf, fetched 2026-10-26 (unofficial)
[S6] mkazhdan/IsoSurfaceExtraction, https://github.com/mkazhdan/IsoSurfaceExtraction, fetched 2026-10-27 (unofficial)
[S7] PDF GPU-based Frequency Domain Volume Rendering - TU Wien, https://www.cg.tuwien.ac.at/research/publications/2004/Viola-2004-GPU/Viola-2004-GPU-Paper.pdf, fetched 2026-10-27 (unofficial)
[S8] itu-biai/torchtomo, https://github.com/itu-biai/torchtomo, fetched 2026-10-27 (unofficial)
[S9] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S10] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
[S11] PDF CPU Isosurface Ray Tracing of Adaptive Mesh Refinement Da, https://www.sci.utah.edu/~feng/assets/papers/Fen-AMR-VIS18.pdf, fetched 2026-10-26 (unofficial)
[S12] SVGS: Enhancing Gaussian Splatting Using Primitives With Spatially Varying Colors., https://doi.org/10.1109/tvcg.2026.3690745, fetched 2026-10-26
