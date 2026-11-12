# Batched Marching Cubes SDF Extraction

Also called: MC extraction, grid-based isosurface extraction

## Overview
Extracts a 0-level isosurface from a voxelized Signed Distance Function (SDF) grid using the Marching Cubes algorithm, specifically designed to handle dense grid evaluation without causing GPU Out-Of-Memory (OOM) errors [S1, unofficial]. It differs from standard Marching Cubes by decoupling the neural network inference step from the mesh generation step, processing the grid points in fixed-size batches [S1, unofficial].

## How it works
**Inputs:**
- A neural network `net` capable of evaluating SDF values `net.sdf(pts)` [S1, unofficial].
- Grid resolution `MC_SAMPLE_SIZE` (integer, typically 256 or 512) [S1, unofficial].
- Batch size `BATCH_POINTS` (integer, fixed at $2^{20}$ in the reference implementation) [S1, unofficial].
- Spatial extent `CANVAS_SIZE` (float, default 1.0) [S1, unofficial].
- Normalization radius `training_data.R` (float) [S1, unofficial].

**Outputs:**
- A triangular mesh (vertices and faces) representing the 0-level isosurface [S1, unofficial].

**Data Structures:**
- `s`: 1D tensor of shape `[MC_SAMPLE_SIZE]` containing linearly spaced coordinates from `-CANVAS_SIZE` to `CANVAS_SIZE` [S1, unofficial].
- `grid_x, grid_y, grid_z`: 3D tensors of shape `[MC_SAMPLE_SIZE, MC_SAMPLE_SIZE, MC_SAMPLE_SIZE]` generated via meshgrid [S1, unofficial].
- `pts`: 2D tensor of shape `[M, 3]` where $M = MC\_SAMPLE\_SIZE^3$, containing all grid coordinates flattened [S1, unofficial].
- `sdfs_flat`: 1D tensor of shape `[M]` storing the computed SDF values [S1, unofficial].
- `sdfs`: 3D tensor of shape `[MC_SAMPLE_SIZE, MC_SAMPLE_SIZE, MC_SAMPLE_SIZE]` holding the SDF values in grid order [S1, unofficial].

**Steps:**
1. Generate the 1D coordinate vector `s` using `torch.linspace(-CANVAS_SIZE, CANVAS_SIZE, MC_SAMPLE_SIZE)` [S1, unofficial].
2. Create the 3D meshgrid `grid_x, grid_y, grid_z` from `s` using `indexing='ij'` [S1, unofficial].
3. Stack the grid tensors and reshape them into a 2D tensor `pts` of shape `[M, 3]` where $M = MC\_SAMPLE\_SIZE^3$ [S1, unofficial].
4. Initialize `sdfs_flat` as an empty float32 tensor of shape `[M]` on the CPU [S1, unofficial].
5. Iterate through `pts` in batches of size `BATCH_POINTS`:
   - Calculate `start` and `end` indices for the current batch [S1, unofficial].
   - Move the batch `pts[start:end]` to the GPU [S1, unofficial].
   - Evaluate the SDF network: `sdf_batch = net.sdf(pts[start:end].cuda())[:, 0]` [S1, unofficial].
   - Move the results back to the CPU and store in `sdfs_flat[start:end]` [S1, unofficial].
6. Reshape `sdfs_flat` into the 3D tensor `sdfs` of shape `[MC_SAMPLE_SIZE, MC_SAMPLE_SIZE, MC_SAMPLE_SIZE]` [S1, unofficial].
7. Call the Marching Cubes algorithm on `sdfs` with an isovalue of 0: `vertices, faces = mcubes.marching_cubes(sdfs, 0)` [S1, unofficial].
8. Scale the extracted vertices to the physical canvas space:
   $$ v_{scaled} = \frac{v_{raw}}{MC\_SAMPLE\_SIZE - 1.0} \times 2 \times CANVAS\_SIZE - CANVAS\_SIZE $$
   [S1, unofficial].
9. Normalize the scaled vertices by dividing by `training_data.R` [S1, unofficial].
10. Convert the result to a standard mesh format (e.g., numpy arrays or trimesh) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import mcubes
import numpy as np

def batched_mc_extraction(net, mc_sample_size, batch_points, canvas_size, training_data_R):
    # 1. Generate 1D grid
    s = torch.linspace(-canvas_size, canvas_size, mc_sample_size)
    
    # 2. Create 3D meshgrid
    grid_x, grid_y, grid_z = torch.meshgrid(s, s, s, indexing='ij')
    
    # 3. Stack and reshape to [M, 3]
    pts = torch.stack([grid_x, grid_y, grid_z], dim=-1)
    pts = pts.reshape(-1, 3)
    M = pts.shape[0]
    
    # 4. Initialize SDF storage
    sdfs_flat = torch.empty(M, dtype=torch.float32, device='cpu')
    
    # 5. Batched inference
    for start in range(0, M, batch_points):
        end = min(start + batch_points, M)
        # Move batch to GPU, evaluate, move back to CPU
        pts_batch = pts[start:end].cuda()
        sdf_batch = net.sdf(pts_batch)[:, 0]
        sdfs_flat[start:end] = sdf_batch.cpu()
        
    # 6. Reshape to 3D grid
    sdfs = sdfs_flat.reshape(mc_sample_size, mc_sample_size, mc_sample_size)
    
    # 7. Marching Cubes
    vertices_raw, faces = mcubes.marching_cubes(sdfs.numpy(), 0.0)
    
    # 8. Scale vertices
    vertices_scaled = vertices_raw / (mc_sample_size - 1.0) * 2 * canvas_size - canvas_size
    
    # 9. Normalize
    vertices_final = vertices_scaled / training_data_R
    
    # 10. Return mesh
    return vertices_final, faces
```

**Parameters:**
- `MC_SAMPLE_SIZE`: Grid resolution. Typical values are 256 or 512 [S1, unofficial].
- `BATCH_POINTS`: Inference batch size. Fixed at $2^{20}$ in the reference implementation [S1, unofficial].
- `CANVAS_SIZE`: Spatial extent of the grid. Default is 1.0 [S1, unofficial].

## Operating characteristics
- **Memory:** The method splits memory usage into two parts: the neural network weights (constant) and the batch of points being evaluated. By fixing `BATCH_POINTS` to $2^{20}$, it prevents GPU OOM errors that would occur if the entire dense grid ($512^3 \approx 1.34 \times 10^8$ points) were evaluated simultaneously [S1, unofficial].
- **Time Complexity:** $O(M)$ where $M = MC\_SAMPLE\_SIZE^3$, due to the linear evaluation of each grid point [S1, unofficial]. The constant factor depends on the neural network inference speed and the batch size efficiency.
- **Strengths:** Preserves fine geometric detail without extraction error inherent to lower-resolution approximations [S1, unofficial]. Handles dense grids that would otherwise crash the GPU [S1, unofficial].
- **Limitations:** Requires significant CPU-to-GPU data transfer overhead for each batch [S1, unofficial]. The fixed batch size may not be optimal for all GPU architectures or network sizes [S1, unofficial].

## Background
This method is derived from the TetraSDF project, which uses neural networks to represent 3D shapes as SDFs [S1, unofficial]. It addresses the practical challenge of extracting meshes from high-resolution neural SDF grids on consumer or limited hardware [S1, unofficial].

## Sources
[S1] naver-ai/TetraSDF, https://github.com/naver-ai/TetraSDF, fetched 2026-10-27 (unofficial)
