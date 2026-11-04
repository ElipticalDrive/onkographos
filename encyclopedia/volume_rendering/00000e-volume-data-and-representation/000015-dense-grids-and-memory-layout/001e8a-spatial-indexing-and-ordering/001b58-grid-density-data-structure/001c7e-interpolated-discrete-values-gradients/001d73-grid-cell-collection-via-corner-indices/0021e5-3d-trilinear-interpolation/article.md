# 3D Trilinear Interpolation

Also called: 3D Linear Interpolation, Trilinear Interpolant, Trilinear Interpolation (Standard), Trilinear Interpolation on Regular Mesh, Trilinear Voxel Interpolation, Voxel Trilinear Interpolation

## Overview
3D Trilinear Interpolation estimates the value of a scalar or vector field at an arbitrary point $P=(x,y,z)$ within a grid cell by combining the values at the cell's eight corners [S1][S2, S3, unofficial]. It solves the problem of sampling dense volumetric data at sub-voxel resolution, providing smoother results than nearest-neighbor sampling [S4, S5, unofficial]. It is distinguished from higher-order methods like [Cubic Volume Interpolation](4278-cubic-volume-interpolation.md) by its use of only 8 neighbors and linear basis functions, which ensures $C^0$ continuity on regular rectilinear grids but lacks the derivative continuity of cubic methods [S1][S2, unofficial].

## How it works
**Inputs:**
- A point $P = (x, y, z)$ in world or grid coordinates [S1][S6, unofficial].
- A dense 3D grid of values $V$ with dimensions $N_x \times N_y \times N_z$ [S3, S7, unofficial].
- Grid origin $O = (o_x, o_y, o_z)$ and voxel spacings $D = (d_x, d_y, d_z)$ [S6, unofficial]. If the grid is unit-spaced and origin-aligned, these are implicit.

**Outputs:**
- A scalar value $F(P)$ or vector field value at $P$ [S1][S7, unofficial].

**Data Structures:**
- The grid $V$ is typically stored as a contiguous array or multi-dimensional array [S3, unofficial]. For a grid with dimensions $N_x, N_y, N_z$, access is often $V[z][y][x]$ or flattened with stride-based indexing [S3, unofficial].
- Corner indices are integers $x_0, x_1, y_0, y_1, z_0, z_1$ where $x_1 = x_0 + 1$, etc. [S5, unofficial].

**Steps:**

1. **Coordinate Normalization:** If the input coordinates are in world space, normalize them to voxel space.
   $$ xyz = \frac{P - O}{D} $$
   [S6, unofficial]

2. **Cell Identification:** Determine the integer coordinates of the lower corner of the enclosing voxel.
   $$ x_0 = \lfloor x \rfloor, \quad y_0 = \lfloor y \rfloor, \quad z_0 = \lfloor z \rfloor $$
   [S5, S6, unofficial]
   The upper corner indices are:
   $$ x_1 = x_0 + 1, \quad y_1 = y_0 + 1, \quad z_1 = z_0 + 1 $$
   [S5, unofficial]

3. **Fractional Parts (Weights):** Compute the fractional offset within the voxel.
   $$ \alpha = x - x_0, \quad \beta = y - y_0, \quad \gamma = z - z_0 $$
   [S5, unofficial]
   Alternatively defined as ratios $R_x, R_y, R_z \in [0, 1]$ [S7, unofficial].

4. **Boundary Check:** If any coordinate is outside the valid grid range (e.g., $x < 0$ or $x \ge N_x$), return a default value (typically 0) or clamp indices [S5, unofficial]. Note: Clamping indices may cause artifacts at boundaries; returning 0 is common for sparse or bounded volumes [S5, unofficial].

5. **Corner Value Retrieval:** Retrieve the 8 corner values from the grid $V$. The indexing order must be consistent. A common mapping (matching [S6, unofficial]) is:
   - $v_{000} = V(x_0, y_0, z_0)$
   - $v_{100} = V(x_1, y_0, z_0)$
   - $v_{010} = V(x_0, y_1, z_0)$
   - $v_{110} = V(x_1, y_1, z_0)$
   - $v_{001} = V(x_0, y_0, z_1)$
   - $v_{101} = V(x_1, y_0, z_1)$
   - $v_{011} = V(x_0, y_1, z_1)$
   - $v_{111} = V(x_1, y_1, z_1)$
   [S5, S6, unofficial]

6. **Interpolation:** Perform linear interpolation along each axis [S4, unofficial]. This can be done sequentially or via the explicit polynomial form [S1].
   
   *Sequential Method (X, then Y, then Z):*
   $$
   \begin{aligned}
   V_{z0} &= (1-\alpha)(1-\beta)v_{000} + \alpha(1-\beta)v_{100} + (1-\alpha)\beta v_{010} + \alpha\beta v_{110} \\
   V_{z1} &= (1-\alpha)(1-\beta)v_{001} + \alpha(1-\beta)v_{101} + (1-\alpha)\beta v_{011} + \alpha\beta v_{111} \\
   F(P) &= (1-\gamma)V_{z0} + \gamma V_{z1}
   \end{aligned}
   $$
   [S3, S5, unofficial]

   *Explicit Polynomial Form:*
   The value is approximated by:
   $$ F(P) = a + bx + cy + dz + exy + fxz + gyz + hxyz $$
   where constants $a \dots h$ are determined by the corner values [S1]. For a unit cube aligned with axes starting at $(x_0, y_0, z_0)$, the weights for the 8 corners are products of $(1-\alpha)$ or $\alpha$, $(1-\beta)$ or $\beta$, and $(1-\gamma)$ or $\gamma$ [S5, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def trilinear_interpolation(grid, x, y, z, origin=(0.0, 0.0, 0.0), spacing=(1.0, 1.0, 1.0)):
    """
    Performs 3D trilinear interpolation at point (x, y, z).
    grid: 3D array-like, shape (Nx, Ny, Nz).
    x, y, z: float coordinates in world space.
    origin: tuple (ox, oy, oz) of grid origin in world space.
    spacing: tuple (dx, dy, dz) of voxel dimensions.
    """
    # 1. Normalize to voxel space
    vx = (x - origin[0]) / spacing[0]
    vy = (y - origin[1]) / spacing[1]
    vz = (z - origin[2]) / spacing[2]

    # 2. Boundary check
    if (vx < 0 or vy < 0 or vz < 0 or 
        vx >= grid.shape[0] or vy >= grid.shape[1] or vz >= grid.shape[2]):
        return 0.0

    # 3. Compute floor coordinates and fractional parts
    x0 = math.floor(vx)
    y0 = math.floor(vy)
    z0 = math.floor(vz)
    
    x1 = x0 + 1
    y1 = y0 + 1
    z1 = z0 + 1
    
    alpha = vx - x0
    beta = vy - y0
    gamma = vz - z0
    
    # 4. Retrieve corner values
    v000 = grid[x0, y0, z0]
    v100 = grid[x1, y0, z0]
    v010 = grid[x0, y1, z0]
    v110 = grid[x1, y1, z0]
    v001 = grid[x0, y0, z1]
    v101 = grid[x1, y0, z1]
    v011 = grid[x0, y1, z1]
    v111 = grid[x1, y1, z1]
    
    # 5. Interpolate
    # Interpolate along X
    v00 = (1 - alpha) * v000 + alpha * v100
    v01 = (1 - alpha) * v001 + alpha * v101
    v10 = (1 - alpha) * v010 + alpha * v110
    v11 = (1 - alpha) * v011 + alpha * v111
    
    # Interpolate along Y
    v0 = (1 - beta) * v00 + beta * v10
    v1 = (1 - beta) * v01 + beta * v11
    
    # Interpolate along Z
    result = (1 - gamma) * v0 + gamma * v1
    
    return result
```

## Subtopics

### [Trilinear Interpolation (Compressed/Linked)](2629-trilinear-interpolation-compressed-linked.md)
This method performs trilinear interpolation on a compressed sparse grid using a links array to map voxel indices to compressed value indices, handling boundary checks via masking or a zero-value hack. It builds on [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md) by adding a link lookup step that translates grid coordinates to compressed indices while retaining the regular 8-corner neighborhood structure.

### [Cubic Volume Interpolation](4278-cubic-volume-interpolation.md)
Cubic Volume Interpolation extends standard trilinear interpolation by using cubic mixture weights for each axis to sample a 3D texture at 8 corners defined by those weights. It provides higher-order continuity and smoother gradients compared to linear methods, at the cost of increased computational complexity and memory bandwidth usage.

### Trilinear Gradient Interpolation
Computes the gradient of the field at point $P$ by interpolating the gradients stored at the 8 corner points of the enclosing voxel using the same trilinear weights [S8, unofficial]. The result is typically normalized from $[-1, 1]$ to $[0, 1]$ if the gradient data is encoded in that range [S8, unofficial].

### Grid Querying with Interpolation
In some volume rendering pipelines (e.g., EgoNeRF), trilinear interpolation is used to query density $\sigma$ and color $c$ from feature grids along camera rays [S9]. To reduce computational burden, this can be replaced by linear/bilinear interpolation of vector/matrix factors [S9].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample point, involving a constant number of arithmetic operations and memory accesses [S4, S6, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space per sample [G]. The grid storage is $O(N_x N_y N_z)$ [S3, unofficial].
- **Performance:** Significantly faster than tricubic interpolation, which requires 64 samples [S2, unofficial]. On rectilinear grids, it is efficient; on curvilinear grids, precalculation of coefficients can take ~110 seconds, and intersection testing becomes the bottleneck [S1].
- **Strengths:** Simple implementation, low computational cost, $C^0$ continuity on regular grids [S1][S5, unofficial].
- **Limitations:** Artifacts such as "3D crosses" or diamonds appear at large magnification factors due to the linear kernel [S4, unofficial]. Continuity is not guaranteed for irregular, rotated, or curvilinear grids [S1]. Degenerate cases may occur where the trilinear function cannot be fit [S1].

## Background
Trilinear interpolation was developed as a standard method for volume rendering of curvilinear and rectilinear volumes [S1]. It is used in raycasting renderers to sample dense volumetric data at sub-voxel resolution [S3, S5, unofficial]. The method assumes the underlying function varies linearly along edges, quadratically across faces, and cubically within the cell [S1]. It is a fundamental component of raycasting renderers and is used in modern neural rendering pipelines like EgoNeRF for querying feature grids [S9].

## Sources
[S1] Direct volume rendering of curvilinear volumes, https://doi.org/10.1145/99308.99318, fetched 2026-10-28
[S2] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
[S3] kkhese/Volume-Rendering, https://github.com/kkhese/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S4] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S5] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S6] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
[S7] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
[S8] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
[S9] Balanced Spherical Grid for Egocentric View Synthesis, https://doi.org/10.1109/cvpr52729.2023.01592, fetched 2026-10-29
