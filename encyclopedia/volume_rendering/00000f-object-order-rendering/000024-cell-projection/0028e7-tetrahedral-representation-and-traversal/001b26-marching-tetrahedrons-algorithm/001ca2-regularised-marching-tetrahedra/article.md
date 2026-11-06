# Regularised Marching Tetrahedra


## Overview
Regularised Marching Tetrahedra (RMT) is an iso-surface extraction algorithm that combines the topological consistency of the Marching Tetrahedra Algorithm with vertex clustering to generate simplified surfaces with improved triangle aspect ratios [S1]. Unlike standard Marching Tetrahedra, which can produce thin, degenerate triangles that degrade interpolated shading, RMT reduces the triangle count by approximately 70% while preserving the original topology by basing clustering on the tetrahedral lattice [S1].

## How it works
**Inputs:** A scalar field defined on a 3D grid (or implicit surface definition) and an iso-value threshold.
**Outputs:** A triangular mesh representing the iso-surface.

**Data Structures:**
1.  **Lattice:** A tessellation of the data volume into tetrahedra. The lattice is constructed from the underlying grid cells. Each grid cell is decomposed into tetrahedra. The specific decomposition must be consistent to ensure topological correctness across cell boundaries.
2.  **Vertex Clusters:** A mapping from original sample points (grid vertices) to clustered representative points. Since RMT clusters *before* triangulation, the primary data structure is the set of unique clustered vertices.
3.  **Edge Intersections:** For each tetrahedral edge crossing the iso-surface, the intersection point is calculated.

**Steps:**

1.  **Construct Lattice:**
    Decompose the volume grid into tetrahedra. Ensure that the decomposition is consistent across shared faces and edges of the original grid cells to prevent topological holes or overlaps [S1].

2.  **Vertex Clustering:**
    Perform vertex clustering on the sampled data vertices *before* triangulation.
    *   Identify vertices that are spatially close to each other.
    *   Cluster these vertices into single representative points.
    *   **Crucial Constraint:** The clustering must be based on the original tetrahedral lattice structure. Vertices belonging to the same tetrahedron or sharing a common edge/face in the lattice are considered for clustering together. This specific constraint preserves the topological consistency of the Marching Tetrahedra algorithm, which is not guaranteed by generic vertex clustering [S1].
    *   The position of the clustered vertex is typically the centroid or average of the clustered original vertices [G].

3.  **Triangulation:**
    For each tetrahedron in the lattice:
    *   Determine which edges intersect the iso-surface by comparing the scalar values at the tetrahedron's vertices against the iso-value.
    *   Calculate the intersection points on these edges using linear interpolation.
    *   Use the standard Marching Tetrahedra lookup table to determine the connectivity of the triangles, but use the *clustered* vertex positions instead of the original grid vertex positions.
    *   This results in a mesh of near-regular triangles [S1].

4.  **Gradient and Curvature Estimation (Optional/Post-processing):**
    To estimate the gradient or curvature at a surface vertex:
    *   Identify all tetrahedra that contain the edge associated with that vertex.
    *   Use only the scalar data from the vertices of these specific tetrahedra to compute the gradient/curvature [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def regularised_marching_tetrahedra(grid, iso_value):
    # 1. Construct Tetrahedral Lattice
    # Decompose each grid cell into tetrahedra.
    # Ensure consistent decomposition across cell boundaries.
    tetrahedra = decompose_grid_into_tetrahedra(grid)
    
    # 2. Vertex Clustering
    # Cluster original grid vertices based on the tetrahedral lattice topology.
    # Vertices close in space and connected in the lattice are merged.
    clustered_vertices = cluster_vertices(grid.vertices, tetrahedra)
    
    # Map original vertex indices to clustered vertex indices/positions
    vertex_map = build_vertex_map(grid.vertices, clustered_vertices)
    
    triangles = []
    
    # 3. Triangulation
    for tet in tetrahedra:
        # Get scalar values for the 4 vertices of this tetrahedron
        # Use the original grid values, but we will use clustered positions for output
        vals = [grid.values[tet.v[i]] for i in range(4)]
        
        # Determine intersection edges
        intersecting_edges = find_intersecting_edges(vals, iso_value)
        
        if not intersecting_edges:
            continue
            
        # Calculate intersection points
        # Note: We use the CLUSTERED positions for the final mesh vertices
        intersection_points = []
        for edge in intersecting_edges:
            v0_idx, v1_idx = edge
            # Get clustered positions
            p0 = clustered_vertices[vertex_map[v0_idx]]
            p1 = clustered_vertices[vertex_map[v1_idx]]
            
            # Linear interpolation for intersection point
            # t = (iso_value - val0) / (val1 - val0)
            val0 = vals[v0_idx]
            val1 = vals[v1_idx]
            t = (iso_value - val0) / (val1 - val0)
            
            p_intersect = lerp(p0, p1, t)
            intersection_points.append(p_intersect)
            
        # Generate triangles based on the standard MT configuration
        # for the specific edge configuration
        tri = generate_triangles_from_edges(intersecting_edges, intersection_points)
        triangles.extend(tri)
        
    return triangles

def cluster_vertices(original_vertices, tetrahedra):
    # Group vertices that are spatially close and topologically connected
    # via the tetrahedral lattice.
    # Return a list of unique representative points.
    # Implementation detail: Use a spatial hash or grid to find neighbors,
    # then merge based on lattice connectivity.
    pass

def build_vertex_map(original_vertices, clustered_vertices):
    # Map each original vertex index to the index of its representative
    # in the clustered_vertices list.
    pass
```

**Parameters:**
*   **Iso-value:** The scalar threshold for surface extraction.
*   **Clustering Threshold:** The maximum distance between vertices to be considered for clustering. Not explicitly given in sources, but implied by "spatially close" [S1].

**Edge Cases and Pitfalls:**
*   **Clustering Topology:** If clustering is not strictly bound by the tetrahedral lattice, topological errors (holes, non-manifold edges) may occur [S1].
*   **Accuracy vs. Simplification:** The vertex clustering method is not optimal for minimizing surface area, volume, or surface curvature errors. More accurate positioning would require principal curvature calculations, which are more complex [S1].
*   **Gradient Estimation:** Gradient estimation must be limited to the local tetrahedral neighborhood to maintain consistency with the simplified mesh [S1].

## Subtopics

### [Regularised Marching Tetrahedra (Simplified)](7413-regularised-marching-tetrahedra-simplified.md)
An iso-surface extraction algorithm that iterates over a volume grid to extract intersections within tetrahedral cells, specifically designed for objects represented using radial basis functions. It differs from its parent, Regularised Marching Tetrahedra, by omitting the vertex clustering step, thereby lacking the mesh simplification and improved triangle aspect ratios provided by the full method. It is chosen over Marching Squares to avoid topological ambiguities.

## Operating characteristics
*   **Triangle Count:** Typically reduces the triangle count by 70% compared to Marching Tetrahedra (MT) and by 40% compared to Marching Cubes (MC) [S1].
*   **Performance:** The additional time required for clustering is offset by the reduction in points and triangles stored, resulting in a fast algorithm that can be generated in nearly the same time as MT alone [S1].
*   **Visual Quality:** Significantly improves interpolated shading (especially Gouraud shading) by eliminating sharp edges and points caused by thin/small triangles in MT and MC [S1].
*   **Accuracy:** Volume and surface area error differences compared to MT are much less than the error inherent in the sampling resolution [S1].
*   **Limitations:** The vertex clustering is not optimal for minimizing surface area, volume, or curvature errors [S1].

## Background
Regularised Marching Tetrahedra was introduced to address the issues of mesh simplification and poor triangle aspect ratios in standard iso-surface extraction methods. While mesh simplification algorithms exist, they often fail to maintain topology. RMT combines the topological robustness of Marching Tetrahedra with the simplification benefits of vertex clustering, specifically tailored to preserve topology by leveraging the tetrahedral lattice structure [S1]. It is suitable for implicit surfaces, thresholded medical data, and surfaces created from object cross-sections [S1].

## Sources
[S1] Regularised marching tetrahedra: improved iso-surface extraction, https://doi.org/10.1016/s0097-8493(99)00076-x, fetched 2026-10-27
