# NDC Ground Truth LOD Generation via Least-Squares Plane Fitting

Also called: LOD ground truth computation

## Overview
This method generates multi-resolution ground truth data for Neural Distance Fields (NDC) by fitting planes to surface intersection points within grid cells [S1, unofficial]. It solves the problem of creating accurate per-cell labels (signs and plane parameters) from high-resolution voxel and intersection data, which are then used to train a CNN to predict Marching Cubes configurations [S1, unofficial]. It differs from gradient-based methods by explicitly using computed intersection points and their normals for least-squares fitting [S1, unofficial].

## How it works
**Inputs:**
1. Object file (`.obj`).
2. External executables: `VOXGen`, `SDFGen`, `IntersectionXYZpn` [S1, unofficial].
3. Configuration parameters: `grid_size_list` (e.g., `[32, 64]`), `cell_voxel_size` (8.0) [S1, unofficial].

**Outputs:**
HDF5 file containing datasets for each `grid_size` in `grid_size_list` [S1, unofficial]:
* `{grid_size}_int`: Shape `[grid_size+1, grid_size+1, grid_size+1, 1]`, `np.uint8`. Stores the sign grid [S1, unofficial].
* `{grid_size}_float`: Shape `[grid_size+1, grid_size+1, grid_size+1, 3]`, `np.float32`. Stores plane fitting results (initialized to -10.0) [S1, unofficial].
* `{grid_size}_sdf`: Shape `[grid_size+1, grid_size+1, grid_size+1]`, `np.float32`. Downsampled SDF [S1, unofficial].
* `{grid_size}_voxel`: Shape `[grid_size+1, grid_size+1, grid_size+1]`, `np.uint8`. Downsampled voxels [S1, unofficial].

**Data Structures:**
* `reference_V`: Buffer storing intersection points and normals for a cell. Layout: `[N, 6]`, where columns 0-2 are positions $(X, Y, Z)$ and columns 3-5 are normals $(n_x, n_y, n_z)$ [S1, unofficial].
* `LOD_gt_int`: Integer array for sign bits [S1, unofficial].
* `LOD_gt_float`: Float array for plane parameters [S1, unofficial].

**Steps:**
1. **Initialize Output Arrays:** Create `LOD_gt_int` and `LOD_gt_float` with shape `[grid_size+1, grid_size+1, grid_size+1, num_params]` for each `grid_size` in `grid_size_list` [S1, unofficial]. Initialize `LOD_gt_float` with -10.0 [S1, unofficial].
2. **Generate Raw Data:** For each object, run `./VOXGen <obj> 1024 0`, `./SDFGen <obj> 128 0`, and `./IntersectionXYZpn <obj> 128 0` to generate voxel, SDF, and intersection files [S1, unofficial].
3. **Read Input Data:** Load the SDF (129×129×129), intersection points (X, Y, Z planes with normals), and voxels (1025×1025×1025) [S1, unofficial].
4. **Downsample for Target Resolution:** For each `grid_size` in `grid_size_list`:
   a. Calculate downscale factor: $downscale = 1024 // (grid\_size \times cell\_voxel\_size)$ [S1, unofficial].
   b. Downsample voxels: Sample `voxel_1025` at stride `downscale` [S1, unofficial].
   c. Downsample SDF: Sample `sdf_129` at stride `downscale` [S1, unofficial].
   d. Downsample intersections: Filter points where coordinates are multiples of `downscale`, then divide coordinates by `downscale` [S1, unofficial].
5. **Compute Input Voxel Grid:** Compute the input voxel grid for the target `grid_size` [S1, unofficial].
6. **Organize Intersections:** Organize downsampled intersection points into a cell-based data structure using pointers and data arrays [S1, unofficial].
7. **Compute Sign Grid:** $tmp\_sign = (tmp\_sdf \le 0).astype(np.uint8)$ [S1, unofficial]. Store `tmp_sign` in `LOD_gt_int` [S1, unofficial].
8. **Iterate Over Grid Cells:** For each cell $(i, j, k)$ from $0$ to $grid\_size-1$:
   a. Check surface intersection: If $\max(tmp\_sign[i:i+2, j:j+2, k:k+2]) \neq \min(tmp\_sign[i:i+2, j:j+2, k:k+2])$, the cell intersects the surface; proceed to plane fitting; otherwise skip [S1, unofficial].
   b. Retrieve points: Get intersection points and normals into `reference_V` [S1, unofficial].
   c. Handle empty cells: If `reference_V` is empty (length 0), skip the cell [S1, unofficial].
   d. Construct linear system: $A = reference\_V[:, 3:6]$, $b = \sum_{d=0}^{2} (A_{:,d} \times reference\_V[:, d])$ (dot product of normals and positions) [S1, unofficial].
   e. Regularization: Initialize `reg_scale = 1e-6`. Calculate `reg_pos` as the mean position of intersection points [S1, unofficial]. Add regularization rows to $A$ and $b$ to avoid singular matrices [S1, unofficial].
   f. Solve least squares: $x = \text{lstsq}(A, b)[0]$ [S1, unofficial].
   g. Boundary adjustment: If any component of $x$ is outside $[0, 1]$, enter an adjustment loop [S1, unofficial]. Initialize `tmp_scale = 1`. While $x$ is out of bounds: increase `reg_scale` by `tmp_scale`, recompute $A$ and $b$, solve for $x$ [S1, unofficial]. If `reg_scale > 1000`, break and reset `reg_scale = 1e-6` [S1, unofficial]. If still out of bounds, decrease `reg_scale` by `tmp_scale / 2` in a loop of 10 iterations, adjusting `reg_scale` and solving again [S1, unofficial]. If still out of bounds after the loop, perform one final solve with the current `reg_scale` [S1, unofficial].
   h. Store result: Store $x$ into `LOD_gt_float[i, j, k]` [S1, unofficial].
9. **Save Data:** Save all generated arrays to an HDF5 file with the specified dataset names and shapes [S1, unofficial].

**Parameters:**
* `cell_voxel_size`: Size of the base voxel grid in world units, set to 8 [S1, unofficial].
* `num_of_int_params`: Number of integer parameters per grid cell, set to 1 (sign) [S1, unofficial].
* `num_of_float_params`: Number of float parameters per grid cell, set to 3 (plane coefficients/position) [S1, unofficial].
* `grid_size_list`: List of grid resolutions to generate, specifically `[32, 64]` [S1, unofficial].

**Pitfalls:**
* Precision and degenerate input: If no intersection points are found for a cell, the linear system is undefined; skip the cell [S1, unofficial].
* Boundary adjustment loop: Ill-conditioned intersection sets can cause the regularization loop to iterate many times or hit the `reg_scale > 1000` limit, requiring careful handling to avoid infinite loops or numerical instability [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import h5py
import subprocess
import os

def load_generated_files(obj_path):
    """Run external executables and load generated arrays."""
    subprocess.run(['./VOXGen', obj_path, '1024', '0'], check=True)
    subprocess.run(['./SDFGen', obj_path, '128', '0'], check=True)
    subprocess.run(['./IntersectionXYZpn', obj_path, '128', '0'], check=True)
    # Standard file loading placeholders; in practice, read the binary/text outputs
    sdf_129 = np.load('SDFGen_output.npy')  # Shape (129, 129, 129)
    voxels_1025 = np.load('VOXGen_output.npy')  # Shape (1025, 1025, 1025)
    intersections = np.load('IntersectionXYZpn_output.npy')  # Shape (N, 6)
    return sdf_129, voxels_1025, intersections

def get_input_voxel(grid_size, voxels_ds):
    """Compute input voxel grid for the target grid_size [G]."""
    # Standard padding/alignment step for CNN input
    return np.pad(voxels_ds, ((1, 1), (1, 1), (1, 1)), mode='constant')

def get_intersection_points_normals_in_cells(intersections_ds, grid_size):
    """Organize downsampled intersection points into a cell-based data structure [G]."""
    cells = {}
    for pt in intersections_ds:
        ix, iy, iz = int(pt[0]), int(pt[1]), int(pt[2])
        ix = max(0, min(ix, grid_size - 1))
        iy = max(0, min(iy, grid_size - 1))
        iz = max(0, min(iz, grid_size - 1))
        key = (ix, iy, iz)
        if key not in cells:
            cells[key] = []
        cells[key].append(pt)
    return cells

def solve_least_squares(A, b, reg_scale, reg_pos):
    """Add regularization and solve Ax=b."""
    A_reg = np.vstack([A, np.eye(3) * reg_scale])
    b_reg = np.concatenate([b, reg_pos * reg_scale])
    x, _, _, _ = np.linalg.lstsq(A_reg, b_reg, rcond=None)
    return x

def generate_ndc_gt(obj_path, grid_size_list, cell_voxel_size=8.0, output_hdf5_path="output.hdf5"):
    sdf_129, voxels_1025, intersections = load_generated_files(obj_path)
    
    with h5py.File(output_hdf5_path, 'w') as f:
        for grid_size in grid_size_list:
            downscale = 1024 // (grid_size * cell_voxel_size)
            
            voxels_ds = voxels_1025[::downscale, ::downscale, ::downscale]
            sdf_ds = sdf_129[::downscale, ::downscale, ::downscale]
            
            valid_mask = (intersections[:, 0] % downscale == 0) & \
                         (intersections[:, 1] % downscale == 0) & \
                         (intersections[:, 2] % downscale == 0)
            intersections_ds = intersections[valid_mask] / downscale
            
            input_voxel = get_input_voxel(grid_size, voxels_ds)
            cell_intersections = get_intersection_points_normals_in_cells(intersections_ds, grid_size)
            
            tmp_sign = (sdf_ds <= 0).astype(np.uint8)
            
            lod_gt_int = np.zeros((grid_size + 1, grid_size + 1, grid_size + 1, 1), dtype=np.uint8)
            lod_gt_int[:, :, :, 0] = tmp_sign
            
            lod_gt_float = np.full((grid_size + 1, grid_size + 1, grid_size + 1, 3), -10.0, dtype=np.float32)
            
            for i in range(grid_size):
                for j in range(grid_size):
                    for k in range(grid_size):
                        corner_signs = tmp_sign[i:i+2, j:j+2, k:k+2]
                        if np.max(corner_signs) != np.min(corner_signs):
                            ref_v = np.array(cell_intersections.get((i, j, k), []))
                            
                            if ref_v.shape[0] == 0:
                                continue
                            
                            A = ref_v[:, 3:6]
                            b = np.sum(A * ref_v[:, 0:3], axis=1)
                            
                            reg_scale = 1e-6
                            reg_pos = np.mean(ref_v[:, 0:3], axis=0)
                            
                            x = solve_least_squares(A, b, reg_scale, reg_pos)
                            
                            if not np.all((x >= 0) & (x <= 1)):
                                tmp_scale = 1.0
                                while not np.all((x >= 0) & (x <= 1)):
                                    reg_scale += tmp_scale
                                    if reg_scale > 1000:
                                        reg_scale = 1e-6
                                        break
                                    x = solve_least_squares(A, b, reg_scale, reg_pos)
                                
                                if not np.all((x >= 0) & (x <= 1)):
                                    for _ in range(10):
                                        reg_scale -= tmp_scale / 2
                                        x = solve_least_squares(A, b, reg_scale, reg_pos)
                                    x = solve_least_squares(A, b, reg_scale, reg_pos)
                            
                            lod_gt_float[i, j, k] = x
            
            f.create_dataset(f'{grid_size}_int', data=lod_gt_int)
            f.create_dataset(f'{grid_size}_float', data=lod_gt_float)
            f.create_dataset(f'{grid_size}_sdf', data=sdf_ds)
            f.create_dataset(f'{grid_size}_voxel', data=voxels_ds)
```

## Operating characteristics
* **Time Complexity:** Dominated by iterating over all grid cells and solving the least-squares system for surface-intersecting cells [S1, unofficial]. The cell iteration is $O(grid\_size^3)$ [S1, unofficial]. The least-squares solve is $O(N)$ per cell, where $N$ is the number of intersection points, which is typically small [S1, unofficial].
* **Memory Complexity:** $O(grid\_size^3)$ for storing the ground truth arrays [S1, unofficial]. The HDF5 file size scales linearly with the number of grid resolutions and objects [S1, unofficial].
* **Strengths:** Provides explicit geometric ground truth (plane parameters) that is robust for training CNNs to predict Marching Cubes configurations [S1, unofficial]. Handles multi-resolution LODs efficiently [S1, unofficial].
* **Limitations:** Relies on external executables for SDF and intersection generation [S1, unofficial]. The regularization and boundary adjustment loop can be computationally expensive if many cells have ill-conditioned intersection sets [S1, unofficial].
* **When to use:** When training Neural Distance Fields that require explicit plane fitting ground truth for surface representation [S1, unofficial].
* **When not to use:** When real-time generation is required, or when the external executables are unavailable [S1, unofficial].

## Background
This method is part of the Neural Distance Fields (NDC) framework, specifically designed to generate training data for CNNs that predict Marching Cubes configurations [S1, unofficial]. It was introduced in the context of the NDC project [S1, unofficial]. The method uses explicit intersection points and normals for plane fitting, distinguishing it from methods that might use SDF gradients directly [S1, unofficial].

## Sources
[S1] czq142857/NDC, https://github.com/czq142857/NDC, fetched 2026-10-26 (unofficial)
