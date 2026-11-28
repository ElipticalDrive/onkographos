# Voxel Center Precomputation
## Overview
Precomputes the 3D world coordinates of voxel centers for a regular grid defined by its origin, physical size, and dimensions [S1, unofficial].
## How it works
**Inputs:**
- `grid_origin`: Scene origin coordinates $[x_{min}, y_{min}, z_{min}]$ [S1, unofficial].
- `voxel_size`: Physical size of a single voxel $[s_x, s_y, s_z]$ [S1, unofficial].
- `grid_dims`: Voxel grid dimensions $[D, H, W]$ (depth, height, width) [S1, unofficial].

**Outputs:**
- `centers`: A tensor of shape $(D, H, W, 3)$ containing the world coordinates of each voxel center [S1, unofficial].

**Data Structures:**
- The output is a dense 4D tensor where the last dimension holds the $(x, y, z)$ coordinates and the first three dimensions correspond to the grid indices $(d, h, w)$ [S1, unofficial].

**Steps:**
1. Extract dimensions $D, H, W$ from `grid_dims` [S1, unofficial].
2. Compute x-coordinates: $x = \text{grid\_origin}[0] + \text{arange}(W) \times \text{voxel\_size}[0] + \text{voxel\_size}[0]/2$ [S1, unofficial].
3. Compute y-coordinates: $y = \text{grid\_origin}[1] + \text{arange}(H) \times \text{voxel\_size}[1] + \text{voxel\_size}[1]/2$ [S1, unofficial].
4. Compute z-coordinates: $z = \text{grid\_origin}[2] + \text{arange}(D) \times \text{voxel\_size}[2] + \text{voxel\_size}[2]/2$ [S1, unofficial].
5. Stack coordinates into a grid using `meshgrid` with 'ij' indexing and return tensor of shape $(D, H, W, 3)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def precompute_voxel_centers(grid_origin, voxel_size, grid_dims):
    # Extract dimensions
    D, H, W = grid_dims
    # Extract origin and size components
    x_min, y_min, z_min = grid_origin
    sx, sy, sz = voxel_size
    # Compute 1D coordinate arrays
    x_coords = x_min + np.arange(W) * sx + sx / 2.0
    y_coords = y_min + np.arange(H) * sy + sy / 2.0
    z_coords = z_min + np.arange(D) * sz + sz / 2.0
    # Stack into grid using meshgrid with 'ij' indexing
    # Note: meshgrid with 'ij' indexing expects inputs in (depth, height, width) order to produce (D, H, W) shape [G]
    Z_grid, Y_grid, X_grid = np.meshgrid(z_coords, y_coords, x_coords, indexing='ij')
    # Return tensor of shape (D, H, W, 3)
    return np.stack((X_grid, Y_grid, Z_grid), axis=-1)
```

**Parameters:**
- `grid_origin`: Scene origin coordinates $[x_{min}, y_{min}, z_{min}]$ [S1, unofficial].
- `voxel_size`: Physical size of a single voxel $[s_x, s_y, s_z]$ [S1, unofficial].
- `grid_dims`: Voxel grid dimensions $[D, H, W]$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- Not given in the sources.

## Subtopics

### [Voxel Center Computation from Indices](11239-voxel-center-computation-from-indices.md)
Computes the 3D world-space coordinates of voxel centers given their discrete indices, bounding box origins, and voxel sizes. It solves the problem of mapping discrete grid positions to continuous physical space for ray casting, shading, or crosshair placement. It is distinguished from [Voxel Grid Coordinate Mapping](15198-voxel-grid-coordinate-mapping.md) by focusing specifically on center point calculation rather than handling linear buffer indexing schemes or geometry-based vertex generation.

## Operating characteristics

## Background

## Sources
[S1] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
