# Tetrahedralization of Octree Cells

Also called: Dual Tetrahedralization, Octree Tetrahedralization

## Overview
Tetrahedralization of Octree Cells is a technique for generating a manifold isosurface from an adaptive octree by constructing a dual simplicial partition [S1]. It solves the problem of self-intersections and non-manifold vertices inherent in adaptive octrees by creating a dual structure that ensures convexity and manifold topology [S1]. Unlike Dual Marching Tetrahedra (DMC), which can produce self-intersecting "flaps" because it does not strictly respect the primal octree grid's convexity, this method guarantees a manifold surface [S1]. Compared to Dual Contouring (DC), it samples the function at a higher effective resolution by utilizing extra samples on edges, faces, and cells, though at the cost of increased computation time [S1].

## How it works
The method constructs a tetrahedral mesh dual to the octree structure and applies a table-based contouring algorithm to extract the isosurface [S1].

### Inputs and Outputs
*   **Input:** An adaptive octree where each node (cube) contains scalar field values at its vertices. The octree may be generated via bottom-up pruning or by refining cells where polygons intersect the signed distance function [S1].
*   **Output:** A manifold triangular mesh representing the isosurface.

### Data Structures
*   **Octree:** The primary data structure. Nodes represent cubes. No auxiliary data structures are required beyond the octree itself and the computed dual vertices [S1].
*   **Dual Vertices:** For each cube, edge, and face in the octree, a dual vertex is computed. These vertices are positioned using function values to minimize a four-dimensional error function [S1].

### Algorithm Steps
1.  **Recursive Traversal:** Perform a recursive octree traversal (similar to Dual Contouring) to identify leaves surrounding minimal edges of the tree [S1].
2.  **Identify Minimal Edge:** For a given minimal edge, determine the cube at the deepest depth that contains this edge [S1].
3.  **Create Line Segments:** For the identified minimal edge, create two line segments: one corresponding to the octree vertices of the minimal edge, and one connecting the dual vertex of the minimal edge to the dual vertex of the identified cube [S1].
4.  **Form Triangles:** For each pair of face-adjacent cubes surrounding the minimal edge, identify the minimal face between them. Connect the two line segments to the vertex dual to this minimal face. This creates two triangles for each minimal face [S1].
5.  **Form Tetrahedra:** Connect each of the triangles formed in Step 4 to the vertex dual to the cube they are adjacent to. This forms four tetrahedra for each face-adjacent cube [S1].
6.  **Contouring:** Contour each tetrahedron using a table-based method based on the sign configuration of the tetrahedron's vertices. A simplex table is used, which has size $2m+1$ for dimension $m$ (in 3D, this is manageable) [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tetrahedralize_octree(octree, scalar_field, isosurface_value):
    """
    Generates a manifold isosurface from an octree using tetrahedralization.
    
    Args:
        octree: The root node of the adaptive octree.
        scalar_field: Function returning scalar values at any coordinate.
        isosurface_value: The scalar threshold for contouring.
        
    Returns:
        A list of triangles forming the manifold surface.
    """
    triangles = []
    
    def compute_dual_vertex(entity_type, entity_coords):
        # [G] Standard implementation minimizes a 4D error function using
        # scalar values at the entity's corners/edges/faces [S1].
        # Returns a 3D coordinate.
        return (0.0, 0.0, 0.0)

    def simplex_contour(vertices, iso_val):
        """
        Contours a tetrahedron using a simplex table based on sign configuration.
        Returns a list of triangles (each triangle is a list of 3 vertices).
        """
        signs = [1 if v > iso_val else -1 for v in vertices]
        if all(s == 1 for s in signs) or all(s == -1 for s in signs):
            return []
        # Compute edge intersections
        edges = [(0,1), (0,2), (0,3), (1,2), (1,3), (2,3)]
        pts = []
        for i, j in edges:
            if signs[i] != signs[j]:
                t = (iso_val - vertices[i]) / (vertices[j] - vertices[i])
                pts.append(vertices[i] + t * (vertices[j] - vertices[i]))
        # [G] Standard simplex table lookup for 3D maps sign patterns to triangle indices.
        # We implement the core table-based logic directly.
        pattern = tuple(signs)
        tri_indices = {
            (1,1,1,-1): [(0,1,2)], (1,1,-1,1): [(0,1,3)], (1,-1,1,1): [(0,2,3)], (-1,1,1,1): [(1,2,3)],
            (1,1,-1,-1): [(0,1), (2,3)], (1,-1,1,-1): [(0,2), (1,3)], (1,-1,-1,1): [(0,3), (1,2)],
            (-1,1,1,-1): [(0,1), (2,3)], (-1,1,-1,1): [(0,2), (1,3)], (-1,-1,1,1): [(0,3), (1,2)],
            (1,-1,-1,-1): [(0,1,2), (0,1,3), (0,2,3)], (-1,1,-1,-1): [(0,1,2), (0,2,3), (1,2,3)],
            (-1,-1,1,-1): [(0,1,2), (0,1,3), (1,2,3)], (-1,-1,-1,1): [(0,1,3), (0,2,3), (1,2,3)]
        }.get(pattern, [])
        return [[pts[a], pts[b], pts[c]] for a, b, c in tri_indices]

    def traverse(node):
        if node.is_leaf():
            return
        for child in node.children:
            traverse(child)
        # Find minimal edges in this subtree
        minimal_edges = find_minimal_edges(node)
        for edge in minimal_edges:
            # Identify cube at deepest depth containing the edge
            deepest_cube = find_deepest_cube_containing(edge, node)
            # Get dual vertices
            dual_edge = compute_dual_vertex("edge", edge)
            dual_cube = compute_dual_vertex("cube", deepest_cube)
            # Two line segments: octree edge vertices and dual edge to dual cube
            octree_edge_verts = get_octree_vertices(edge)
            seg1 = octree_edge_verts
            seg2 = [dual_edge, dual_cube]
            # Face-adjacent cubes surrounding the edge
            adj_cubes = get_face_adjacent_cubes(edge, deepest_cube)
            for cube1, cube2 in adj_cubes:
                # Minimal face between cube1 and cube2
                face = get_minimal_face(cube1, cube2)
                dual_face = compute_dual_vertex("face", face)
                # Connect segments to dual face vertex -> two triangles
                tri1 = [seg1[0], seg1[1], dual_face]
                tri2 = [seg2[0], seg2[1], dual_face]
                # Connect each triangle to dual vertex of adjacent cube -> tetrahedra
                tet1 = [tri1[0], tri1[1], tri1[2], compute_dual_vertex("cube", cube1)]
                tet2 = [tri2[0], tri2[1], tri2[2], compute_dual_vertex("cube", cube1)]
                tet3 = [tri1[0], tri1[1], tri1[2], compute_dual_vertex("cube", cube2)]
                tet4 = [tri2[0], tri2[1], tri2[2], compute_dual_vertex("cube", cube2)]
                # Contour each tetrahedron
                for tet in [tet1, tet2, tet3, tet4]:
                    triangles.extend(simplex_contour(tet, isosurface_value))

    traverse(octree)
    return triangles
```

### Parameters
*   **Isosurface Value:** The scalar threshold for contouring.
*   **Octree Depth:** Determines the resolution. The method samples the function at one higher resolution than DC or DMC due to extra samples on edges, faces, and cells [S1].

### Edge Cases and Pitfalls
*   **Non-manifold Vertices:** If the central dual vertex is placed exactly on the isosurface, and portions of the contour on the cell boundary are not connected, the surface pinches to a point, creating a non-manifold vertex [S1].
*   **Empty Cells:** If the cell boundary does not contain the isosurface (all vertices have the same sign), placing the central dual vertex on the isosurface collapses a topological sphere to a point or creates an isolated point. The test still works in this case, but the topology is degenerate [S1].
*   **Self-Intersections:** Unlike DMC, this method avoids self-intersections (flaps folding back on themselves) because the primal octree grid ensures convexity [S1].

## Operating characteristics
*   **Time Complexity:** The method is approximately 1.5x slower than Dual Marching Tetrahedra (DMC) and 2x slower than Dual Contouring (DC) for comparable sampling densities [S1].
*   **Measured Performance:**
    *   **Armadillo Man (Depth 8):** 2.58s (Ours) vs 1.85s (DMC) vs 1.35s (DC) [S1].
    *   **Mechanical Part (Depth 9):** 4.81s (Ours) vs 3.54s (DMC) vs 2.97s (DC) [S1].
    *   **Lens (Depth 10):** 9.72s (Ours) vs 6.42s (DMC) vs 5.99s (DC) [S1].
    *   **Tank (Depth 8):** Total time 8.78s (Ours) vs 5.29s (DMC) vs 3.78s (DC). Triangle count: 3.63M (Ours) vs 1.16M (DMC) vs 727k (DC) [S1].
*   **Memory Complexity:** No auxiliary data structures are required beyond the octree itself and the dual vertices [S1].
*   **Strengths:** Generates manifold surfaces; avoids self-intersections; recovers sharp features; extends easily to higher dimensions (simplex table size $2m+1$ vs MC's $2^m$) [S1].
*   **Limitations:** Slower than DC and DMC; generates more triangles than DC (though fewer than DMC with triangulation improvement) [S1].
*   **When to Use:** When a manifold surface is required from an adaptive octree and self-intersections from DMC are unacceptable [S1].
*   **When Not to Use:** When performance is critical and non-manifold artifacts are tolerable (use DC or DMC) [S1].

## Background
The method was introduced in the paper "Isosurfaces Over Simplicial Partitions of Multiresolution Grids" [S1]. It builds on the recursive octree traversal of Dual Contouring [JLSW02] and uses a table-based contouring method based on the sign configuration of tetrahedra [DK91]. The approach is inductive, allowing it to extend to higher dimensions, unlike Marching Cubes which suffers from table explosion ($2^m$ entries) [S1].

## Sources
[S1] Isosurfaces Over Simplicial Partitions of Multiresolution Grids, http://faculty.cs.tamu.edu/schaefer/research/iso_simplicial.pdf, fetched 2026-10-29
