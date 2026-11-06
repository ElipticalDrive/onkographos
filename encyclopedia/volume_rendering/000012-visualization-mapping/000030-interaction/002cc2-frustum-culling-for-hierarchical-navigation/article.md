# Frustum Culling for Hierarchical Navigation

Also called: Context-aware Frustum Culling, Focus and Context with Interactive Filtering, Frustum Culling in Navigation, Interactive Filtering

## Overview
Frustum Culling for Hierarchical Navigation is an interaction technique that reduces visual clutter and aids data exploration by selectively rendering nodes based on their position relative to the 3D viewing frustum [S1]. It sets itself apart from standard geometric culling by simultaneously managing the visual state of coordinated 2D views (overview maps and treemaps), ensuring that culling in 3D triggers specific contextual updates in 2D rather than simple invisibility [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The current 3D viewing frustum (defined by camera position, orientation, and clipping planes), the current Level of Detail (LOD) hierarchy of nodes, and the current state of the 2D coordinated views (Overview Map and Treemap) [S1].
*   **Output:** A rendered 3D volume with only visible nodes, and updated 2D views where culled nodes are visually distinguished but not selectable [S1].

**Data Structures:**
*   **Node Hierarchy:** A tree structure where each node at the current LOD has a spatial location (point) and a corresponding representation in the 2D views (point in Overview, rectangle in Treemap) [S1].
*   **Viewing Frustum:** A set of six planes defining the visible volume in 3D space [S1].
*   **Visual State Flags:** Per-node flags indicating visibility and selection eligibility [S1].

**Steps:**
1.  **Frustum Intersection Test:** For each node in the current LOD, determine if its spatial location lies inside the 3D viewing frustum [S1].
2.  **Culling Decision:** Identify nodes that are completely outside the viewing frustum [S1].
3.  **3D Rendering Update:**
    *   For nodes inside the frustum: Render them normally in the 3D rendering window [S1].
    *   For nodes outside the frustum: Do not draw their primary visual markers (e.g., squares) in the 3D rendering window [S1].
4.  **Selection State Update:**
    *   For nodes inside the frustum: Keep them active for selection [S1].
    *   For nodes outside the frustum: Deactivate them from selection [S1].
5.  **Overview Map Update:**
    *   For nodes inside the frustum: Draw their corresponding point markers [S1].
    *   For nodes outside the frustum: Do not draw their point markers (squares) [S1]. However, draw line segments passing through their point locations to indicate the cut and provide context [S1].
6.  **Treemap Update:**
    *   For nodes inside the frustum: Render their rectangle regions normally [S1].
    *   For nodes outside the frustum: Fill their corresponding rectangle regions with a specific context color (grey) [S1]. The rectangles are still drawn to provide structural context [S1].

**Parameters:**
*   **Context Color:** Grey is used to fill the rectangle regions of culled nodes in the treemap [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_hierarchical_navigation(nodes, frustum, overview_map, treemap):
    """
    Updates the 3D view, Overview Map, and Treemap based on frustum culling.
    
    Args:
        nodes: List of Node objects, each with:
            - position: (x, y, z) tuple
            - id: Unique identifier
            - overview_point: (x, y) tuple for 2D overview
            - treemap_rect: (x, y, w, h) tuple for 2D treemap
        frustum: Frustum object with method is_inside(point)
        overview_map: 2D canvas object with methods draw_square(point), draw_line_segment(point)
        treemap: 2D canvas object with method fill_rectangle(rect, color)
    """
    for node in nodes:
        is_visible = frustum.is_inside(node.position)
        
        if is_visible:
            # 3D View: Render node normally
            render_3d_node(node)
            
            # Overview Map: Draw selection marker
            overview_map.draw_square(node.overview_point)
            
            # Treemap: Render normally (assumed default behavior for visible nodes)
            # Note: Specific fill color for visible nodes is not detailed in sources,
            # assuming default or previous state.
            treemap.fill_rectangle(node.treemap_rect, color="default")
        else:
            # 3D View: Cull node (do not draw square)
            # No action required for 3D rendering if default is "do not draw"
            
            # Overview Map: Deactivate selection (no square), but draw context line
            # overview_map.draw_square(node.overview_point) # Explicitly NOT called
            overview_map.draw_line_segment(node.overview_point)
            
            # Treemap: Fill with context color (grey)
            treemap.fill_rectangle(node.treemap_rect, color="grey")
```

**Edge Cases and Pitfalls:**
*   **Precision:** Frustum intersection tests must handle floating-point precision issues near frustum boundaries to avoid flickering or incorrect culling [S1].
*   **Context Clarity:** The line segments in the overview map and grey rectangles in the treemap must be visually distinct from active nodes to avoid confusion, but not so dominant as to obscure the structure [S1].
*   **LOD Consistency:** The culling decision is based on the current LOD. If the LOD changes, the culling state must be recalculated [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of nodes in the current LOD, as each node requires a frustum intersection test and state updates [S1].
*   **Memory Complexity:** Minimal additional memory, primarily for storing the visual state flags and the frustum definition [S1].
*   **Performance:** Improves rendering performance by reducing the number of primitives drawn in the 3D view and coordinated 2D views [S1].
*   **Strengths:** Reduces visual clutter, aids orientation by providing context for culled data, and prevents selection of irrelevant data [S1].
*   **Limitations:** Relies on the accuracy of the frustum culling algorithm; context lines and grey rectangles may become cluttered in very dense hierarchies [S1].
*   **When to Use:** When navigating large hierarchical datasets where visual clutter hinders exploration and context is needed for orientation [S1].
*   **When Not to Use:** When the user requires full visibility of all nodes regardless of position, or when the overhead of maintaining coordinated view states is prohibitive [S1].

## Background
This technique was introduced in the "Hierarchical Navigation Interface" paper, which leverages multiple coordinated views to assist in the exploration of large and high-dimensional datasets [S1]. It is part of a broader family of focus and context techniques used in visualization to show portions of data in detail while keeping the rest at a lower resolution for orientation [S1]. The specific implementation described uses interactive filtering to manage the visual state of nodes across 3D and 2D views during navigation [S1].

## Sources
[S1] Hierarchical Navigation Interface: Leveraging Multiple Coordinated Views, https://academicweb.nd.edu/~cwang11/papers/iv05-hni.pdf, fetched 2026-10-30
