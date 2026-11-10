# 2D Stitching

Also called: Stitching 2D Grids

## Overview
2D Stitching is an index-based tessellation scheme that fills topological gaps between dual grids of different Adaptive Mesh Refinement (AMR) levels to generate crack-free isosurfaces [S1]. It merges coarse and fine grid levels seamlessly by generating "stitch cells" (triangles and deformed quadrilaterals) without subdividing existing boundary elements [S1]. Unlike constrained Delaunay triangulation, which requires subdividing quadrilateral boundary faces into triangles (causing cracks in 3D Marching Cubes extraction), this method preserves the quadrilateral topology of the grid boundaries [S1].

## How it works
**Inputs:**
*   Coarse grid boundary edges and vertices.
*   Fine grid boundary edges and vertices.
*   Refinement ratio $r$ (integer, typically 2 or 4) [S1].
*   Boolean flags indicating whether coarse grid points are refined by other fine grids [S1].

**Outputs:**
*   A set of stitch cells (triangles and deformed quadrilaterals) connecting the fine grid boundary to the coarse grid boundary [S1].

**Data Structures:**
*   The stitch mesh is a collection of polygons.
*   Vertices are existing grid points; no new vertices are generated [S1].
*   Cells are defined by their vertex indices.

**Steps:**

1.  **Initialize Stitch Cells:** Create an empty list for stitch cells [S1].

2.  **Process Boundary Edges:** Iterate over each edge $e$ on the boundary of the fine grid [S1].
    *   Determine the connectivity to the coarse grid based on the refinement ratio $r$ [S1].
    *   **Case A: General Refinement Ratio $r$**
        *   Group the fine grid edges into sequences of length $r$ [S1].
        *   For each sequence of $(r-1)$ consecutive fine edge segments, connect them to a single common coarse-grid point [S1].
        *   Connect the $r$-th fine edge to a coarse edge [S1].
        *   *Note:* For $r=2$, this simplifies to alternating connections: each fine edge connects to either a coarse vertex or a coarse edge [S1].
    *   **Case B: Check Coarse Point Refinement Status**
        *   If a fine edge connects to a coarse point $P$:
            *   If $P$ is **not refined**: Connect the fine edge to $P$ using the standard tessellation (forming triangles or deformed quadrilaterals) [S1].
            *   If $P$ **is refined**: The fine edge must connect to another fine edge to form a rectilinear cell (quadrilateral) instead of a triangle [S1].
        *   If a fine edge connects to a coarse edge $E$ with endpoints $P_1, P_2$:
            *   If **both** $P_1$ and $P_2$ are refined: Connect the fine edge to another fine edge [S1].
            *   If **only one** endpoint is refined: Treat as an edge case using the same tessellation logic as other edge cases (consistent with the "one endpoint refined" scenario) [S1].
            *   If **neither** is refined: Connect to the coarse edge normally [S1].

3.  **Process Boundary Vertices:** Iterate over each vertex $v$ on the boundary of the fine grid [S1].
    *   Identify the adjacent coarse grid points $P_0, P_1$ that bound the region where $v$ resides [S1].
    *   **Case A: Coarse Points Not Refined**
        *   Connect $v$ to the coarse grid via two triangles [S1].
        *   Connect each edge of the local fine grid structure to the two coarse edges "visible" from it [S1].
    *   **Case B: At Least One Coarse Point is Refined**
        *   "Promote" the border vertex $v$ to a border-edge segment [S1].
        *   Connect $v$ to the other refined grid point [S1].
        *   Treat this segment as an edge and apply the edge connection configurations (Step 2) [S1].

4.  **Merge Adjacent Grids:** If multiple fine grids adjoin, ensure that the stitch cells generated for shared boundaries are consistent and merged [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def stitch_2d_grids(coarse_boundary, fine_boundary, refinement_ratio_r):
    """
    Generates stitch cells between coarse and fine 2D AMR grids.
    
    Args:
        coarse_boundary: List of coarse grid boundary edges and vertices.
        fine_boundary: List of fine grid boundary edges and vertices.
        refinement_ratio_r: Integer refinement ratio (e.g., 2, 4).
        
    Returns:
        List of stitch cells (polygons defined by vertex indices).
    """
    stitch_cells = []
    
    # Helper to check if a coarse point is refined by another fine grid
    def is_refined(coarse_point):
        # [G] Implementation depends on specific AMR data structure.
        # Returns True if the coarse point is covered by a finer level grid.
        pass

    # 1. Process Fine Grid Edges
    for fine_edge in fine_boundary.edges:
        # Determine corresponding coarse connection target
        # [G] Logic to map fine edge to coarse edge/vertex based on geometry and r
        coarse_target = map_fine_to_coarse(fine_edge, coarse_boundary, refinement_ratio_r)
        
        if coarse_target.type == 'POINT':
            p = coarse_target.point
            if is_refined(p):
                # Connect to another fine edge to form rectilinear cell
                # [G] Find adjacent fine edge to pair with
                other_fine_edge = find_adjacent_fine_edge(fine_edge)
                stitch_cells.append(create_rectilinear_cell([fine_edge, other_fine_edge]))
            else:
                # Connect to coarse point
                stitch_cells.append(create_triangle_or_quad(fine_edge, p))
                
        elif coarse_target.type == 'EDGE':
            p1, p2 = coarse_target.edge.endpoints
            if is_refined(p1) and is_refined(p2):
                # Connect to another fine edge
                other_fine_edge = find_adjacent_fine_edge(fine_edge)
                stitch_cells.append(create_rectilinear_cell([fine_edge, other_fine_edge]))
            elif is_refined(p1) or is_refined(p2):
                # One endpoint refined: treat as edge case
                # [G] Use consistent tessellation for mixed refinement status
                stitch_cells.append(create_edge_case_cell(fine_edge, coarse_target.edge))
            else:
                # Neither refined: connect to coarse edge
                stitch_cells.append(create_quad(fine_edge, coarse_target.edge))

    # 2. Process Fine Grid Vertices
    for fine_vertex in fine_boundary.vertices:
        # Identify adjacent coarse points bounding the vertex
        coarse_points = get_adjacent_coarse_points(fine_vertex, coarse_boundary)
        
        refined_count = sum(1 for p in coarse_points if is_refined(p))
        
        if refined_count == 0:
            # Connect via two triangles to visible coarse edges
            visible_coarse_edges = get_visible_coarse_edges(fine_vertex, coarse_boundary)
            stitch_cells.append(create_triangles_for_vertex(fine_vertex, visible_coarse_edges))
        else:
            # Promote vertex to edge segment
            # Connect to the other refined grid point
            other_refined_point = get_other_refined_point(coarse_points)
            promoted_edge = create_edge(fine_vertex, other_refined_point)
            # Apply edge stitching logic
            # [G] Recursively or iteratively handle the promoted edge as an edge case
            stitch_cells.extend(stitch_edge(promoted_edge, coarse_boundary, refinement_ratio_r))
            
    return stitch_cells
```

**Parameters:**
*   Refinement ratio $r$: Integer specifying how many fine grid cells fit into a coarse grid cell. Typical values are 2 and 4 [S1].

**Pitfalls:**
*   **Quadrilateral Subdivision:** Do not subdivide quadrilaterals formed by stitching. Subdivision is not unique and causes cracks in 3D isosurface extraction [S1].
*   **Vertex Generation:** Do not generate new vertices. Use only existing grid vertices to maintain compatibility with dual grid structures [S1].
*   **Consistency:** Ensure that edge cases (where only one coarse endpoint is refined) use the same tessellation logic as other edge cases to avoid topological inconsistencies [S1].

## Subtopics

### [3D AMR Grid Stitching](10250-3d-amr-grid-stitching.md)
Generalizes the 2D index-based approach to 3D volumes, handling vertex, edge, and face interactions in three dimensions using stitch cells like pyramids and deformed prisms.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of boundary elements, as it uses an index-based approach that leverages the regular structure of the grid boundaries [S1].
*   **Memory Complexity:** Proportional to the number of stitch cells generated, which is bounded by the perimeter of the fine grid [S1].
*   **Strengths:** Efficient, avoids the computational cost and complexity of constrained Delaunay triangulation, and guarantees crack-free isosurfaces in 3D by preserving quadrilateral boundary faces [S1].
*   **Limitations:** Specific to AMR grids with integer refinement ratios; requires careful handling of mixed refinement states at boundaries [S1].

## Background
The method was introduced in the context of extracting crack-free isosurfaces from Adaptive Mesh Refinement (AMR) data [S1]. It addresses the problem of gaps between dual grids of different hierarchy levels, which would otherwise cause discontinuities in the extracted surface [S1]. The approach was chosen over constrained Delaunay triangulation because the latter requires subdividing quadrilateral boundary faces, which is incompatible with Marching Cubes extraction in 3D [S1].

## Sources
[S1] Extraction of Crack-free Isosurfaces from Adaptive Mesh Refinement Data, https://doi.org/10.2312/vissym/vissym01/025-034, fetched 2026-10-26
