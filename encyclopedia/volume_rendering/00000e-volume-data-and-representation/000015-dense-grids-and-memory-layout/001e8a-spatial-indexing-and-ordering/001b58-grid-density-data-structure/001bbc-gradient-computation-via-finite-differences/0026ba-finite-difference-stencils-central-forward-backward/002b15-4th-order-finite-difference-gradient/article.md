# 4th-Order Finite Difference Gradient
Also called: 4th-Order Accurate Finite Difference Scheme, Grid Normal Computation via 4th-Order Finite Differences

## Overview
The 4th-Order Finite Difference Gradient computes the gradient (normal vector) of a scalar field, typically a Signed Distance Function (SDF) grid, at every voxel center using a 4th-order accurate finite difference scheme [S1, unofficial]. This method provides smooth normals essential for ray marching and shading in volume rendering [S1, unofficial]. It distinguishes itself from lower-order siblings like the [2nd-Order Finite Difference Gradient](13052-2nd-order-finite-difference-gradient.md) by employing a wider stencil and boundary extrapolation to maintain high accuracy even at domain edges, whereas the 2nd-order method uses only immediate neighbors [G].

## How it works
**Inputs and Outputs:**
- Input: A 3D scalar grid $V$ of dimensions $N_x \times N_y \times N_z$, where $V_{i,j,k}$ is the value at voxel $(i,j,k)$ [S1, unofficial].
- Input: `voxel_size` $h$, the physical spacing between voxels (assumed isotropic) [S1, unofficial].
- Output: Three 3D grids $G_x, G_y, G_z$ representing the partial derivatives $\frac{\partial V}{\partial x}, \frac{\partial V}{\partial y}, \frac{\partial V}{\partial z}$ [S1, unofficial].

**Data Structures:**
- The grid $V$ is stored as a dense 3D array or tensor [G].
- Boundary handling requires accessing indices outside $[0, N-1]$ [G].

**Algorithm:**
The gradient is computed by approximating the central difference using forward and backward approximations that are extended to the boundaries using a 4th-order extrapolation formula [S1, unofficial].

1.  **Define Boundary Indices:**
    Let $n_x = N_x - 1$, $n_y = N_y - 1$, $n_z = N_z - 1$ [S1, unofficial].

2.  **Compute X-Axis Gradient:**
    -   Compute forward approximation $X_1$:
        -   Interior: $V_{i+1, j, k}$ for $i \in [0, n_x-1]$.
        -   Boundary ($i=n_x$): Extrapolate $V_{n_x+1, j, k}$ using the 4th-order formula:
            $$ V_{n_x+1} = 3V_{n_x} - 3V_{n_x-1} + V_{n_x-2} $$
        -   Concatenate: $X_1 = [V_{1,\cdot,\cdot}, \dots, V_{n_x,\cdot,\cdot}, V_{n_x+1,\cdot,\cdot}]$.
    -   Compute backward approximation $X_2$:
        -   Boundary ($i=0$): Extrapolate $V_{-1, j, k}$ using the 4th-order formula:
            $$ V_{-1} = -3V_{0} + 3V_{1} + V_{2} $$
        -   Interior: $V_{i-1, j, k}$ for $i \in [1, n_x]$.
        -   Concatenate: $X_2 = [V_{-1,\cdot,\cdot}, V_{0,\cdot,\cdot}, \dots, V_{n_x-1,\cdot,\cdot}]$.
    -   Calculate $G_x$:
        $$ G_x = \frac{X_1 - X_2}{2h} $$

3.  **Compute Y-Axis Gradient:**
    -   Repeat logic for $Y_1$ and $Y_2$ along the $y$-dimension.
    -   $Y_1$ interior: $V_{i, j+1, k}$. Boundary $j=n_y$: $V_{i, n_y+1, k} = 3V_{i, n_y, k} - 3V_{i, n_y-1, k} + V_{i, n_y-2, k}$.
    -   $Y_2$ interior: $V_{i, j-1, k}$. Boundary $j=0$: $V_{i, -1, k} = -3V_{i, 0, k} + 3V_{i, 1, k} + V_{i, 2, k}$.
    -   Calculate $G_y = \frac{Y_1 - Y_2}{2h}$.

4.  **Compute Z-Axis Gradient:**
    -   Repeat logic for $Z_1$ and $Z_2$ along the $z$-dimension.
    -   $Z_1$ interior: $V_{i, j, k+1}$. Boundary $k=n_z$: $V_{i, j, n_z+1} = 3V_{i, j, n_z} - 3V_{i, j, n_z-1} + V_{i, j, n_z-2}$.
    -   $Z_2$ interior: $V_{i, j, k-1}$. Boundary $k=0$: $V_{i, j, -1} = -3V_{i, j, 0} + 3V_{i, j, 1} + V_{i, j, 2}$.
    -   Calculate $G_z = \frac{Z_1 - Z_2}{2h}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_4th_order_gradient(grid, voxel_size):
    """
    Computes the 4th-order finite difference gradient of a 3D grid.
    
    Args:
        grid: 3D numpy array of shape (Nx, Ny, Nz)
        voxel_size: float, physical spacing h
        
    Returns:
        gx, gy, gz: 3D arrays of shape (Nx, Ny, Nz)
    """
    Nx, Ny, Nz = grid.shape
    gx = np.zeros_like(grid)
    gy = np.zeros_like(grid)
    gz = np.zeros_like(grid)
    
    # X-axis gradient
    x1_interior = grid[1:, :, :]
    x1_boundary = (3 * grid[Nx-1, :, :] - 3 * grid[Nx-2, :, :] + grid[Nx-3, :, :]).reshape(1, Ny, Nz)
    X1 = np.concatenate([x1_interior, x1_boundary], axis=0)
    
    x2_interior = grid[:-1, :, :]
    x2_boundary = (-3 * grid[0, :, :] + 3 * grid[1, :, :] + grid[2, :, :]).reshape(1, Ny, Nz)
    X2 = np.concatenate([x2_boundary, x2_interior], axis=0)
    
    gx = (X1 - X2) / (2 * voxel_size)
    
    # Y-axis gradient
    y1_interior = grid[:, 1:, :]
    y1_boundary = (3 * grid[:, Ny-1, :] - 3 * grid[:, Ny-2, :] + grid[:, Ny-3, :]).reshape(Nx, 1, Nz)
    Y1 = np.concatenate([y1_interior, y1_boundary], axis=1)
    
    y2_interior = grid[:, :-1, :]
    y2_boundary = (-3 * grid[:, 0, :] + 3 * grid[:, 1, :] + grid[:, 2, :]).reshape(Nx, 1, Nz)
    Y2 = np.concatenate([y2_boundary, y2_interior], axis=1)
    
    gy = (Y1 - Y2) / (2 * voxel_size)
    
    # Z-axis gradient
    z1_interior = grid[:, :, 1:]
    z1_boundary = (3 * grid[:, :, Nz-1] - 3 * grid[:, :, Nz-2] + grid[:, :, Nz-3]).reshape(Nx, Ny, 1)
    Z1 = np.concatenate([z1_interior, z1_boundary], axis=2)
    
    z2_interior = grid[:, :, :-1]
    z2_boundary = (-3 * grid[:, :, 0] + 3 * grid[:, :, 1] + grid[:, :, 2]).reshape(Nx, Ny, 1)
    Z2 = np.concatenate([z2_boundary, z2_interior], axis=2)
    
    gz = (Z1 - Z2) / (2 * voxel_size)
    
    return gx, gy, gz
```

**Parameters:**
- `voxel_size`: The physical size of a single voxel. Used to normalize the finite difference result to physical units [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Small Grids:** If any dimension has fewer than 3 voxels, the extrapolation formula $V_{-1} = -3V_0 + 3V_1 + V_2$ or $V_{n+1} = 3V_n - 3V_{n-1} + V_{n-2}$ requires indices that do not exist. The method assumes $N \ge 3$ for all dimensions [G].
- **Precision:** The extrapolation coefficients (3, -3, 1) can amplify noise in the grid values, particularly near boundaries [G].

## Subtopics

### [2nd-Order Finite Difference Gradient](13052-2nd-order-finite-difference-gradient.md)
Uses only immediate neighbors for gradient estimation. Choose this when computational cost is critical and high-frequency noise is not a concern, as it offers a trade-off between cost and smoothness compared to the 4th-order method [G].

## Operating characteristics
- **Time Complexity:** $O(N_x N_y N_z)$, as each voxel is processed a constant number of times [G].
- **Memory Complexity:** $O(N_x N_y N_z)$ for storing the input grid and three output gradient grids [G].
- **Performance:** Provides smooth normals beneficial for ray marching stability and shading quality [S1, unofficial].
- **Strengths:** High accuracy (4th-order) even at boundaries; smooth gradients [G].
- **Limitations:** Requires at least 3 voxels per dimension; sensitive to noise due to extrapolation coefficients; higher computational overhead than 2nd-order [G].
- **When to use:** When high-quality normals are required for SDF rendering and the grid resolution is sufficient ($N \ge 3$) [G].
- **When not to use:** For very low-resolution grids or when noise robustness is prioritized over accuracy [G].

## Background
The method is implemented in the context of differentiable rendering of Signed Distance Functions, specifically in the SDFDiff framework [S1, unofficial]. It was presented in a 2020 paper (CVPR 2020) by Yue Jiang et al., aiming to provide accurate gradients for optimizing SDFs from images [S1, unofficial]. The 4th-order finite difference scheme is a standard numerical analysis technique adapted here for grid-based volume data [G].

## Sources
[S1] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
