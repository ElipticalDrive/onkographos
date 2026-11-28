# Lopes' Marching Cubes with Critical Isovalues

Also called: Lopes' MC, Topologically Correct Marching Cubes based on Lopes

## Overview
Lopes' Marching Cubes with Critical Isovalues is a topologically correct surface extraction method that handles critical isovalues by using topological polygons (TPs), shoulder points, and inflection points to correctly represent tunnels and degenerate cases in trilinear cells [S1]. It builds on [Topologically Correct Marching Cubes](1368-topologically-correct-marching-cubes.md) by providing robust handling of cases where the isovalue coincides with critical values of the trilinear interpolant, specifically addressing the topological ambiguity in Case 13j [S1]. Unlike Nielson's method, which guarantees correctness only when the isovalue differs from the critical value, Lopes' approach produces better results for isovalues close to critical isovalues, though it may generate more triangles [S1].

## How it works
**Inputs:** A scalar field defined on a 3D grid, an isovalue $I$, and the trilinear interpolant $F(x, y, z)$ within each cell.
**Outputs:** A triangle mesh approximating the isosurface $F(x, y, z) = I$.

**Data Structures:**
*   **Cell Configuration:** Determined by vertex polarities (sign of $F(v) - I$) and contour topology on faces.
*   **Topological Polygon (TP) LUT:** Maps configurations to TPs. Each TP entry includes:
    *   Vertices on cell faces.
    *   Number of "loop-back" faces (faces containing two edges of the same TP, counting opposite faces only once) [S1].
    *   For Case 13j: A flag indicating if the TP is used only for a tunnel connecting positive vertices, only for negative vertices, or "irrelevant" (part of both) [S1].
*   **Inflection Points:** Up to six points per cell, associated with faces, where two derivatives of $F$ vanish [S1].
*   **Shoulder Points:** Points on cell faces derived from bilinear interpolants [S1].
*   **Bi-shoulder Points:** Points in the cell interior derived from sweeping planes [S1].

**Steps:**

1.  **Configuration Selection:** For each cell, determine the configuration using the Asymptotic Decider based on vertex polarities and contour topology on cell faces [S1].
2.  **TP Retrieval:** Retrieve the corresponding Topological Polygon(s) from the LUT for the selected configuration [S1].
3.  **Refinement with Shoulder Points:** Refine each TP by adding shoulder points [S1].
    *   A shoulder point is computed as the intersection of the isocontour with the line connecting the midpoint of a linear contour approximation on a face and the location of the face saddle [S1].
    *   In degenerate cases where the contour consists of two perpendicular lines, merge the shoulder points of the two hyperbolic arcs at the face saddle location [S1].
4.  **Triangulation of TPs:** Triangulate each TP individually by connecting its vertices to internal contour points [S1].
    *   **Case A: TP contains at least one loop-back face.** Triangulate using inflection points associated with those loop-back faces [S1]. A triangulation LUT based on the number of vertices and loop-back faces can be used [S1].
    *   **Case B: TP has no loop-back faces.**
        *   Check for tunnels: Determine if all six inflection points are in the cell's interior [S1].
        *   **Tunnel Exists:** Connect the TP to the polyline formed by the six interior inflection points [S1].
            *   *Correction for Case 13j:* If the configuration is 13j, determine the tunnel type (connecting positive or negative vertices) using Nielson's criterion [S1]. Connect only those TPs whose LUT flag matches the detected tunnel type (or if the flag is "irrelevant") [S1].
        *   **No Tunnel:** Connect the TP's edges to a "bi-shoulder point" [S1].
            *   Compute bi-shoulder points iteratively by sweeping planes through the cell, starting from faces intersected by the TP [S1].
            *   Use a selection scheme to locate the most appropriate bi-shoulder point among non-unique candidates [S1].

**Formulas:**
*   **Shoulder Point:** Let $M$ be the midpoint of the linear contour approximation on a face, and $S$ be the face saddle location. The shoulder point $P$ is the intersection of the line segment $MS$ with the isocontour on that face [S1].
*   **Inflection Point:** Locations where $\nabla^2 F(x, y, z) = 0$ (two derivatives vanish) on the isosurface [S1].

**Parameters:**
*   **Inflection Points:** Up to six per cell [S1].
*   **Loop-back Faces:** Defined as faces containing two edges of the same TP, counting opposite faces only once [S1].

**Edge Cases and Pitfalls:**
*   **Case 13j:** Lopes' original method incorrectly connects three TPs to inflection points in this case, resulting in invalid triangulation [S1]. This is corrected by using Nielson's criterion to distinguish tunnel types and filtering TPs based on LUT flags [S1].
*   **Critical Isovalues:** The method is designed to handle isovalues close to critical values where standard Marching Cubes fails [S1].
*   **Bi-shoulder Point Uniqueness:** Bi-shoulder points are not unique; a selection scheme is required [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def march_cubes_lopes_critical(field, isovalue, grid):
    """
    Performs topologically correct Marching Cubes with critical isovalues.
    field: 3D array of scalar values.
    isovalue: The threshold for surface extraction.
    grid: Grid dimensions (nx, ny, nz).
    """
    triangles = []
    
    # Precompute inflection points for all cells if needed, or compute on demand
    # Inflection points are where two derivatives of F vanish.
    # For a trilinear cell, these can be computed analytically or numerically.
    
    for x in range(grid.nx - 1):
        for y in range(grid.ny - 1):
            for z in range(grid.nz - 1):
                cell = get_cell(field, x, y, z)
                config = asympotic_decider(cell, isovalue)
                
                # Retrieve TPs from LUT
                tps = get_tps_from_lut(config)
                
                # Refine TPs with shoulder points
                refined_tps = []
                for tp in tps:
                    refined_tp = refine_tp_with_shoulder_points(tp, cell)
                    refined_tps.append(refined_tp)
                
                # Triangulate TPs
                for tp in refined_tps:
                    if tp.has_loop_back_faces():
                        # Use inflection points associated with loop-back faces
                        inflection_pts = get_inflection_points_for_faces(tp.loop_back_faces, cell)
                        tp_triangles = triangulate_tp_with_inflection_points(tp, inflection_pts)
                    else:
                        # Check for tunnel
                        all_inflection_interior = check_all_inflection_points_interior(cell)
                        
                        if all_inflection_interior:
                            # Tunnel exists
                            inflection_polyline = get_inflection_points_interior(cell)
                            
                            # Handle Case 13j correction
                            if config == 13j:
                                tunnel_type = determine_tunnel_type_nielson(cell, isovalue)
                                if tp.flag_matches_tunnel(tunnel_type):
                                    tp_triangles = connect_tp_to_polyline(tp, inflection_polyline)
                                else:
                                    tp_triangles = [] # Skip this TP
                            else:
                                tp_triangles = connect_tp_to_polyline(tp, inflection_polyline)
                        else:
                            # No tunnel, use bi-shoulder point
                            bi_shoulder = compute_bi_shoulder_point(tp, cell)
                            tp_triangles = connect_tp_to_bi_shoulder(tp, bi_shoulder)
                    
                    triangles.extend(tp_triangles)
    
    return triangles

def refine_tp_with_shoulder_points(tp, cell):
    """
    Refines a Topological Polygon by adding shoulder points.
    """
    new_vertices = []
    for edge in tp.edges:
        face = edge.face
        # Get linear contour approximation midpoint
        m = get_linear_contour_midpoint(edge, cell)
        # Get face saddle location
        s = get_face_saddle(face, cell)
        # Compute shoulder point as intersection of line MS with isocontour
        shoulder = intersect_line_isocontour(m, s, face, cell)
        
        # Handle degenerate case: if contour is two perpendicular lines,
        # merge shoulder points at saddle location
        if is_degenerate_contour(face, cell):
            shoulder = s
        
        new_vertices.append(shoulder)
    
    tp.vertices.extend(new_vertices)
    return tp

def compute_bi_shoulder_point(tp, cell):
    """
    Computes a bi-shoulder point by sweeping planes through the cell.
    """
    # Start from faces intersected by the TP
    start_faces = tp.intersected_faces
    
    # Iteratively sweep planes
    candidates = []
    for face in start_faces:
        # Sweep plane parallel to coordinate system planes
        plane = sweep_plane(face, cell)
        # Find shoulder points on this plane
        shoulder_pts = find_shoulder_points_on_plane(plane, cell)
        candidates.extend(shoulder_pts)
    
    # Selection scheme to find the most appropriate bi-shoulder point
    best_bi_shoulder = select_best_bi_shoulder(candidates, tp)
    return best_bi_shoulder

def determine_tunnel_type_nielson(cell, isovalue):
    """
    Uses Nielson's criterion to determine if the tunnel connects positive or negative vertices.
    """
    # Check sign of trilinear interpolant in the cuboid containing DeV[T]
    # DeV[T] is DeVella's necklace
    dev_t = get_devella_necklace(cell)
    sign = get_trilinear_sign(dev_t, cell, isovalue)
    if sign > 0:
        return "positive"
    else:
        return "negative"
```

## Operating characteristics
*   **Time Complexity:** Depends on the number of cells and the complexity of computing inflection points and bi-shoulder points. The iterative sweeping for bi-shoulder points adds overhead compared to standard MC [S1].
*   **Memory Complexity:** Requires storage for TPs, inflection points, and LUTs. The LUT size is larger than standard MC due to additional flags and TP definitions [S1].
*   **Performance:** Lopes' method produces better results for isovalues close to critical isovalues compared to Nielson's method [S1]. Nielson's method produces fewer triangles but guarantees correctness only when the isovalue differs from the critical value [S1].
*   **Strengths:** Handles critical isovalues correctly; robust topology for degenerate cases [S1].
*   **Limitations:** More complex implementation; potentially more triangles than Nielson's method; Case 13j requires special handling [S1].

## Background
Lopes' Marching Cubes with Critical Isovalues is based on the works of Lopes [15] and Nielson [19] [S1]. Lopes' method constructs an isosurface using topological polygons and shoulder points [S1]. Nielson provided a comprehensive analysis of trilinear interpolant contours, leading to triangulations for all possible configurations [S1]. The implementation described here modifies Lopes' approach to correct flaws in Case 13j using Nielson's criterion [S1]. The method is used in applications requiring accurate surface extraction near critical isovalues, such as exploring scalar fields [S1].

## Sources
[S1] Exploring Scalar Fields Using Critical Isovalues, https://escholarship.org/content/qt8h5134ht/qt8h5134ht.pdf?t=ptt27g, fetched 2026-10-27
