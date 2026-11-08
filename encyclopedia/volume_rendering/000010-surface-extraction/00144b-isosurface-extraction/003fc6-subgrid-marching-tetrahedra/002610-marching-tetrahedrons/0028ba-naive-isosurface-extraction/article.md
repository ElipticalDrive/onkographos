# Naive Isosurface Extraction


## Overview
Naive Isosurface Extraction is a surface extraction algorithm that processes each tetrahedron independently to generate a collection of individual, unconnected polygons [S1, unofficial]. It differs from standard Marching Tetrahedra by iterating through all tetrahedra and their edges to find intersections with an isosurface, rather than using the standard 14-case lookup table based on vertex signs [S1, unofficial].

## How it works
**Inputs:**
- A tetrahedral mesh containing $N$ tetrahedra [S1, unofficial].
- A scalar field defined at the vertices of the mesh [S1, unofficial].
- A scalar isovalue $f_{iso}$ defining the level set [S1, unofficial].

**Outputs:**
- A polygonal surface representation consisting of a list of vertices and a list of polygons [S1, unofficial].

**Data Structures:**
- **Input Tetrahedron:** Defined by 4 vertex indices and associated scalar values [S1, unofficial].
- **Output Points:** A dynamic list of 3D coordinates [S1, unofficial].
- **Output Cells:** A dynamic list of polygons, where each polygon is defined by a list of vertex indices into the Output Points list [S1, unofficial].

**Algorithm Steps:**

1.  **Initialize Output:** Create empty lists for output vertices and polygons [S1, unofficial].
2.  **Loop over Tetrahedra:** For each tetrahedron $T_i$ in the input mesh:
    a.  **Check Intersection:** Call `IsCellOnLevelSet(T_i, f_{iso})` [S1, unofficial].
        -   Retrieve scalar values for all vertices of $T_i$ [S1, unofficial].
        -   Determine if the level set intersects $T_i$ [S1, unofficial].
        -   If not intersected, continue to the next tetrahedron [S1, unofficial].
    b.  **Compute Intersection Polygon:** Call `ComputeSimpleIntersection(T_i, f_{iso})` [S1, unofficial].
        -   Initialize an empty list `pointIds` to store vertex indices of the resulting polygon [S1, unofficial].
        -   **Loop over Edges:** Iterate through all 6 edges of the tetrahedron [S1, unofficial].
        -   **Check Edge Intersection:** For each edge, call `IsCellOnLevelSet(edge, f_{iso})` [S1, unofficial].
        -   **Compute Intersection Point:** If the edge is intersected, call `ComputeEdgeIntersection(edge, f_{iso})` [S1, unofficial].
            -   Calculate the intersection point $P$ using linear interpolation [G].
            -   Add $P$ to the Output Points list and record its index [S1, unofficial].
            -   Append the index to `pointIds` [S1, unofficial].
        -   **Order Vertices:** Call `ReOrderTetEdges(pointIds, T_i)` to ensure consistent winding [S1, unofficial].
            -   This function re-orders the list of intersected edges such that the list describes a consistent winding around the tetrahedron [S1, unofficial].
        -   **Insert Polygon:** Call `Output->InsertNextCell(VTK_POLYGON, pointIds)` [S1, unofficial].
3.  **Finalize:** Return the constructed output surface [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def naive_isosurface_extraction(tet_mesh, vertex_coords, scalar_field, isovalue):
    """
    Extracts isosurface from a tetrahedral mesh using the naive approach.
    
    Args:
        tet_mesh: List of tetrahedra, each defined as [v0, v1, v2, v3] (vertex indices).
        vertex_coords: Array of 3D coordinates indexed by vertex index.
        scalar_field: Array of scalar values indexed by vertex index.
        isovalue: The scalar value defining the isosurface.
        
    Returns:
        output_points: List of 3D coordinates.
        output_polys: List of polygons, each defined as a list of vertex indices.
    """
    output_points = []
    output_polys = []
    
    def is_cell_on_level_set(cell_vertices, isovalue):
        scalars = [scalar_field[v] for v in cell_vertices]
        return min(scalars) <= isovalue <= max(scalars)

    def compute_edge_intersection(v0, v1, f0, f1, isovalue):
        if abs(f1 - f0) < 1e-10:
            return ((v0[0] + v1[0]) / 2.0, (v0[1] + v1[1]) / 2.0, (v0[2] + v1[2]) / 2.0)
        t = (isovalue - f0) / (f1 - f0)
        return (v0[0] + t * (v1[0] - v0[0]), v0[1] + t * (v1[1] - v0[1]), v0[2] + t * (v1[2] - v0[2]))

    def reorder_tet_edges(intersected_edges):
        # Re-orders the list of intersected edges to form a consistent winding around the tetrahedron [G]
        if len(intersected_edges) < 2:
            return intersected_edges
        adj = {i: [] for i in range(len(intersected_edges))}
        for i, (e1, e2) in enumerate(intersected_edges):
            for j, (e3, e4) in enumerate(intersected_edges):
                if i != j and (e1 == e3 or e1 == e4 or e2 == e3 or e2 == e4):
                    adj[i].append(j)
        visited = [False] * len(intersected_edges)
        cycle = []
        curr = 0
        while not visited[curr]:
            visited[curr] = True
            cycle.append(intersected_edges[curr])
            nxt = adj[curr][0] if adj[curr] else -1
            if nxt == -1 or visited[nxt]:
                break
            curr = nxt
        return cycle

    for tet in tet_mesh:
        v0, v1, v2, v3 = tet
        if not is_cell_on_level_set(tet, isovalue):
            continue
            
        point_ids = []
        intersected_edges = []
        
        edges = [(v0, v1), (v0, v2), (v0, v3), (v1, v2), (v1, v3), (v2, v3)]
        
        for va, vb in edges:
            fa = scalar_field[va]
            fb = scalar_field[vb]
            if (fa - isovalue) * (fb - isovalue) < 0:
                pa = vertex_coords[va]
                pb = vertex_coords[vb]
                p_intersect = compute_edge_intersection(pa, pb, fa, fb, isovalue)
                idx = len(output_points)
                output_points.append(p_intersect)
                point_ids.append(idx)
                intersected_edges.append((va, vb))
        
        ordered_edges = reorder_tet_edges(intersected_edges)
        edge_to_idx = {e: point_ids[i] for i, e in enumerate(intersected_edges)}
        final_ids = [edge_to_idx[e] for e in ordered_edges]
        output_polys.append(final_ids)

    return output_points, output_polys
```

**Parameters:**
- `isovalue`: The scalar threshold defining the level set [S1, unofficial].
- `tolerance`: Not explicitly defined in the naive source, but precision issues with floating-point comparisons in `is_cell_on_level_set` can cause cracks [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Duplicate Vertices:** The naive approach creates a new vertex for every intersection on every tetrahedron, leading to duplicate vertices at shared edges and non-manifold surfaces with cracks [S1, unofficial].
- **Inconsistent Winding:** If the order of vertices in the generated polygon is inconsistent, visual artifacts like cracks appear [S1, unofficial].
- **Performance:** The approach is slow compared to optimized implementations because it does not reuse vertex data and may perform redundant calculations [S1, unofficial].

## Subtopics

### Adaptive Octree Marching Tetrahedra
Adaptive Octree Marching Tetrahedra is a variant that uses an adaptive octree structure to refine the mesh only where necessary, reducing the number of tetrahedra processed [S1, unofficial]. It differs from Naive Isosurface Extraction by dynamically adjusting resolution based on scalar field gradients, rather than processing a uniform or pre-defined tetrahedral mesh [S1, unofficial]. Choose this when high-resolution details are localized and computational resources are limited [S1, unofficial]. [Link to sibling article](3897-adaptive-octree-marching-tetrahedra.md).

### [Naive Tetrahedral Surfacing](6833-naive-tetrahedral-surfacing.md)
Naive Tetrahedral Surfacing is an isosurface extraction method that processes each tetrahedron independently to generate a collection of individual, unconnected polygons. It differs from standard Marching Tetrahedra by iterating through all tetrahedra and their edges to find intersections with an isosurface, rather than using the standard 14-case lookup table based on vertex signs. This approach builds on [Marching Tetrahedrons](3720-marching-tetrahedrons.md) by operating on the tetrahedral subgrid structure. [Link to child article](6833-naive-tetrahedral-surfacing.md).

## Operating characteristics
- **Time Complexity:** $O(N \cdot E)$, where $N$ is the number of tetrahedra and $E$ is the number of edges per tetrahedron (constant 6) [S1, unofficial]. However, due to the lack of vertex reuse and potential inefficiencies in polygon construction, it is significantly slower than optimized methods [S1, unofficial].
- **Memory Complexity:** $O(V_{out} + F_{out})$, where $V_{out}$ is the number of output vertices and $F_{out}$ is the number of output polygons [S1, unofficial]. Since vertices are duplicated, $V_{out}$ can be up to 6 times larger than necessary for a manifold surface [S1, unofficial].
- **Measured Performance:** The naive implementation is notably slower than VTK's optimized implementation, as noted in the exercise comparison [S1, unofficial].
- **Strengths:** Simple to implement; no complex data structures for neighbor lookup or vertex merging required [S1, unofficial].
- **Limitations:** Produces non-manifold surfaces with cracks due to duplicate vertices and inconsistent winding [S1, unofficial]. Not suitable for applications requiring manifold connectivity or efficient rendering [S1, unofficial].
- **When to Use:** Educational purposes or simple visualization where manifold connectivity is not critical [S1, unofficial].
- **When Not to Use:** Production rendering, simulation, or any application requiring manifold surfaces or high performance [S1, unofficial].

## Background
The naive approach to isosurface extraction from tetrahedral meshes is described in visualization exercises by Julien Tierny [S1, unofficial]. It serves as a baseline for understanding the challenges of surface extraction, such as vertex duplication and winding consistency, before introducing more sophisticated techniques like vertex reuse or lookup tables [S1, unofficial].

## Sources
[S1] Julien Tierny - Visualization Exercise - Isosurface Extraction, https://julien-tierny.github.io/visualizationExerciseIso.html, fetched 2026-10-30 (unofficial)
