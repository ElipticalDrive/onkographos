# NeRF Density Field Mesh Extraction

Also called: Density Field Meshing

## Overview
NeRF Density Field Mesh Extraction extracts a triangle mesh from a trained Neural Radiance Field (NeRF) by discretizing the scene volume into a 3D grid, querying the learned density field $\sigma$ at each voxel, and applying the marching cubes algorithm at a specific iso-value threshold [S1, unofficial]. It solves the problem of converting a continuous, volumetric radiance representation—which lacks explicit geometry—into a discrete, watertight mesh suitable for standard graphics pipelines [S1, unofficial]. It differs from its siblings, such as MISE [MISE](977-mise-meshing-isosurfaces-by-extracting.md) and FlexiCubes [FlexiCubes](3669-flexicubes.md), by relying on a fixed, uniform grid query of the density field rather than adaptive sampling or learned cube templates, making it computationally simpler but potentially less efficient for high-resolution details [S1, unofficial].

## How it works
**Inputs:**
- A trained NeRF model (typically a coarse/fine hierarchy, though the extraction step often uses a single query function) [S1, unofficial].
- `scene_scale`: The half-extent of the bounding box containing the scene [S1, unofficial].
- `resolution`: The side length of the 3D voxel grid (default 128) [S1, unofficial].
- `sigma_threshold`: The density iso-value for surface extraction (default 50.0) [S1, unofficial].
- `chunk`: Batch size for querying the neural network (default 32768) [S1, unofficial].

**Outputs:**
- `verts`: A tensor of vertex positions in world coordinates.
- `faces`: A tensor of triangle indices.
- `normals`: Vertex normals (optional, computed by marching cubes).

**Data Structures:**
- `sigma_grid`: A 3D tensor of shape `[resolution, resolution, resolution]` storing the density values $\sigma$ at each grid point.
- `pts`: A 2D tensor of shape `[N, 3]` containing the 3D coordinates of all grid points, where $N = resolution^3$.

**Steps:**

1.  **Grid Generation:**
    Create a 1D linear space of coordinates ranging from `-scene_scale` to `scene_scale` with `resolution` points:
    $$ lin = \text{linspace}(-scene\_scale, scene\_scale, resolution) $$
    Generate a 3D grid using meshgrid with `indexing="ij"`:
    $$ xx, yy, zz = \text{meshgrid}(lin, lin, lin, indexing="ij") $$
    Stack the coordinates into a single tensor. Note that NeRF conventions often swap the Y and Z axes relative to standard Cartesian grid indexing. The implementation stacks `[xx, zz, yy]` to match this convention:
    $$ pts = \text{stack}([xx, zz, yy], dim=-1).reshape(-1, 3) $$
    The shape of `pts` is `[resolution^3, 3]`.

2.  **Density Query:**
    Query the NeRF model for density $\sigma$ at each point in `pts`. To manage memory, process `pts` in chunks of size `chunk`. For each chunk `i`:
    $$ \_, \sigma_i = \text{model}(pts[i:i+chunk], \text{dummy\_dirs}[i:i+chunk]) $$
    Here, `dummy_dirs` are placeholder view directions, as density $\sigma$ in a standard NeRF is often view-independent or queried with dummy rays. Concatenate all $\sigma_i$ to form a 1D array of size `resolution^3`.

3.  **Reshaping:**
    Reshape the concatenated density array into the 3D grid structure:
    $$ \sigma\_grid = \sigma\_array.reshape(resolution, resolution, resolution) $$

4.  **Marching Cubes:**
    Apply the marching cubes algorithm to `sigma_grid` at the specified `sigma_threshold`:
    $$ verts\_local, faces, normals, \_ = \text{marching\_cubes}(\sigma\_grid, level=sigma\_threshold) $$
    The resulting `verts_local` are in grid index coordinates (integers from 0 to `resolution-1`).

5.  **Coordinate Rescaling:**
    Transform the vertices from grid index space to world coordinates. The mapping from index $v \in [0, resolution-1]$ to world coordinate $w \in [-scene\_scale, scene\_scale]$ is:
    $$ w = \frac{v}{resolution - 1} \times (2 \times scene\_scale) - scene\_scale $$
    Apply this to all vertices:
    $$ verts = \frac{verts\_local}{resolution - 1} \times (2 \times scene\_scale) - scene\_scale $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
from skimage import measure

def extract_mesh_from_nerf(coarse_model, scene_scale, resolution=128, sigma_threshold=50.0, chunk=32768):
    """
    Extract a mesh from a NeRF density field using marching cubes.
    
    Args:
        coarse_model: A callable NeRF model that returns (color, sigma) given points and directions.
        scene_scale: Half-extent of the scene bounding box.
        resolution: Voxel grid resolution.
        sigma_threshold: Iso-value for surface extraction.
        chunk: Batch size for model queries.
        
    Returns:
        verts: Tensor of shape [V, 3] in world coordinates.
        faces: Tensor of shape [F, 3] of vertex indices.
    """
    # 1. Create 1D coordinate array
    lin = torch.linspace(-scene_scale, scene_scale, resolution, device="cpu")
    
    # 2. Create 3D grid
    xx, yy, zz = torch.meshgrid(lin, lin, lin, indexing="ij")
    
    # 3. Stack and reshape to points
    # Note: NeRF convention often swaps Y and Z. We stack [xx, zz, yy].
    pts = torch.stack([xx, zz, yy], dim=-1).reshape(-1, 3)
    
    # 4. Query density sigma in chunks
    # Dummy directions are required if the model expects them, but sigma is view-independent.
    dummy_dirs = torch.zeros_like(pts)
    
    sigma_list = []
    num_points = pts.shape[0]
    
    for i in range(0, num_points, chunk):
        end_idx = min(i + chunk, num_points)
        # Query model: returns (color, sigma)
        # We only need sigma.
        _, sigma = coarse_model(pts[i:end_idx], dummy_dirs[i:end_idx])
        sigma_list.append(sigma.detach().cpu())
        
    # Concatenate and reshape to [resolution, resolution, resolution]
    sigma_grid = torch.cat(sigma_list, dim=0).reshape(resolution, resolution, resolution).numpy()
    
    # 5. Run marching cubes
    # measure.marching_cubes returns (verts, faces, normals, values)
    verts_local, faces, normals, _ = measure.marching_cubes(sigma_grid, level=sigma_threshold)
    
    # 6. Rescale vertices from [0, resolution] to world coordinates
    # Formula: v_world = (v_local / (resolution - 1)) * (2 * scene_scale) - scene_scale
    verts = verts_local / (resolution - 1) * (2 * scene_scale) - scene_scale
    
    return verts, faces
```

**Parameters:**
- `resolution`: Voxel grid resolution. Default: 128 [S1, unofficial].
- `scene_scale`: Half-extent of the query box. Default: 1.5 [S1, unofficial].
- `sigma_threshold`: Iso-value for the surface. Default: 50.0 [S1, unofficial].
- `chunk`: Number of points queried per batch. Default: 32768 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Y/Z Axis Swap:** The grid creation must swap Y and Z axes (`[xx, zz, yy]`) to match the NeRF coordinate convention, otherwise the extracted mesh will be rotated or distorted [S1, unofficial].
- **Density Threshold Sensitivity:** The choice of `sigma_threshold` significantly affects the mesh quality. Too low results in noisy, fragmented surfaces; too high results in a shrunken or missing surface [S1, unofficial].
- **Memory Usage:** Querying a high-resolution grid (e.g., 256^3) requires significant memory for both the grid tensor and the intermediate batched queries. Chunking is essential to avoid out-of-memory errors [S1, unofficial].

## Subtopics

### [NeRF-based Mesh Extraction with Color Fusion](7372-nerf-based-mesh-extraction-with-color-fusion.md)
This method extracts a textured 3D mesh from a trained NeRF by combining isosurface extraction with a color fusion strategy that aggregates information from multiple training views. It solves the problem of generating a watertight, color-accurate mesh from a volumetric representation that lacks explicit geometry and consistent surface color. It differs from basic NeRF mesh extraction [Parent](4104-nerf-density-field-mesh-extraction.md) by replacing simple vertex shading with a multi-view color fusion process that weights colors by occlusion and depth, or by using vertex normals to query the NeRF, thereby reducing artifacts from single-view sampling.

### Differentiable Marching Tetrahedrons (DMTet)
A differentiable surface extraction method used to create a triangle mesh from a Signed Distance Field (SDF) [S2, unofficial]. Introduced by Shen et al. in 2021 [S2, unofficial], it is used in frameworks like JOC to generate the triangle mesh, initialized by the SDF from the neural field [S2, unofficial]. Unlike the non-differentiable marching cubes used in standard NeRF mesh extraction, DMTet allows for gradient-based optimization of the mesh geometry during training [S2, unofficial].

### Neural SDF Mesh Extraction with Texture Synthesis
This variant extracts a mesh from an optimized neural SDF using the marching cube algorithm [S3]. It then computes per-vertex UV mapping using the Blender Smart UV Project tool [S3]. To generate material textures, it densely samples points on the triangle meshes using trilinearly interpolated UV coordinates, queries the material networks at these sampled surface points to get material parameters, and splats these per-point material parameters to texture images using their interpolated UV coordinates [S3].

## Operating characteristics
- **Time Complexity:** $O(R^3 \cdot C)$, where $R$ is the resolution and $C$ is the cost of querying the NeRF model per point. The marching cubes step is $O(R^3)$ but is typically negligible compared to the neural network queries [S1, unofficial].
- **Memory Complexity:** $O(R^3)$ for storing the density grid and the point coordinates. Chunking reduces peak memory usage for the model queries but does not change the asymptotic grid storage requirement [S1, unofficial].
- **Measured Performance:** At resolution 128, the method is fast enough for post-processing but becomes prohibitive at higher resolutions (e.g., 256 or 512) due to the cubic growth in query points [S1, unofficial].
- **Strengths:** Simple to implement; leverages well-established marching cubes algorithm; produces watertight meshes [S1, unofficial].
- **Limitations:** Uniform grid resolution limits detail in sparse regions; requires careful tuning of `sigma_threshold`; color extraction is not inherent to the basic method, requiring additional steps like color fusion [S1, unofficial].
- **When to use:** When a quick, approximate mesh is needed from a pre-trained NeRF for visualization or simple downstream tasks [S1, unofficial].
- **When not to use:** When high-fidelity geometry is required in sparse regions (use MISE [MISE](977-mise-meshing-isosurfaces-by-extracting.md) or DMTet [DMTet](3669-flexicubes.md)) or when differentiable geometry optimization is needed [S2, unofficial].

## Background
The method originates from the standard practice of visualizing NeRF outputs, where the density field is treated as a volumetric scalar field [S1, unofficial]. It was popularized by early NeRF implementations and tutorials that demonstrated how to convert the learned implicit representation into an explicit mesh for broader compatibility with 3D software [S1, unofficial]. The approach builds on the foundational NeRF paper [Mildenhall et al. 2020] and the classic marching cubes algorithm [Lorensen and Cline 1987].

## Sources
[S1] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S2] Joint Optimization of Triangle Mesh, Material, and Light from Neural Fields with Neural Radiance Cache, https://doi.org/10.48550/arxiv.2305.16800, fetched 2026-10-26 (unofficial)
[S3] IRON: Inverse Rendering by Optimizing Neural SDFs and Materials from Photometric Images, https://doi.org/10.1109/cvpr52688.2022.00548, fetched 2026-10-26
