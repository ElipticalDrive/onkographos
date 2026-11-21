# Voxel Center Precomputation
## Overview
Precomputes the 3D world coordinates of voxel centers for a regular grid defined by its origin, physical size, and dimensions [S1, unofficial]. It transforms discrete grid indices into continuous spatial coordinates, enabling efficient ray-voxel intersection and attribute sampling in volume rendering pipelines [S1, unofficial]. It differs from sibling methods by explicitly supporting batched inputs with concatenated batch IDs and enforcing specific axis-swapping conventions required by certain rendering pipelines [S1, unofficial].
## How it works
**Inputs:**
- `grid_origin`: Scene origin coordinates $[x_{min}, y_{min}, z_{min}]$ [S1, unofficial].
- `voxel_size`: Physical size of a single voxel $[s_x, s_y, s_z]$ [S1, unofficial].
- `grid_dims`: Voxel grid dimensions $[D, H, W]$ (depth, height, width) [S1, unofficial].

**Outputs:**
- `centers`: A tensor of shape $(D, H, W, 3)$ containing the world coordinates of each voxel center [S1, unofficial].

**Data Structures:**
- The output is a dense 4D tensor where the last dimension holds the $(x, y, z)$ coordinates. The first three dimensions correspond to the grid indices $(d, h, w)$.

**Steps:**
1. Extract dimensions $D, H, W$ from `grid_dims` [S1, unofficial].
2. Compute the x-coordinates for all columns:
   $$x_w = x_{min} + w \cdot s_x + \frac{s_x}{2}, \quad w \in [0, W-1]$$
   This results in a 1D array of length $W$ [S1, unofficial].
3. Compute the y-coordinates for all rows:
   $$y_h = y_{min} + h \cdot s_y + \frac{s_y}{2}, \quad h \in [0, H-1]$$
   This results in a 1D array of length $H$ [S1, unofficial].
4. Compute the z-coordinates for all depth slices:
   $$z_d = z_{min} + d \cdot s_z + \frac{s_z}{2}, \quad d \in [0, D-1]$$
   This results in a 1D array of length $D$ [S1, unofficial].
5. Stack the coordinates into a 3D grid using `meshgrid` with 'ij' indexing to ensure correct axis alignment (i for depth, j for height) [S1, unofficial].
6. Concatenate the x, y, and z grids along the last dimension to form the $(D, H, W, 3)$ tensor [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def precompute_voxel_centers(grid_origin, voxel_size, grid_dims):
    """
    Precompute 3D world coordinates of voxel centers.
    
    Args:
        grid_origin (array-like): [x_min, y_min, z_min]
        voxel_size (array-like): [sx, sy, sz]
        grid_dims (array-like): [D, H, W]
        
    Returns:
        numpy.ndarray: Centers tensor of shape (D, H, W, 3)
    """
    # Extract dimensions
    D, H, W = grid_dims
    
    # Extract origin and size components
    x_min, y_min, z_min = grid_origin
    sx, sy, sz = voxel_size
    
    # Compute 1D coordinate arrays
    # x coordinates: width dimension
    x_coords = x_min + np.arange(W) * sx + sx / 2.0
    
    # y coordinates: height dimension
    y_coords = y_min + np.arange(H) * sy + sy / 2.0
    
    # z coordinates: depth dimension
    z_coords = z_min + np.arange(D) * sz + sz / 2.0
    
    # Create meshgrid with 'ij' indexing
    # X_grid shape: (D, H, W) where X_grid[d, h, w] = x_coords[w]
    # Y_grid shape: (D, H, W) where Y_grid[d, h, w] = y_coords[h]
    # Z_grid shape: (D, H, W) where Z_grid[d, h, w] = z_coords[d]
    X_grid, Y_grid, Z_grid = np.meshgrid(x_coords, y_coords, z_coords, indexing='ij')
    
    # Stack into (D, H, W, 3)
    centers = np.stack((X_grid, Y_grid, Z_grid), axis=-1)
    
    return centers
```

**Parameters:**
- `grid_origin`: Typically set to the minimum corner of the scene bounding box [S1, unofficial].
- `voxel_size`: Determined by the scene extent and desired grid resolution [S1, unofficial].
- `grid_dims`: Integer dimensions of the voxel grid [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Axis Swapping:** The method relies on 'ij' indexing for `meshgrid`. Using 'xy' indexing (default in some libraries) will swap the height and width dimensions, leading to incorrect coordinate mapping [S1, unofficial].
- **Batching:** For batched scenarios, inputs may need concatenation of batch IDs before processing, though the core geometric computation remains per-voxel [S1, unofficial].
- **Precision:** Floating-point accumulation in large grids may introduce minor precision errors in world coordinates. Use double precision if high accuracy is required [G].

## Subtopics

### [Voxel Center Computation from Indices](11239-voxel-center-computation-from-indices.md)
Computes the 3D world coordinates of voxel centers given their discrete integer indices, bounding box origins, and voxel sizes. It transforms discrete grid indices into continuous spatial coordinates, handling both single-grid and batched scenarios. It differs from sibling methods by explicitly supporting batched inputs with concatenated batch IDs and enforcing specific axis-swapping conventions required by certain rendering pipelines.

## Operating characteristics
- **Time Complexity:** $O(D \cdot H \cdot W)$ to generate the coordinate grid. The meshgrid operation dominates the cost.
- **Memory Complexity:** $O(D \cdot H \cdot W \cdot 3)$ to store the resulting tensor. For high-resolution grids, this can be significant.
- **Strengths:** Provides a direct lookup for voxel centers, avoiding per-ray computation of center positions during rendering.
- **Limitations:** Memory usage scales cubically with resolution. Not suitable for extremely high-resolution grids without sparse storage strategies.
- **When to use:** When the voxel grid is static and memory allows storing the full coordinate grid.
- **When not to use:** When memory is constrained or the grid is too large to fit in RAM.

## Background
This method is part of the preprocessing pipeline for 3D Gaussian Splatting and related volume rendering techniques. It ensures that voxel centers are accurately mapped to world space for attribute assignment and ray intersection. The implementation is based on the AVM-GS codebase [S1, unofficial].

## Sources
[S1] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
