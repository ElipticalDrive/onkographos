# VoxelGrid Mesh Generation

Also called: getAllCellMeshWithCond, getAllCellMeshWithIds

## Overview
VoxelGrid Mesh Generation constructs a combined triangle mesh from a sparse set of voxel linear IDs or a predicate-defined subset within a `VoxelGrid` [S2, unofficial][S3, unofficial]. It transforms a base cell mesh into world space for each selected voxel and adjusts triangle indices to maintain connectivity across the flattened output array [S2, unofficial][S3, unofficial]. Unlike boundary-detection methods, it does not share vertices between voxels and generates a full independent geometry block for every selected voxel [S1, unofficial].

## How it works
**Inputs:**
1.  **VoxelGrid**: A data structure containing `std::vector<CellType> data` storing voxels linearly, with access via linear index `operator[](size_t)` or 3D coordinates `operator()(int x, int y, int z)` calling `getCellId(v)` [S3, unofficial].
2.  **Selection Criteria**: Either a `std::vector<std::size_t> cell_ids` containing linear voxel indices, or a predicate function `FuncType` (taking a `CellType` and returning a boolean) used via `detect_non_empty_cells(func)` to iterate all cells and check the predicate [S3, unofficial].
3.  **Base Mesh**: A reference mesh geometry. If a boolean flag `filled` is true, the faceted (solid faces) mesh (`baseCellMeshFilled`) is used; otherwise, the wireframe mesh (`baseCellMesh`) is used [S3, unofficial].
4.  **Cell Size**: The physical size of a single voxel in world units [S2, unofficial].

**Outputs:**
*   A mesh object containing two flat arrays:
    *   `vs`: A vertex array of size $N_v^{\text{total}} = N_{\text{nonZero}} \times N_v$, where $N_{\text{nonZero}}$ is the number of selected voxels and $N_v$ is the vertex count of the base mesh [S2, unofficial][S3, unofficial].
    *   `ts`: A triangle array of size $N_t^{\text{total}} = N_{\text{nonZero}} \times N_t$, where $N_t$ is the triangle count of the base mesh [S2, unofficial][S3, unofficial].

**Data Structures:**
*   **Index Offset Vector**: A 3-component unsigned integer vector `offsetT` defined as `Vector3u(numV, numV, numV)` [S2, unofficial]. This vector is applied uniformly to all three vertex indices of each triangle to shift them to the correct vertex block for the current voxel instance [S2, unofficial].

**Steps:**
1.  **Identify Voxels**: Determine the set of voxel indices to process. If using a predicate, iterate all cells and collect those returning true. If using IDs, use the provided list [S3, unofficial]. Let $N_{\text{nonZero}}$ be the count of selected voxels.
2.  **Select Base Geometry**: Choose `baseCellMeshFilled` if `filled` is true, else `baseCellMesh` [S3, unofficial]. Let $N_v$ be the number of vertices and $N_t$ be the number of triangles in this base mesh.
3.  **Allocate Output Arrays**: Initialize `vs` with size $N_{\text{nonZero}} \times N_v$ and `ts` with size $N_{\text{nonZero}} \times N_t$ [S2, unofficial][S3, unofficial].
4.  **Iterate Selected Voxels**: For each selected voxel index $i$ from $0$ to $N_{\text{nonZero}} - 1$:
    a.  Retrieve the voxel's integer 3D coordinates `cell` $(x, y, z)$ [S3, unofficial].
    b.  Compute the world-space vertex offset: $\text{offsetV} = \text{cell} \times \text{cellSize}$ [S2, unofficial][S3, unofficial].
    c.  **Transform Vertices**: For each vertex index $v$ in the base mesh ($0 \le v < N_v$):
        $$v_{\text{out}}[i \cdot N_v + v] = v_{\text{base}}[v] + \text{offsetV}$$
        where $v_{\text{base}}[v]$ is the position of the $v$-th vertex in the base mesh [S2, unofficial].
    d.  **Transform Triangles**: For each triangle index $t$ in the base mesh ($0 \le t < N_t$):
        $$t_{\text{out}}[i \cdot N_t + t] = t_{\text{base}}[t] + i \cdot N_v$$
        where $t_{\text{base}}[t]$ is the tuple of three vertex indices for the $t$-th triangle in the base mesh, and the index offset is computed as $i \times N_v$ [S2, unofficial][S3, unofficial].
5.  **Assign and Return**: Assign the filled `vs` and `ts` arrays to the output mesh object and return it [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_grid_mesh(voxel_grid, cell_ids_or_predicate, filled, cell_size):
    """
    Generates a combined triangle mesh from a list of voxel linear IDs or predicate.
    
    Args:
        voxel_grid: Object with data vector and get_coords method.
        cell_ids_or_predicate: Either a list of linear indices or a callable predicate.
        filled: Boolean. True for faceted mesh, False for wireframe.
        cell_size: Float or tuple. Size of one voxel in world units.
        
    Returns:
        Mesh object with vertices (vs) and triangles (ts).
    """
    # Step 1: Identify voxels
    if callable(cell_ids_or_predicate):
        selected_indices = [idx for idx in range(voxel_grid.total_cells()) 
                            if cell_ids_or_predicate(voxel_grid[idx])]
    else:
        selected_indices = cell_ids_or_predicate
    
    num_non_zero = len(selected_indices)
    if num_non_zero == 0:
        return Mesh(vertices=[], triangles=[])

    # Step 2: Select base geometry
    if filled:
        base_mesh = get_base_cell_mesh_filled()
    else:
        base_mesh = get_base_cell_mesh()
        
    num_v = len(base_mesh.vertices())
    num_t = len(base_mesh.triangles())
    
    # Step 3: Allocate output arrays
    vs = [None] * (num_non_zero * num_v)
    ts = [None] * (num_non_zero * num_t)
    
    # Step 4: Iterate and transform
    for i, linear_id in enumerate(selected_indices):
        cell_coords = voxel_grid.get_coords(linear_id)
        
        # Compute world-space vertex offset
        if isinstance(cell_size, (int, float)):
            offset_v = (cell_coords[0] * cell_size, 
                        cell_coords[1] * cell_size, 
                        cell_coords[2] * cell_size)
        else:
            offset_v = (cell_coords[0] * cell_size[0], 
                        cell_coords[1] * cell_size[1], 
                        cell_coords[2] * cell_size[2])
        
        # Transform vertices
        base_vertices = base_mesh.vertices()
        for v in range(num_v):
            base_v = base_vertices[v]
            vs[i * num_v + v] = (base_v[0] + offset_v[0], 
                                 base_v[1] + offset_v[1], 
                                 base_v[2] + offset_v[2])
            
        # Transform triangles
        base_triangles = base_mesh.triangles()
        index_offset = i * num_v
        for t in range(num_t):
            base_t = base_triangles[t]
            ts[i * num_t + t] = (base_t[0] + index_offset, 
                                 base_t[1] + index_offset, 
                                 base_t[2] + index_offset)
    
    # Step 5: Assign and return
    return Mesh(vertices=vs, triangles=ts)

class BaseMesh:
    def __init__(self, vertices, triangles):
        self._vertices = vertices
        self._triangles = triangles
    def vertices(self): return self._vertices
    def triangles(self): return self._triangles

class Mesh:
    def __init__(self, vertices, triangles):
        self.vertices = vertices
        self.triangles = triangles

def get_base_cell_mesh_filled():
    # Returns a faceted (solid faces) base mesh
    return BaseMesh(vertices=[(0,0,0), (1,0,0), (1,1,0), (0,1,0),
                              (0,0,1), (1,0,1), (1,1,1), (0,1,1)],
                    triangles=[(0,1,2), (0,2,3), (4,5,6), (4,6,7),
                               (0,1,5), (0,5,4), (2,3,7), (2,7,6),
                               (0,3,7), (0,7,4), (1,2,6), (1,6,5)])

def get_base_cell_mesh():
    # Returns a wireframe base mesh
    return BaseMesh(vertices=[(0,0,0), (1,0,0), (1,1,0), (0,1,0),
                              (0,0,1), (1,0,1), (1,1,1), (0,1,1)],
                    triangles=[])
```

**Parameters:**
*   `filled`: Boolean flag. If true, generates a faceted mesh (solid faces); if false, generates a wireframe mesh [S3, unofficial].
*   `cell_ids`: A vector of linear voxel indices specifying which voxels to include in the output mesh [S3, unofficial].
*   `func`: A template predicate function `FuncType` that takes a `CellType` voxel and returns a boolean. Used to filter which voxels are included in the mesh [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Index Collision**: Triangle vertex indices must be incremented by $i \times N_v$ to prevent triangles from referencing vertices belonging to previously processed cells [S2, unofficial][S3, unofficial]. Failure to apply the offset results in a corrupted mesh where all voxels share the same vertex positions [S2, unofficial].
*   **Overlapping Geometry**: Since this method generates a mesh per voxel without boundary culling, adjacent voxels will produce overlapping triangles [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N_{\text{nonZero}} \times (N_v + N_t))$, where $N_{\text{nonZero}}$ is the number of selected voxels, and $N_v, N_t$ are the vertex and triangle counts of the base mesh [S2, unofficial][S3, unofficial].
*   **Memory Complexity**: $O(N_{\text{nonZero}} \times (N_v + N_t))$ for storing the output vertex and triangle arrays [S2, unofficial][S3, unofficial].
*   **Measured Performance**: Not given in the sources.
*   **Scaling**: Linear scaling with the number of selected voxels and base mesh complexity [S2, unofficial].
*   **Strengths**: Not given in the sources.
*   **Limitations**: Not given in the sources.
*   **When to use**: Not given in the sources.
*   **When not to use**: Not given in the sources.

## Background
The method originates from implementations within the SIBR framework, specifically utilizing `sibr::Vector3u` for index offset calculations [S2, unofficial][S3, unofficial]. It is referenced in the context of voxel-based spatial indexing and level-of-detail management in 3D Gaussian Splatting extensions [S2, unofficial][S3, unofficial]. The implementation details are derived directly from the source code of these repositories [S2, unofficial][S3, unofficial].

## Sources
[S1] bigmat18/cuda-mesh-voxelization, https://github.com/bigmat18/cuda-mesh-voxelization, fetched 2026-10-26 (unofficial)
[S2] 3DGS-FLoD/flod, https://github.com/3DGS-FLoD/flod, fetched 2026-10-27 (unofficial)
[S3] whu-lyh/flod, https://github.com/whu-lyh/flod, fetched 2026-10-27 (unofficial)
