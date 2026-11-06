# NeRF Density Field Mesh Extraction

Also called: Density Field Meshing

## Overview
Extracts a triangle mesh from a trained Neural Radiance Field (NeRF) by evaluating the learned volume density $\sigma$ on a 3D grid and applying the marching cubes algorithm at a specific threshold [S1, unofficial]. It differs from adaptive methods like MISE [MISE (Meshing Isosurfaces by Extracting)](977-mise-meshing-isosurfaces-by-extracting.md) by using a fixed uniform grid, which is simpler to implement but less efficient for complex geometries, and from differentiable methods like FlexiCubes [FlexiCubes](3669-flexicubes.md) by producing a static mesh without gradient-based optimization of the surface vertices [Parent](6371-neural-and-learning-based-extraction.md).

## How it works
**Inputs:**
- `coarse_model`: A trained NeRF MLP (or hierarchy) capable of querying volume density $\sigma$ given 3D coordinates.
- `resolution`: Integer, the side length of the cubic query grid (default 128) [S1, unofficial].
- `scene_scale`: Float, half-extent of the bounding box containing the scene (default 1.5) [S1, unofficial].
- `sigma_threshold`: Float, the density value defining the isosurface (default 50.0) [S1, unofficial].
- `chunk`: Integer, number of points to query per batch to manage memory (default 32768) [S1, unofficial].

**Outputs:**
- `verts`: Array of shape $[V, 3]$ containing vertex coordinates in world space.
- `faces`: Array of shape $[F, 3]$ containing triangle indices.
- `normals`: Array of shape $[V, 3]$ containing vertex normals (optional, computed by marching cubes).

**Data Structures:**
- `lin`: 1D tensor of shape $[resolution]$.
- `pts`: 2D tensor of shape $[resolution^3, 3]$.
- `sigma_grid`: 3D tensor of shape $[resolution, resolution, resolution]$.

**Steps:**
1.  **Grid Generation**: Create a 1D linear space from $-scene\_scale$ to $scene\_scale$ with `resolution` points:
    $$ lin = \text{linspace}(-scene\_scale, scene\_scale, resolution) $$
    Create a 3D meshgrid using `lin` for x, y, and z axes. Note that NeRF conventions often swap Y and Z axes relative to standard Cartesian coordinates. Stack the coordinates into a point cloud tensor `pts` of shape $[resolution^3, 3]$ [S1, unofficial].
    *Pitfall*: Ensure Y and Z axes are swapped if the NeRF model expects coordinates in the format $(x, z, y)$ or similar, as standard meshgrid indexing may differ from the training data convention [S1, unofficial].

2.  **Density Querying**: Evaluate the density $\sigma$ for all points in `pts` using the trained `coarse_model`. To avoid out-of-memory errors, process `pts` in batches of size `chunk`. For each batch $i$, query the model:
    $$ \sigma_i = \text{coarse\_model}(pts[i:i+chunk]) $$
    Concatenate the results to form a flat array of densities.

3.  **Reshaping**: Reshape the flat density array into a 3D tensor `sigma_grid` of shape $[resolution, resolution, resolution]$ [S1, unofficial].

4.  **Marching Cubes**: Apply the marching cubes algorithm to `sigma_grid` with `level = sigma_threshold` to extract the isosurface. This returns vertices, faces, normals, and values [S1, unofficial].

5.  **Coordinate Rescaling**: The vertices returned by marching cubes are in grid indices $[0, resolution-1]$. Rescale them to world coordinates:
    $$ verts_{world} = \frac{verts_{grid}}{resolution - 1} \times (2 \times scene\_scale) - scene\_scale $$
    This maps the grid range $[0, resolution-1]$ to $[-scene\_scale, scene\_scale]$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
from skimage import measure

def extract_mesh(coarse_model, resolution=128, scene_scale=1.5, sigma_threshold=50.0, chunk=32768):
    # 1. Grid Generation
    lin = torch.linspace(-scene_scale, scene_scale, resolution, device=coarse_model.device)
    xx, yy, zz = torch.meshgrid(lin, lin, lin, indexing="ij")
    # NeRF convention often swaps Y and Z; adjust if necessary
    pts = torch.stack([xx, zz, yy], dim=-1).reshape(-1, 3)
    
    # 2. Density Querying
    sigma_list = []
    with torch.no_grad():
        for i in range(0, pts.shape[0], chunk):
            batch_pts = pts[i:i+chunk]
            # Query density; dummy_dirs may be required by the model signature
            _, sigma = coarse_model(batch_pts, batch_pts) 
            sigma_list.append(sigma)
    sigma_flat = torch.cat(sigma_list, dim=0)
    
    # 3. Reshaping
    sigma_grid = sigma_flat.reshape(resolution, resolution, resolution)
    
    # 4. Marching Cubes
    verts, faces, normals, _ = measure.marching_cubes(sigma_grid.cpu().numpy(), level=sigma_threshold)
    
    # 5. Coordinate Rescaling
    verts = verts / (resolution - 1) * (2 * scene_scale) - scene_scale
    
    return verts, faces, normals
```

## Subtopics

### [NeRF-based Mesh Extraction with Color Fusion](7372-nerf-based-mesh-extraction-with-color-fusion.md)
Extracts a textured 3D mesh from a trained NeRF by combining isosurface extraction with a color fusion strategy that aggregates information from multiple training views [NeRF-based Mesh Extraction with Color Fusion](7372-nerf-based-mesh-extraction-with-color-fusion.md). It solves the problem of generating a watertight, color-accurate mesh from a volumetric representation that lacks explicit geometry and consistent surface color. It differs from basic NeRF mesh extraction [Parent](4104-nerf-density-field-mesh-extraction.md) by replacing simple vertex shading with a multi-view color fusion process that weights colors by occlusion and depth, or by using vertex normals to query the NeRF, thereby reducing artifacts from single-view sampling [NeRF-based Mesh Extraction with Color Fusion](7372-nerf-based-mesh-extraction-with-color-fusion.md).

### Differentiable Marching Tetrahedrons (DMTet)
A differentiable surface extraction method used to create a triangle mesh from a Signed Distance Field (SDF) [S2, unofficial]. Introduced by Shen et al. in 2021 [S2, unofficial]. It is used in frameworks like JOC to generate triangle meshes initialized by the SDF from a neural field, allowing for joint optimization of geometry, material, and light [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N \cdot Q)$, where $N$ is the number of grid points ($resolution^3$) and $Q$ is the cost of a single NeRF query. Marching cubes is $O(N)$.
- **Memory Complexity**: $O(N)$ for storing the density grid and intermediate query results. High-resolution grids (e.g., 256+) can exceed GPU memory, necessitating chunked querying [S1, unofficial].
- **Performance**: Fast and deterministic. Performance depends heavily on the resolution and the speed of the NeRF inference.
- **Strengths**: Simple to implement; produces watertight meshes; leverages existing NeRF training infrastructure.
- **Limitations**: Fixed grid resolution may miss fine details or include noise; requires careful tuning of `sigma_threshold`; Y/Z axis swapping pitfalls [S1, unofficial]; does not support differentiable optimization of the mesh geometry itself.
- **When to use**: When a quick, static mesh is needed from a pre-trained NeRF and high-fidelity differentiable optimization is not required.
- **When not to use**: When adaptive resolution is needed (use MISE) or when the mesh needs to be optimized for downstream tasks like rendering or physics (use DMTet or FlexiCubes).

## Background
Originated from the standard post-processing pipeline for Neural Radiance Fields (NeRF) [S1, unofficial]. The method leverages the fact that NeRFs learn a volume density field $\sigma(\mathbf{x})$, which can be treated as an implicit surface definition when thresholded. It builds on the classic marching cubes algorithm [S1, unofficial].

## Sources
[S1] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S2] Joint Optimization of Triangle Mesh, Material, and Light from Neural Fields with Neural Radiance Cache, https://doi.org/10.48550/arxiv.2305.16800, fetched 2026-10-26 (unofficial)
