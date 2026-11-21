# Fast Solid Voxelization via Planar Raycasting


## Overview
Fast Solid Voxelization via Planar Raycasting is a rasterization-based method that generates a solid voxel mesh from an arbitrary polyhedron by casting rays along the three world coordinate planes (XY, YZ, XZ) [S1, unofficial]. It solves the problem of converting polygonal meshes into dense 3D grids by determining voxel occupancy through intersection parity, prioritizing raw computational speed over memory efficiency [S1, unofficial]. It differs from its sibling [A-Buffer Solid Voxelization](1189-a-buffer-solid-voxelization.md) and [3D Grid Rasterization](903-3d-grid-rasterization.md) by using pre-computed plane buffers and bit-shifting accumulation rather than per-voxel raycasting or hierarchical data structures, making it significantly faster but less accurate for complex internal structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A polygonal mesh (polyhedron) defined by vertices and faces, typically loaded from .stl or .obj files [S1, unofficial].
*   **Output:** A 3D grid of `uint8_t` values representing voxel states (0: empty, 1: shell, 2+: interior depth) [S1, unofficial].

**Data Structures:**
The method relies on three pre-computed 3D buffers to accelerate raycasting. Each buffer stores face indices for a specific projection plane.
*   `buffer3d<mesh::face<id_t>> _xy_plane_buffer`: Stores face indices projected onto the XY plane. Dimensions are $D_x \times D_y \times 1$ (conceptually 2D, stored in 3D structure).
*   `buffer3d<mesh::face<id_t>> _yz_plane_buffer`: Stores face indices projected onto the YZ plane.
*   `buffer3d<mesh::face<id_t>> _xz_plane_buffer`: Stores face indices projected onto the XZ plane.
*   `voxel_data_t`: A struct containing:
    *   `_arr_dim`: `glm::vec3` representing the voxel grid dimensions.
    *   `_voxels`: `buffer3d<uint8_t>` initialized to 0.
    *   `_mesh`: The input `mesh::polyhedron<base_t>`.

**Algorithm Steps:**

1.  **Initialization:**
    *   Calculate `min_face_area` as `poly.avg_face_area() / 10` [S1, unofficial].
    *   Determine grid dimensions `_dim` by taking the ceiling of the polyhedron's bounding box dimensions: `_dim = glm::ceil(_polyhedron.dim())` [S1, unofficial].
    *   Call `prepare_index_buffers(poly, dim, _xy_plane_buffer, _yz_plane_buffer, _xz_plane_buffer, min_face_area)` to populate the plane buffers with face indices based on their projected overlap with grid cells [S1, unofficial].

2.  **Raycasting Loop:**
    *   Initialize `voxel_data_t buf` with dimensions `_dim` and a 3D `uint8_t` array set to 0 [S1, unofficial].
    *   Execute `planar_raycast<yzx>(buf)`.
    *   Execute `planar_raycast<xzy>(buf)`.
    *   Execute `planar_raycast<xyz>(buf)` [S1, unofficial].

3.  **Planar Raycast Implementation (`planar_raycast<mode>`):**
    *   Iterate over the 2D plane dimensions $(i, j)$ corresponding to the mode (e.g., for YZX, $i$ corresponds to Y, $j$ to Z) [S1, unofficial].
    *   Parallelize this iteration using OpenMP (`#pragma omp parallel for`) [S1, unofficial].
    *   For each pixel $(i, j)$, retrieve the sorted list of intersection indices from the pre-computed plane buffer using `checks::raycast::get_intersections_fast<mode>` [S1, unofficial].
    *   Initialize `is_in = false` and `from = 0` [G].
    *   Iterate through the sorted intersection indices `inters`:
        *   Clamp `inters` to the valid voxel range $[0, \text{dim}-1]$ using `constrain(0, _dim-1, inters)` [S1, unofficial].
        *   Mark the voxel at the intersection as shell type (value 1) [S1, unofficial].
        *   If `from < inters`:
            *   Iterate through intermediate voxels $k$ from `from` to `inters - 1` [S1, unofficial].
            *   Update voxel value:
                *   If `voxel != 0`: `voxel = voxel << 1` (left-shift by 1) [S1, unofficial].
                *   If `voxel == 0`: `voxel = 1` [S1, unofficial].
            *   Check if the current seed position holds an interior voxel. If not, update the seed position to the current voxel coordinates $(k, i, j)$ depending on the plane mode [S1, unofficial].
        *   Set `from = inters + 1` [S1, unofficial].
        *   Toggle `is_in = !is_in` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from typing import List, Tuple

def prepare_index_buffers(mesh, dim, xy_buf, yz_buf, xz_buf, min_face_area):
    """
    Populates plane buffers with face indices based on projected overlap.
    Implementation detail: Not given in the sources.
    """
    pass

def constrain(val, min_val, max_val):
    return max(min_val, min(val, max_val))

def get_intersections_fast(mode, i, j, plane_buffer, mesh):
    """
    Retrieves sorted intersection indices for a specific pixel on a plane.
    Implementation detail: Not given in the sources.
    """
    pass

def planar_raycast(mode, buf, mesh, dim):
    """
    Performs raycasting for a specific plane orientation.
    mode: 'yzx', 'xzy', or 'xyz'
    """
    # Determine plane dimensions based on mode
    if mode == 'yzx':
        dim_i, dim_j = dim.y, dim.z
        get_coords = lambda k, i, j: (k, i, j) # x=k, y=i, z=j
    elif mode == 'xzy':
        dim_i, dim_j = dim.x, dim.z
        get_coords = lambda k, i, j: (i, k, j) # x=i, y=k, z=j
    elif mode == 'xyz':
        dim_i, dim_j = dim.x, dim.y
        get_coords = lambda k, i, j: (i, j, k) # x=i, y=j, z=k
    else:
        raise ValueError("Unknown mode")

    # Parallel iteration over 2D plane
    for i in range(dim_i):
        for j in range(dim_j):
            # Get sorted intersection indices
            inters = get_intersections_fast(mode, i, j, 
                                            _plane_buffers[mode], mesh)
            
            if not inters:
                continue

            is_in = False
            from_idx = 0
            
            for inters_idx in inters:
                # Clamp to valid range
                current_idx = constrain(inters_idx, 0, dim[mode_to_axis(mode)] - 1)
                
                # Mark shell
                coords = get_coords(current_idx, i, j)
                buf.voxels[coords] = 1
                
                # Process interval between shells
                if from_idx < current_idx:
                    for k in range(from_idx, current_idx):
                        coords = get_coords(k, i, j)
                        voxel_val = buf.voxels[coords]
                        if voxel_val != 0:
                            buf.voxels[coords] = voxel_val << 1
                        else:
                            buf.voxels[coords] = 1
                        
                        # Seed position update logic
                        # Note: Specific seed logic is not fully detailed in sources
                        # [G] Assuming standard interior tracking
                        pass
                
                from_idx = current_idx + 1
                is_in = not is_in

def voxelize(mesh, dim):
    """
    Main voxelization function.
    """
    # Calculate min face area
    min_face_area = mesh.avg_face_area() / 10.0
    
    # Initialize buffers
    xy_buf = buffer3d(mesh::face<id_t>, dim.x, dim.y, 1)
    yz_buf = buffer3d(mesh::face<id_t>, dim.y, dim.z, 1)
    xz_buf = buffer3d(mesh::face<id_t>, dim.x, dim.z, 1)
    
    # Prepare buffers
    prepare_index_buffers(mesh, dim, xy_buf, yz_buf, xz_buf, min_face_area)
    
    # Initialize voxel data
    buf = voxel_data_t()
    buf.arr_dim = dim
    buf.voxels = buffer3d<uint8_t>(dim.x, dim.y, dim.z, 0)
    buf.mesh = mesh
    
    # Execute raycasts
    planar_raycast('yzx', buf, mesh, dim)
    planar_raycast('xzy', buf, mesh, dim)
    planar_raycast('xyz', buf, mesh, dim)
    
    return buf
```

**Parameters:**
*   `seed_size`: Initial size for the seed tracking mechanism, default value 128 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Infill Issues:** The method casts rays along world coordinate planes rather than checking every cell individually. This can miss internal structures not aligned with these projections, leading to incorrect infill [S1, unofficial].
*   **Precision:** Uses `FLT_EPSILON * 10` as a precision bias for geometric checks [S1, unofficial].
*   **Memory:** Allocates complete 3D buffers, which is not memory efficient for large resolutions [S1, unofficial].

## Operating characteristics
**Time Complexity:**
The method is optimized for pure speed. Parallelization over the 2D plane dimensions using OpenMP reduces wall-clock time significantly [S1, unofficial].

**Memory Complexity:**
High memory usage due to the allocation of complete 3D buffers for voxel data and plane index buffers. It is less memory efficient than [A-Buffer Solid Voxelization](1189-a-buffer-solid-voxelization.md) which uses quad trees and projections [S1, unofficial].

**Measured Performance:**
On an Intel i7 4770k (4 physical cores) with a Stanford Bunny mesh (112k faces):
*   64³ resolution: 59 ms
*   128³ resolution: 71 ms
*   256³ resolution: 134 ms
*   512³ resolution: 409 ms
*   1024³ resolution: 2918 ms
*   2048³ resolution: 28509 ms [S1, unofficial]

**Strengths and Limitations:**
*   **Strengths:** Very fast for solid voxelization; simple implementation; parallelizable.
*   **Limitations:** Not memory efficient; less accurate for complex internal structures; prone to missing infill not aligned with world axes; does not handle edge collisions as robustly as the '--safe' mode [S1, unofficial].

**When to Use:**
Use when raw speed is the primary concern and the mesh is relatively simple or does not require precise internal structure representation. Avoid when memory is constrained or high accuracy for complex internal geometries is required.

## Background
The method is part of the 3DStuff voxelizer project, an open-source tool for converting polygonal meshes into voxel data [S1, unofficial]. It was developed as a high-performance alternative to more memory-intensive voxelization techniques, specifically targeting the '--fast' mode use case for generating solid voxel models [S1, unofficial].

## Sources
[S1] 3DStuff/voxelizer, https://github.com/3DStuff/voxelizer, fetched 2026-10-27 (unofficial)
