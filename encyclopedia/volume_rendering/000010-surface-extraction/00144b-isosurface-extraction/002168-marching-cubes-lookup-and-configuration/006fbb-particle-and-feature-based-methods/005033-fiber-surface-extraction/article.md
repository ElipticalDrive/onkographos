# Fiber Surface Extraction
Also called: Fiber Surface Control Polygon (FSCP) extraction, Fiber Surface of a Line, Isosurface of weighted sum

## Overview
Fiber Surface Extraction generalizes isosurface extraction from univariate scalar fields to bivariate vector fields by extracting the preimage of a curve in the range space [S1]. It sets itself apart from standard Marching Cubes by replacing the scalar isovalue comparison with a point-in-polygon test in the range space and computing edge intersections via range-space line intersection rather than scalar interpolation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A bivariate function $f: \mathbb{R}^3 \to \mathbb{R}^2$, defined as $f(x) = (f_1(x), f_2(x))$; a computational mesh $M$ in the domain $\text{Dom } f$ (typically a cubic grid); and a Fiber Surface Control Polygon (FSCP) $P$ in the range $\text{Ran } f$ [S1]. The FSCP is a closed loop of line segments defining the boundary in the 2D range [S1]. Field values are computed on the fly during extraction rather than stored [S1].
*   **Output:** A triangular mesh approximating the fiber surface $f^{-1}(P)$ in the domain [S1].

**Data Structures:**
The method relies on the standard Marching Cubes data structures inherited from the parent [G].
1.  **Edge Intersection Table (`edgeTable`)**: 256 entries, 12-bit bitmasks indicating which of the 12 cube edges are intersected [G].
2.  **Triangle Connection Table (`triangleTable`)**: $256 \times 16$ array of edge indices grouped in triplets, terminated by `-1` [G].
3.  **Vertex Classification**: A binary classification of the 8 cube vertices as "black" (inside $P$) or "white" (outside $P$) [S1].

**Special Case: Lines in the Range**
If the FSCP degenerates to a straight line in the range, defined by $\vec{n} \cdot p = c$ where $\vec{n} = (n_1, n_2)$ is the normal vector and $c$ is the scaled perpendicular distance from the origin, the fiber surface is equivalent to the isosurface of the derived scalar field $g(x) = \vec{n} \cdot f(x) = n_1 f_1(x) + n_2 f_2(x)$ at isovalue $c$ [S1]. In this case, standard Marching Cubes is used directly with $g(x)$ [S1]. The gradient of the derived field is $\nabla g = n_1 \nabla f_1 + n_2 \nabla f_2$, allowing normal vectors to be computed via linear combination of the component gradients [S1].

**General Algorithm (Arbitrary FSCP):**
For an arbitrary closed polygon $P$, the algorithm adapts Marching Cubes as follows:

1.  **Vertex Classification (Stage I):** For each cell $C$ in mesh $M$, and for each vertex $V$ of $C$, evaluate $f(V)$ [S1]. Perform a point-in-polygon test to determine if $f(V)$ lies inside $P$ [S1]. If inside, classify $V$ as "black"; otherwise, classify as "white" [S1].
2.  **Case Index Calculation:** Compute the 8-bit case index $C$ based on the black/white classification of the 8 vertices, using the standard bit assignment convention [G].
3.  **Lookup:** Retrieve the intersection mask from `edgeTable[C]` and the triangle configuration from `triangleTable[C]` [G].
4.  **Edge Intersection and Interpolation (Stages III/IV):** For each triangle in the MC case, and for each intersected cell edge defined by vertices $u$ and $v$:
    *   Map vertices to the range: $p_u = f(u)$ and $p_v = f(v)$ [S1].
    *   Find the intersection point $w$ of the line segment $p_u p_v$ in the range with the polygon boundary $P$ [S1].
    *   Determine the parameter $t \in [0, 1]$ along the segment $f(u)f(v)$ such that $w = (1-t)f(u) + t f(v)$ [S1].
    *   Interpolate the spatial position in the domain: $e = u + t(v - u)$ [S1].
    *   Interpolate the normal vector at $e$ [S1].

**Pseudocode:**
The following pseudocode implements the general FSCP extraction [S1]. It uses the signed distance field of the FSCP to compute the intersection parameter $t$ efficiently, as recommended by the sources [S1]. Standard computational geometry routines are marked [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_fiber_surface(mesh, f, polygon):
    # Standard MC edge-to-vertex mapping [G]
    EDGE_VERTICES = [
        (0, 1), (1, 2), (2, 3), (3, 0),  # Bottom face
        (4, 5), (5, 6), (6, 7), (7, 4),  # Top face
        (0, 4), (1, 5), (2, 6), (3, 7)   # Vertical edges
    ]
    # LUTs provided by parent Marching Cubes Lookup and Configuration [G]
    # edge_table: list of 256 ints (12-bit masks)
    # triangle_table: list of 256 lists (16 ints, -1 terminated)
    
    triangles = []
    for cell in mesh.cells:
        # Stage I: Vertex Classification
        vertex_flags = []
        for v in cell.vertices:
            p = f(v)
            if point_in_polygon(p, polygon):  # [G]
                vertex_flags.append(1)
            else:
                vertex_flags.append(0)
        
        # Case Index Calculation
        case_index = sum(flag << i for i, flag in enumerate(vertex_flags))
        
        # Lookup
        edge_mask = edge_table[case_index]
        tri_indices = triangle_table[case_index]
        
        # Edge Intersections via Signed Distance Field [S1]
        edge_t = {}
        for e_idx in range(12):
            if (edge_mask >> e_idx) & 1:
                v1_idx, v2_idx = EDGE_VERTICES[e_idx]
                v1, v2 = cell.vertices[v1_idx], cell.vertices[v2_idx]
                p1, p2 = f(v1), f(v2)
                d1 = signed_distance(p1, polygon)  # [G]
                d2 = signed_distance(p2, polygon)  # [G]
                denom = d1 - d2
                if abs(denom) > 1e-9:
                    t = d1 / denom
                    edge_t[e_idx] = t
                else:
                    edge_t[e_idx] = 0.0  # Degenerate edge parallel to contour
        
        # Generate Triangles
        for i in range(0, len(tri_indices), 3):
            if tri_indices[i] == -1:
                break
            e1, e2, e3 = tri_indices[i], tri_indices[i+1], tri_indices[i+2]
            t1, t2, t3 = edge_t[e1], edge_t[e2], edge_t[e3]
            
            v1_1, v1_2 = EDGE_VERTICES[e1]
            v2_1, v2_2 = EDGE_VERTICES[e2]
            v3_1, v3_2 = EDGE_VERTICES[e3]
            
            # Interpolate domain positions
            e1_pos = cell.vertices[v1_1] + t1 * (cell.vertices[v1_2] - cell.vertices[v1_1])
            e2_pos = cell.vertices[v2_1] + t2 * (cell.vertices[v2_2] - cell.vertices[v2_1])
            e3_pos = cell.vertices[v3_1] + t3 * (cell.vertices[v3_2] - cell.vertices[v3_1])
            
            # Flat shading normal
            normal = cross(e2_pos - e1_pos, e3_pos - e1_pos)  # [G]
            normal = normalize(normal)  # [G]
            triangles.append((e1_pos, e2_pos, e3_pos, normal))
            
    return triangles
```

**Edge Cases and Pitfalls:**
*   **Performance:** Standard geometric tests (point-in-polygon, line-polygon intersection) are slow [S1]. The sources recommend mitigating this by precomputing the signed distance field of the FSCP, where interpolating the distance field yields the intersection parameter $t$ directly without explicit geometric tests [S1].
*   **Topology:** If the domain of $f$ does not touch the boundary of the range polygon $P$, the resulting fiber surface is a closed 2-manifold [S1].
*   **Ambiguity:** The standard Marching Cubes ambiguity cases (46, 47, 98, 99, 210, 213, 214, 215) still apply [G]. The fixed triangulation in the LUT may cause cracks if adjacent voxels resolve the ambiguity differently [G].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of cells in the mesh, as each cell is processed independently [S1]. The cost per cell is dominated by the point-in-polygon test and segment-polygon intersection, which are more expensive than the scalar comparisons in standard Marching Cubes [S1].
*   **Strengths:** Allows visualization of complex relationships in bivariate data, such as texture boundaries or color regions, which are not accessible via scalar isosurfaces [S1]. It generalizes naturally from lines (weighted sums) to arbitrary polygons [S1].
*   **Limitations:** The topology of the fiber surface depends on the topology of the polygon in the range and the mapping $f$ [S1].
*   **When to use:** When the user wants to extract surfaces based on the relationship between two scalar fields (e.g., "extract the surface where the RGB color is on the boundary of this texture patch") [S1].

## Background
Fiber Surface Extraction was introduced by Hamish Carr, Zhao Geng, Julien Tierny, Amit Chattopadhyay, and Aaron Knoll in the paper "Fiber Surfaces: Generalizing Isosurfaces to Bivariate Data" [S1]. The method generalizes the concept of isosurfaces to bivariate functions, allowing the extraction of surfaces defined by arbitrary curves in the range space [S1].

## Sources
[S1] Fiber Surfaces: Generalizing Isosurfaces to Bivariate Data, https://doi.org/10.1111/cgf.12636, fetched 2026-10-26
