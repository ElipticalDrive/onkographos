# Incoherent Projection


## Overview
Incoherent projection is a cell projection method for general convex hexahedra that renders the 2D projection of each cell as an arrangement of polygons by analyzing each cell individually in screen space, thereby avoiding case analysis for the high number of projection topologies [S2, unofficial]. It builds on [Cell projection](26-cell-projection.md) by extending the coherent projection technique for rectilinear grids to irregular hexahedra, where different cells may fall into different nondegenerate or degenerate projection topologies [S1]. Unlike coherent projection, which assumes all cells share the same topology, incoherent projection handles the complexity of general grids by processing each cell's projection geometry independently [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A single convex hexahedron defined by its 8 vertices in 3D space, and a projection matrix (orthogonal or perspective) to convert them to screen space $(x, y, z)$.
*   **Output:** A set of 2D polygons in screen space that comprise the arrangement of the cell's projection, along with their associated $z$-depths for sorting [S2, unofficial].

**Data Structures:**
The algorithm maintains three primary data structures during the sweep-line process [S2, unofficial]:
1.  **Event Queue:** A priority queue of vertex events. The "minimum" element is the unprocessed vertex with the minimum $y$-value.
2.  **Active Edges:** An $x$-sorted list of active edges. An edge is active if it connects a processed vertex to an unprocessed vertex.
3.  **Current Boundary:** A polygon represented as an edge list, surrounding the processed vertices and edges.

**Algorithm Steps:**
The method follows a standard sweep-line pattern [S2, unofficial].

1.  **Initialization:**
    *   Project the hexahedron vertices to screen space $(x, y, z)$ [S2, unofficial].
    *   Initialize the event queue with the original hexahedron vertices, sorted by $y$-value [S2, unofficial].
    *   Initialize the active-edge list and current boundary as empty [S2, unofficial].

2.  **Sweep Loop:**
    While the event queue is not empty, perform the following [S2, unofficial]:

    1.  Retrieve the next vertex from the event queue:
        ```python
        nextVertex = eventQueue.getMin()
        currentY = nextVertex.y
        ```
    2.  Calculate the current $x$-position for all active edges at `currentY`:
        ```python
        for edge in ActiveEdges:
            edge.currentX = edge.xAtY(currentY)
        ```
    3.  Check if the active edges remain sorted by `currentX`. If they are still sorted:
        *   Remove edges that end at `nextVertex` from the active list. Call this set `remEdges` [S2, unofficial].
        *   Determine the $z$-distance of `nextVertex` from the face into which it projects [S2, unofficial].
        *   Update the current boundary with `remEdges` to create polygons of the projection (see Boundary Update below) [S2, unofficial].
        *   Insert edges that begin at `nextVertex` into the active-edge list, ensuring they are $x$-ordered if `currentY` increases slightly [S2, unofficial].
        *   Remove `nextVertex` from the event queue [S2, unofficial].
    4.  If the active edges are **not** sorted (indicating an intersection of edge projections):
        *   Find the $(x, y)$ coordinates where two previously adjacent active edges intersect [S2, unofficial].
        *   Create a new "intersection vertex" (`newVertex`) at this location. Note that these vertices do not correspond to original hexahedron vertices but are produced by the intersection of edge projections [S1].
        *   Chop off the two intersecting edges so they end at `newVertex`. The cut-off parts become new edges starting at `newVertex` [S2, unofficial].
        *   Insert these new edges into the active-edge list in the correct $x$-order [S2, unofficial].
        *   Insert `newVertex` into the event queue (it will be the minimum) [S2, unofficial].

**Boundary Update Logic:**
When `remEdges` are removed from the active list, the current boundary is updated to form polygons [S2, unofficial]:
1.  Insert the first removed edge and its reversal into the current boundary edge list. This forms a planar, non-convex, non-simple "needle" polygon (no edges cross) [S2, unofficial].
2.  If a second edge is removed, insert it similarly. This completes a counterclockwise polygon, which is then "spliced out" (extracted as a final output polygon) from the current boundary [S2, unofficial].
3.  The last vertex of the cell typically has a third removed edge, which creates a second polygon to be spliced out [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def incoherent_projection(hexahedron_vertices, projection_matrix):
    # 1. Project to screen space
    screen_vertices = [project(v, projection_matrix) for v in hexahedron_vertices]
    
    # 2. Initialize data structures
    # Event Queue: Priority queue sorted by y-coordinate
    event_queue = PriorityQueue()
    for v in screen_vertices:
        event_queue.push(v)
        
    # Active Edges: List of edges (v_start, v_end), sorted by x at current sweep line
    active_edges = []
    
    # Current Boundary: List of edges forming the boundary of processed area
    current_boundary = []
    
    output_polygons = []
    
    while not event_queue.is_empty():
        next_vertex = event_queue.get_min()
        current_y = next_vertex.y
        
        # Calculate current x for all active edges at current_y
        for edge in active_edges:
            edge.current_x = edge.x_at_y(current_y)
            
        # Check if active edges are still sorted by current_x
        if is_sorted(active_edges, key=lambda e: e.current_x):
            # Case: No intersection of edge projections
            rem_edges = []
            for edge in active_edges:
                if edge.v_end == next_vertex:
                    rem_edges.append(edge)
            
            # Remove rem_edges from active list
            for edge in rem_edges:
                active_edges.remove(edge)
                
            # Determine z-distance for sorting/compositing
            z_depth = next_vertex.z
            
            # Update current boundary and extract polygons
            # This involves splicing out counterclockwise polygons formed by rem_edges
            polygons = update_boundary(current_boundary, rem_edges)
            output_polygons.extend(polygons)
            
            # Insert new edges starting at next_vertex
            new_edges = get_edges_starting_at(next_vertex, hexahedron_edges)
            for edge in new_edges:
                insert_sorted(active_edges, edge)
                
        else:
            # Case: Edge projections intersect
            # Find intersection of two adjacent active edges
            edge1, edge2 = find_intersecting_adjacent_edges(active_edges)
            intersection_point = edge1.intersect(edge2)
            
            # Create new intersection vertex
            new_vertex = Vertex(intersection_point.x, intersection_point.y, 
                                interpolate_z(edge1, edge2, intersection_point))
            
            # Chop off intersecting edges
            chop_edge(edge1, new_vertex)
            chop_edge(edge2, new_vertex)
            
            # Create new edges starting from new_vertex
            new_edge1 = Edge(new_vertex, edge1.v_end)
            new_edge2 = Edge(new_vertex, edge2.v_end)
            
            # Insert new edges into active list in correct x-order
            insert_sorted(active_edges, new_edge1)
            insert_sorted(active_edges, new_edge2)
            
            # Insert new vertex into event queue
            event_queue.push(new_vertex)
            
    return output_polygons

def update_boundary(boundary, rem_edges):
    polygons = []
    # Insert first removed edge and its reversal to form a needle
    if len(rem_edges) >= 1:
        e1 = rem_edges[0]
        boundary.append(e1)
        boundary.append(e1.reverse())
        
    # If second edge is removed, it completes a counterclockwise polygon
    if len(rem_edges) >= 2:
        e2 = rem_edges[1]
        boundary.append(e2)
        boundary.append(e2.reverse())
        # Splice out the completed counterclockwise polygon
        poly = extract_counterclockwise_polygon(boundary)
        if poly:
            polygons.append(poly)
            
    # Handle subsequent edges similarly...
    return polygons
```

**Parameters and Assumptions:**
*   **Assumption 1:** No two adjacent edges have zero length [S2, unofficial].
*   **Assumption 2:** No two adjacent faces have zero area [S1][S2, unofficial].
*   These assumptions allow tetrahedra and pentahedra to be represented as degenerate hexahedra [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Projection Degeneracies:** Occur when two projection vertices have the same $x$-value or $y$-value. These are handled by assuming a slight rotation of the screen space that does not change any nondegenerate topology [S2, unofficial].
*   **Geometric Degeneracies:** Occur when two points coincide in 3-space (e.g., zero-length edges). The solution perturbs the hexahedron to give the edge some length while maintaining convexity.
    *   If the edge connects two faces with positive area, their planes intersect in a well-defined line that determines the direction to "pull apart" the coinciding vertices [S2, unofficial].
    *   If the edge is adjacent to one face of zero area, a more complex perturbation is required, leveraging the fact that coinciding vertices and the face of positive area form a triangle [S2, unofficial].
*   **Intersection Vertices:** Vertices in the projection that do not correspond to original hexahedron vertices but are produced by the intersection of edge projections must be handled dynamically during the sweep [S1].

## Operating characteristics
*   **Complexity:** The method is described as the "most careful" and "most expensive" of the projection methods presented [S2, unofficial]. The sweep-line algorithm for a hexahedron (8 vertices, 12 edges) is constant time per cell, but the overhead of dynamic intersection detection and boundary management is significant compared to coherent projection [S2, unofficial].
*   **Strengths:** It handles general convex hexahedra without requiring case analysis for the high number of projection topologies (which is significantly higher for irregular hexahedra than for rectilinear grids) [S1][S2, unofficial]. It also handles non-orthogonal perspective projections as easily as orthogonal ones [S1].
*   **Limitations:** High computational cost per cell due to the individual analysis of each cell's projection topology [S2, unofficial].
*   **When to use:** When rendering general curvilinear grids with irregular hexahedral cells where coherent projection's assumption of uniform topology fails [S1][S2, unofficial].

## Background
Incoherent projection was introduced in the context of rapid exploration of curvilinear grids using direct volume rendering [S1]. It builds upon the "coherent projection" technique for rectilinear grids [WVG91, cited in S2] and extends it to general convex hexahedra [S2, unofficial]. The method was developed to address the difficulty of handling the significantly higher number of nondegenerate and degenerate projection topologies present in irregular hexahedral grids, where different cells do not fall into the same case from any given viewpoint [S1].

## Sources
[S1] Rapid Exploration of Curvilinear Grids Using Direct Volume Rendering, https://ntrs.nasa.gov/api/citations/19930015755/downloads/19930015755.pdf, fetched 2026-10-24
[S2] NASA Technical Reports Server (NTRS) 19930015755: Rapid exploration of curvilinear grids using direct volume rendering, https://archive.org/details/NASA_NTRS_Archive_19930015755, fetched 2026-10-26 (unofficial)
