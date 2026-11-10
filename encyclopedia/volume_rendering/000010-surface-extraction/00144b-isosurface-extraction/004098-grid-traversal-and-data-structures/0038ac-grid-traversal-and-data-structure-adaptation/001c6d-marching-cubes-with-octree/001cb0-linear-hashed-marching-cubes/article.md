# Linear Hashed Marching Cubes


## Overview
Linear Hashed Marching Cubes is a surface extraction method that converts volumetric isosurfaces into triangle meshes by applying the Marching Cubes algorithm over a linear hashed octree data structure [S1, unofficial]. It serves as a loose implementation of the "Fast Generation of Pointerless Octree Duals" technique, designed to accelerate mesh generation compared to standard grid-based marching cubes [S1, unofficial]. By replacing fixed-grid storage with a linear hashed octree, the method efficiently handles sparse or unbounded datasets without the iteration overhead of dense arrays or the query latency of traditional pointer-based octrees [S1, unofficial].

## How it works
**Inputs:**
- A volumetric scalar field (implicit function).
- An isovalue threshold for surface extraction.
- `max_depth`: The maximum depth of the internal octree, defining the resolution limit. The tree spans a cubic grid of size $2^{\text{max\_depth}}$ in each direction [S1, unofficial].

**Outputs:**
- A triangle mesh containing vertices and indices representing the isosurface.

**Data Structures:**
1.  **ImplicitOctree**: A linear hashed octree traversal structure. Unlike `PrimalGrid`, it uses sparse storage. Nodes are indexed using `MortonKey` (Z-order curve keys) rather than `GridKey` (3D integer coordinates) [S1, unofficial].
2.  **MeshTopologyBuilder**: Accumulates vertex positions and face indices. It stores vertices with associated `MortonKey` identifiers to facilitate potential adjacency or deduplication logic, though the core extraction focuses on topology generation [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization**:
    - Create an `ImplicitOctree` instance with the specified `max_depth`.
    - Create a `MeshTopologyBuilder` to store the resulting mesh geometry.

2.  **Tree Traversal**:
    - Traverse the `ImplicitOctree`. The traversal visits only occupied or relevant nodes.
    - For each visited cube (leaf node or active cell), retrieve:
        - `keys`: The Morton keys of the cube's corners.
        - `corners`: The 3D spatial coordinates of the 8 corners.
        - `values`: The scalar field values at the 8 corners.

3.  **Cube Classification**:
    - For the current cube, determine the **cube index** (0–255) by comparing each corner's signed distance value against the isovalue.
    - Let $v_i$ be the value at corner $i$ and $I$ be the isovalue. If $v_i < I$, the corner is inside; otherwise, it is outside. The cube index is the bitmask formed by these 8 comparisons.

4.  **Edge Intersection Calculation**:
    - Using the cube index and the corner values, identify the 12 edges of the cube that are crossed by the isosurface.
    - For each crossed edge connecting corners $A$ and $B$ with values $v_A$ and $v_B$, calculate the intersection point $P$ using linear interpolation:
      $$ P = A + \frac{I - v_A}{v_B - v_A} (B - A) $$
    - Store the 12 resulting vertex positions.

5.  **Mesh Generation (Marching)**:
    - Iterate through the triangles defined by the cube index lookup table (standard Marching Cubes topology).
    - For each triangle defined by vertices $(a, b, c)$:
        - Add vertex $a$ to the `MeshTopologyBuilder` with its position and associated `MortonKey`.
        - Add vertex $b$ to the `MeshTopologyBuilder`.
        - Add vertex $c$ to the `MeshTopologyBuilder`.
        - Add a face (triangle) referencing the indices of $a, b, c$ in the builder.

6.  **Finalization**:
    - After traversal completes, call `build()` on the `MeshTopologyBuilder`.
    - Extract the final vertex buffer and index buffer from the builder to form the mesh.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_hashed_marching_cubes(field, isovalue, max_depth):
    # Initialize structures
    octree = ImplicitOctree(max_depth=max_depth)
    mesh_builder = MeshTopologyBuilder()

    # Traverse the linear hashed octree
    # The traversal yields visited cubes with their corner data
    for cube in octree.traverse():
        corners = cube.corners      # List of 8 vec3 positions
        values = cube.values        # List of 8 float scalar values
        keys = cube.keys            # List of 8 MortonKey identifiers

        # Classify cube
        cube_index = 0
        for i in range(8):
            if values[i] < isovalue:
                cube_index |= (1 << i)

        # Find edge crossings
        edge_vertices = []
        for edge in MARCHING_CUBES_EDGES[cube_index]:
            i, j = edge
            p1 = corners[i]
            p2 = corners[j]
            v1 = values[i]
            v2 = values[j]
            
            # Linear interpolation
            t = (isovalue - v1) / (v2 - v1)
            intersection = p1 + t * (p2 - p1)
            edge_vertices.append(intersection)

        # March the cube: add triangles
        for triangle in MARCHING_CUBES_TRIANGLES[cube_index]:
            v_idx_a = triangle[0]
            v_idx_b = triangle[1]
            v_idx_c = triangle[2]

            # Add vertices to builder
            # Note: MortonKey is stored for potential deduplication/adjacency
            mesh_builder.add_vertex(edge_vertices[v_idx_a], keys[v_idx_a])
            mesh_builder.add_vertex(edge_vertices[v_idx_b], keys[v_idx_b])
            mesh_builder.add_vertex(edge_vertices[v_idx_c], keys[v_idx_c])
            
            # Add face
            mesh_builder.add_face()

    # Build final mesh
    mesh = mesh_builder.build()
    return mesh
```

**Parameters:**
- `max_depth`: Integer. The maximum depth of the octree. Determines the finest resolution ($2^{\text{max\_depth}}$) [S1, unofficial].
- `isovalue`: Float. The threshold for surface extraction.

**Edge Cases and Pitfalls:**
- **Precision**: Linear interpolation assumes the scalar field is locally linear between grid points. Highly oscillatory fields may produce artifacts.
- **Empty Regions**: The linear hashed octree handles empty regions by not storing nodes, but the traversal must correctly skip these to maintain performance [S1, unofficial].
- **Boundary Conditions**: The tree spans a cubic grid. Input fields must be defined within this bounds, or boundary handling must be implemented in the `ImplicitOctree` construction [S1, unofficial].

## Subtopics

## Operating characteristics
- **Performance**: Faster than standard marching cubes on dense grids due to the sparse nature of the linear hashed octree, which avoids iterating over empty space [S1, unofficial].
- **Memory**: Memory usage is proportional to the number of active cells (near the isosurface) rather than the total volume, making it suitable for sparse data [S1, unofficial].
- **Complexity**: Not given in the sources.
- **Strengths**: Efficient for unbounded or sparse datasets; avoids iteration overhead of dense arrays [S1, unofficial].
- **Limitations**: Hash map overhead may impact performance for very dense, fully occupied volumes compared to raw arrays [S1, unofficial].

## Background
The method is described as a loose implementation of "Fast Generation of Pointerless Octree Duals" [S1, unofficial]. It was presented in a GitHub repository by swiftcoder, focusing on efficient isosurface extraction using linear hashed octrees [S1, unofficial].

## Sources
[S1] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
