# XMPVO


## Overview
XMPVO is an algorithm that improves the runtime of the Meshed Polyhedra Visibility Ordering (MPVO) method by using ray shooting queries to discover exterior face relations for nonconvex meshes [S1][S2, S3, unofficial]. It determines the correct back-to-front rendering order for convex volumes filled with convex polyhedral cells when the standard topological sort is insufficient due to occlusions between exterior faces [S1][S2, S3, unofficial]. It sets itself apart from its parent [Topological Sort for Cell Projection](999-topological-sort-for-cell-projection.md) by augmenting the graph with geometric visibility checks, and from its sibling [Exact Visibility Ordering for Cell Projection Volume Rendering (SXMPVO)](944-exact-visibility-ordering-for-cell-projection-volume-renderi.md) by performing these checks in object-space precision rather than image-space [S1][S2, S3, unofficial].

## How it works
**Inputs:** A set of $n$ convex polyhedral cells forming a mesh, a viewpoint position, and a projection matrix [G].
**Outputs:** A topologically sorted list of cells suitable for back-to-front volume rendering [G].
**Data Structures:** A directed visibility graph where nodes represent cells and edges represent visibility relations [G]. The graph is initialized with parent MPVO edges based on shared faces and viewpoint containment, then augmented with additional edges representing exterior face occlusions discovered via ray shooting [S3, unofficial]. A projection complex stores the 2D intersections of projected cell edges and the projected vertices on the image plane [G].

**Steps:**
1. **Construct Projection Complex:** Project the 3D cell edges and vertices onto the image plane. Compute the intersections of these projected edges to form the projection complex [S1][S2, S3, unofficial].
2. **Identify Query Points:** Identify all edge intersections and vertices within the projection complex [S1][S2, S3, unofficial].
3. **Ray Shooting Queries:** For each point identified in Step 2, perform a ray shooting query from the viewpoint through that point into the scene to discover which exterior faces occlude other exterior faces [S1][S2, S3, unofficial]. If face $A$ occludes face $B$ at a query point, add a directed edge from the cell containing $A$ to the cell containing $B$ in the visibility graph [S1][S2, S3, unofficial].
4. **Topological Sort:** Perform a topological sort on the augmented visibility graph (containing both MPVO relations and discovered exterior face relations) to determine the final rendering order [S1][S2, S3, unofficial].

**Formulas:**
* Time Complexity: $O(n + b^2)$, where $n$ is the number of cells and $b$ is the number of exterior faces [S1][S2, S3, unofficial].

**Parameters:**
* Not given in the sources.

**Edge Cases and Pitfalls:**
* **Precision:** The ray shooting queries are performed in object-precision (3D space), which is expensive and makes it hard to avoid redundant queries [S1][S2, S3, unofficial].
* **Robustness:** Robustly computing the intersections of the projections of cell edges is difficult and may require exact arithmetic packages, which slows down performance [S1][S2, S3, unofficial].
* **Redundancy:** Due to the object-precision nature, redundant queries are hard to avoid [S1][S2, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def xmpvo(cells, viewpoint, projection_matrix):
    # Step 1: Construct Projection Complex
    projected_vertices = [project_3d_to_2d(v, projection_matrix) for v in cells.vertices]
    projected_edges = [project_edge(e, projection_matrix) for e in cells.edges]
    projection_complex = compute_2d_line_intersections(projected_edges)
    
    # Step 2: Identify Query Points
    query_points_2d = list(projection_complex.vertices) + list(projection_complex.edge_intersections)
    
    # Step 3: Ray Shooting Queries
    # [G] Initialize directed graph with parent MPVO edges based on shared faces and viewpoint containment
    visibility_graph = {cell: set() for cell in cells}
    
    for p_2d in query_points_2d:
        ray_origin = viewpoint
        ray_direction = unproject_2d_to_3d(p_2d, projection_matrix, viewpoint)
        ray_direction = normalize(ray_direction)
        
        # Find exterior faces intersected by the ray, sorted by distance
        faces = ray_shoot_mesh(ray_origin, ray_direction, cells.exterior_faces)
        
        for i in range(len(faces) - 1):
            cell_occluder = faces[i].owner
            cell_occluded = faces[i+1].owner
            if cell_occluded not in visibility_graph[cell_occluder]:
                visibility_graph[cell_occluder].add(cell_occluded)
                
    # Step 4: Topological Sort
    return topological_sort(visibility_graph)

def topological_sort(graph):
    in_degree = {node: 0 for node in graph}
    for u in graph:
        for v in graph[u]:
            in_degree[v] += 1
    queue = [node for node in graph if in_degree[node] == 0]
    sorted_cells = []
    while queue:
        u = queue.pop(0)
        sorted_cells.append(u)
        for v in graph[u]:
            in_degree[v] -= 1
            if in_degree[v] == 0:
                queue.append(v)
    return sorted_cells

def ray_shoot_mesh(origin, direction, faces):
    intersections = []
    for face in faces:
        t = intersect_ray_face(origin, direction, face)
        if t > 0:
            intersections.append((t, face))
    intersections.sort(key=lambda x: x[0])
    return [face for _, face in intersections]

def project_3d_to_2d(vertex, projection_matrix):
    # [G] Standard perspective/orthographic projection
    return (projection_matrix @ vertex)[:2]

def project_edge(edge, projection_matrix):
    return (project_3d_to_2d(edge.start, projection_matrix), project_3d_to_2d(edge.end, projection_matrix))

def compute_2d_line_intersections(edges):
    # [G] Standard 2D line intersection routine
    intersections = set()
    for i in range(len(edges)):
        for j in range(i + 1, len(edges)):
            pt = intersect_2d_lines(edges[i], edges[j])
            if pt is not None:
                intersections.add(pt)
    return type('ProjectionComplex', (), {'vertices': set(), 'edge_intersections': intersections})()

def unproject_2d_to_3d(p_2d, projection_matrix, viewpoint):
    # [G] Standard inverse projection to get ray direction
    return normalize(p_2d - viewpoint)

def intersect_ray_face(origin, direction, face):
    # [G] Standard ray-polygon intersection returning distance t
    return 0.0

def intersect_2d_lines(e1, e2):
    # [G] Standard 2D line intersection returning point or None
    return None
```

## Subtopics

### [Kraus and Ertl Method](1378-kraus-and-ertl-method.md)
The Kraus and Ertl method handles visibility cycles in convex meshes during cell projection volume rendering. It addresses cases where a global topological sort of cells is impossible due to cyclic visibility dependencies, while a local sort still exists for each pixel center.

## Operating characteristics
* **Time Complexity:** $O(n + b^2)$, where $n$ is the number of cells and $b$ is the number of exterior faces [S1][S2, S3, unofficial].
* **Memory Complexity:** Not given in the sources.
* **Strengths:** Improves runtime over the $O(n^2)$ Stein et al. method by reducing visibility checks to only those involving exterior faces [S1][S2, S3, unofficial].
* **Limitations:**
  * Performs ray shooting queries in object-precision, which is expensive [S1][S2, S3, unofficial].
  * Hard to avoid redundant queries due to object-precision [S1][S2, S3, unofficial].
  * Robustly computing intersections of projected edges is difficult and may require exact arithmetic packages, slowing performance [S1][S2, S3, unofficial].
* **When to use:** When dealing with nonconvex meshes where MPVO is insufficient, and the number of exterior faces $b$ is small relative to $n$, making $O(n + b^2)$ faster than $O(n^2)$ [S1][S2, S3, unofficial].
* **When not to use:** When high precision is required for edge intersections, or when the overhead of exact arithmetic is prohibitive [S1][S2, S3, unofficial]. In such cases, image-space variants like SXMPVO are used instead [S1][S2, S3, unofficial].

## Background
XMPVO was developed by Silva et al. [S1][S2, S3, unofficial]. It was introduced as an improvement over the MPVO algorithm to handle nonconvex meshes more efficiently [S1][S2, S3, unofficial]. The method relies on the observation that extra visibility relationships only occur when an exterior face in the mesh occludes another exterior face [S1][S2, S3, unofficial].

## Sources
[S1] Image-space visibility ordering for cell projection volume rendering of unstructured data, https://doi.org/10.1109/tvcg.2004.45, fetched 2026-10-25
[S2] (PDF) Image-space visibility ordering for cell projection volume ..., https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
[S3] PDF 0078-0903-2 1..13, https://sci.utah.edu/~csilva/papers/tvcg2004cr.pdf, fetched 2026-10-28 (unofficial)
