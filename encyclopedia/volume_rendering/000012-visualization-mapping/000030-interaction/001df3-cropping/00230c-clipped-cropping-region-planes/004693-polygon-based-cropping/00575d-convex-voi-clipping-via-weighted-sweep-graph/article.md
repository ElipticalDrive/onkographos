# Convex VOI Clipping via Weighted Sweep Graph

Also called: Clipper, WSG-based clipping

## Overview
This method is a CPU-based clipping algorithm that intersects a sequence of parallel sampling planes against a user-defined convex Volume of Interest (VOI) to reduce the computational load on the GPU [S1]. It solves the problem of unnecessary volume rendering computation in empty or irrelevant regions by transmitting only the clipped, valid portions of sampling planes to the GPU [S1]. It sets itself apart from general convex clipping algorithms by exploiting the coherence between parallel planes and using a Weighted Sweep Graph (WSG) to efficiently manage topological changes (vertex events) during the sweep, avoiding the recomputation of edge lists from scratch for every plane [S1].

## How it works
**Inputs:**
1.  A convex VOI defined by a set of cutting planes. Each cutting plane is defined by a normal vector $\mathbf{n}$ and a point $\mathbf{p}$ on the plane.
2.  A sequence of parallel sampling planes. Each sampling plane is defined by a normal vector $\mathbf{n}_{sample}$ (parallel to the viewing direction) and a distance $d_i$ from the VOI center (or a reference origin).

**Outputs:**
A sequence of convex polygons, one for each sampling plane that intersects the VOI. Each polygon is defined by an ordered list of vertices lying on the edges of the VOI.

**Data Structures:**
*   **VOI Representation:** The convex VOI is implicitly defined by the intersection of half-spaces created by the cutting planes. The geometry is tracked via its edges and vertices.
*   **Weighted Sweep Graph (WSG):** A graph structure that encodes the topological structure of the convex volume. It maintains:
    *   The current list of edges intersected by the sweep plane, ordered to form the convex polygon.
    *   Vertex events, which indicate when the sweep plane passes through a vertex of the VOI.
    *   Connections between vertices and edges to facilitate updates.

**Algorithm Steps:**

1.  **VOI Definition:**
    The user defines the convex VOI starting from a basic convex shape (e.g., a cube). Cutting planes are added to slice the volume. Each cutting plane is specified by:
    *   A normal vector $\mathbf{n}$, controlled by dragging a line endpoint from the VOI center.
    *   A reference point, controlled by a slider specifying the distance from the VOI center.
    The resulting VOI is the intersection of the initial shape and all added half-spaces [S1].

2.  **Initialization:**
    Initialize the WSG with the topology of the initial convex shape (e.g., the cube). The WSG stores the edges and their connectivity. Identify the initial set of edges that intersect the first sampling plane (or the plane closest to the VOI center) [S1].

3.  **Sweep Preparation:**
    Sort the sampling planes by their distance from the VOI center along the viewing direction. Sort the vertices of the VOI by their projection onto the viewing direction to identify the order of vertex events (sweep events) [S1].

4.  **Sweep Loop:**
    For each sampling plane $i$ in the sorted order:
    a.  **Check for Vertex Events:** Determine if any vertex of the VOI lies between the current plane $i$ and the previous plane $i-1$. This is done by comparing the signed distance of the vertex to the plane normal against the plane positions [S1].
    b.  **Update Edge List (if Vertex Event):** If a vertex $v$ is encountered:
        i.  Identify the edges incident to $v$ in the WSG.
        ii. Remove edges that are no longer intersected by the sweep plane (those "behind" the vertex relative to the sweep direction).
        iii. Add edges that become intersected (those "in front" of the vertex).
        iv. Ensure the edge list remains ordered to form a convex polygon. The WSG maintains this order [S1].
    c.  **Compute Polygon Vertices:**
        i.  For each edge $e$ in the current ordered edge list, compute the intersection point $P$ between the edge $e$ and the current sampling plane $i$.
        ii. The edge $e$ is defined by two vertices $A$ and $B$. The intersection point $P$ is calculated as:
        $$ P = A + \frac{d_i - \mathbf{n}_{sample} \cdot A}{\mathbf{n}_{sample} \cdot (B - A)} (B - A) $$
        where $d_i$ is the signed distance of the plane from the origin, and $\mathbf{n}_{sample}$ is the plane normal. Note: The denominator is the dot product of the plane normal and the edge direction vector. If the denominator is zero, the edge is parallel to the plane and does not intersect (or lies on it) [S1].
        iii. Collect all intersection points $P$. These points form the vertices of the convex polygon for plane $i$.
    d.  **Output:** Send the resulting polygon (list of vertices) to the GPU for volume rendering [S1].

5.  **Termination:**
    Stop when all sampling planes have been processed.

**Parameters:**
*   **VOI Center:** The reference point for user interaction (normal vector direction and plane distance) [S1].
*   **Plane Normal:** The viewing direction, which is parallel for all sampling planes [S1].

**Edge Cases and Pitfalls:**
*   **Parallel Edges:** If an edge is parallel to the sampling plane, the intersection formula yields a division by zero. Such edges do not contribute a unique intersection point and should be handled by checking if the edge lies on the plane (in which case it might be part of the polygon boundary) or is strictly outside [S1].
*   **Vertex Coincidence:** If a vertex lies exactly on a sampling plane, it is a vertex event. The edge list update must be handled carefully to ensure the polygon remains closed and convex. The WSG structure is designed to detect these events [S1].
*   **Precision:** Floating-point precision issues may arise when determining if a vertex is between two planes or if an intersection point lies exactly on an edge. Robust geometric predicates are recommended [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convex_voi_clipping_via_wsg(voi_edges, voi_vertices, cutting_planes, sampling_planes):
    """
    voi_edges: List of edges, each edge is a tuple (vertex_index_A, vertex_index_B)
    voi_vertices: List of vertices, each vertex is a 3D point (x, y, z)
    cutting_planes: List of cutting planes, each plane is (normal, point_on_plane)
    sampling_planes: List of sampling planes, each plane is (normal, distance_from_origin)
    Returns: List of polygons, each polygon is a list of 3D points
    """
    
    # Step 1: Define VOI implicitly via cutting planes
    # The VOI is the intersection of half-spaces defined by cutting_planes
    # For simplicity, we assume voi_edges and voi_vertices are pre-computed
    # based on the cutting_planes. In practice, this requires a constructive
    # solid geometry (CSG) or incremental clipping step.
    
    # Step 2: Initialize WSG
    # WSG maintains the current list of edges intersected by the sweep plane
    # and their order.
    current_edges = get_initial_intersected_edges(voi_edges, voi_vertices, sampling_planes[0])
    # Sort current_edges to form a convex polygon
    current_edges = sort_edges_to_polygon(current_edges, voi_vertices)
    
    # Precompute vertex distances along the sweep direction (sampling plane normal)
    sweep_normal = sampling_planes[0][0]
    vertex_distances = [dot(v, sweep_normal) for v in voi_vertices]
    
    # Sort sampling planes by distance
    sorted_plane_indices = sorted(range(len(sampling_planes)), 
                                  key=lambda i: sampling_planes[i][1])
    
    polygons = []
    
    for idx in sorted_plane_indices:
        plane_normal, plane_dist = sampling_planes[idx]
        
        # Step 4a: Check for vertex events
        # Find vertices that lie between the previous plane and the current plane
        # For the first plane, there is no previous plane, so no events
        if idx != sorted_plane_indices[0]:
            prev_plane_dist = sampling_planes[sorted_plane_indices[sorted_plane_indices.index(idx) - 1]][1]
            # Determine which vertices are crossed
            # A vertex is crossed if its distance is between prev_plane_dist and plane_dist
            # Note: This assumes the sweep direction is consistent with the normal
            crossed_vertices = []
            for v_idx, v_dist in enumerate(vertex_distances):
                if (prev_plane_dist < v_dist < plane_dist) or (plane_dist < v_dist < prev_plane_dist):
                    crossed_vertices.append(v_idx)
            
            # Step 4b: Update Edge List for each crossed vertex
            for v_idx in crossed_vertices:
                current_edges = update_edge_list_for_vertex(current_edges, v_idx, voi_edges, voi_vertices, plane_normal)
                # Re-sort edges to maintain polygon order
                current_edges = sort_edges_to_polygon(current_edges, voi_vertices)
        
        # Step 4c: Compute Polygon Vertices
        polygon_vertices = []
        for edge in current_edges:
            v_a_idx, v_b_idx = edge
            v_a = voi_vertices[v_a_idx]
            v_b = voi_vertices[v_b_idx]
            
            # Compute intersection of edge (v_a, v_b) with plane
            # Plane equation: dot(x, plane_normal) = plane_dist
            # Edge parametric: x(t) = v_a + t * (v_b - v_a)
            # dot(v_a + t * (v_b - v_a), plane_normal) = plane_dist
            # t = (plane_dist - dot(v_a, plane_normal)) / dot(v_b - v_a, plane_normal)
            
            denom = dot(v_b - v_a, plane_normal)
            if abs(denom) < 1e-9:
                # Edge is parallel to the plane. 
                # If the edge lies on the plane, it might be part of the polygon.
                # For simplicity, skip if not exactly on plane.
                if abs(dot(v_a, plane_normal) - plane_dist) < 1e-9:
                    # Edge is on the plane. Add both endpoints? 
                    # This is a degenerate case. Usually, vertices are added via adjacent edges.
                    pass
                continue
            
            t = (plane_dist - dot(v_a, plane_normal)) / denom
            
            # Clamp t to [0, 1] to ensure intersection is on the segment
            if 0 <= t <= 1:
                intersection_point = v_a + t * (v_b - v_a)
                polygon_vertices.append(intersection_point)
        
        # Step 4d: Output
        # polygon_vertices should be in order. If not, sort them.
        # The WSG update should maintain order, but numerical errors might occur.
        polygon_vertices = sort_polygon_vertices(polygon_vertices)
        polygons.append(polygon_vertices)
        
    return polygons

def get_initial_intersected_edges(voi_edges, voi_vertices, plane):
    """Find edges intersecting the first plane."""
    plane_normal, plane_dist = plane
    intersected = []
    for edge in voi_edges:
        v_a_idx, v_b_idx = edge
        v_a = voi_vertices[v_a_idx]
        v_b = voi_vertices[v_b_idx]
        
        denom = dot(v_b - v_a, plane_normal)
        if abs(denom) < 1e-9:
            continue
        
        t = (plane_dist - dot(v_a, plane_normal)) / denom
        if 0 <= t <= 1:
            intersected.append(edge)
    return intersected

def update_edge_list_for_vertex(current_edges, v_idx, voi_edges, voi_vertices, plane_normal):
    """
    Update the current edge list when a vertex v_idx is crossed.
    Remove edges incident to v_idx that are no longer intersected,
    and add edges that become intersected.
    """
    # Find edges incident to v_idx
    incident_edges = [e for e in voi_edges if v_idx in e]
    
    # For each incident edge, determine if it should be in the current list
    # This depends on the sweep direction and the position of the vertex relative to the plane
    # The WSG structure helps identify which edges to add/remove.
    # Simplified logic:
    # Edges that were intersecting before the vertex and are now "behind" the vertex are removed.
    # Edges that were not intersecting before and are now "in front" are added.
    
    new_edges = []
    for edge in current_edges:
        if v_idx not in edge:
            # Edge not incident to the vertex, keep it
            new_edges.append(edge)
        else:
            # Edge incident to the vertex. Check if it should be kept.
            # This requires knowing the sweep direction and the vertex's position.
            # For a convex VOI, as the plane sweeps past a vertex, the set of intersected edges
            # changes by removing the edges that are "behind" the vertex and adding those "in front".
            # This logic is complex and depends on the specific topology.
            # The WSG is designed to handle this. Here, we assume the WSG provides the correct
            # set of edges to keep/add/remove.
            # For the purpose of this pseudocode, we assume the WSG update is a black box
            # that returns the new set of edges.
            pass
            
    # In a real implementation, the WSG would provide the new edge list directly.
    # Here, we simulate the update by recomputing the intersection for the current plane
    # using the new set of edges determined by the WSG.
    # Since we don't have the full WSG implementation, we assume the edge list is updated
    # by the WSG logic described in the source.
    
    # Re-compute intersection for the current plane with the updated edge list
    # This is a simplification. The WSG should maintain the edge list incrementally.
    return current_edges # Placeholder for WSG update logic

def sort_edges_to_polygon(edges, vertices):
    """Sort edges to form a convex polygon."""
    if not edges:
        return []
    
    # Build adjacency list
    adj = {}
    for e in edges:
        v1, v2 = e
        if v1 not in adj:
            adj[v1] = []
        if v2 not in adj:
            adj[v2] = []
        adj[v1].append(v2)
        adj[v2].append(v1)
    
    # Start from an arbitrary edge
    start_v = edges[0][0]
    path = [start_v]
    current_v = start_v
    visited = {start_v}
    
    while len(path) < len(edges) + 1: # +1 to close the loop
        neighbors = adj.get(current_v, [])
        next_v = None
        for n in neighbors:
            if n not in visited:
                next_v = n
                break
        
        if next_v is None:
            # If no unvisited neighbor, we might be at the end of a chain or the loop is closed
            # Check if we can close the loop
            if path[0] in neighbors:
                next_v = path[0]
            else:
                break
        
        path.append(next_v)
        visited.add(next_v)
        current_v = next_v
        
    return path

def sort_polygon_vertices(vertices):
    """Sort vertices to form a convex polygon."""
    if len(vertices) < 3:
        return vertices
    
    # Compute centroid
    centroid = sum(vertices) / len(vertices)
    
    # Sort by angle relative to centroid
    def angle(v):
        return atan2(v[1] - centroid[1], v[0] - centroid[0])
    
    return sorted(vertices, key=angle)

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def sub(v1, v2):
    return (v1[0]-v2[0], v1[1]-v2[1], v1[2]-v2[2])

def add(v1, v2):
    return (v1[0]+v2[0], v1[1]+v2[1], v1[2]+v2[2])

def scale(v, s):
    return (v[0]*s, v[1]*s, v[2]*s)

def atan2(y, x):
    import math
    return math.atan2(y, x)
```

## Subtopics

## Operating characteristics
*   **Time Complexity:** The complexity depends on the number of sampling planes $N$, the number of edges $E$ in the VOI, and the number of vertex events $V$. For each plane, the edge list update is $O(V)$ in the worst case (if many vertices are crossed), and the intersection computation is $O(E)$. With coherence, the average case is much faster, as most planes only require $O(1)$ updates.
*   **Memory Complexity:** $O(E + V)$ to store the WSG and the VOI topology.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Reduces GPU computation by clipping empty regions. Exploits coherence between parallel planes. Interactive VOI definition.
*   **Limitations:** CPU-based, so it adds CPU overhead. Limited to convex VOIs.
*   **When to use:** When the VOI is convex and the user wants to interactively define the clipping region. When the GPU is the bottleneck and can benefit from reduced sampling.
*   **When not to use:** When the VOI is non-convex. When the CPU is the bottleneck.

## Background
The method was proposed in the paper "Real-time interactive visualization and manipulation of the volumetric data using GPU-based methods" [S1]. It addresses the need for efficient clipping of volumetric data to improve rendering performance. The WSG is a novel data structure introduced in this work to manage the topological changes during the sweep.

## Sources
[S1] Real-time interactive visualization and manipulation of the volumetric data using GPU-based methods, https://doi.org/10.1117/12.536578, fetched 2026-10-26
