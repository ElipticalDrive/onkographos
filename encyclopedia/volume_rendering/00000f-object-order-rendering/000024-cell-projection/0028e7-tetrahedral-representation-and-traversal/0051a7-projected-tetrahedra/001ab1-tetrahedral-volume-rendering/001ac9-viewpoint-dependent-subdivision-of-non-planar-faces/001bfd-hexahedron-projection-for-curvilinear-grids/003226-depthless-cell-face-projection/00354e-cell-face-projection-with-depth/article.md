# Cell Face Projection with Depth


## Overview
Cell Face Projection with Depth is a projection-based direct volume rendering method for curvilinear grids that addresses the lack of depth consideration in the depthless method [S2, unofficial]. It maps sample locations to screen space and calculates depth, average color, and opacity for vertices, allowing opacity to be taken into consideration more accurately than its sibling [Depthless Cell Face Projection](6692-depthless-cell-face-projection.md) [S2, unofficial].

## How it works
**Inputs:** Curvilinear grid cell data (vertex positions, data values); current viewpoint.
**Outputs:** Screen-space image with color and opacity.

**Data Structures:**
*   **Cell Data Structure:** Stores precomputed vertex attributes for the current viewpoint.
    *   Fields: `vertex_count`, `vertices[]`.
    *   `vertices[i]` contains:
        *   `screen_pos`: 2D screen coordinates.
        *   `depth`: Average depth through the cell at this projected location.
        *   `color`: Average color.
        *   `opacity`: Average opacity.
        *   `is_on_convex_hull`: Boolean flag.
        *   `is_coalesced`: Boolean flag (true if two vertices map to the same screen location).

**Steps:**

1.  **Identify Vertex Addresses and Locations:** Determine the vertex addresses and spatial locations for each cell [S1].
2.  **Map to Screen Space:** Map all sample locations (vertices) of the cell to screen space based on the current viewpoint [S2, unofficial].
3.  **Identify Convex Hull:** For each cell, identify the vertices that lie on the convex hull of the projected cell [S2, unofficial].
4.  **Process Convex Hull Vertices:**
    *   **Check Coalescence:** Determine if convex hull vertices are coalesced (i.e., two distinct vertices map to the same screen space location) [S2, unofficial].
    *   **If Not Coalesced:** The depth through the cell at that point is zero. The vertex does not contribute color or opacity [S2, unofficial].
    *   **If Coalesced (e.g., looking straight on at a regular cell):**
        *   Record a depth for that vertex [S2, unofficial].
        *   Calculate the data value as the average of the two coalesced vertices' data values [S2, unofficial].
        *   Map this average data value to color and opacity [S2, unofficial].
    *   **Depth Recording:** If a vertex appears on the convex hull from certain angles, record its depth; if it appears on the opposite face, take the average of the two vertex depths [S1].
5.  **Process Interior Vertices:**
    *   For interior vertices, locate the point at the same projected location on the opposite cell exterior [S2, unofficial].
    *   Find data values for that point by interpolation [S2, unofficial].
    *   Calculate depth and average color and opacity for these interior vertices [S2, unofficial].
    *   Store this information in the vertex data structure [S2, unofficial].
6.  **Store Precomputed Data:** This information (depth, color, opacity) is calculated whenever the viewpoint is changed and stored with the cell data structure [S2, unofficial].
7.  **Rendering:**
    *   Draw each face once for each cell (twice overall, except on the boundary) [S2, unofficial].
    *   Scale color and opacity based on the depth of the vertex during rendering [S1].
    *   Scaling intensity can be used to take care of intensity problems this may cause [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_cell(cell, viewpoint):
    """
    Precompute depth, color, and opacity for a cell given a viewpoint.
    Returns a CellData structure.
    """
    # Step 1 & 2: Map vertices to screen space
    screen_vertices = []
    for v in cell.vertices:
        pos_3d = v.position
        screen_pos = project_to_screen(pos_3d, viewpoint)
        data_val = v.data_value
        screen_vertices.append({
            'original_index': v.index,
            'screen_pos': screen_pos,
            'data_value': data_val,
            'pos_3d': pos_3d
        })

    # Step 3: Identify convex hull vertices in screen space
    hull_indices = compute_convex_hull_indices(screen_vertices)
    
    # Step 4 & 5: Calculate attributes for each vertex
    vertex_data = []
    
    for i, v in enumerate(screen_vertices):
        is_hull = i in hull_indices
        depth = 0.0
        color = 0.0
        opacity = 0.0
        
        if is_hull:
            # Check for coalescence with other hull vertices
            coalesced_pair = find_coalesced_pair(v, screen_vertices, hull_indices)
            
            if coalesced_pair is None:
                # Not coalesced: depth is zero, no contribution
                depth = 0.0
                color = 0.0
                opacity = 0.0
            else:
                # Coalesced: average data values, record depth
                other_v = coalesced_pair
                avg_data = (v['data_value'] + other_v['data_value']) / 2.0
                depth = calculate_depth(v['pos_3d'], other_v['pos_3d'], viewpoint) # [G] Depth calculation logic inferred from context
                color, opacity = map_data_to_color_opacity(avg_data)
        else:
            # Interior vertex
            # Locate point on opposite cell exterior at same projected location
            opposite_point_3d = find_opposite_exterior_point(v['screen_pos'], cell, viewpoint) # [G] Interpolation logic
            opposite_data = interpolate_data(opposite_point_3d, cell)
            
            depth = calculate_depth(v['pos_3d'], opposite_point_3d, viewpoint) # [G] Depth calculation logic inferred from context
            color, opacity = map_data_to_color_opacity(opposite_data)
            
        vertex_data.append({
            'screen_pos': v['screen_pos'],
            'depth': depth,
            'color': color,
            'opacity': opacity,
            'is_hull': is_hull
        })
        
    return CellData(vertices=vertex_data, hull_indices=hull_indices)

def render_cell(cell_data):
    """
    Render a preprocessed cell.
    """
    # Step 7: Draw faces
    # Draw each face once for each cell (twice overall, except on boundary)
    faces = get_faces_from_hull_indices(cell_data.hull_indices)
    
    for face in faces:
        vertices_in_face = [cell_data.vertices[i] for i in face]
        
        # Scale color and opacity based on depth
        for v in vertices_in_face:
            scaled_color = scale_by_depth(v['color'], v['depth'])
            scaled_opacity = scale_by_depth(v['opacity'], v['depth'])
            
        # Draw polygon with scaled attributes
        draw_polygon(vertices_in_face, scaled_color, scaled_opacity)
```

**Parameters:**
*   Viewpoint: Changes the screen projection and precomputed values.
*   Scaling intensity: A parameter to adjust for intensity problems caused by double drawing [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Coalesced Vertices:** If convex hull vertices are not coalesced, the depth is zero and the vertex contributes nothing [S2, unofficial].
*   **Artifacts:** Produces noticeable artifacts at angles close to 90 degrees for regular cells [S2, unofficial]. These artifacts tend to delineate cell boundaries and probably would not be misinterpreted as data information [S1][S2, unofficial].
*   **Performance:** Much slower for new orientations than the depthless method because precomputation must be recalculated whenever the viewpoint is changed [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Slower than the depthless method for new orientations due to the need to recalculate depth, color, and opacity for all vertices whenever the viewpoint changes [S2, unofficial].
*   **Memory Complexity:** Requires storing precomputed vertex attributes (depth, color, opacity) with the cell data structure for each viewpoint [S2, unofficial].
*   **Accuracy:** Takes opacity into consideration more accurately than the depthless method, providing a more "realistic" rendering assuming a physical, colored medium [S2, unofficial].
*   **Strengths:** More realistic appearance; accounts for depth and opacity.
*   **Limitations:** Noticeable artifacts at near-90-degree angles for regular cells; slower performance for dynamic viewpoints.
*   **When to use:** When realistic opacity handling is prioritized over performance and viewpoint stability.
*   **When not to use:** When rapid exploration with frequent viewpoint changes is required, or when artifacts at specific angles are unacceptable.

## Background
*   **Origin:** Proposed in "Rapid Exploration of Curvilinear Grids Using Direct Volume Rendering" (1993) by NASA researchers [S1][S2, unofficial].
*   **Development:** Developed as an improvement over the "Depthless Cell Face Projection" method to address the lack of depth consideration in compositing [S2, unofficial].
*   **Usage:** Used for direct volume rendering of curvilinear grids, particularly in scientific visualization contexts where realistic opacity is desired [S2, unofficial].

## Sources
[S1] Rapid Exploration of Curvilinear Grids Using Direct Volume Rendering, https://ntrs.nasa.gov/api/citations/19930015755/downloads/19930015755.pdf, fetched 2026-10-24
[S2] NASA Technical Reports Server (NTRS) 19930015755: Rapid exploration of curvilinear grids using direct volume rendering, https://archive.org/details/NASA_NTRS_Archive_19930015755, fetched 2026-10-26 (unofficial)
