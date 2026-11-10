# LOD Ground Truth Generation from Surface Intersections

Also called: intersection-based LOD construction

## Overview
Constructs a Level-of-Detail (LOD) representation for a scene by downsampling high-resolution binary voxels and Signed Distance Fields (SDFs), extracting surface intersections along axes, and storing them in a sparse pointer/data structure per grid cell [S1, unofficial].

## How it works
**Inputs:**
- High-resolution binary voxel grid (e.g., $1024^3$) [S1, unofficial].
- High-resolution SDF grid (e.g., $128^3$) [S1, unofficial].
- Intersection files (pre-computed or derived) [S1, unofficial].
- Target grid sizes (e.g., $[32, 64]$) [S1, unofficial].

**Outputs:**
- `LOD_gt_int`: 4D array `[grid_size+1, grid_size+1, grid_size+1, 3]`, `uint8`, storing 3 cube edge flags per cell [S1, unofficial].
- `LOD_gt_float`: 4D array `[grid_size+1, grid_size+1, grid_size+1, 3]`, `float32`, storing 1 cube internal plane parameter (P) per cell [S1, unofficial].
- `LOD_intersection_pointer`: 1D array of `int32`, storing pointers/indices to intersection data per cell [S1, unofficial].
- `LOD_intersection_data`: 2D array `[max_len, 6]`, `float32`, storing intersection coordinates (3) and normals (3) [S1, unofficial].

**Data Structures:**
The method uses a sparse representation for intersections to handle varying density across the grid [S1, unofficial]. Each cell in the target LOD grid points to a list of intersections via `LOD_intersection_pointer` [S1, unofficial]. The actual data (position and normal) is stored contiguously in `LOD_intersection_data` [S1, unofficial].

**Parameters:**
- `grid_size_list`: List of target grid resolutions, e.g., `[32, 64]` [S1, unofficial].
- `cell_voxel_size`: Scaling factor for voxel size, set to 8 [S1, unofficial].
- `point_sample_num`: Number of points sampled from mesh for testing, set to 65536 [S1, unofficial].

**Steps:**
1. **Load Data:** Load high-resolution voxel, SDF, and intersection files [S1, unofficial].
2. **Compute Downscale:** For each target grid size in `grid_size_list`, compute the downscale factor relative to the high-resolution input [S1, unofficial].
3. **Downsample:** Downsample the voxel and SDF grids by striding with the computed downscale factor [S1, unofficial].
4. **Extract Intersections:** Identify grid cells where surface intersections occur. Specifically, extract intersections where the coordinate modulo the downscale factor equals 0, then divide the coordinates by the downscale factor to map them to the target LOD grid [S1, unofficial].
5. **Store Intersections:** Store the extracted intersections in `LOD_intersection_pointer` and `LOD_intersection_data`. Each intersection entry contains 6 values: 3 for coordinates and 3 for normals [S1, unofficial].
6. **Handle Empty Cells:** If a cell has no intersections (reference length == 0), skip processing to avoid singular least-squares systems [S1, unofficial].
7. **Plane Parameter Fitting:** For cells with intersections, fit a plane parameter using iterative regularization. If the solution $x$ falls outside $[0,1]$, cap the regularization scale at 1000. If it still fails, fall back to the center point $[0.5, 0.5, 0.5]$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_lod_ground_truth(high_res_voxel, high_res_sdf, target_grid_sizes, cell_voxel_size=8):
    results = {}
    for grid_size in target_grid_sizes:
        downscale = high_res_voxel.shape[0] / grid_size
        stride = int(downscale)

        # Downsample by striding
        downsampled_voxel = [[cell[stride::stride] for cell in row[stride::stride]] for row in high_res_voxel[stride::stride]]
        downsampled_sdf = [[cell[stride::stride] for cell in row[stride::stride]] for row in high_res_sdf[stride::stride]]

        dim = grid_size + 1
        # Initialize arrays per evidence layout: [dim, dim, dim, 3]
        lod_gt_int = [[[ [0, 0, 0] for _ in range(dim)] for _ in range(dim)] for _ in range(dim)]
        lod_gt_float = [[[ [0.0, 0.0, 0.0] for _ in range(dim)] for _ in range(dim)] for _ in range(dim)]
        lod_intersection_pointer = [-1] * (dim * dim * dim)
        lod_intersection_data = []

        current_data_idx = 0

        for x in range(dim):
            for y in range(dim):
                for z in range(dim):
                    # Extract intersections where coordinate % downscale == 0
                    # Divide coordinates by downscale to map to target LOD grid
                    # Sources do not specify the exact intersection file parsing logic.
                    intersections = []

                    # Handle empty cells to avoid singular least-squares systems
                    if not intersections:
                        lod_intersection_pointer[x * dim * dim + y * dim + z] = -1
                        continue

                    # Store intersections in contiguous array
                    for inter in intersections:
                        lod_intersection_data.append(inter)
                        lod_intersection_pointer[x * dim * dim + y * dim + z] = current_data_idx
                        current_data_idx += 1

                    # Plane parameter fitting using iterative regularization
                    reg_scale = 1.0
                    plane_param = [0.5, 0.5, 0.5]
                    # Sources do not specify the exact least-squares solver implementation.
                    # Standard iterative regularization: if solution x falls outside [0,1],
                    # scale regularization until x is in bounds or reg_scale > 1000.
                    # If reg_scale exceeds 1000, fallback to center point [0.5, 0.5, 0.5].

                    lod_gt_float[x][y][z] = plane_param
                    lod_gt_int[x][y][z] = [1, 1, 1]

        results[grid_size] = (lod_gt_int, lod_gt_float, lod_intersection_pointer, lod_intersection_data)
    return results
```

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, unofficial].
- **Memory Complexity:** $O(G^3)$ for the dense arrays (`LOD_gt_int`, `LOD_gt_float`) where $G$ is the grid size [S1, unofficial]. The sparse intersection data is $O(I)$ where $I$ is the total number of intersections [G].

## Background
Originated in the context of neural rendering and implicit surface reconstruction, specifically in the NDC (Normalized Device Coordinates) framework. The method is part of a pipeline for generating high-quality ground truth data for training neural networks to represent 3D scenes [S1, unofficial].

## Sources
[S1] czq142857/NDC, https://github.com/czq142857/NDC, fetched 2026-10-26 (unofficial)
