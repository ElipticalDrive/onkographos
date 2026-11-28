# Implicit Surface Representation

Also called: Grid-based SDF, Tricubic SDF, Zero-level set surface

## Overview
This method represents an implicit surface as a grid-based Signed Distance Function (SDF) using tricubic interpolation to ensure continuous surface normals [S2]. It defines the surface geometry as the zero-level set of a scalar field $f_{implicit}(\mathbf{p})$ [S1, unofficial]. It builds on [SDF Primitive Definitions](1578-sdf-primitive-definitions.md) but replaces analytical formulas with a discrete grid evaluated via higher-order interpolation.

## How it works
**Inputs and Outputs**
*   **Input:** A 3D grid of floating-point values representing the signed distance to the surface at grid vertices [G].
*   **Output:** A continuous scalar field $f_{implicit}(\mathbf{p})$ and its gradient $\nabla f_{implicit}(\mathbf{p})$ for any query point $\mathbf{p} \in \mathbb{R}^3$ [G].

**Data Structures**
The core data structure is a 3D array (grid) of floating-point values [G]. Let $G$ be a grid of size $N_x \times N_y \times N_z$ [G]. Each vertex $(i, j, k)$ stores the distance value $d_{i,j,k}$ at the corresponding grid point [G]. The grid spacing (resolution) is uniform, denoted by $\Delta$ [G].

**Formulas**
The implicit surface $S_{implicit}$ is defined as the set of points where the function evaluates to zero [S1, unofficial]:
$$
S_{implicit} = \left\{ \begin{pmatrix} x \\ y \\ z \end{pmatrix} \in \mathbb{R}^3 \mid f_{implicit}(x, y, z) = 0 \right\}
$$

**Parameters**
*   **Grid Resolution:** Determines the spatial frequency of the SDF [G].
*   **Interpolation Order:** Tricubic (order 3) is specified to ensure continuous normals [S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_tricubic_sdf(grid, p, spacing):
    """
    Evaluates the implicit surface function and its gradient at point p
    using tricubic interpolation on a uniform grid.
    
    Args:
        grid: 3D numpy array of shape (Nx, Ny, Nz) containing distance values.
        p: 3D vector (x, y, z) of the query point.
        spacing: float, the uniform distance between grid vertices.
        
    Returns:
        value: float, the signed distance f(p).
        normal: 3D vector, the gradient df/dp (surface normal).
    """
    # 1. Map world point to grid coordinates
    gx = p[0] / spacing
    gy = p[1] / spacing
    gz = p[2] / spacing
    
    # 2. Find base cell indices and fractional offsets
    ix = int(gx)
    iy = int(gy)
    iz = int(gz)
    tx = gx - ix
    ty = gy - iy
    tz = gz - iz
    
    # 3. Boundary clamping [G]
    ix = max(0, min(ix, grid.shape[0] - 2))
    iy = max(0, min(iy, grid.shape[1] - 2))
    iz = max(0, min(iz, grid.shape[2] - 2))
    
    # 4. Retrieve 8 corner values
    corners = []
    for i in range(2):
        for j in range(2):
            for k in range(2):
                corners.append(grid[ix + i, iy + j, iz + k])
                
    # 5. Tricubic interpolation basis functions and derivatives [G]
    def h0(t): return 2*t**3 - 3*t**2 + 1
    def h1(t): return t**3 - 2*t**2 + t
    def h2(t): return -2*t**3 + 3*t**2
    def h3(t): return t**3 - t**2
    def dh0(t): return 6*t**2 - 6*t
    def dh1(t): return 3*t**2 - 4*t + 1
    def dh2(t): return -6*t**2 + 6*t
    def dh3(t): return 3*t**2 - 2*t
    
    # 6. Evaluate value and gradient via tensor product
    value = 0.0
    grad_x = 0.0
    grad_y = 0.0
    grad_z = 0.0
    
    idx = 0
    for i in range(2):
        for j in range(2):
            for k in range(2):
                vi = [h0(tx), h1(tx), h2(tx), h3(tx)][i]
                vj = [h0(ty), h1(ty), h2(ty), h3(ty)][j]
                vk = [h0(tz), h1(tz), h2(tz), h3(tz)][k]
                dvi = [dh0(tx), dh1(tx), dh2(tx), dh3(tx)][i]
                dvj = [dh0(ty), dh1(ty), dh2(ty), dh3(ty)][j]
                dvk = [dh0(tz), dh1(tz), dh2(tz), dh3(tz)][k]
                
                val = corners[idx]
                value += vi * vj * vk * val
                grad_x += (dvi / spacing) * vj * vk * val
                grad_y += vi * (dvj / spacing) * vk * val
                grad_z += vi * vj * (dvk / spacing) * val
                idx += 1
                
    normal = (grad_x, grad_y, grad_z)
    return value, normal
```

**Edge Cases**
*   **Boundary Conditions:** Points outside the grid bounds require clamping or extrapolation [G].
*   **Precision:** Floating-point errors in interpolation can cause artifacts near the zero-level set [G].

## Operating characteristics
*   **Memory:** Not given in the sources.
*   **Time:** Not given in the sources.
*   **Strengths:** Continuous surface normals enable smooth shading and differentiable rendering without explicit mesh extraction [S2].
*   **Limitations:** Memory usage scales with volume rather than surface area [G]. High resolution is needed for fine details [G].

## Background
This representation is used in differentiable rendering pipelines, specifically in the implementation based on the `cuda_ad_rgb` backend of Mitsuba 3 [S2]. It builds on the principles of [SDF Primitive Definitions](1578-sdf-primitive-definitions.md).

## Sources
[S1] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S2] Path-Space Differentiable Rendering of Implicit Surfaces, https://doi.org/10.1145/3641519.3657473, fetched 2026-10-30
