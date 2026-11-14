# 4th-Order Finite Difference Gradient

Also called: 4th-Order Accurate Finite Difference Scheme, Grid Normal Computation via 4th-Order Finite Differences

## Overview
The 4th-Order Finite Difference Gradient computes the gradient vector (normal) of a scalar field, typically a Signed Distance Function (SDF) grid, at every voxel center using a 4th-order accurate finite difference scheme [S1, unofficial]. This method provides smoother normals for ray marching and shading compared to lower-order approximations. It distinguishes itself from its sibling [2nd-Order Finite Difference Gradient](13052-2nd-order-finite-difference-gradient.md) by employing a wider stencil and specific boundary extrapolation to maintain 4th-order accuracy across the entire domain, including boundaries, whereas the 2nd-order method uses only immediate neighbors [S3].

## How it works
**Inputs and Outputs:**
- Input: A 3D scalar grid $V$ of dimensions $N_x \times N_y \times N_z$, and a scalar `voxel_size` $h$ representing the physical spacing between voxels.
- Output: Three 3D grids $G_x, G_y, G_z$ representing the partial derivatives $\frac{\partial V}{\partial x}, \frac{\partial V}{\partial y}, \frac{\partial V}{\partial z}$ at each voxel center.

**Data Structures:**
- The grid $V$ is stored as a dense 3D array or tensor. Indices are 0-based: $i \in [0, N_x-1]$, $j \in [0, N_y-1]$, $k \in [0, N_z-1]$.

**Algorithm Steps:**
The gradient along each axis is computed by approximating the central difference using forward and backward extrapolations at the boundaries to preserve 4th-order accuracy. For an axis with length $N$ and largest index $n = N-1$:

1.  **Compute Forward Difference Approximation ($F$):**
    Create a shifted version of the grid where the interior values are shifted by one index. At the boundary (index $n$), extrapolate the value using the 4th-order boundary formula:
    $$ V_{n+1} = 3V_n - 3V_{n-1} + V_{n-2} $$
    The forward difference vector $F$ consists of the interior values $V_{i+1}$ for $i < n$, and the extrapolated value at $i=n$.

2.  **Compute Backward Difference Approximation ($B$):**
    Create a shifted version of the grid where the interior values are shifted by one index in the opposite direction. At the boundary (index $0$), extrapolate the value using the symmetric 4th-order boundary formula:
    $$ V_{-1} = -3V_0 + 3V_1 + V_2 $$
    The backward difference vector $B$ consists of the extrapolated value at $i=0$, and the interior values $V_{i-1}$ for $i > 0$.

3.  **Calculate Gradient:**
    The partial derivative along the axis is the difference between the forward and backward approximations, divided by $2h$:
    $$ G = \frac{F - B}{2h} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_4th_order_gradient(grid, voxel_size):
    """
    Computes the 4th-order gradient of a 3D grid.
    grid: 3D array/tensor of shape (Nx, Ny, Nz)
    voxel_size: float, physical spacing h
    Returns: tuple of 3 arrays (Gx, Gy, Gz)
    """
    Nx, Ny, Nz = grid.shape
    
    # --- X-Axis Gradient ---
    n_x = Nx - 1
    # Interior forward shift: grid[1:Nx, :, :]
    # Boundary extrapolation at index n_x: 3*grid[n_x] - 3*grid[n_x-1] + grid[n_x-2]
    # X_1 is the "forward" shifted grid (values at i+1)
    # Note: In the source, X_1 corresponds to the right-shifted values used in central diff numerator
    # Specifically: (V[i+1] - V[i-1]) / 2h
    # X_1 contains V[1...n_x] and the extrapolated V[n_x+1]
    x_forward_interior = grid[1:, :, :]
    x_boundary_extrap = 3 * grid[n_x, :, :] - 3 * grid[n_x-1, :, :] + grid[n_x-2, :, :]
    x_boundary_extrap = x_boundary_extrap.reshape(1, 1, 1) # Reshape for concatenation if needed by framework
    # Concatenate along axis 0. X_1 has shape (Nx, Ny, Nz)
    # X_1[i] = V[i+1] for i < n_x, X_1[n_x] = V[n_x+1]
    X_1 = concatenate_along_axis_0(x_forward_interior, x_boundary_extrap)
    
    # X_2 contains V[-1] and V[0...n_x-1]
    # X_2[0] = V[-1], X_2[i] = V[i-1] for i > 0
    x_boundary_neg_extrap = -3 * grid[0, :, :] + 3 * grid[1, :, :] + grid[2, :, :]
    x_boundary_neg_extrap = x_boundary_neg_extrap.reshape(1, 1, 1)
    x_backward_interior = grid[:n_x, :, :]
    X_2 = concatenate_along_axis_0(x_boundary_neg_extrap, x_backward_interior)
    
    Gx = (X_1 - X_2) / (2 * voxel_size)
    
    # --- Y-Axis Gradient ---
    n_y = Ny - 1
    y_forward_interior = grid[:, 1:, :]
    y_boundary_extrap = 3 * grid[:, n_y, :] - 3 * grid[:, n_y-1, :] + grid[:, n_y-2, :]
    y_boundary_extrap = y_boundary_extrap.reshape(1, 1, 1)
    Y_1 = concatenate_along_axis_1(grid[:, 1:, :], y_boundary_extrap) # Concatenate along axis 1
    
    y_boundary_neg_extrap = -3 * grid[:, 0, :] + 3 * grid[:, 1, :] + grid[:, 2, :]
    y_boundary_neg_extrap = y_boundary_neg_extrap.reshape(1, 1, 1)
    y_backward_interior = grid[:, :n_y, :]
    Y_2 = concatenate_along_axis_1(y_boundary_neg_extrap, y_backward_interior)
    
    Gy = (Y_1 - Y_2) / (2 * voxel_size)
    
    # --- Z-Axis Gradient ---
    n_z = Nz - 1
    z_forward_interior = grid[:, :, 1:]
    z_boundary_extrap = 3 * grid[:, :, n_z] - 3 * grid[:, :, n_z-1] + grid[:, :, n_z-2]
    z_boundary_extrap = z_boundary_extrap.reshape(1, 1, 1)
    Z_1 = concatenate_along_axis_2(grid[:, :, 1:], z_boundary_extrap) # Concatenate along axis 2
    
    z_boundary_neg_extrap = -3 * grid[:, :, 0] + 3 * grid[:, :, 1] + grid[:, :, 2]
    z_boundary_neg_extrap = z_boundary_neg_extrap.reshape(1, 1, 1)
    z_backward_interior = grid[:, :, :n_z]
    Z_2 = concatenate_along_axis_2(z_boundary_neg_extrap, z_backward_interior)
    
    Gz = (Z_1 - Z_2) / (2 * voxel_size)
    
    return (Gx, Gy, Gz)

def concatenate_along_axis_0(a, b):
    # a is (Nx-1, Ny, Nz), b is (1, Ny, Nz) -> result (Nx, Ny, Nz)
    return stack([a, b], axis=0).squeeze(axis=0) # Simplified logic for description

def concatenate_along_axis_1(a, b):
    # a is (Nx, Ny-1, Nz), b is (Nx, 1, Nz) -> result (Nx, Ny, Nz)
    return stack([a, b], axis=1).squeeze(axis=1)

def concatenate_along_axis_2(a, b):
    # a is (Nx, Ny, Nz-1), b is (Nx, Ny, 1) -> result (Nx, Ny, Nz)
    return stack([a, b], axis=2).squeeze(axis=2)
```
*Note: The `concatenate` operations assume the framework handles broadcasting or explicit reshaping of the scalar boundary terms to match the other dimensions. The core logic is the construction of the shifted arrays.*

**Parameters:**
- `voxel_size` ($h$): The physical size of a single voxel. Used to normalize the result to physical units [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Grid Resolution:** The stencil requires at least 3 voxels in each dimension ($N \ge 3$) to compute the boundary extrapolations ($V_{n-2}$ and $V_2$). If $N < 3$, the method is undefined [G].
- **Boundary Accuracy:** The extrapolation formulas $V_{n+1} = 3V_n - 3V_{n-1} + V_{n-2}$ and $V_{-1} = -3V_0 + 3V_1 + V_2$ are derived to maintain 4th-order accuracy at the boundaries for smooth functions. If the field is not smooth near boundaries, accuracy degrades [S1, unofficial].

## Subtopics

### [2nd-Order Finite Difference Gradient](13052-2nd-order-finite-difference-gradient.md)
Uses immediate neighbors with 2nd-order accuracy. Choose when computational cost is critical and high-frequency noise is not a concern.

### [Central Difference Stencils](9968-central-difference-stencils.md)
General central difference methods. The 4th-order method is a specific, higher-accuracy variant of central differencing with specialized boundary handling.

## Operating characteristics
- **Time Complexity:** $O(N_x N_y N_z)$, as it requires a single pass over the grid for each axis. The constant factor is higher than 2nd-order due to the wider stencil and boundary logic, but still linear.
- **Memory Complexity:** $O(N_x N_y N_z)$ for the input grid and $O(N_x N_y N_z)$ for the output gradient grids. Intermediate shifted arrays also require $O(N_x N_y N_z)$ memory.
- **Strengths:** Provides significantly smoother normals than 2nd-order methods, which is beneficial for ray marching stability and shading quality in SDF rendering. Maintains high accuracy even at boundaries.
- **Limitations:** Requires more memory for intermediate buffers. Sensitive to noise in the input grid; smoothing may be required before gradient computation. Requires $N \ge 3$.
- **When to use:** When rendering SDFs or smooth scalar fields where normal quality is paramount and computational resources allow for the extra memory and arithmetic.
- **When not to use:** For very small grids ($N < 3$) or when memory is extremely constrained. For noisy data, pre-filtering is needed.

## Background
The method is implemented in the context of differentiable rendering of Signed Distance Functions, specifically in the `SDFDiff` framework [S1, unofficial]. It leverages the property that 4th-order finite differences provide a better approximation of the derivative for smooth fields, reducing aliasing artifacts in the rendered normals.

## Sources
[S1] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
