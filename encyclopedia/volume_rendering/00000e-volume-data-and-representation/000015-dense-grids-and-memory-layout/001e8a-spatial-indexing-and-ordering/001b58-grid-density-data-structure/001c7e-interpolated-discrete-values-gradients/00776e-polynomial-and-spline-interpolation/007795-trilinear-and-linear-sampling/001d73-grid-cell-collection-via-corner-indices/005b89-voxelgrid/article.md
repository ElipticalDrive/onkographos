# VoxelGrid

Also called: 3D Voxel Grid, Voxel-grid Representation

## Overview
VoxelGrid is a dense, fixed-resolution 3D data structure that stores scene properties (such as density, color, or signed distance) at grid vertices to support efficient feature queries via trilinear interpolation [S4][S5, unofficial]. It serves as a differentiable, learnable replacement for implicit neural networks (MLPs) in neural rendering, enabling direct optimization of volumetric attributes [S2, S5, unofficial]. Unlike sparse structures like octrees, which are difficult to update on the fly in learning frameworks, VoxelGrid offers fast, memory-accessible queries at the cost of high memory consumption for large volumes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D query points $P \in \mathbb{R}^{N \times 3}$ and grid configuration parameters (resolution `shape`, bounding box `range_per_dim`, output dimensionality `d`).
*   **Output:** Interpolated values $V \in \mathbb{R}^{N \times d}$ corresponding to the properties stored in the grid at the query locations.

**Data Structures:**
The grid stores values at the corners of the voxels to facilitate trilinear interpolation.
*   **Storage Shape:** If the logical grid resolution is `shape` = $(N_x, N_y, N_z)$, the internal storage tensor has shape $(N_x+1, N_y+1, N_z+1, d)$ [S5, unofficial].
*   **Fields:**
    *   `grid` (or `data`): A dense tensor (e.g., `torch.Tensor` or `std::vector<CellType>`) holding the learnable or static values [S3, S5, S6, unofficial].
    *   `range_per_dim`: A list of tuples $(low, high)$ defining the bounding box for each dimension [S6, unofficial].
    *   `resolution`: The physical size of each voxel side, or derived from `range_per_dim` and `shape` [S6, unofficial].
    *   `invalid_val`: A default value (e.g., 0) for out-of-bounds or uninitialized voxels [S6, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Define the bounding box for each dimension: $x \in [x_{min}, x_{max}]$, $y \in [y_{min}, y_{max}]$, $z \in [z_{min}, z_{max}]$ [S6, unofficial].
    *   Adjust the range so the span is evenly divisible by the resolution if necessary [S6, unofficial].
    *   Allocate storage tensor of shape $(N_x+1, N_y+1, N_z+1, d)$ [S5, unofficial].
    *   Initialize values: either randomly scaled by `max` (default 1.0), filled with a provided tensor, or set to a specific midpoint (e.g., for log-density fields) [S2, S5, unofficial].

2.  **Coordinate Mapping (Query):**
    *   For each query point $p = (x, y, z)$, map the continuous coordinates to grid indices.
    *   Compute the fractional position within the grid bounds. If the grid is periodic (e.g., longitude), apply wrapping [S2, unofficial].

3.  **Corner Indexing (`bracket`):**
    *   For each dimension $d \in \{x, y, z\}$ with grid size $n_d$:
        *   Compute integer index: $idx = \lfloor coord \rfloor$ [S5, unofficial].
        *   Clamp indices to valid range $[0, n_d - 1]$ to handle boundary cases:
            $$ X_0 = \text{clamp}(idx, 0, n_d - 1) $$
            $$ X_1 = \text{clamp}(idx + 1, 0, n_d - 1) $$
        *   Compute interpolation weight $\alpha$:
            $$ \alpha = \text{clamp}(coord - idx, 0.0, 1.0) $$
        *   Return $(X_0, X_1, \alpha)$ [S5, unofficial].

4.  **Trilinear Interpolation:**
    *   Retrieve values from the storage tensor at the 8 corners defined by $(X_0/X_1, Y_0/Y_1, Z_0/Z_1)$ [S5, unofficial].
    *   Interpolate along the x-axis using weight $a$:
        $$ v_{y0z0} = v(X_0, Y_0, Z_0) \cdot (1-a) + v(X_1, Y_0, Z_0) \cdot a $$
        $$ v_{y1z0} = v(X_0, Y_1, Z_0) \cdot (1-a) + v(X_1, Y_1, Z_0) \cdot a $$
        $$ v_{y0z1} = v(X_0, Y_0, Z_1) \cdot (1-a) + v(X_1, Y_0, Z_1) \cdot a $$
        $$ v_{y1z1} = v(X_0, Y_1, Z_1) \cdot (1-a) + v(X_1, Y_1, Z_1) \cdot a $$
    *   Interpolate along the y-axis using weight $b$:
        $$ v_{z0} = v_{y0z0} \cdot (1-b) + v_{y1z0} \cdot b $$
        $$ v_{z1} = v_{y0z1} \cdot (1-b) + v_{y1z1} \cdot b $$
    *   Interpolate along the z-axis using weight $c$:
        $$ result = v_{z0} \cdot (1-c) + v_{z1} \cdot c $$
    *   The general 1D interpolation formula is $v = v_0(1-\alpha) + v_1\alpha$ [S5, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def bracket(x, n):
    """
    Return the indices of the nearest grid points to x, as well as weights.
    x: tensor of coordinates
    n: grid dimension size
    """
    x0 = torch.floor(x).long()
    # Clamp to [0, n-1] to prevent out-of-bounds access
    X0 = torch.clamp(x0, min=0, max=(n - 1))
    X1 = torch.clamp(x0 + 1, min=0, max=(n - 1))
    # Fractional weight, clamped to [0.0, 1.0]
    alpha = torch.clamp(x - x0.float(), min=0.0, max=1.0)
    return X0, X1, alpha

def interpolate(v0, v1, alpha):
    """
    Interpolate between v0 and v1 using alpha.
    Handles batch dimensions by unsqueezing alpha.
    """
    # Ensure alpha has compatible shape for broadcasting
    alpha_expanded = alpha.unsqueeze(-1)
    return v0 * (1 - alpha_expanded) + v1 * alpha_expanded

class VoxelGrid(nn.Module):
    def __init__(self, shape, d=1, max_val=1.0, fill=None):
        """
        A 3D voxel grid with given `shape` storing corner values for trilinear interpolation.
        
        Args:
            shape: Tuple of integers (Nx, Ny, Nz) defining the logical grid dimensions.
            d: Integer defining the output dimensionality (e.g., 1 for density, 3 for RGB).
            max_val: Float defining the maximum value for random initialization.
            fill: Optional tensor to initialize the grid instead of random values.
        """
        super(VoxelGrid, self).__init__()
        # Storage shape includes +1 in each dimension to store corner values
        storage_shape = tuple(s + 1 for s in shape)
        
        if fill is not None:
            # Repeat fill tensor to match storage shape
            self.grid = nn.Parameter(fill.view(1, 1, 1, d).repeat(*storage_shape, 1))
        else:
            # Initialize with random values scaled by max_val
            self.grid = nn.Parameter(torch.rand(*storage_shape, d, dtype=torch.float32) * max_val)

    def forward(self, P):
        """
        Implement trilinear interpolation at the points P.
        
        Args:
            P: Tensor of shape (..., 3) containing query coordinates (x, y, z).
        Returns:
            Tensor of interpolated values.
        """
        x, y, z = P[..., 0], P[..., 1], P[..., 2]

        # Get indices of the corners and interpolation weights for each dimension
        X0, X1, a = bracket(x, self.grid.shape[0])
        Y0, Y1, b = bracket(y, self.grid.shape[1])
        Z0, Z1, c = bracket(z, self.grid.shape[2])

        # Interpolate in the x direction (along edges parallel to x-axis)
        # y0z0: edge at Y0, Z0
        y0z0 = interpolate(self.grid[X0, Y0, Z0, :], self.grid[X1, Y0, Z0, :], a)
        # y1z0: edge at Y1, Z0
        y1z0 = interpolate(self.grid[X0, Y1, Z0, :], self.grid[X1, Y1, Z0, :], a)
        # y0z1: edge at Y0, Z1
        y0z1 = interpolate(self.grid[X0, Y0, Z1, :], self.grid[X1, Y0, Z1, :], a)
        # y1z1: edge at Y1, Z1
        y1z1 = interpolate(self.grid[X0, Y1, Z1, :], self.grid[X1, Y1, Z1, :], a)

        # Interpolate in the y direction (along edges parallel to y-axis)
        # z0: face at Z0
        z0 = interpolate(y0z0, y1z0, b)
        # z1: face at Z1
        z1 = interpolate(y0z1, y1z1, b)

        # Interpolate in the z direction (final volume interpolation)
        result = interpolate(z0, z1, c)
        
        # Squeeze the last dimension if d=1 to match expected output shape
        return result.squeeze(-1)
```

**Parameters:**
*   `shape`: Tuple of integers $(N_x, N_y, N_z)$ defining the logical grid dimensions [S5, unofficial].
*   `d`: Integer defining the output dimensionality (default 1) [S5, unofficial].
*   `resolution`: Float, the size of each voxel side length [S6, unofficial].
*   `range_per_dim`: List of tuples $(low, high)$ defining the bounding box for each dimension [S6, unofficial].
*   `max`: Float defining the maximum value for random initialization (default 1.0) [S5, unofficial].
*   `invalid_val`: Float, default 0, the value used to mark empty/uninitialized voxels [S6, unofficial].
*   `N_G`: The grid resolution (often used interchangeably with `shape` components) [S4].
*   `C`: The dimension of the outputs [S4].

**Edge Cases and Pitfalls:**
*   **Out-of-Bounds Access:** Query points outside the bounding box must be handled. Indices are clamped to $[0, n-1]$ to prevent crashes, though this may introduce artifacts at boundaries [S5, S6, unofficial].
*   **Memory Efficiency:** Dense voxel grids are memory inefficient for large volumes or high resolutions, as storage scales with $N_x N_y N_z$ [S1, unofficial].
*   **Periodicity:** For domains like longitude, special handling (wrap-padding) is required to maintain continuity across boundaries [S2, unofficial].
*   **Initialization:** Improper initialization (e.g., not centering log-density values) can hinder convergence in learning tasks [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Query time is $O(1)$ per point, independent of grid size, as it involves a fixed number of tensor lookups and arithmetic operations (trilinear interpolation) [S1, S5, unofficial].
*   **Memory Complexity:** $O(N_x N_y N_z \cdot d)$, where $N_x, N_y, N_z$ are grid dimensions and $d$ is the feature dimension. This is a significant limitation for high-resolution scenes [S1, unofficial].
*   **Measured Performance:** In a solar tomography example, a grid of $299 \times 142 \times 154$ (6.5M parameters) was used with AdamW optimization, demonstrating feasibility for moderate resolutions [S2, unofficial].
*   **Strengths:** Fast, differentiable, and easy to integrate into deep learning frameworks (e.g., via PyTorch's `grid_sample` or custom CUDA kernels) [S1, S2, unofficial]. Supports efficient batched queries.
*   **Limitations:** High memory usage makes it unsuitable for very large scenes without sparsification (e.g., octrees) or multi-resolution hierarchies [S1, unofficial]. Fixed resolution limits detail adaptability.
*   **When to Use:** When fast, differentiable queries are needed for moderate-resolution volumes, or as a baseline for neural rendering [S2, S5, unofficial].
*   **When Not to Use:** For very large scenes where memory is constrained, or when adaptive resolution is required (consider Octrees or Hash Grids instead) [S1, unofficial].

## Background
Voxel grids have long been used in computer graphics as the 3D equivalent of pixels to represent volumes, storing geometry, density, or appearance [S1, unofficial]. In the context of neural rendering, they were popularized by methods like DeepVoxels [S1, unofficial] and DVGO (Differentiable Volumes), which replaced MLPs with learnable voxel grids to accelerate training and inference [S5, unofficial]. They are widely used in neural radiance fields (NeRF) variants, autonomous driving scene representations (e.g., Lightning NeRF), and volumetric SDF computations [S4][S6, unofficial].

## Sources
[S1] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S2] Image Fidelity is Not Field Fidelity: Joint Thermodynamic Reconstruction and Error Localization in Neural Tomography, https://doi.org/10.48550/arxiv.2609.28868, fetched 2026-10-26 (unofficial)
[S3] whoiszzj/FDS-GS, https://github.com/whoiszzj/FDS-GS, fetched 2026-10-27 (unofficial)
[S4] Lightning NeRF: Efficient Hybrid Scene Representation for Autonomous Driving, https://doi.org/10.1109/icra57147.2024.10611130, fetched 2026-10-27
[S5] 7.6. Neural Radiance Fields for Drones, https://www.roboticsbook.org/S76_drone_learning.html, fetched 2026-10-29 (unofficial)
[S6] GitHub - UM-ARM-Lab/pytorch_volumetric: Volumetric structures such as ..., https://github.com/UM-ARM-Lab/pytorch_volumetric, fetched 2026-10-30 (unofficial)
