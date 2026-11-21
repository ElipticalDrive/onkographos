# Mesh Voxelization

Also called: STL Mesh Voxelization, Raycasting-based Voxelization, VoxelGrid from Mesh (Ray Method)

## Overview
Mesh Voxelization converts a continuous polygon mesh (typically STL or OBJ) into a discrete 3D grid of voxels, determining the occupancy (solid/empty) and optionally the signed distance field (SDF) for each cell [S1, unofficial][S4, unofficial]. It solves the problem of discretizing surface geometry for volume rendering, simulation, or 3D printing preparation [G]. Unlike [Mesh to Voxel Generation via AABB-Triangle Intersection](3775-mesh-to-voxel-generation-via-aabb-triangle-intersection.md), which relies on bounding box overlap, or [Template-based Voxelization](7964-template-based-voxelization.md), which uses precomputed patterns, this method uses ray casting to robustly determine topological inside/outside status, handling complex concavities and open meshes via multi-directional sampling [S2, unofficial][S4, unofficial].

## How it works
**Inputs**: A triangle mesh (vertices and indices), a target resolution (grid dimensions or voxel size), and optional parameters for interior filling, narrow bands, and double-sided mesh handling [S2, unofficial][S3, unofficial][S4, unofficial].
**Outputs**: A voxel grid (binary occupancy or SDF values), often exported as a point cloud of voxel centers, a binary volume (binvox), or a mesh representation of the voxel surface [S2, unofficial][S3, unofficial][S4, unofficial].

### Data Structures
1.  **Mesh**: List of triangles, each defined by 3 vertices $(V_1, V_2, V_3)$ in 3D space [S2, unofficial][S4, unofficial].
2.  **Voxel Grid**: A 3D array or hash map indexed by $(i, j, k)$ corresponding to spatial coordinates [G].
    *   *Layout*: Bounding box $(bb_{min}, bb_{max})$ derived from mesh vertices [S2, unofficial][S4, unofficial]. Spacing $s = \max(bb_x, bb_y, bb_z) / (dim - 2)$ for a cubic grid of size $dim$ [S2, unofficial]. Origin shifted by $-0.5 \cdot s$ [S2, unofficial].
3.  **Ray**: Defined by origin $O$ and direction $D$ [G].

### Algorithm Steps

1.  **Preprocessing**:
    *   Load mesh vertices and triangles. Validate that index arrays are multiples of 3 [S4, unofficial].
    *   Compute the Axis-Aligned Bounding Box (AABB) of the mesh: $bb_{min}, bb_{max}$ [S2, unofficial][S4, unofficial].
    *   Calculate grid dimensions: $N_x = \lceil (bb_{max,x} - bb_{min,x}) / resolution \rceil$, similarly for $N_y, N_z$ [S4, unofficial].
    *   If a `narrow_band` is specified, expand bounds: $bb_{min} -= band$, $bb_{max} += band$ [S4, unofficial].

2.  **Surface Voxelization (Ray Casting)**:
    For each voxel center $P$ within the grid bounds:
    *   Cast one or more rays from $P$ to determine if it is inside the mesh [S2, unofficial][S4, unofficial].
    *   **Single Ray (Closed Mesh)**: Cast a ray along a fixed axis (e.g., +X) [S4, unofficial]. Count intersections with mesh triangles.
        *   Use Möller–Trumbore intersection test for each triangle [S2, unofficial].
        *   If the mesh is double-sided (e.g., 2D surfaces with thickness), check if $(intersections / 2) \% 2 == 1$ [S2, unofficial].
        *   Otherwise, use parity: $is\_inside = (intersections \% 2) != 0$ [S4, unofficial].
    *   **Multi-Directional Sampling (Open/Complex Mesh)**: If the mesh is not watertight or `samples > 1`, cast $N$ rays in random directions distributed on a sphere [S2, unofficial].
        *   Generate direction from spherical coordinates $(\theta, z)$:
            $$
            \begin{aligned}
            dir.x &= \sqrt{1-z^2} \cos(\theta) \\
            dir.y &= \sqrt{1-z^2} \sin(\theta) \\
            dir.z &= \sqrt{1-z^2} \cos(\theta) \quad \text{(Note: Source uses cos for z, likely intended sin for one component, but implemented as cos)}
            \end{aligned}
            $$
            Where $\theta = u_1 \cdot 2\pi$, $z = 2u_2 - 1$ for uniform randoms $u_1, u_2 \in [0,1]$ [S2, unofficial].
        *   Vote for inside/outside status based on the majority of ray results [S2, unofficial].

3.  **Interior Filling (Optional)**:
    *   If `fill_interior` is enabled, fill enclosed empty voxels [S3, unofficial].
    *   The implementation calls `voxel_grid.fill()` to fill enclosed empty voxels, catching exceptions for problematic meshes [S3, unofficial].
    *   Alternatively, perform 3D scan-filling along all three axes (X, Y, Z) and intersect the results to eliminate singular point errors [S5, unofficial].

4.  **Signed Distance Field (SDF) Computation (Optional)**:
    *   For each voxel center $P$, compute the distance to the nearest mesh surface: $d = \text{mesh.distance}(P)$ [S4, unofficial].
    *   Assign SDF value: $SDF = -d$ if $is\_inside$, else $SDF = d$ [S4, unofficial].
    *   Apply narrow band filter: Keep voxel only if $|SDF| \le band$ or if `surface_only` is true [S4, unofficial].

5.  **Output Generation**:
    *   Extract centers of occupied voxels.
    *   Export to formats such as `.obj` (vertices), `.binvox` (binary volume), `.csv`/`.txt` (x, y, z, voxel_size), or `.ply` [S2, unofficial][S3, unofficial][S4, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def voxelize_mesh(mesh_vertices, mesh_triangles, resolution, samples=1, fill_interior=False, narrow_band=None, double_thick=False):
    # 1. Preprocessing
    bb_min = [min(v[i] for v in mesh_vertices) for i in range(3)]
    bb_max = [max(v[i] for v in mesh_vertices) for i in range(3)]
    
    if narrow_band is not None:
        bb_min = [bb_min[i] - narrow_band for i in range(3)]
        bb_max = [bb_max[i] + narrow_band for i in range(3)]
        
    size = [bb_max[i] - bb_min[i] for i in range(3)]
    nx = math.ceil(size[0] / resolution)
    ny = math.ceil(size[1] / resolution)
    nz = math.ceil(size[2] / resolution)
    
    occupied_voxels = []
    
    # 2. Iterate over grid
    for iy in range(ny):
        for iz in range(nz):
            y = bb_min[1] + (iy * resolution) + (resolution * 0.5)
            z = bb_min[2] + (iz * resolution) + (resolution * 0.5)
            
            for ix in range(nx):
                x = bb_min[0] + (ix * resolution) + (resolution * 0.5)
                point = [x, y, z]
                
                is_inside = False
                
                if samples == 1:
                    ray_origin = [bb_min[0] - 1.0, y, z]
                    ray_dir = [1.0, 0.0, 0.0]
                    max_dist = (bb_max[0] - ray_origin[0]) + 1.0
                    
                    intersections = count_intersections(mesh_triangles, ray_origin, ray_dir, max_dist)
                    
                    # Handle double-thick meshes
                    if double_thick:
                        is_inside = (intersections // 2) % 2 == 1
                    else:
                        is_inside = (intersections % 2) != 0
                else:
                    votes_inside = 0
                    for _ in range(samples):
                        theta = random.uniform(0, 1) * 2 * math.pi
                        z_coord = 2 * random.uniform(0, 1) - 1
                        dir_x = math.sqrt(1 - z_coord**2) * math.cos(theta)
                        dir_y = math.sqrt(1 - z_coord**2) * math.sin(theta)
                        dir_z = math.sqrt(1 - z_coord**2) * math.cos(theta) # Source specific
                        ray_dir = [dir_x, dir_y, dir_z]
                        
                        intersections = count_intersections(mesh_triangles, point, ray_dir, 1e6)
                        if (intersections % 2) != 0:
                            votes_inside += 1
                    
                    is_inside = votes_inside > (samples // 2)
                
                if is_inside:
                    sdf_val = 0.0
                    if narrow_band is not None:
                        # Standard point-to-triangle distance calculation [G]
                        dist = mesh_distance(mesh_triangles, point)
                        sdf_val = -dist if is_inside else dist
                        if abs(sdf_val) > narrow_band:
                            continue
                    
                    occupied_voxels.append((x, y, z, sdf_val))
                    
    # 3. Interior Filling (if requested)
    # Standard flood-fill or scan-fill to enclose empty voxels [G]
    if fill_interior:
        occupied_voxels = fill_interior_voxels(occupied_voxels, bb_min, bb_max, resolution)
        
    return occupied_voxels

def count_intersections(triangles, origin, direction, max_dist):
    count = 0
    current_pos = origin[:]
    max_hits = 100000 
    
    for _ in range(max_hits):
        hit_point, hit_dist = find_nearest_intersection(triangles, current_pos, direction)
        
        if hit_dist is None or hit_dist > max_dist:
            break
            
        count += 1
        
        # Advance ray origin to prevent re-hitting the same face
        current_pos = [current_pos[i] + direction[i] * (hit_dist + 1e-4) for i in range(3)]
        
    return count

def find_nearest_intersection(triangles, origin, direction):
    closest_dist = float('inf')
    closest_point = None
    
    for tri in triangles:
        v1, v2, v3 = tri
        e1 = [v2[i] - v1[i] for i in range(3)]
        e2 = [v3[i] - v1[i] for i in range(3)]
        h = [direction[1]*e2[2] - direction[2]*e2[1], 
             direction[2]*e2[0] - direction[0]*e2[2], 
             direction[0]*e2[1] - direction[1]*e2[0]]
        a = e1[0]*h[0] + e1[1]*h[1] + e1[2]*h[2]
        
        if abs(a) < 1e-8:
            continue
            
        inv_a = 1.0 / a
        f = [direction[i] * inv_a for i in range(3)]
        s = [origin[i] - v1[i] for i in range(3)]
        u = (s[0]*h[0] + s[1]*h[1] + s[2]*h[2]) * inv_a
        
        if u < 0.0 or u > 1.0:
            continue
            
        q = [s[1]*e1[2] - s[2]*e1[1], 
             s[2]*e1[0] - s[0]*e1[2], 
             s[0]*e1[1] - s[1]*e1[0]]
        v = (direction[0]*q[0] + direction[1]*q[1] + direction[2]*q[2]) * inv_a
        
        if v < 0.0 or u + v > 1.0:
            continue
            
        t = (e2[0]*q[0] + e2[1]*q[1] + e2[2]*q[2]) * inv_a
        
        if t > 1e-8:
            if t < closest_dist:
                closest_dist = t
                closest_point = [origin[i] + direction[i] * t for i in range(3)]
                
    if closest_point is not None:
        return closest_point, closest_dist
    return None, None

def mesh_distance(triangles, point):
    # Not given in the sources. [G]
    return 0.0

def fill_interior_voxels(voxels, bb_min, bb_max, resolution):
    # Not given in the sources. [G]
    return voxels
```

### Parameters
*   `resolution`: Voxel grid spacing (float). Must be $> 10^{-6}$ to prevent division by zero or excessive memory [S4, unofficial]. Default varies (e.g., 0.5 in `voxelizer-rs`, 3.0 in GUI) [S3, unofficial][S4, unofficial].
*   `samples`: Number of sample rays per voxel (int). Default 1 (single ray). Higher values handle open meshes [S2, unofficial].
*   `fill_interior`: Boolean. If true, fills enclosed empty voxels. Default True [S3, unofficial].
*   `narrow_band`: Float. Distance threshold for SDF. Expands bounds and filters voxels. Default None [S4, unofficial].
*   `surface_only`: Boolean. If true, retains only voxels intersecting the surface, ignoring interior. Default False [S4, unofficial].
*   `double_thick`: Boolean. Handles double-sided geometry by checking `(intersections/2)%2 == 1`. Default False [S2, unofficial].

### Edge Cases and Pitfalls
*   **Ray Re-hitting**: Floating-point precision can cause a ray to intersect the same face multiple times. Mitigated by advancing the ray origin by a small epsilon ($10^{-4}$) after each hit [S4, unofficial].
*   **Open Meshes**: Single-ray casting fails for non-watertight meshes. Mitigated by multi-directional sampling and voting [S2, unofficial].
*   **Double-Thick Meshes**: Standard odd-even rule fails. Requires specific parity check `(intersections/2)%2 == 1` [S2, unofficial].
*   **Precision Ordering**: Intersection points may be returned out of order due to precision errors. Sorting is required if exact distance ordering is needed, though simple counting often suffices for parity [S4, unofficial].
*   **Memory Usage**: High resolutions generate massive files. A 256x256x256 resolution can produce >4.7 GB OBJ files [S2, unofficial]. Use binary formats (binvox) for high res [S2, unofficial].

## Subtopics

### [Solid Voxelization based on Octree](2062-solid-voxelization-based-on-octree.md)
Solid Voxelization based on Octree converts a polygon mesh (specifically OBJ format) into a hierarchical octree representation to determine the solid/empty status and distance to surface for each leaf node. It differs from fixed-grid voxelization by using an octree structure for adaptive resolution and from ray-casting-only methods by combining spatial partitioning with ray casting for sign determination. It builds on [Mesh Voxelization](6514-mesh-voxelization.md) by providing the specific octree-based implementation for mesh inputs.

### Solid Voxelization via 3D Scan-Fill
An extension of surface voxelization for closed objects that fills the inner region of the object using a 3D scan-filling operation [S5, unofficial]. It traverses the volume slice by slice and line by line, maintaining an inside/outside flag for each scan line that toggles when crossing the object boundary [S5, unofficial]. To eliminate errors from singular points, it performs scan-filling along all three axis directions and intersects the results [S5, unofficial].

## Operating characteristics
*   **Memory Usage**: High resolutions generate massive files. A 256x256x256 resolution can produce >4.7 GB OBJ files [S2, unofficial]. Use binary formats (binvox) for high res [S2, unofficial].
*   **Performance**: GPU acceleration (CUDA) provides massive speedups (400-2750x) over CPU. On an NVIDIA GTX 970, a 512^3 voxelization of a teapot (2464 triangles) took 15.64 seconds [S2, unofficial]. A 1024^3 sphere (960 triangles) took 49.93 seconds [S2, unofficial].
*   **Strengths**: Robust handling of complex geometry; produces watertight volumes; supports SDF generation [S2, unofficial][S4, unofficial].
*   **Limitations**: Computationally expensive for high resolutions without acceleration; sensitive to mesh quality (non-manifold edges, double-sided faces); large output file sizes for dense grids [S2, unofficial][S4, unofficial].

## Background
Implementations often leverage GPU parallelism (CUDA) to handle the massive number of ray-triangle intersection tests required for high-resolution grids [S2, unofficial]. The Möller–Trumbore algorithm is the standard efficient method for ray-triangle intersection in this context [S2, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] kctess5/voxelizer, https://github.com/kctess5/voxelizer, fetched 2026-10-26 (unofficial)
[S3] Guda108/3D-Voxelization-GUI, https://github.com/Guda108/3D-Voxelization-GUI, fetched 2026-10-27 (unofficial)
[S4] voxelizer-rs, https://github.com/lhemerly/voxelizer-rs, fetched 2026-10-28 (unofficial)
[S5] (PDF) Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
