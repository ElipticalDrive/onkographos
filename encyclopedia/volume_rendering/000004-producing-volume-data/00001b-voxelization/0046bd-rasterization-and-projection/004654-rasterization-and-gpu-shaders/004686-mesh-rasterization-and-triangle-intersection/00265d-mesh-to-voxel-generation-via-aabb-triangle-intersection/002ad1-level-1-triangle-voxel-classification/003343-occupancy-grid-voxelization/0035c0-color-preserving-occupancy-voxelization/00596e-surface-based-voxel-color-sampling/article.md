# Surface-based Voxel Color Sampling

Also called: Surface Voxel Color Assignment

## Overview
Surface-based Voxel Color Sampling assigns colors to voxels by sampling the nearest point on the mesh surface rather than the voxel center, avoiding color errors when voxel centers fall inside empty space [S1, unofficial]. It differs from center-based sampling by ensuring that surface voxels receive colors derived from the actual geometry intersection rather than potentially empty interior points [S1, unofficial].

## How it works
**Inputs:**
- A filled voxel grid (binary occupancy mask) [S1, unofficial].
- A mesh object containing geometry, and optionally vertex colors or texture UVs [S1, unofficial].
- Voxel grid parameters: `pitch` (voxel size), `voxel_origins` (world coordinate of the grid origin) [S1, unofficial].
- Resolution `N` (assumed cubic grid $N \times N \times N$) [S1, unofficial].

**Outputs:**
- A color array or voxel data structure containing RGB values for each surface voxel [S1, unofficial].
- Voxels not on the surface may be left empty or assigned a default color [S1, unofficial].

**Data Structures:**
- `filled_mask`: Boolean array of shape $(N, N, N)$, true if the voxel intersects the mesh [S1, unofficial].
- `surface_mask`: Boolean array of shape $(N, N, N)$, true if the voxel is on the surface [S1, unofficial].
- `world_points`: Array of shape $(K, 3)$, world coordinates of the centers of surface voxels [S1, unofficial].
- `closest_points`: Array of shape $(K, 3)$, world coordinates of the nearest points on the mesh surface [S1, unofficial].
- `face_indices`: Array of shape $(K,)$, indices of the mesh faces containing the closest points [S1, unofficial].
- `colors`: Array of shape $(K, 3)$, RGB values for each surface voxel [S1, unofficial].

**Steps:**

1. **Identify Surface Voxels:**
   Erode the `filled_mask` to remove interior voxels, then subtract the eroded mask from the filled mask to isolate the surface layer [S1, unofficial].
   $$ \text{eroded} = \text{erode}(\text{filled}) $$
   $$ \text{surface} = \text{filled} \ \& \ \sim \text{eroded} $$
   *Note: The erosion operation typically uses a 3x3x3 kernel or checks all 26 neighbors. If a voxel has any empty neighbor, it is considered surface [G].*

2. **Convert to World Coordinates:**
   For each voxel index $(i, j, k)$ where `surface[i, j, k]` is true, compute the world center coordinate [S1, unofficial]:
   $$ \mathbf{p}_{world} = (i, j, k) \times \text{pitch} + \text{voxel\_origins} + \frac{\text{pitch}}{2} $$
   Collect all such $\mathbf{p}_{world}$ into `world_points` [S1, unofficial].

3. **Nearest Surface Query:**
   Perform a nearest-neighbor query on the mesh surface for each point in `world_points` [S1, unofficial]:
   $$ \text{closest\_points}, \_, \text{face\_indices} = \text{mesh.nearest.on\_surface}(\text{world\_points}) $$
   This returns the closest point on the mesh geometry and the index of the triangle (face) containing that point [S1, unofficial].

4. **Color Assignment:**
   Assign colors based on available mesh data [S1, unofficial]:

   **Case A: Vertex Colors**
   If the mesh has vertex colors, for each surface voxel $m$ with corresponding face index $f_m$:
   - Let $V_f$ be the set of 3 vertices of face $f_m$ [G].
   - Let $C_v$ be the color of vertex $v$ [G].
   - Compute the average color:
     $$ \mathbf{c}_m = \frac{1}{3} \sum_{v \in V_{f_m}} C_v $$
   - Clamp and cast to 8-bit unsigned integer:
     $$ \text{colors}[m] = \text{clamp}(\mathbf{c}_m, 0, 255).astype(\text{uint8}) $$

   **Case B: Texture UVs**
   If the mesh has texture UVs, for each surface voxel $m$ with corresponding face index $f_m$:
   - Let $UV_v$ be the texture coordinate of vertex $v$ of face $f_m$ [G].
   - Compute the average UV:
     $$ \mathbf{uv}_{avg} = \frac{1}{3} \sum_{v \in V_{f_m}} UV_v $$
   - Map to texture pixel coordinates $(t_x, t_y)$ using texture dimensions $T_w \times T_h$:
     $$ t_x = \text{clip}(\mathbf{uv}_{avg}[0] \times T_w, 0, T_w - 1) $$
     $$ t_y = \text{clip}((1 - \mathbf{uv}_{avg}[1]) \times T_h, 0, T_h - 1) $$
     *Note: The y-coordinate is inverted to match typical image coordinate systems [G].*
   - Sample the texture at $(t_x, t_y)$ to get $\mathbf{c}_m$ [G].

   **Case C: No Visual Data**
   If the mesh has no vertex colors or textures, assign a default gray color [S1, unofficial]:
   $$ \mathbf{c}_m = (180, 180, 180) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def surface_based_voxel_color_sampling(filled_mask, mesh, pitch, voxel_origins, resolution):
    """
    Assign colors to surface voxels by sampling the nearest point on the mesh surface.
    
    Args:
        filled_mask: Boolean array (N, N, N) indicating occupied voxels.
        mesh: Object with method nearest.on_surface(points) returning (closest_points, _, face_indices).
              Also has attributes faces, vertex_colors (optional), vertex_uvs (optional), 
              texture_image (optional), texture_width, texture_height.
        pitch: Float, size of one voxel.
        voxel_origins: Array (3,), world coordinates of the grid origin (min corner).
        resolution: Int, N (grid is NxNxN).
        
    Returns:
        colors: Array (N, N, N, 3), RGB values for all voxels. Empty voxels are (0,0,0).
    """
    N = resolution
    colors = np.zeros((N, N, N, 3), dtype=np.uint8)
    
    # Step 1: Identify surface voxels
    from scipy.ndimage import binary_erosion
    eroded_mask = binary_erosion(filled_mask, structure=np.ones((3, 3, 3)))
    surface_mask = filled_mask & ~eroded_mask
    
    surface_indices = np.argwhere(surface_mask) # Shape (K, 3)
    if len(surface_indices) == 0:
        return colors
        
    # Step 2: Convert voxel indices to world coordinates
    world_points = (surface_indices * pitch) + voxel_origins + (pitch / 2.0)
    
    # Step 3: Find closest point on mesh surface
    closest_points, _, face_indices = mesh.nearest.on_surface(world_points)
    
    # Step 4: Assign colors
    if hasattr(mesh, 'vertex_colors') and mesh.vertex_colors is not None:
        # Vertex colors case
        face_vertex_indices = mesh.faces[face_indices]
        vertex_colors_for_faces = mesh.vertex_colors[face_vertex_indices]
        avg_colors = vertex_colors_for_faces.mean(axis=1)
        colors[tuple(surface_indices.T)] = np.clip(avg_colors, 0, 255).astype(np.uint8)
        
    elif hasattr(mesh, 'vertex_uvs') and mesh.vertex_uvs is not None and hasattr(mesh, 'texture_image'):
        # Texture UV case
        face_vertex_indices = mesh.faces[face_indices]
        vertex_uvs_for_faces = mesh.vertex_uvs[face_vertex_indices]
        avg_uvs = vertex_uvs_for_faces.mean(axis=1)
        
        tw = mesh.texture_width
        th = mesh.texture_height
        px = np.clip(avg_uvs[:, 0] * tw, 0, tw - 1).astype(int)
        py = np.clip((1 - avg_uvs[:, 1]) * th, 0, th - 1).astype(int)
        
        sampled_colors = mesh.texture_image[py, px]
        colors[tuple(surface_indices.T)] = sampled_colors
        
    else:
        # Default gray color
        default_color = np.array([180, 180, 180], dtype=np.uint8)
        colors[tuple(surface_indices.T)] = default_color
        
    return colors
```

**Parameters:**
- `pitch`: Voxel size in world units. Not given in the sources.
- `voxel_origins`: World coordinate of the minimum corner of the voxel grid. Not given in the sources.
- Default color: RGB (180, 180, 180) gray [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Voxel Centers in Empty Space:** If center-based sampling is used, voxel centers may land inside empty space, causing incorrect color sampling. This method avoids that by sampling the nearest surface point [S1, unofficial].
- **Degenerate Faces:** If a face has coincident vertices, the average color/UV may be undefined. Standard mesh processing should handle this [G].
- **Texture Bounds:** UVs should be clamped to [0, 1] before mapping to texture coordinates to avoid out-of-bounds access [G].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** In the context of the source pipeline (M4 Pro, 48 GB RAM), voxelization is Stage 3, running in parallel with GPU inference. Performance is bottlenecked by the overall pipeline throughput (~5 seconds per emoji) rather than this specific sampling step [S1, unofficial].
- **Strengths:** Accurate color assignment for surface voxels; avoids artifacts from empty-space center sampling [S1, unofficial].
- **Limitations:** Requires a mesh with surface geometry; does not handle volume interiors (only surface voxels are colored) [G].

## Background
This method is part of a pipeline for generating voxel assets from images (e.g., emojis) using neural networks (TripoSR) and subsequent voxelization [S1, unofficial]. It is used in the "image-to-voxel" project to produce Minecraft-like voxel representations with correct surface colors [S1, unofficial].

## Sources
[S1] mohamedmansour/image-to-voxel, https://github.com/mohamedmansour/image-to-voxel, fetched 2026-10-26 (unofficial)
