# Barycentric Mapping for Material Interfaces

Also called: Barycentric Voronoi Interface Reconstruction

## Overview
Barycentric Mapping for Material Interfaces is an algorithm for reconstructing piecewise two-manifold boundary surfaces between multiple materials in a grid where each cell contains fractional volumetric information for each material [S1]. It transforms the reconstruction problem to analyze a dual data set where vertices have associated barycentric coordinate tuples representing material fractions [S1]. It sets itself apart from standard isosurface extraction by operating directly on volume-fraction data in unstructured triangular meshes, using the geometric properties of Voronoi cells in barycentric space to define interfaces rather than scalar field thresholds [S1]. Unlike SLIC, Youngs, or Gueffier algorithms which produce generally discontinuous interfaces and are limited to 2D or specific normal estimates, this algorithm produces a crack-free piecewise two-manifold, handles any number of materials per element, works on any grid structure (via tetrahedral conversion), and does not rely on application-specific hydrodynamics knowledge [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A grid $S$ (rectilinear, curvilinear, or unstructured) where each cell $C$ has an associated tuple $(v_1, v_2, \dots, v_m)$ representing the fractional volume of each of $m$ materials, such that $\sum_{i=1}^{m} v_i = 1$ [S1].
*   **Output:** A triangulated surface approximation of the boundaries between materials [S1].

**Data Structures:**
1.  **Dual Tetrahedral Mesh:** Constructed from the original grid. Each vertex in the dual mesh corresponds to the center of an original element and is associated with a barycentric coordinate tuple $(v_1, \dots, v_m)$ [S1].
2.  **BSP Tree:** Used to store the tetrahedron (or simplex) to facilitate the clipping algorithm described by Samet [S1].

**Steps:**

1.  **Preprocessing:** Subdivide each polyhedral cell of the input grid into tetrahedra to create a tetrahedral grid [S1].
2.  **Dual Mesh Construction:** Construct a dual tetrahedral mesh. Each vertex in the dual mesh represents the center of an original element and holds the barycentric coordinate tuple $(v_1, \dots, v_m)$ [S1].
3.  **Mapping to Barycentric Space:** For each tetrahedral cell $T$ in the dual mesh, map the tetrahedron into an $m$-simplex representing $m$-dimensional barycentric space, where $m$ is the number of materials [S1]. Let the image of $T$ in barycentric space be $T'$ [S1].
4.  **Intersection Calculation:** Calculate the intersections of the mapped tetrahedron edges (or faces, depending on dimensionality) with the edges of Voronoi cells in the barycentric $m$-simplex [S1]. Voronoi cells represent regions where one material dominates others locally [S1].
    *   For $m=2$, the boundary is defined by $v_1 = v_2 = 1/2$. Find the point on each edge where this equality holds using linear interpolation [S1].
    *   For $m=3$, the Voronoi cells $V_1, V_2, V_3$ are bounded by the edges of the triangle and line segments $l_{12}, l_{13}, l_{23}$ defined by:
        $$ v_1 = v_2 \text{ and } v_3 \le 1/3 $$
        $$ v_1 = v_3 \text{ and } v_2 \le 1/3 $$
        $$ v_2 = v_3 \text{ and } v_1 \le 1/3 $$
        [S1]
    *   For $m \ge 4$, boundaries are defined by hyperplanes $v_i = v_j$ for $i \neq j$. Use a clipping algorithm against these hyperplanes [S1].
5.  **Mapping Back:** Map the calculated intersection points from barycentric space back to the original physical space [S1]. For 2D triangular grids, map intersections from $T'$ back to $T$ using the same linear parameters used to determine intersections on the edges of $T$ [S1]. For 3D tetrahedral grids, map intersections back to $T$ using linear interpolation [S1].
6.  **Triangulation:** Triangulate the mapped intersection points to form the boundary surface approximation [S1].
    *   **Case $m=3$, 3D Tetrahedra:**
        *   If no edge of $T'$ intersects $l_{12}, l_{13}, l_{23}$, no boundaries exist [S1].
        *   If edges intersect but the center $c(1/3, 1/3, 1/3)$ is not inside any face of $T'$, calculate intersection line segments for each triangular face. Determine a triangulation from these segments by following the marching tetrahedra algorithm [S1].
        *   If the center $c$ lies inside two faces of $T'$, two faces have a single material boundary line segment connecting two edge intersection points, and two faces have three material boundary line segments meeting in the interior. Map intersections back to $T$. Determine a valid triangulation of the boundary surface using the face segments and the line segment connecting the two interior points [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def barycentric_mapping(grid, num_materials):
    """
    Reconstruct material interfaces from volume-fraction data.
    
    Args:
        grid: List of cells. Each cell is a list of vertices.
              Each vertex has attributes: position (x,y,z), 
              fractions (list of floats summing to 1).
        num_materials: Integer m.
        
    Returns:
        List of triangles (each triangle is a list of 3 vertices).
    """
    # Step 1: Preprocess grid into tetrahedra
    tetrahedra = preprocess_to_tetrahedra(grid)
    
    all_triangles = []
    
    for tet in tetrahedra:
        # tet is a list of 4 vertices: [v0, v1, v2, v3]
        # Each vertex v has .pos and .fractions (list of length m)
        
        if num_materials == 2:
            all_triangles.extend(process_two_materials(tet))
        elif num_materials == 3:
            all_triangles.extend(process_three_materials(tet))
        else:
            all_triangles.extend(process_general_materials(tet, num_materials))
            
    return all_triangles

def process_two_materials(tet):
    """
    For m=2, boundary is v1 = v2 = 1/2.
    """
    triangles = []
    intersection_points = []
    
    edges = [(0,1), (0,2), (0,3), (1,2), (1,3), (2,3)]
    for i, j in edges:
        v0, v1 = tet[i], tet[j]
        f0 = v0.fractions[0]
        f1 = v1.fractions[0]
        
        denom = f1 - f0
        if abs(denom) < 1e-9:
            continue 
            
        t = (0.5 - f0) / denom
        
        if 0.0 <= t <= 1.0:
            pos = lerp(v0.pos, v1.pos, t)
            intersection_points.append(pos)
            
    # Marching tetrahedra connection: form triangles from intersection points
    if len(intersection_points) >= 3:
        for k in range(1, len(intersection_points) - 1):
            triangles.append([intersection_points[0], intersection_points[k], intersection_points[k+1]])
            
    return triangles

def process_three_materials(tet):
    """
    For m=3, map to barycentric triangle T'.
    Voronoi boundaries: v1=v2 & v3<=1/3, v1=v3 & v2<=1/3, v2=v3 & v1<=1/3.
    """
    triangles = []
    intersection_points = []
    
    edges = [(0,1), (0,2), (0,3), (1,2), (1,3), (2,3)]
    for i, j in edges:
        v0, v1 = tet[i], tet[j]
        f0 = v0.fractions
        f1 = v1.fractions
        
        ts = find_voronoi_intersections(f0, f1)
        for t, _ in ts:
            pos = lerp(v0.pos, v1.pos, t)
            intersection_points.append(pos)
            
    # Triangulate intersection points on the tetrahedron boundary
    if len(intersection_points) >= 3:
        for k in range(1, len(intersection_points) - 1):
            triangles.append([intersection_points[0], intersection_points[k], intersection_points[k+1]])
            
    return triangles

def find_voronoi_intersections(fa, fb):
    """
    Find t in [0,1] such that the interpolated fraction tuple lies on a Voronoi boundary.
    Returns list of (t, line_type).
    """
    results = []
    
    # v1 = v2
    diff_v1 = fb[0] - fa[0]
    diff_v2 = fb[1] - fa[1]
    denom = diff_v1 - diff_v2
    if abs(denom) > 1e-9:
        t = (fa[1] - fa[0]) / denom
        if 0.0 <= t <= 1.0:
            v3_at_t = fa[2] + t * (fb[2] - fa[2])
            if v3_at_t <= 1.0/3.0 + 1e-9:
                results.append((t, '12'))
                
    # v1 = v3
    diff_v3 = fb[2] - fa[2]
    denom = diff_v1 - diff_v3
    if abs(denom) > 1e-9:
        t = (fa[2] - fa[0]) / denom
        if 0.0 <= t <= 1.0:
            v2_at_t = fa[1] + t * (fb[1] - fa[1])
            if v2_at_t <= 1.0/3.0 + 1e-9:
                results.append((t, '13'))
                
    # v2 = v3
    denom = diff_v2 - diff_v3
    if abs(denom) > 1e-9:
        t = (fa[2] - fa[1]) / denom
        if 0.0 <= t <= 1.0:
            v1_at_t = fa[0] + t * (fb[0] - fa[0])
            if v1_at_t <= 1.0/3.0 + 1e-9:
                results.append((t, '23'))
                
    return results

def process_general_materials(tet, m):
    """
    For m >= 4, use BSP clipping against hyperplanes vi = vj.
    """
    # Map tetrahedron T to T' in m-simplex.
    # Clip T' against hyperplanes vi = vj using BSP tree [S1].
    # Map intersections back to physical space.
    # Triangulate resulting points.
    # Implementation follows the same intersection pattern as m=3.
    return []

def lerp(a, b, t):
    return a + t * (b - a)

def preprocess_to_tetrahedra(grid):
    # Subdivide polyhedral cells into tetrahedra [G]
    return []
```

**Parameters:**
*   $m$: The number of materials present in the data set. The algorithm must treat any number of materials per element/vertex [S1].

**Edge Cases and Pitfalls:**
*   If only one material is present in a cell, no boundary surface exists for that cell [S1].
*   If no edge of $T'$ intersects the Voronoi boundary lines ($l_{12}, l_{13}, l_{23}$), no material boundaries exist in $T$ [S1].
*   The algorithm can miss material boundaries in tetrahedra, similar to other isosurface-type algorithms [S1].

## Subtopics

### [Barycentric Mapping for Material Interfaces (2D Triangular Grids)](9890-barycentric-mapping-for-material-interfaces-2d-triangular-gr.md)
This method determines material interface line segments within a single 2D triangular grid cell by mapping the triangle's barycentric coordinates to a reference barycentric triangle and intersecting with Voronoi cell boundaries. It sets itself apart from standard isosurface extraction by operating directly on volume-fraction data in unstructured triangular meshes, using the geometric properties of Voronoi cells in barycentric space to define interfaces rather than scalar field thresholds.

## Operating characteristics
*   **Time Complexity:** The algorithm runs in effectively the same time as the marching cubes/tetrahedra algorithm [S1]. It traverses the cells of a grid and calculates a polygonal representation of the material boundaries for each cell [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Measured Performance:** Not given in the sources [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Produces a crack-free piecewise two-manifold, handles any number of materials per element, works on any grid structure (via tetrahedral conversion), and does not rely on application-specific hydrodynamics knowledge [S1].
*   **Limitations:** Can miss material boundaries in tetrahedra [S1]. Requires preprocessing to convert non-tetrahedral grids to tetrahedral grids [S1].
*   **When to use:** When reconstructing material interfaces from data sets with volume-fraction information, especially for multiple materials and unstructured grids [S1].
*   **When not to use:** When only one material is present (no boundary) or when high-fidelity boundary detection is critical and missing boundaries are unacceptable [S1].

## Background
Barycentric Mapping for Material Interfaces was introduced by K. Bonnell, M.A. Duchaineau, D.R. Schikore, B. Hamann, and K.I. Joy from Lawrence Livermore National Laboratory and UC Davis [S1]. It was submitted to IEEE Visualization, Salt Lake City, UT, October 8-13, 2000, with a preprint dated March 29, 2000 [S1]. It addresses the need for geometric approximations of fluid interfaces in Eulerian flow simulations, where grid cells contain fractional volumetric information for each material [S1].

## Sources
[S1] Constructing Material Interfaces from Data Sets with Volume-Fraction Information, https://digital.library.unt.edu/ark:/67531/metadc739076/?q=marching cubes, fetched 2026-10-28
