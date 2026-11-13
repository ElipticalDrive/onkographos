# Solid Voxelization based on Quadtree

Also called: Safe Voxelization with Quad Trees, safe rasterizer

## Overview
Solid Voxelization based on Quadtree is a memory-efficient voxelization technique that determines the solid, shell, and background status of voxels by storing plane-wise intersection distances rather than allocating a full 3D buffer during computation [S1, unofficial]. It distinguishes itself from sibling methods like Octree-based Solid Voxelization by using 2D quadtree spatial acceleration structures on projected planes to handle huge models or high target resolutions where other voxelizers fail due to memory constraints [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A mesh defined as `mesh::polyhedron<base_t>`, typically imported from `.stl` or `.obj` files [S1, unofficial]. The mesh contains a vertex buffer (`_vertices`) and an index buffer (`_indices._buffer`) representing triangular faces [S1, unofficial].
*   **Output:** A `raster_results` structure containing `_arr_dim` (the voxel grid dimensions), `_voxels` (intersection data), and `_mesh` (the input mesh) [S1, unofficial]. The final voxel types (interior, shell, background) are derived from the intersection counts [S1, unofficial].

**Data Structures**
The method utilizes three 2D quadtree structures to accelerate ray casting:
1.  `_xy_tree`: Stores faces projected onto the XY plane.
2.  `_yz_tree`: Stores faces projected onto the YZ plane.
3.  `_xz_tree`: Stores faces projected onto the XZ plane.

Each quadtree node contains a buffer of face indices. The quadtree is configured with a constant `_branch_size` of 128 [S1, unofficial]. Coordinates are scaled by a constant `_tree_scale` of 2.0f for insertion and search [S1, unofficial].

**Parameters**
*   `_tree_scale`: `2.0f` (float). Scales coordinates for quadtree insertion and search [S1, unofficial].
*   `_branch_size`: `128` (size_t). The maximum number of elements per quadtree node [S1, unofficial].
*   `_tree_search_steps`: Not explicitly defined in the notes, but used in the search function [S1, unofficial].

**Algorithm Steps**

1.  **Initialization and Normalization**
    *   Normalize and scale the input polyhedron vertices by the provided scale vector [S1, unofficial].
    *   Calculate the polyhedron dimension: `_polyhedron_dim = glm::ceil(_polyhedron.dim()) + 1.0f` [S1, unofficial]. The `+1.0f` handles the edge case where a face lies exactly on the array boundary, ensuring voxels can be drawn [S1, unofficial].
    *   Calculate half-dimensions: `dim_half = _polyhedron_dim / 2.0f` [S1, unofficial].
    *   Initialize the three quadtrees (`_xy_tree`, `_yz_tree`, `_xz_tree`) with boundaries:
        *   `_xy_tree`: `dim_half.xy() * _tree_scale`
        *   `_yz_tree`: `dim_half.yz() * _tree_scale`
        *   `_xz_tree`: `dim_half.xz() * _tree_scale` [S1, unofficial].

2.  **Tree Population**
    *   Iterate through each face `f` in `_polyhedron._indices._buffer` [S1, unofficial].
    *   Retrieve the three vertices `v1, v2, v3` of the face and scale them by `_tree_scale` [S1, unofficial].
    *   Calculate the integer bounding box `[min, max]` of the projected face for each plane [S1, unofficial].
    *   Insert the face index `f` into the respective quadtrees for all integer coordinates within the bounding box:
        *   Insert into `_xy_tree` for `x` in `[min.x, max.x]` and `y` in `[min.y, max.y]` [S1, unofficial].
        *   Insert into `_xz_tree` for `x` in `[min.x, max.x]` and `z` in `[min.z, max.z]` [S1, unofficial].
        *   Insert into `_yz_tree` for `y` in `[min.y, max.y]` and `z` in `[min.z, max.z]` [S1, unofficial].

3.  **Plane Rasterization (Intersection Calculation)**
    The method performs parallel loops over each 2D plane to calculate ray intersections. For each pixel in the plane, a ray is cast through the mesh, and intersection distances are stored.

    *   **YZ Plane Rasterization:**
        *   Parallel loop over `y` in `[0, _polyhedron_dim.y)` and `z` in `[0, _polyhedron_dim.z)` [S1, unofficial].
        *   Calculate 2D position: `p = glm::vec2(y, z) * _tree_scale` [S1, unofficial].
        *   Find the relevant face buffer in `_yz_tree` using `p` and `_tree_search_steps` [S1, unofficial].
        *   Get intersections `inters` by calling `get_intersections_safe<yzx>(glm::vec2(y, z), _polyhedron._vertices, buffer)` [S1, unofficial].
        *   Store results in `r.yz[y][z]` [S1, unofficial].

    *   **XZ Plane Rasterization:**
        *   Parallel loop over `x` in `[0, _polyhedron_dim.x)` and `z` in `[0, _polyhedron_dim.z)` [S1, unofficial].
        *   Calculate 2D position: `p = glm::vec2(x, z) * _tree_scale` [S1, unofficial].
        *   Find the relevant face buffer in `_xz_tree` [S1, unofficial].
        *   Get intersections `inters` by calling `get_intersections_safe<xzy>(glm::vec2(x, z), _polyhedron._vertices, buffer)` [S1, unofficial].
        *   Store results in `r.xz[x][z]` [S1, unofficial].

    *   **XY Plane Rasterization:**
        *   Parallel loop over `x` in `[0, _polyhedron_dim.x)` and `y` in `[0, _polyhedron_dim.y)` [S1, unofficial].
        *   Calculate 2D position: `p = glm::vec2(x, y) * _tree_scale` [S1, unofficial].
        *   Find the relevant face buffer in `_xy_tree` [S1, unofficial].
        *   Get intersections `inters` by calling `get_intersections_safe<xyz>(glm::vec2(x, y), _polyhedron._vertices, buffer)` [S1, unofficial].
        *   Store results in `r.xy[x][y]` [S1, unofficial].

4.  **Intersection Calculation (Safe Mode)**
    The `get_intersections_safe` function performs robust ray-triangle intersection checks, including edge collision detection [S1, unofficial]. For a given 2D position `p` and a set of candidate faces:
    *   Construct the 3D ray origin `pos_3d` and direction `dir` based on the swizzle mode (e.g., for `yzx`, `dir = (0,1,0)` and `pos_3d = (p2.x, ray_start, p2.y)`) [S1, unofficial].
    *   For each face, perform ray-triangle intersection:
        ```python
        bary_pos = glm::intersectRayTriangle(pos_3d, dir, v1, v2, v3)
        if bary_pos is valid and distance > 0:
            inters_dist.insert(glm::round(distance + ray_start))
        ```
    *   Return the set of rounded integer intersection distances [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Boundary Faces:** If a face lies exactly on the end of the array, a voxel cannot be drawn without increasing the dimension. The algorithm adds `1.0f` to the calculated dimension to accommodate this [S1, unofficial].
*   **Edge Collisions:** The safe mode includes additional checks for edge collisions to ensure accuracy, which makes it slower than the fast mode but more robust [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method is parallelized and described as one of the fastest CPU rasterizers [S1, unofficial]. However, the safe mode is slower than the fast mode due to edge collision checks and quadtree overhead [S1, unofficial].
*   **Memory Complexity:** The primary advantage is memory efficiency. It avoids allocating a huge 3D buffer array by saving plane-wise intersections in 2D structures [S1, unofficial]. This allows it to handle huge models or high target resolutions where other voxelizers fail due to memory issues [S1, unofficial].
*   **Measured Performance:** On an Intel i7 4770k CPU with a Stanford Bunny mesh (112k faces):
    *   64³: 118 ms
    *   128³: 298 ms
    *   256³: 1089 ms
    *   512³: 4483 ms
    *   1024³: 17933 ms [S1, unofficial]
*   **Strengths:** Memory efficiency, robustness (edge checks), ability to handle high resolutions.
*   **Limitations:** Slower than the `--fast` mode due to safety checks and quadtree traversal overhead [S1, unofficial].

## Background
This method is part of the `3DStuff/voxelizer` project, an open-source voxelization tool [S1, unofficial]. It was developed as a solution to memory limitations in existing CPU-based voxelizers, providing a trade-off between performance and memory usage for large-scale 3D models [S1, unofficial].

## Sources
[S1] 3DStuff/voxelizer, https://github.com/3DStuff/voxelizer, fetched 2026-10-27 (unofficial)
