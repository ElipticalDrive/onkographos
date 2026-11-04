# Sampled DMC

Also called: Linear Hashed Marching Cubes

## Overview
Sampled DMC is a mesh generation technique that adapts the Dual Marching Cubes (DMC) algorithm to hashed octrees using Morton keys [S1, unofficial]. It constructs a dual grid where vertices are located at the center of each octree leaf node, allowing for adaptive resolution isosurfacing without requiring a uniform grid [S1, unofficial]. Unlike standard Marching Cubes implementations that expect Signed Distance Functions (SDFs) as direct input, Sampled DMC takes the data octree itself, making it suitable for storing and extracting meshes from sampled points, complex SDFs, or destructible environments [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A `HashedOctree<T>` where `T` implements the `McNode` trait, providing `density()` (distance from node center to contour) and `normal()` (normalized gradient of the SDF) for each node. A `scale` parameter (`f32`) applied to vertex positions.
*   **Output:** A `Mesh` struct containing a `Vec<Vertex>` and `Vec<u32>` indices. Vertices are in counter-clockwise order.

**Data Structures:**
*   `McNode`: Trait requiring `density(&self) -> f32` and `normal(&self) -> Vector3<f32>`.
*   `NodeData`: Struct with `density: f32` and `normal: Vector3<f32>`.
*   `Vertex`: Struct with `position: Point3<f32>` and `normal: Vector3<f32>`. Normals are normalized.
*   `DualGrid`: Constructed from the octree. Contains `volumes`, where each volume is an array of 8 `MortonKey`s representing the corners of a dual cell.
*   `CellVertexData`: Struct with `positions: [Point3<f32>; 12]` and `normals: [Vector3<f32>; 12]`.

**Numbered Steps:**

1.  **Construct Dual Grid:** Call `DualGrid::from_octree(octree)`.
    *   Get all leaf nodes of the octree.
    *   For each leaf, determine its vertex keys.
    *   For each vertex key, find neighboring leaves at the same level.
    *   Filter neighbors to ensure they match the current leaf's level and are not the source vertex itself.
2.  **Process Volumes in Parallel:** Iterate over `dual.volumes` using a parallel iterator (e.g., `rayon`).
3.  **Fetch Volume Data:** For each volume, call `fetch_volume_data(volume, octree)`.
    *   Initialize `cube_index = 0` and an array of 8 `NodeData`.
    *   Map volume vertex indices to Marching Cubes vertex indices using `VOLUME_TO_MC_VERTEX: [3, 2, 7, 6, 0, 1, 4, 5]`.
    *   If any vertex key is `MortonKey::none()`, return immediately (boundary handling).
    *   Retrieve `density` and `normal` from the octree for each vertex.
    *   If `density > 0.0`, set the corresponding bit in `cube_index`.
    *   Store density and normal in the `nodes` array.
4.  **Filter Visibility:** Call `is_cell_visible(cube_index)`. If `cube_index` is 0 (all negative) or 0xFF (all positive), skip the volume.
5.  **Calculate Interpolated Data:** For visible volumes, call `calculate_mc_vertex_data(volume, nodes, cube_index, scale)`.
    *   Get the edge mask from `tables::EDGE_TABLE[cube_index]`.
    *   For each of the 12 edges, if the edge is active (bit set in mask):
        *   Get the two volume vertex indices from `MC_CUBE_EDGES`.
        *   Calculate the interpolation factor `R` using the densities of the two vertices.
        *   Interpolate the position and normal using `R`.
        *   Scale the position by `vertex_position_scale`.
6.  **Obtain Vertices:** Call `obtain_mc_vertices(cell_data, cube_index)`.
    *   Get triangle indices from `tables::TRI_TABLE[cube_index]`.
    *   Chunk the table into groups of 3.
    *   For each valid chunk, create `Vertex` structs using the interpolated positions and normals.
7.  **Collect Results:** Collect all vertices into a `Vec`. Generate indices as `(0..vertices.len() as u32).collect()`.

**Formulas:**
*   **Interpolation Factor:**
    $$ R = \frac{0 - p_1}{p_2 - p_1} $$
    Where $p_1$ and $p_2$ are the densities at the two endpoints of the edge. This estimates the relative value $R$ such that $p_1 + (p_2 - p_1) \cdot R == 0$.
    *   Special cases:
        *   If $|p_1| < 0.00001$, return $p_1$.
        *   If $|p_2| < 0.00001$, return $p_2$.
        *   If $|p_1 - p_2| < 0.00001$, return $p_1$.
*   **Position Interpolation:**
    $$ \text{pos} = (\text{pos}_1 \oplus \text{pos}_2, R) \cdot \text{scale} $$
    Where $\oplus$ denotes linear interpolation.
*   **Normal Interpolation:**
    $$ \text{norm} = \text{norm}_1 \oplus \text{norm}_2, R $$
    (Note: The source code does not explicitly normalize the interpolated normal, but the `McNode` trait requires normalized normals, and the `Vertex` struct stores normalized normals. The implementation relies on the input normals being normalized.)

**Parameters:**
*   `scale`: `f32`. A scaling factor applied to vertex positions.

**Pitfalls:**
*   **Boundary Handling:** If any vertex in a volume is `MortonKey::none()`, the volume is on the edge of the octree. The function returns immediately, skipping that volume. This may result in missing mesh parts at the boundary.
*   **Cell Visibility:** Cells with `cube_index == 0` or `cube_index == 0xFF` are filtered out. This is correct for isosurfacing but must be understood to avoid expecting triangles in uniform regions.
*   **Vertex Merging:** The algorithm does not merge vertices. Indices are simply incremental integers. This can lead to higher vertex counts and potential visual artifacts at shared edges if normals are not consistent.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mesh_from_octree(octree: HashedOctree, scale: f32) -> Mesh:
    dual = DualGrid.from_octree(octree)
    vertices = []
    
    for volume in dual.volumes.par_iter():
        nodes, cube_index = fetch_volume_data(volume, octree)
        
        if not is_cell_visible(cube_index):
            continue
            
        cell_data = calculate_mc_vertex_data(volume, nodes, cube_index, scale)
        vertices.extend(obtain_mc_vertices(cell_data, cube_index))
        
    indices = list(range(len(vertices)))
    return Mesh(vertices, indices)

def fetch_volume_data(volume: [MortonKey; 8], octree: HashedOctree) -> (VolumeData):
    VOLUME_TO_MC_VERTEX = [3, 2, 7, 6, 0, 1, 4, 5]
    nodes = [None] * 8
    cube_index = 0
    
    for volume_i, key in enumerate(volume):
        mc_i = VOLUME_TO_MC_VERTEX[volume_i]
        
        if key == MortonKey.none():
            return VolumeData(nodes, 0) # Skip boundary
        
        node_data = octree.value(key)
        density = node_data.density()
        normal = node_data.normal()
        
        if density > 0.0:
            cube_index |= (1 << mc_i)
            
        nodes[volume_i] = NodeData(density, normal)
        
    return VolumeData(nodes, cube_index)

def calculate_mc_vertex_data(volume: [MortonKey; 8], nodes: [NodeData; 8], cube_index: u8, scale: f32) -> CellVertexData:
    MC_CUBE_EDGES = [
        (4, 5), (1, 5), (0, 1), (0, 4),
        (6, 7), (3, 7), (2, 3), (2, 6),
        (4, 6), (5, 7), (1, 3), (0, 2)
    ]
    
    positions = [None] * 12
    normals = [None] * 12
    edges = tables.EDGE_TABLE[cube_index]
    
    for i in range(12):
        if (edges & (1 << i)) > 0:
            v1_i, v2_i = MC_CUBE_EDGES[i]
            factor = interpolation_factor(nodes[v1_i].density, nodes[v2_i].density)
            
            pos1 = volume[v1_i].position()
            pos2 = volume[v2_i].position()
            positions[i] = (pos1.interpolate(pos2, factor) * scale)
            
            norm1 = nodes[v1_i].normal
            norm2 = nodes[v2_i].normal
            normals[i] = norm1.interpolate(norm2, factor)
            
    return CellVertexData(positions, normals)

def obtain_mc_vertices(cell_data: CellVertexData, cube_index: u8) -> [Vertex]:
    tri_table = tables.TRI_TABLE[cube_index]
    vertices = []
    
    for chunk in tri_table.chunks(3):
        if chunk[0] == -1:
            break
            
        v1 = cell_data.positions[chunk[0]]
        v2 = cell_data.positions[chunk[1]]
        v3 = cell_data.positions[chunk[2]]
        
        n1 = cell_data.normals[chunk[0]]
        n2 = cell_data.normals[chunk[1]]
        n3 = cell_data.normals[chunk[2]]
        
        vertices.append(Vertex(v1, n1))
        vertices.append(Vertex(v2, n2))
        vertices.append(Vertex(v3, n3))
        
    return vertices

def interpolation_factor(p1: f32, p2: f32) -> f32:
    if abs(p1) < 0.00001:
        return p1
    if abs(p2) < 0.00001:
        return p2
    if abs(p1 - p2) < 0.00001:
        return p1
    return (0.0 - p1) / (p2 - p1)
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of dual volumes (leaf nodes in the octree). Parallel processing is used via `rayon`.
*   **Memory Complexity:** Linear with respect to the number of dual volumes. Each volume is processed independently, and vertices are collected into a single vector.
*   **Performance:** Fast due to parallel processing and efficient data structures. The use of Morton keys allows for efficient spatial locality.
*   **Strengths:** Adaptive resolution via octree; no need for uniform grid; suitable for dynamic/destructible environments; parallelizable.
*   **Limitations:** No vertex merging (higher vertex count); boundary handling skips cells; requires `McNode` implementation; not suitable for non-octree data.
*   **When to use:** When working with octree-based data structures, adaptive resolution is needed, or the environment is dynamic.
*   **When not to use:** When vertex merging is critical for performance or visual quality, or when working with uniform grid data.

## Background
Sampled DMC is an implementation of the Dual Marching Cubes technique, also known as Linear Hashed Marching Cubes [S1, unofficial]. It was developed as part of the `dmc` crate, available on crates.io [S1, unofficial]. The method adapts the original Marching Cubes algorithm, which uses a uniform grid, to hashed octrees using Morton keys [S1, unofficial]. This allows for adaptive resolution isosurfacing, which is particularly useful for complex SDFs, destructible environments, or objects with different levels of detail (LODs) [S1, unofficial]. The crate uses `cgmath` for vector math and `rayon` for parallel processing [S1, unofficial].

## Sources
[S1] aleokdev/dmc, https://github.com/aleokdev/dmc, fetched 2026-10-27 (unofficial)
