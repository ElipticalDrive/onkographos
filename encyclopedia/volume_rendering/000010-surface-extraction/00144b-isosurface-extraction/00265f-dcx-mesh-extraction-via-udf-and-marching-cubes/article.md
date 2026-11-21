# DCX Mesh Extraction via UDF and Marching Cubes

Also called: BVH UDF to Mesh, UDF-based Marching Cubes

## Overview
This method extracts a triangular mesh from a BVH (Bounding Volume Hierarchy) structure by computing an Unsigned Distance Field (UDF) on a regular grid, converting it to a Signed Distance Field (SDF) via flood fill, and applying Marching Cubes [S1, unofficial]. It differs from Dual Contouring [778-dual-contouring.md] by avoiding complex optimization and vertex positioning; instead, it relies on a standard isosurface extractor on a dense grid, making it simpler but potentially less efficient for high-resolution outputs.

## How it works
**Inputs:**
- A BVH structure supporting unsigned distance queries.
- An integer `resolution` defining the grid dimensions ($N \times N \times N$).

**Outputs:**
- A `trimesh.Trimesh` object containing vertices, faces, and normals.

**Data Structures:**
- `udf`: A 3D array of shape $(N, N, N)$ of type `float32`, initialized to zero.
- `occ_mask`: A boolean mask of shape $(N, N, N)$ indicating occupied voxels.
- `sdf`: A 3D array of shape $(N, N, N)$ of type `float32`, representing the signed distance field.

**Steps:**

1.  **Grid Initialization:**
    Define the spatial range $[-0.5, 0.5]$ for X, Y, and Z axes. Split this range into chunks of size $N=32$ for processing efficiency, though the final grid resolution is determined by the input `resolution` parameter [S1, unofficial]. Create a meshgrid from the coordinate ranges and flatten it to form a points tensor of shape $(num\_points, 3)$, where $num\_points = N^3$ [S1, unofficial].

2.  **UDF Query:**
    Query the BVH for the unsigned distance to the surface for all grid points. The query returns a sub-UDF array. Reshape this result to $(N, N, N)$ and store it in the `udf` array [S1, unofficial].

3.  **Occupancy Determination:**
    Define occupancy using a threshold. A voxel is considered occupied if its UDF value is less than $2 / resolution$ [S1, unofficial].
    $$ occ = udf < \frac{2}{resolution} $$
    Convert `occ` to a tensor and perform a flood fill operation starting from the corner voxel $(0, 0, 0)$ to identify connected components [S1, unofficial].
    - Identify the `empty_label` corresponding to the background (the label at `floodfill_mask[0, 0, 0]`) [S1, unofficial].
    - Create an `empty_mask` where `floodfill_mask == empty_label`.
    - Create the final `occ_mask` as the inverse of the empty mask:
      $$ occ\_mask = \neg empty\_mask $$

4.  **UDF to SDF Conversion:**
    Initialize `sdf` as a copy of `udf`. Invert the sign of the distance values for all occupied regions:
    $$ sdf[occ\_mask] = -sdf[occ\_mask] $$
    Unoccupied regions retain their positive unsigned distance values [S1, unofficial].

5.  **Marching Cubes:**
    Apply the Marching Cubes algorithm to the `sdf` array with an isovalue of $0$. The spacing for the grid is uniform:
    $$ spacing = \left(\frac{1}{resolution}, \frac{1}{resolution}, \frac{1}{resolution}\right) $$
    This yields vertices `verts`, faces `faces`, normals `normals`, and values `values` [S1, unofficial].

6.  **Coordinate Adjustment:**
    The raw Marching Cubes output assumes a grid from $0$ to $N-1$. Adjust vertices to match the original $[-0.5, 0.5]$ bounding box:
    $$ verts = verts - 0.5 + \frac{0.5}{resolution - 1} $$
    $$ verts = verts \times \frac{resolution}{resolution - 1} $$
    Construct the final mesh using these adjusted vertices and the extracted faces [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
# Assuming cubvh, marching_cubes, and trimesh are available in the environment
# as per the source implementation context [S1, unofficial]

def extract_mesh_from_bvh(bvh, resolution):
    # 1. Initialize UDF array
    udf = np.zeros((resolution, resolution, resolution), dtype=np.float32)
    
    # Define coordinate ranges [-0.5, 0.5]
    # Note: The source splits into chunks of N=32 for processing, 
    # but the final grid is resolution x resolution x resolution.
    # We assume a helper to generate meshgrid points for the full resolution.
    xs = np.linspace(-0.5, 0.5, resolution)
    ys = np.linspace(-0.5, 0.5, resolution)
    zs = np.linspace(-0.5, 0.5, resolution)
    xs, ys, zs = np.meshgrid(xs, ys, zs, indexing='ij')
    points = np.stack([xs, ys, zs], axis=-1).reshape(-1, 3)
    
    # 2. Query BVH for unsigned distance
    # BVH.unsigned_distance returns (distances, u, v, w) or similar
    # We only need distances here.
    sub_udf, _, _ = bvh.unsigned_distance(points, return_uvw=False)
    
    # Reshape and store in udf
    udf = sub_udf.reshape(resolution, resolution, resolution)
    
    # 3. Determine occupancy
    # Threshold: 2 / resolution
    occ = udf < (2.0 / resolution)
    
    # Convert to torch tensor for CUDA flood fill
    occ_tensor = torch.from_numpy(occ).cuda()
    
    # Perform flood fill
    # cubvh.floodfill expects a boolean mask or similar input
    floodfill_mask = cubvh.floodfill(occ_tensor)
    
    # Identify empty label from corner [0,0,0]
    empty_label = floodfill_mask[0, 0, 0].item()
    
    # Create masks
    empty_mask = (floodfill_mask == empty_label)
    occ_mask = ~empty_mask
    
    # 4. Convert UDF to SDF
    sdf = udf.copy()
    # Invert sign for occupied regions
    # Note: occ_mask is boolean, need to align with numpy indexing
    # floodfill_mask is CUDA tensor, convert back to numpy boolean mask
    occ_mask_np = ~empty_mask.cpu().numpy()
    sdf[occ_mask_np] *= -1
    
    # 5. Apply Marching Cubes
    # spacing is 1/resolution
    spacing = (1.0/resolution, 1.0/resolution, 1.0/resolution)
    verts, faces, normals, values = marching_cubes(sdf, 0, spacing=spacing)
    
    # 6. Adjust vertex coordinates
    # Formula from source: verts = verts - 0.5 + (0.5/(resolution-1))
    # Then: verts = verts * (resolution/(resolution-1))
    verts = verts - 0.5 + (0.5 / (resolution - 1))
    verts = verts * (resolution / (resolution - 1))
    
    # Create trimesh
    mesh = trimesh.Trimesh(verts, faces)
    
    return mesh
```

**Parameters:**
- `resolution`: Integer defining the grid resolution for UDF sampling. Typical values are not specified in the sources, but the implementation uses chunks of size 32 for intermediate processing [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Flood Fill Background:** The background (empty) label is determined by the voxel at $(0, 0, 0)$. If the object touches the boundary or the coordinate system is shifted, this assumption may fail [S1, unofficial].
- **Occupancy Threshold:** The threshold $2/resolution$ is critical. Voxels with UDF less than this value are considered inside. This is a very tight threshold, effectively treating only the immediate surface vicinity as occupied [S1, unofficial].
- **Precision:** The UDF is stored in `float32`. For very high resolutions, precision loss in distance queries may affect the accuracy of the SDF conversion [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by the BVH distance query for $N^3$ points and the flood fill operation. Marching Cubes is $O(N^3)$ but typically fast.
- **Memory Complexity:** $O(N^3)$ for the UDF, SDF, and mask arrays.
- **Strengths:** Simple to implement; leverages robust BVH distance queries; avoids the complexity of dual contouring optimization.
- **Limitations:** Dense grid sampling is memory-intensive for high resolutions; the fixed occupancy threshold may not adapt well to varying surface densities; quality depends on Marching Cubes artifacts.
- **When to use:** When a quick, robust mesh extraction from a BVH is needed without fine-tuning optimization parameters.
- **When not to use:** When high-resolution meshes are required with limited memory, or when surface normals need to be precisely preserved (Dual Contouring [778-dual-contouring.md] is better suited).

## Background
This method is part of the DCx (Dual Contouring over Expanded Cubes) project, specifically utilizing a UDF-based approach combined with Marching Cubes for mesh extraction [S1, unofficial]. It serves as an alternative to the primary Dual Contouring implementation within the same codebase, offering a simpler pipeline at the cost of potential quality and efficiency trade-offs.

## Sources
[S1] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
