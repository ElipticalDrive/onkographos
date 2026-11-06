# VoxelGrid Mesh Generation

Also called: getAllCellMeshWithCond, getAllCellMeshWithIds

## Overview
VoxelGrid Mesh Generation constructs a combined triangle mesh from a sparse set of voxel linear IDs or a predicate-defined subset within a `VoxelGrid`. It transforms a base voxel mesh (either wireframe or faceted) into world space for each selected voxel and adjusts triangle indices to maintain connectivity across the flattened output array [S1, S2, unofficial]. Unlike [Boolean Voxel Meshing](971-boolean-voxel-meshing.md), which detects faces at voxel boundaries to produce a watertight or manifold surface, this method generates a non-manifold, blocky mesh where each voxel contributes its own independent geometry, potentially resulting in overlapping triangles if voxels are adjacent [S1, S2, unofficial]. It is distinct from [Blocky Surface Extraction](6418-blocky-surface-extraction.md) in that it does not necessarily filter for boundary voxels but can include any voxels satisfying a condition or ID list [S2, unofficial].

## How it works
**Inputs:**
1.  **VoxelGrid**: A data structure containing `std::vector<CellType> data` storing voxels linearly. Access is via linear index `operator[](size_t)` or 3D coordinates `operator()(int x, int y, int z)` which calls `getCellId(v)` [S2, unofficial].
2.  **Selection Criteria**: Either a `std::vector<std::size_t> cell_ids` containing linear voxel indices, or a predicate function `FuncType` (taking a `CellType` and returning a boolean) used via `detect_non_empty_cells(func)` to iterate all cells and check the predicate [S2, unofficial].
3.  **Base Mesh**: A reference mesh geometry. If a boolean flag `filled` is true, the faceted (solid faces) mesh (`baseCellMeshFilled`) is used; otherwise, the wireframe mesh (`baseCellMesh`) is used [S1, S2, unofficial].
4.  **Cell Size**: The physical size of a single voxel in world units [S2, unofficial].

**Outputs:**
*   A mesh object containing two arrays:
    *   `vs`: A flat array of vertices of size $N_v^{total} = N_{nonZero} \times N_v$, where $N_{nonZero}$ is the number of selected voxels and $N_v$ is the vertex count of the base mesh [S1, S2, unofficial].
    *   `ts`: A flat array of triangles of size $N_t^{total} = N_{nonZero} \times N_t$, where $N_t$ is the triangle count of the base mesh [S1, S2, unofficial].

**Data Structures:**
*   **Index Offset Vector**: A 3-component unsigned integer vector `offsetT` defined as `Vector3u(numV, numV, numV)`. This vector is added uniformly to all three vertex indices of each triangle to shift them to the correct vertex block for the current voxel instance [S1, S2, unofficial].

**Steps:**
1.  **Identify Voxels**: Determine the set of voxel indices to process. If using a predicate, iterate all cells and collect those returning true. If using IDs, use the provided list [S2, unofficial]. Let $N_{nonZero}$ be the count of selected voxels.
2.  **Select Base Geometry**: Choose `baseCellMeshFilled` if `filled` is true, else `baseCellMesh` [S2, unofficial]. Let $N_v$ be the number of vertices and $N_t$ be the number of triangles in this base mesh.
3.  **Allocate Output Arrays**: Initialize `vs` with size $N_{nonZero} \times N_v$ and `ts` with size $N_{nonZero} \times N_t$ [S2, unofficial].
4.  **Iterate Selected Voxels**: For each selected voxel index $i$ from $0$ to $N_{nonZero} - 1$:
    a.  Retrieve the voxel's integer 3D coordinates `cell` $(x, y, z)$ [S2, unofficial].
    b.  Compute the world-space vertex offset: $\text{offsetV} = \text{cell} \times \text{cellSize}$ [S1, S2, unofficial].
    c.  **Transform Vertices**: For each vertex index $v$ in the base mesh ($0 \le v < N_v$):
        $$v_{out}[i \cdot N_v + v] = v_{base}[v] + \text{offsetV}$$
        where $v_{base}[v]$ is the position of the $v$-th vertex in the base mesh [S1, S2, unofficial].
    d.  **Transform Triangles**: For each triangle index $t$ in the base mesh ($0 \le t < N_t$):
        $$t_{out}[i \cdot N_t + t] = t_{base}[t] + \text{offsetT}$$
        where $t_{base}[t]$ is the tuple of three vertex indices for the $t$-th triangle in the base mesh, and $\text{offsetT} = (N_v, N_v, N_v)$ [S1, S2, unofficial].
5.  **Assign and Return**: Assign the filled `vs` and `ts` arrays to the output mesh object and return it [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_grid_mesh(voxel_grid, cell_ids_or_predicate, filled, cell_size):
    """
    Generates a combined triangle mesh from a list of voxel linear IDs or predicate.
    
    Args:
        voxel_grid: Object with data vector and getCellId/getCoords methods.
        cell_ids_or_predicate: Either a list of linear indices or a callable predicate.
        filled: Boolean. True for faceted mesh, False for wireframe.
        cell_size: Float or Vector3. Size of one voxel in world units.
        
    Returns:
        Mesh object with vertices (vs) and triangles (ts).
    """
    # Step 1: Identify voxels
    if callable(cell_ids_or_predicate):
        # Predicate-based selection
        selected_indices = []
        # Assuming voxel_grid has a way to iterate all cells or get total count
        # This part depends on specific VoxelGrid API, generic iteration assumed
        for idx in range(voxel_grid.total_cells()):
            cell = voxel_grid[idx]
            if cell_ids_or_predicate(cell):
                selected_indices.append(idx)
    else:
        # ID-based selection
        selected_indices = cell_ids_or_predicate
    
    num_non_zero = len(selected_indices)
    if num_non_zero == 0:
        return Mesh(vertices=[], triangles=[])

    # Step 2: Select base geometry
    if filled:
        base_mesh = get_base_cell_mesh_filled()
    else:
        base_mesh = get_base_cell_mesh()
        
    num_v = base_mesh.num_vertices()
    num_t = base_mesh.num_triangles()
    
    # Step 3: Allocate output arrays
    # Using flat lists for vertices and triangles
    vs = [None] * (num_non_zero * num_v)
    ts = [None] * (num_non_zero * num_t)
    
    # Precompute offsetT as a vector of num_v
    offset_t = (num_v, num_v, num_v)
    
    # Step 4: Iterate and transform
    for i, linear_id in enumerate(selected_indices):
        # Get 3D coordinates from linear ID
        # Implementation detail: voxel_grid.get_coords(linear_id) or similar
        cell_coords = voxel_grid.get_coords(linear_id)
        
        # Compute world-space vertex offset
        # cell_size can be scalar or vector; assume vector multiplication or scalar broadcast
        if isinstance(cell_size, (int, float)):
            offset_v = (cell_coords.x * cell_size, 
                        cell_coords.y * cell_size, 
                        cell_coords.z * cell_size)
        else:
            offset_v = (cell_coords.x * cell_size.x, 
                        cell_coords.y * cell_size.y, 
                        cell_coords.z * cell_size.z)
        
        # Transform vertices
        base_vertices = base_mesh.vertices()
        for v in range(num_v):
            base_v = base_vertices[v]
            vs[i * num_v + v] = (base_v.x + offset_v[0], 
                                 base_v.y + offset_v[1], 
                                 base_v.z + offset_v[2])
            
        # Transform triangles
        base_triangles = base_mesh.triangles()
        for t in range(num_t):
            base_t = base_triangles[t]
            # base_t is a tuple of 3 vertex indices
            ts[i * num_t + t] = (base_t[0] + offset_t[0], 
                                 base_t[1] + offset_t[1], 
                                 base_t[2] + offset_t[2])
    
    # Step 5: Assign and return
    return Mesh(vertices=vs, triangles=ts)
```

**Parameters:**
*   `filled`: Boolean flag. If true, generates a faceted mesh (solid faces); if false, generates a wireframe mesh [S1, S2, unofficial].
*   `cell_ids`: A vector of linear voxel indices specifying which voxels to include in the output mesh [S1, unofficial].
*   `func`: A template predicate function `FuncType` that takes a `CellType` voxel and returns a boolean. Used to filter which voxels are included in the mesh [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Index Collision**: Triangle vertex indices must be incremented by $i \times N_v$ (stored in `offsetT` as `Vector3u(numV, numV, numV)`) to prevent triangles from referencing vertices belonging to previously processed cells [S1, S2, unofficial]. Failure to do so results in a corrupted mesh where all voxels share the same vertex positions [S1, S2, unofficial].
*   **Overlapping Geometry**: Since this method generates a mesh per voxel without boundary culling, adjacent voxels will produce overlapping triangles. This is intentional for blocky rendering but incorrect for watertight surface extraction [S1, S2, unofficial].
*   **Precision**: World-space offsets are computed using floating-point arithmetic. For very large voxel grids, precision loss in vertex positions may occur if `cell_size` or coordinates are large [G].

## Operating characteristics
*   **Time Complexity**: $O(N_{nonZero} \times (N_v + N_t))$, where $N_{nonZero}$ is the number of selected voxels, and $N_v, N_t$ are the vertex and triangle counts of the base mesh. The complexity is linear with respect to the number of output primitives [S1, S2, unofficial].
*   **Memory Complexity**: $O(N_{nonZero} \times (N_v + N_t))$ for storing the output vertex and triangle arrays [S1, S2, unofficial].
*   **Strengths**: Simple implementation; efficient for rendering blocky structures where per-voxel independence is desired; supports both wireframe and solid representations via a single base mesh [S1, S2, unofficial].
*   **Limitations**: Produces non-manifold geometry; inefficient for surface extraction due to redundant internal faces; not suitable for physics or collision detection requiring watertight meshes [S1, S2, unofficial].
*   **When to use**: When a blocky, non-manifold visualization of a voxel set is required, or when the voxel grid is sparse and the overhead of boundary detection is unjustified [S2, unofficial].
*   **When not to use**: When a watertight, manifold surface is needed; use [Boolean Voxel Meshing](971-boolean-voxel-meshing.md) or [Blocky Surface Extraction](6418-blocky-surface-extraction.md) instead [S1, S2, unofficial].

## Background
The method is implemented in the SIBR framework, specifically within the `VoxelGrid` mesh generation utilities. It is referenced in the context of 3D Gaussian Splatting extensions, particularly in the `3DGS-FLoD` and `whu-lyh/flod` repositories, which utilize voxel-based representations for level-of-detail management or spatial indexing [S1, S2, unofficial]. The implementation details are derived from the source code of these repositories [S1, S2, unofficial].

## Sources
[S1] 3DGS-FLoD/flod, https://github.com/3DGS-FLoD/flod, fetched 2026-10-27 (unofficial)
[S2] whu-lyh/flod, https://github.com/whu-lyh/flod, fetched 2026-10-27 (unofficial)
