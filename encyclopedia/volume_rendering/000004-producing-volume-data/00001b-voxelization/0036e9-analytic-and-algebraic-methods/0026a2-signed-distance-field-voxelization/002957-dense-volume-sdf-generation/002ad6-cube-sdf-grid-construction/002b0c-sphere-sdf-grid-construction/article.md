# Sphere SDF Grid Construction
Also called: Sphere Field Generation, Torus SDF Grid Construction

## Overview
Constructs a discrete Signed Distance Field (SDF) grid for a sphere centered in the grid using its analytical implicit surface equation, vectorized with PyTorch broadcasting [S1, unofficial]. It generates a signed distance field for a sphere centered in the grid, used as a test volume for Marching Tetrahedra [S2, unofficial]. It differs from its sibling [Cube SDF Grid Construction](4883-cube-sdf-grid-construction.md) by using spherical symmetry and Euclidean distance rather than box-based min/max operations [G].

## How it works
**Inputs:**
- `grid_res`: Integer, the resolution of the grid in each dimension (assumed cubic, $N \times N \times N$).
- `bounding_box_min`: Scalar, the minimum coordinate of the bounding box.
- `bounding_box_max`: Scalar, the maximum coordinate of the bounding box.
- `radius`: Scalar, the radius of the sphere. Default value is 0.6 in normalized coordinates [S2, unofficial], or specific values like 1.6 (big) or 1.0 (small) in physical coordinates [S1, unofficial].
- `radius_big`, `radius_small`: Scalars, used for torus construction (major and minor radii) [S1, unofficial].

**Outputs:**
- `grid`: A 3D tensor of shape `(grid_res, grid_res, grid_res)` containing the SDF values.

**Data Structures:**
- `linear_space`: 1D array of length `grid_res` containing evenly spaced coordinates from `bounding_box_min` to `bounding_box_max` [S1, unofficial].
- `x_dim`, `y_dim`, `z_dim`: 3D tensors of shape `(grid_res, grid_res, grid_res)` representing the physical coordinates of each voxel center, created via broadcasting [S1, unofficial].
- `grid`: 3D tensor of shape `(grid_res, grid_res, grid_res)` storing the SDF values.

**Steps:**

1.  **Compute Voxel Size:**
    Calculate the distance between adjacent voxel centers:
    $$ \text{voxel\_size} = \frac{\text{bounding\_box\_max} - \text{bounding\_box\_min}}{\text{grid\_res} - 1} $$
    [S1, unofficial]

2.  **Generate Coordinate Grid (Vectorized Approach):**
    Create a 1D array of coordinates:
    $$ \text{linear\_space} = \text{linspace}(\text{bounding\_box\_min}, \text{bounding\_box\_max}, \text{grid\_res}) $$
    [S1, unofficial]
    
    Broadcast this array into 3D tensors to represent the coordinates of every voxel:
    $$ x\_dim[i, j, k] = \text{linear\_space}[i] $$
    $$ y\_dim[i, j, k] = \text{linear\_space}[j] $$
    $$ z\_dim[i, j, k] = \text{linear\_space}[k] $$
    [S1, unofficial]

3.  **Compute SDF for Sphere:**
    Calculate the Euclidean distance from the origin and subtract the radius:
    $$ \text{grid}[i, j, k] = \sqrt{x\_dim[i, j, k]^2 + y\_dim[i, j, k]^2 + z\_dim[i, j, k]^2} - \text{radius} $$
    [S1, unofficial]
    
    For specific test cases, `radius` is set to 1.6 for a "big" sphere or 1.0 for a "small" sphere [S1, unofficial]. In normalized coordinates $[-1, 1]$, the default radius is 0.6 [S2, unofficial].

4.  **Compute SDF for Torus (Variant):**
    If constructing a torus, the formula differs:
    $$ \text{grid}[i, j, k] = \sqrt{(\sqrt{y\_dim[i, j, k]^2 + z\_dim[i, j, k]^2} - \text{radius\_big})^2 + x\_dim[i, j, k]^2} - \text{radius\_small} $$
    [S1, unofficial]
    Where `radius_big` = 1.5 and `radius_small` = 0.5 [S1, unofficial].

5.  **Return Grid:**
    Return the computed grid. If CUDA is available, move the tensor to the GPU [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def grid_construction_sphere(grid_res, bounding_box_min, bounding_box_max, radius=1.6):
    """
    Constructs a discrete SDF grid for a sphere.
    
    Args:
        grid_res (int): Resolution of the grid (N x N x N).
        bounding_box_min (float): Minimum coordinate of the bounding box.
        bounding_box_max (float): Maximum coordinate of the bounding box.
        radius (float): Radius of the sphere. Default 1.6.
        
    Returns:
        torch.Tensor: 3D tensor of shape (grid_res, grid_res, grid_res) with SDF values.
    """
    # Step 1: Compute voxel size (used for normalization if needed)
    voxel_size = (bounding_box_max - bounding_box_min) / (grid_res - 1)
    
    # Step 2: Generate coordinate grid using PyTorch broadcasting
    # Create 1D array of coordinates
    linear_space = torch.linspace(bounding_box_min, bounding_box_max, grid_res)
    
    # Broadcast to 3D grids
    # x_dim[i, j, k] = linear_space[i]
    x_dim = linear_space.view(-1, 1, 1).repeat(1, grid_res, grid_res)
    # y_dim[i, j, k] = linear_space[j]
    y_dim = linear_space.view(1, -1, 1).repeat(grid_res, 1, grid_res)
    # z_dim[i, j, k] = linear_space[k]
    z_dim = linear_space.view(1, 1, -1).repeat(grid_res, grid_res, 1)
    
    # Step 3: Compute SDF for sphere
    # grid = sqrt(x^2 + y^2 + z^2) - radius
    grid = torch.sqrt(x_dim**2 + y_dim**2 + z_dim**2) - radius
    
    # Step 4: Return grid, moving to CUDA if available
    if torch.cuda.is_available():
        grid = grid.cuda()
        
    return grid

def grid_construction_torus(grid_res, bounding_box_min, bounding_box_max, radius_big=1.5, radius_small=0.5):
    """
    Constructs a discrete SDF grid for a torus.
    
    Args:
        grid_res (int): Resolution of the grid (N x N x N).
        bounding_box_min (float): Minimum coordinate of the bounding box.
        bounding_box_max (float): Maximum coordinate of the bounding box.
        radius_big (float): Major radius. Default 1.5.
        radius_small (float): Minor radius. Default 0.5.
        
    Returns:
        torch.Tensor: 3D tensor of shape (grid_res, grid_res, grid_res) with SDF values.
    """
    # Step 1: Generate coordinate grid
    linear_space = torch.linspace(bounding_box_min, bounding_box_max, grid_res)
    
    # Broadcast to 3D grids
    x_dim = linear_space.view(-1, 1, 1).repeat(1, grid_res, grid_res)
    y_dim = linear_space.view(1, -1, 1).repeat(grid_res, 1, grid_res)
    z_dim = linear_space.view(1, 1, -1).repeat(grid_res, grid_res, 1)
    
    # Step 2: Compute SDF for torus
    # grid = sqrt((sqrt(y^2 + z^2) - radius_big)^2 + x^2) - radius_small
    grid = torch.sqrt(
        (torch.sqrt(y_dim**2 + z_dim**2) - radius_big)**2 + x_dim**2
    ) - radius_small
    
    # Step 3: Return grid
    if torch.cuda.is_available():
        grid = grid.cuda()
        
    return grid
```

**Edge Cases and Pitfalls:**
- **Precision:** The use of `torch.sqrt` and broadcasting requires sufficient memory for $N^3$ tensors [G].
- **Centering:** The formulas assume the sphere is centered at the origin $(0,0,0)$ of the coordinate system [G]. If the bounding box is not symmetric around the origin, the sphere will not be centered in the grid [G].

## Operating characteristics
- **Time Complexity:** $O(N^3)$, where $N$ is `grid_res` [G].
- **Memory Complexity:** $O(N^3)$ for storing the coordinate grids and the SDF grid [G].
- **Measured Performance:** Not given in the sources.
- **Strengths:** Vectorized implementation allows for efficient computation on GPUs [G]. Simple and analytically exact for spheres and tori [G].
- **Limitations:** High memory usage for high-resolution grids [G]. Assumes the object is centered at the origin of the coordinate system [G].
- **When to use:** When generating test volumes for volume rendering algorithms like Marching Tetrahedra, or when a simple, analytically defined SDF is required [G].
- **When not to use:** When the object is complex or not centered at the origin, or when memory is constrained [G].

## Background
The method is implemented in the `CVPR2020-SDFDiff` repository [S1, unofficial]. It is also used in the `Marching-Tetrahedra-on-CT-headscan` repository as a test volume for Marching Tetrahedra [S2, unofficial].

## Sources
[S1] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
[S2] Torbenhjl/Marching-Tetrahedra-on-CT-headscan, https://github.com/Torbenhjl/Marching-Tetrahedra-on-CT-headscan, fetched 2026-10-27 (unofficial)
