# 4th-Order Finite Difference Gradient

Also called: 4th-Order Accurate Finite Difference Scheme, Grid Normal Computation via 4th-Order Finite Differences

## Overview
This method computes the gradient (normal vector) of a Signed Distance Function (SDF) grid at every voxel center using a 4th-order accurate finite difference scheme [S1, unofficial]. It provides smooth normals essential for ray marching and shading in volume rendering [S1, unofficial]. It sets itself apart from the standard central difference (2nd-order) by using a wider stencil and boundary extrapolation to maintain high-order accuracy even at volume edges, unlike the [Sobel Operator](1416-sobel-operator.md) which uses fixed convolution kernels, or the [Forward Difference Gradient](3362-forward-difference-gradient.md) which is only 1st-order accurate.

## How it works
**Inputs:**
- `grid`: A 3D tensor of shape $(N_x, N_y, N_z)$ containing scalar SDF values at voxel centers.
- `voxel_size`: The physical spacing $h$ between voxels (assumed isotropic).
- `grid_res_x`, `grid_res_y`, `grid_res_z`: The integer dimensions of the grid.

**Outputs:**
- A list of three 3D tensors `[grid_normal_x, grid_normal_y, grid_normal_z]`, each of shape $(N_x, N_y, N_z)$, representing the partial derivatives $\frac{\partial V}{\partial x}$, $\frac{\partial V}{\partial y}$, and $\frac{\partial V}{\partial z}$.

**Data Structures:**
- The grid is stored as a dense 3D array. No additional auxiliary structures are required beyond the input grid and temporary tensors for the shifted slices.

**Steps:**
1.  Define the largest valid index for each dimension: $n_x = N_x - 1$, $n_y = N_y - 1$, $n_z = N_z - 1$.
2.  Compute the x-axis gradient components $X_1$ and $X_2$:
    -   $X_1$ is the forward-shifted grid concatenated with an extrapolated boundary value at the upper bound.
    -   $X_2$ is an extrapolated boundary value at the lower bound concatenated with the backward-shifted grid.
3.  Calculate the x-axis normal: $G_x = \frac{X_1 - X_2}{2 \cdot \text{voxel\_size}}$.
4.  Repeat steps 2-3 for the y-axis (shifting along dimension 1) and z-axis (shifting along dimension 2).

**Formulas:**
The core approximation for an interior voxel $i$ in a dimension with spacing $h$ is the central difference:
$$ \frac{\partial V}{\partial x_i} \approx \frac{V_{i+1} - V_{i-1}}{2h} $$

At the boundaries, 4th-order accuracy is maintained by extrapolating the ghost points using the 4th-order finite difference boundary condition [S1, unofficial]:
$$ V_{n+1} = 3V_n - 3V_{n-1} + V_{n-2} $$
$$ V_{-1} = -3V_0 + 3V_1 + V_2 $$

For the x-axis:
$$ X_1 = [V_{1}, V_{2}, \dots, V_{n_x}, \underbrace{3V_{n_x} - 3V_{n_x-1} + V_{n_x-2}}_{\text{extrapolated } V_{n_x+1}}] $$
$$ X_2 = [\underbrace{-3V_0 + 3V_1 + V_2}_{\text{extrapolated } V_{-1}}, V_0, V_1, \dots, V_{n_x-1}] $$
$$ G_x = \frac{X_1 - X_2}{2h} $$

**Parameters:**
- `voxel_size`: Physical size of a single voxel. Typical values depend on the scene scale, e.g., $0.01$ to $1.0$ units.

**Edge Cases and Pitfalls:**
-   **Grid Resolution:** The stencil requires at least 3 voxels in each dimension to compute the boundary extrapolation ($V_{n-2}$ and $V_2$). If $N < 3$, the method fails or requires fallback to lower-order schemes [G].
-   **Precision:** The extrapolation coefficients ($3, -3, 1$) can amplify noise in the SDF grid at boundaries. Smoothing the input SDF before gradient computation is often recommended [G].
-   **Boundary Conditions:** This specific implementation uses one-sided extrapolation. Other methods might use zero-padding or clamping, which reduces accuracy at the boundary to 1st or 2nd order.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_4th_order_gradient(grid, voxel_size, nx, ny, nz):
    """
    Computes the gradient of a 3D SDF grid using 4th-order finite differences.
    
    Args:
        grid: 3D numpy/torch array of shape (nx, ny, nz)
        voxel_size: float, physical spacing h
        nx, ny, nz: int, grid dimensions
        
    Returns:
        gx, gy, gz: 3D arrays of shape (nx, ny, nz)
    """
    # Initialize output arrays
    gx = np.zeros_like(grid)
    gy = np.zeros_like(grid)
    gz = np.zeros_like(grid)
    
    # --- X-axis ---
    # Interior: central difference
    # gx[1:-1, :, :] = (grid[2:, :, :] - grid[:-2, :, :]) / (2 * voxel_size)
    # Note: The source implementation uses a concatenation approach that effectively
    # computes (X1 - X2) / 2h where X1 is shifted forward and X2 is shifted backward.
    # This is equivalent to central difference for interior points.
    
    # Boundary extrapolation for x-axis
    # Upper boundary (index nx): V_{nx+1} = 3*V_{nx} - 3*V_{nx-1} + V_{nx-2}
    # Lower boundary (index 0): V_{-1} = -3*V_0 + 3*V_1 + V_2
    
    # Construct X1: [V_1 ... V_{nx}, V_{nx+1}]
    # Construct X2: [V_{-1}, V_0 ... V_{nx-1}]
    
    # Using slicing for clarity (assuming numpy-like indexing)
    # Interior part of X1: grid[1:, :, :]
    # Interior part of X2: grid[:-1, :, :]
    
    # Extrapolated values
    x_upper_extrap = 3 * grid[nx-1, :, :] - 3 * grid[nx-2, :, :] + grid[nx-3, :, :]
    x_lower_extrap = -3 * grid[0, :, :] + 3 * grid[1, :, :] + grid[2, :, :]
    
    # Concatenate
    X1 = np.concatenate([grid[1:, :, :], x_upper_extrap[np.newaxis, :, :]], axis=0)
    X2 = np.concatenate([x_lower_extrap[np.newaxis, :, :], grid[:-1, :, :]], axis=0)
    
    gx = (X1 - X2) / (2 * voxel_size)
    
    # --- Y-axis ---
    y_upper_extrap = 3 * grid[:, ny-1, :] - 3 * grid[:, ny-2, :] + grid[:, ny-3, :]
    y_lower_extrap = -3 * grid[:, 0, :] + 3 * grid[:, 1, :] + grid[:, 2, :]
    
    Y1 = np.concatenate([grid[:, 1:, :], y_upper_extrap[np.newaxis, :, :]], axis=1)
    Y2 = np.concatenate([y_lower_extrap[np.newaxis, :, :], grid[:, :-1, :]], axis=1)
    
    gy = (Y1 - Y2) / (2 * voxel_size)
    
    # --- Z-axis ---
    z_upper_extrap = 3 * grid[:, :, nz-1] - 3 * grid[:, :, nz-2] + grid[:, :, nz-3]
    z_lower_extrap = -3 * grid[:, :, 0] + 3 * grid[:, :, 1] + grid[:, :, 2]
    
    Z1 = np.concatenate([grid[:, :, 1:], z_upper_extrap[np.newaxis, :, :]], axis=2)
    Z2 = np.concatenate([z_lower_extrap[np.newaxis, :, :], grid[:, :, :-1]], axis=2)
    
    gz = (Z1 - Z2) / (2 * voxel_size)
    
    return gx, gy, gz
```

## Subtopics

### Sobel Operator
Uses fixed 3x3x3 convolution kernels. Faster but lower order accuracy. Choose for real-time performance where smoothness is less critical. [Siblings](1416-sobel-operator.md)

### Jacobian Estimation via Central Differences
Focuses on the full Jacobian matrix rather than just the gradient vector. Useful for deformation analysis. [Siblings](2650-jacobian-estimation-via-central-differences.md)

### Forward Difference Gradient
Uses only forward neighbors. 1st-order accurate. Simpler to implement but less smooth. [Siblings](3362-forward-difference-gradient.md)

### Gauss-Seidel with Successive Over-Relaxation (SOR)
An iterative solver for linear systems, not a direct gradient estimator. Used for solving Poisson equations for SDF reconstruction. [Siblings](6904-gauss-seidel-with-successive-over-relaxation-sor.md)

## Operating characteristics
-   **Time Complexity**: $O(N)$ where $N = N_x N_y N_z$ is the number of voxels. Each voxel is accessed a constant number of times.
-   **Memory Complexity**: $O(N)$ for the input grid and $O(N)$ for the output gradients. Temporary tensors for concatenation add $O(N)$ overhead.
-   **Measured Performance**: Performance depends on the implementation (CPU vs GPU). The tensor concatenation approach in [S1, unofficial] is optimized for GPU execution via PyTorch.
-   **Strengths**: High-order accuracy (4th-order) provides very smooth normals, reducing artifacts in ray marching. Maintains accuracy at boundaries.
-   **Limitations**: Requires at least 3 voxels per dimension. Boundary extrapolation can be sensitive to noise. More complex to implement than simple central differences.
-   **When to Use**: When high-quality normals are critical for rendering fidelity, such as in high-resolution SDF rendering or neural rendering pipelines.
-   **When Not to Use**: For very low-resolution grids (< 3 voxels) or when computational resources are extremely constrained and lower-order accuracy is acceptable.

## Background
The method is implemented in the `SDFDiff` repository by Yue Jiang et al. [S1, unofficial]. It is used in the context of differentiable rendering of Signed Distance Functions, where smooth gradients are necessary for stable optimization and high-quality visualization. The 4th-order boundary extrapolation formula is a standard technique in numerical analysis for maintaining high-order accuracy in finite difference schemes on bounded domains.

## Sources
[S1] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
