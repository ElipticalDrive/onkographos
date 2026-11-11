# NeRF Density Field Mesh Extraction

Also called: Density Field Meshing

## Overview
This method extracts a triangle mesh from a trained Neural Radiance Field (NeRF) by querying the learned density field $\sigma$ on a 3D grid and applying the marching cubes algorithm at a specific threshold [S1, unofficial]. It converts a volumetric, implicit representation into an explicit geometric mesh [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained NeRF model (typically a coarse/fine hierarchy), the scene bounding box defined by `scene_scale`, and extraction parameters (`resolution`, `sigma_threshold`, `chunk`) [S1, unofficial].
*   **Output:** A triangle mesh defined by vertices `verts`, faces `faces`, and normals `normals` in world coordinates [S1, unofficial].

**Data Structures:**
*   **Grid:** A 3D tensor of shape `[resolution, resolution, resolution]` storing scalar density values $\sigma$ [S1, unofficial].
*   **Points:** A 2D tensor of shape `[N, 3]` where $N = resolution^3$, containing the 3D coordinates of grid vertices [S1, unofficial].

**Steps:**
1.  **Grid Generation:** Create a 1D linearly spaced array `lin` from `-scene_scale` to `scene_scale` with `resolution` points. Generate a 3D meshgrid using `torch.meshgrid(lin, lin, lin, indexing="ij")` to produce coordinate tensors `xx`, `yy`, `zz`. Stack these into a point tensor `pts` of shape `[resolution^3, 3]`. The Y and Z axes are swapped during stacking (`[xx, zz, yy]`) to match the NeRF coordinate convention [S1, unofficial].
2.  **Density Query:** Query the NeRF model for density $\sigma$ at each point in `pts`. To manage memory, this is done in batches of size `chunk`. For each batch `i`, pass `pts[i:i+chunk]` to the coarse NeRF model (directions are dummy/ignored for density-only query) to obtain $\sigma$. Concatenate results to form the full density array [S1, unofficial].
3.  **Reshaping:** Reshape the concatenated 1D density array into a 3D grid of shape `[resolution, resolution, resolution]` [S1, unofficial].
4.  **Marching Cubes:** Apply the marching cubes algorithm to the density grid using `sigma_threshold` as the isovalue. This returns vertices, faces, normals, and a gradient array [S1, unofficial].
5.  **Coordinate Rescaling:** The marching cubes algorithm returns vertices in grid index space `[0, resolution-1]`. Rescale these to world coordinates using the formula:
    $$ \text{verts}_{world} = \frac{\text{verts}_{grid}}{\text{resolution} - 1} \times (2 \times \text{scene\_scale}) - \text{scene\_scale} $$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
from skimage import measure

def extract_mesh(coarse_model, resolution=128, scene_scale=1.5, sigma_threshold=50.0, chunk=32768, device="cpu"):
    """
    Extract a mesh from a NeRF density field using marching cubes.
    
    Args:
        coarse_model: The trained NeRF model (coarse stage).
        resolution: Voxel grid resolution.
        scene_scale: Half-extent of the query box.
        sigma_threshold: Iso-value for the surface.
        chunk: Number of points queried per batch.
        device: Device to run computations on.
        
    Returns:
        verts: [V, 3] World space vertices.
        faces: [F, 3] Face indices.
        normals: [V, 3] Vertex normals.
    """
    # 1. Create 1D coordinate array
    lin = torch.linspace(-scene_scale, scene_scale, resolution, device=device)
    
    # 2. Create 3D grid
    xx, yy, zz = torch.meshgrid(lin, lin, lin, indexing="ij")
    
    # 3. Stack and reshape to points (swap Y/Z for NeRF convention)
    pts = torch.stack([xx, zz, yy], dim=-1).reshape(-1, 3)
    
    # 4. Query density sigma in chunks
    sigma_list = []
    num_points = pts.shape[0]
    
    # Dummy directions tensor (shape [N, 3]) - values do not affect density query
    dummy_dirs = torch.zeros_like(pts) 
    
    for i in range(0, num_points, chunk):
        end_idx = min(i + chunk, num_points)
        batch_pts = pts[i:end_idx]
        batch_dirs = dummy_dirs[i:end_idx]
        
        # Query the coarse model
        # [G] Implementation detail: The specific return signature of coarse_model
        # depends on the NeRF implementation. Here we assume it returns a tuple
        # where the second element is sigma.
        _, sigma = coarse_model(batch_pts, batch_dirs)
        
        sigma_list.append(sigma)
        
    # 5. Concatenate and reshape to [resolution, resolution, resolution]
    sigma_grid = torch.cat(sigma_list, dim=0).reshape(resolution, resolution, resolution)
    
    # 6. Run marching cubes
    # [G] skimage.measure.marching_cubes expects a numpy array
    verts_grid, faces, normals, _ = measure.marching_cubes(sigma_grid.cpu().numpy(), level=sigma_threshold)
    
    # 7. Rescale vertices from [0, resolution] to world coordinates
    # Formula: v_world = (v_grid / (res - 1)) * (2 * scale) - scale
    verts = verts_grid / (resolution - 1) * (2 * scene_scale) - scene_scale
    
    return verts, faces, normals
```

**Parameters:**
*   `resolution`: Voxel grid resolution. Default: 128 [S1, unofficial].
*   `scene_scale`: Half-extent of the query box. Default: 1.5 [S1, unofficial].
*   `sigma_threshold`: Iso-value for the surface. Default: 50.0 [S1, unofficial].
*   `chunk`: Number of points queried per batch. Default: 32768 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Coordinate Convention:** The Y and Z axes must be swapped during grid creation to match the NeRF coordinate system (Y-up, Z-forward) [S1, unofficial].
*   **Threshold Selection:** The `sigma_threshold` is critical; too low yields a noisy or incomplete mesh, while too high causes it to shrink or disappear [S1, unofficial].
*   **Memory:** Querying the entire grid at once can exceed GPU memory, making chunking necessary for high resolutions [S1, unofficial].

## Subtopics

### Differentiable Marching Tetrahedrons (DMTet)
A differentiable surface extraction method used to create a triangle mesh from a Signed Distance Field (SDF) [S2, unofficial]. Introduced by Shen et al. in 2021, it is used in methods like JOC to generate triangle meshes initialized by the SDF from a neural field, allowing for joint optimization of geometry, material, and light through backpropagation [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ where $N$ is the number of grid points ($resolution^3$) and $M$ is the cost of a single NeRF forward pass [G]. The marching cubes step is $O(resolution^3)$ but is negligible compared to the neural query [G].
*   **Memory Complexity:** $O(N)$ for storing the grid and intermediate query results. Chunking reduces peak memory usage for the query phase [G].
*   **Performance:** Depends heavily on the NeRF model size and resolution. A 128^3 grid with a standard MLP NeRF can take several minutes to query on a modern GPU [G].
*   **Strengths:** Simple to implement; produces watertight meshes; leverages mature marching cubes implementations [G].
*   **Limitations:** Fixed resolution leads to aliasing or excessive memory usage; sensitive to the choice of `sigma_threshold`; does not adapt to surface complexity [G].
*   **When to use:** When a quick, approximate mesh is needed from a standard NeRF without complex optimization [G].
*   **When not to use:** When high-fidelity geometry is required (use MISE or FlexiCubes) or when the scene is very large (requires octree-based approaches) [G].

## Background
This method bridges the gap between implicit neural representations and explicit mesh-based rendering [G]. It is commonly implemented as a post-processing step following NeRF training [S1, unofficial].

## Sources
[S1] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S2] Joint Optimization of Triangle Mesh, Material, and Light from Neural Fields with Neural Radiance Cache, https://doi.org/10.48550/arxiv.2305.16800, fetched 2026-10-26 (unofficial)
