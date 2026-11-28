# Fast Solid Voxelization via Planar Raycasting


## Overview
Fast Solid Voxelization via Planar Raycasting is a rasterization-based method that generates a solid voxel mesh from an arbitrary polyhedron by casting rays along the three world coordinate planes (XY, YZ, XZ) [S1, unofficial]. It solves the problem of converting polygonal meshes into dense 3D grids by determining voxel occupancy through intersection parity, prioritizing raw computational speed over memory efficiency [S1, unofficial]. It differs from the '--safe' rasterizer by being optimized for pure speed but being less memory efficient and less accurate for complex internal structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A polygonal mesh (polyhedron) defined by vertices and faces, typically loaded from .stl or .obj files [S1, unofficial].
*   **Output:** A 3D grid of `uint8_t` values representing voxel states (0: empty, 1: shell, 2+: interior depth) [S1, unofficial].

**Data Structures:**
The method relies on three pre-computed 3D buffers to accelerate raycasting [S1, unofficial]. Each buffer stores face indices for a specific projection plane [S1, unofficial].
*   `buffer3d<mesh::face<id_t>> _xy_plane_buffer`, `_yz_plane_buffer`, `_xz_plane_buffer`: Three 3D buffers storing face indices for each plane orientation, used to accelerate raycasting [S1, unofficial].
*   `voxel_data_t`: A struct `raster_results<buffer3d<uint8_t>, mesh::polyhedron<base_t>>` containing `_arr_dim` (`glm::vec3`), `_voxels` (`buffer3d<uint8_t>`), and `_mesh` (`mesh::polyhedron<base_t>`) [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Calculate `min_face_area` as `poly.avg_face_area() / 10` [S1, unofficial]. Determine grid dimensions `_dim` by taking the ceiling of the polyhedron's bounding box dimensions: `_dim = glm::ceil(_polyhedron.dim())` [S1, unofficial]. Call `prepare_index_buffers(poly, dim, _xy_plane_buffer, _yz_plane_buffer, _xz_plane_buffer, min_face_area)` to populate internal buffers and set `_polyhedron` [S1, unofficial].
2.  **Buffer Setup:** Create `voxel_data_t buf` with dimensions `_dim`, a 3D `uint8_t` buffer initialized to 0, and the polyhedron [S1, unofficial].
3.  **Raycasting Loop:** Execute `planar_raycast<yzx>(buf)`, then `planar_raycast<xzy>(buf)`, then `planar_raycast<xyz>(buf)` [S1, unofficial].
4.  **Planar Raycast Implementation (`planar_raycast<mode>`):**
    *   Iterate over the 2D plane dimensions $(i, j)$ corresponding to the mode (e.g., for YZX, $i$ corresponds to Y, $j$ to Z) [S1, unofficial].
    *   Parallelize this iteration using OpenMP (`#pragma omp parallel for`) [S1, unofficial].
    *   For each pixel $(i, j)$, retrieve the sorted list of intersection indices from the pre-computed plane buffer using `checks::raycast::get_intersections_fast<mode>` [S1, unofficial].
    *   Initialize `is_in = false` and `from = 0` [G].
    *   Iterate through the sorted intersection indices `inters`:
        *   Clamp `inters` to the valid voxel range $[0, \text{dim}-1]$ using `constrain(0, \text{dim}-1, \text{inters})` [S1, unofficial].
        *   Mark the voxel at the intersection as shell type (value 1) [S1, unofficial].
        *   If `from < inters`:
            *   Iterate through intermediate voxels $k$ from `from` to `inters - 1` [S1, unofficial].
            *   Update voxel value: if `voxel != 0`, left-shift by 1 (`voxel = voxel << 1`); if `voxel == 0`, set to 1 [S1, unofficial].
            *   Check if the current seed position holds an interior voxel. If not, update the seed position to the current voxel coordinates $(k, i, j)$ depending on the plane mode [S1, unofficial].
        *   Set `from = inters + 1` [S1, unofficial].
        *   Toggle `is_in = !is_in` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def constrain(val, min_val, max_val):
    return max(min_val, min(val, max_val))

def planar_raycast(mode, buf, mesh, dim, plane_buffer):
    # Map mode to 2D iteration dimensions and 3D coordinate generation
    if mode == 'yzx':
        dim_i, dim_j = dim.y, dim.z
        get_coords = lambda k, i, j: (k, i, j)
    elif mode == 'xzy':
        dim_i, dim_j = dim.x, dim.z
        get_coords = lambda k, i, j: (i, k, j)
    elif mode == 'xyz':
        dim_i, dim_j = dim.x, dim.y
        get_coords = lambda k, i, j: (i, j, k)
    else:
        raise ValueError("Unknown mode")

    # Parallelize iteration over 2D plane dimensions using OpenMP
    for i in range(dim_i):
        for j in range(dim_j):
            # Retrieve sorted intersection indices from pre-computed plane buffer
            inters = get_intersections_fast(mode, i, j, plane_buffer, mesh)
            if not inters:
                continue

            is_in = False
            from_idx = 0

            for inters_idx in inters:
                # Clamp to valid voxel range [0, dim-1]
                current_idx = constrain(inters_idx, 0, dim[mode] - 1)

                # Mark corresponding voxel as shell type
                coords = get_coords(current_idx, i, j)
                buf.voxels[coords] = 1

                # Iterate through intermediate voxels between shells
                if from_idx < current_idx:
                    for k in range(from_idx, current_idx):
                        coords = get_coords(k, i, j)
                        voxel_val = buf.voxels[coords]
                        if voxel_val != 0:
                            buf.voxels[coords] = voxel_val << 1
                        else:
                            buf.voxels[coords] = 1

                        # Check if current seed position holds interior voxel
                        # [G] Update seed position to (k, i, j) if needed
                        pass

                from_idx = current_idx + 1
                is_in = not is_in

def voxelize(mesh, dim):
    # Constructor initialization
    min_face_area = mesh.avg_face_area() / 10.0
    _dim = [int(d) for d in dim] # glm::ceil approximation

    # Initialize plane buffers
    xy_buf = buffer3d(mesh.face, _dim[0], _dim[1], 1)
    yz_buf = buffer3d(mesh.face, _dim[1], _dim[2], 1)
    xz_buf = buffer3d(mesh.face, _dim[0], _dim[2], 1)

    # Populate plane buffers
    prepare_index_buffers(mesh, _dim, xy_buf, yz_buf, xz_buf, min_face_area)

    # Initialize voxel data buffer
    buf = voxel_data_t()
    buf.arr_dim = _dim
    buf.voxels = buffer3d(uint8, _dim[0], _dim[1], _dim[2], 0)
    buf.mesh = mesh

    # Execute planar raycasts
    planar_raycast('yzx', buf, mesh, _dim, yz_buf)
    planar_raycast('xzy', buf, mesh, _dim, xz_buf)
    planar_raycast('xyz', buf, mesh, _dim, xy_buf)

    return buf
```

**Parameters:**
*   `seed_size`: Initial size for the seed tracking mechanism, default value 128 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Infill Issues:** The method casts rays along world coordinate planes rather than checking every cell individually, potentially missing internal structures not aligned with these projections [S1, unofficial].
*   **Precision:** Uses `FLT_EPSILON * 10` as a precision bias for geometric checks [S1, unofficial].
*   **Memory:** Allocates complete 3D buffers, which is not memory efficient for large resolutions [S1, unofficial].

## Operating characteristics
**Time Complexity:**
The method is optimized for pure speed. Parallelization over the 2D plane dimensions using OpenMP reduces wall-clock time significantly [S1, unofficial].

**Memory Complexity:**
High memory usage due to the allocation of complete 3D buffers for voxel data and plane index buffers [S1, unofficial].

**Measured Performance:**
On an Intel i7 4770k (4 physical cores) with a Stanford Bunny mesh (112k faces):
*   64³ resolution: 59 ms
*   128³ resolution: 71 ms
*   256³ resolution: 134 ms
*   512³ resolution: 409 ms
*   1024³ resolution: 2918 ms
*   2048³ resolution: 28509 ms [S1, unofficial]

**Strengths and Limitations:**
*   **Strengths:** Very fast for solid voxelization; simple implementation; parallelizable [S1, unofficial].
*   **Limitations:** Not memory efficient; less accurate for complex internal structures; prone to missing infill not aligned with world axes; does not handle edge collisions as robustly as the '--safe' mode [S1, unofficial].

**When to Use:**
Use when raw speed is the primary concern and the mesh is relatively simple or does not require precise internal structure representation [S1, unofficial]. Avoid when memory is constrained or high accuracy for complex internal geometries is required [S1, unofficial].

## Background
The method is part of the 3DStuff voxelizer project, an open-source tool for converting polygonal meshes into voxel data [S1, unofficial]. It was developed as a high-performance alternative to more memory-intensive voxelization techniques, specifically targeting the '--fast' mode use case for generating solid voxel models [S1, unofficial].

## Sources
[S1] 3DStuff/voxelizer, https://github.com/3DStuff/voxelizer, fetched 2026-10-27 (unofficial)
