# XMPVO

Also called: Extended Meshed Polyhedra Visibility Ordering

## Overview
XMPVO is an algorithm that improves the runtime of the Meshed Polyhedra Visibility Ordering (MPVO) method by using ray shooting queries to discover exterior face relations for nonconvex meshes [S1][S2, S3, unofficial]. It addresses the problem of determining the correct back-to-front rendering order for convex volumes filled with convex polyhedral cells when the standard topological sort is insufficient due to occlusions between exterior faces [S1][S2, S3, unofficial]. It sets itself apart from its parent [Topological Sort for Cell Projection](999-topological-sort-for-cell-projection.md) by augmenting the graph with geometric visibility checks, and from its sibling [Exact Visibility Ordering for Cell Projection Volume Rendering (SXMPVO)](944-exact-visibility-ordering-for-cell-projection-volume-renderi.md) by performing these checks in object-space precision rather than image-space [S1][S2, S3, unofficial].

## How it works
**Inputs:**
*   A set of $n$ convex polyhedral cells forming a mesh.
*   A viewpoint position.
*   The projection complex of the mesh (the arrangement of projected cell edges and vertices on the image plane).

**Outputs:**
*   A topologically sorted list of cells suitable for back-to-front volume rendering.

**Data Structures:**
*   **Visibility Graph:** A directed graph where nodes represent cells. Edges represent visibility relations.
    *   Edges from the parent MPVO algorithm (based on shared faces and viewpoint containment).
    *   Additional edges representing exterior face occlusions discovered via ray shooting.
*   **Projection Complex:** A data structure storing the 2D intersections of projected cell edges and the projected vertices.

**Steps:**
1.  **Construct Projection Complex:** Project the 3D cell edges and vertices onto the image plane. Compute the intersections of these projected edges to form the projection complex [S1][S2, S3, unofficial].
2.  **Identify Query Points:** Identify all edge intersections and vertices within the projection complex [S1][S2, S3, unofficial].
3.  **Ray Shooting Queries:** For each point identified in Step 2, perform a ray shooting query from the viewpoint through that point into the scene [S1][S2, S3, unofficial].
    *   The purpose is to discover which exterior faces occlude other exterior faces [S1][S2, S3, unofficial].
    *   If face $A$ occludes face $B$ at a query point, add a directed edge from cell containing $A$ to cell containing $B$ in the visibility graph [S1][S2, S3, unofficial].
4.  **Topological Sort:** Perform a topological sort on the augmented visibility graph (containing both MPVO relations and discovered exterior face relations) to determine the final rendering order [S1][S2, S3, unofficial].

**Formulas:**
*   Time Complexity: $O(n + b^2)$, where $n$ is the number of cells and $b$ is the number of exterior faces [S1][S2, S3, unofficial].

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The ray shooting queries are performed in object-precision (3D space), which is expensive and makes it hard to avoid redundant queries [S1][S2, S3, unofficial].
*   **Robustness:** Robustly computing the intersections of the projections of cell edges is difficult and may require exact arithmetic packages, which slows down performance [S1][S2, S3, unofficial].
*   **Redundancy:** Due to the object-precision nature, redundant queries are hard to avoid [S1][S2, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def xmpvo(cells, viewpoint):
    """
    XMPVO algorithm for visibility ordering.
    
    Args:
        cells: List of convex polyhedral cell objects.
        viewpoint: 3D vector representing the camera position.
        
    Returns:
        List of cells sorted back-to-front.
    """
    # Step 1: Construct the projection complex
    # Project all cell edges and vertices onto the image plane.
    # Compute intersections of projected edges.
    projection_complex = construct_projection_complex(cells)
    
    # Step 2: Identify query points
    # Collect all edge intersections and vertices from the projection complex.
    query_points = []
    for edge_intersection in projection_complex.edge_intersections:
        query_points.append(edge_intersection)
    for vertex in projection_complex.vertices:
        query_points.append(vertex)
        
    # Step 3: Perform ray shooting queries to discover exterior face relations
    # Initialize the visibility graph with MPVO relations (from parent algorithm)
    visibility_graph = build_mpvo_graph(cells, viewpoint)
    
    for point in query_points:
        # Create a ray from the viewpoint through the query point
        ray_origin = viewpoint
        ray_direction = normalize(point - viewpoint)
        
        # Perform ray shooting query to find occluding exterior faces
        # This returns a list of faces intersected by the ray, ordered by distance
        intersected_faces = ray_shoot(ray_origin, ray_direction, cells)
        
        # Add visibility edges based on occlusion
        # If face A occludes face B, add edge from cell(A) to cell(B)
        for i in range(len(intersected_faces) - 1):
            face_occluder = intersected_faces[i]
            face_occluded = intersected_faces[i+1]
            
            cell_occluder = get_cell_containing_face(face_occluder, cells)
            cell_occluded = get_cell_containing_face(face_occluded, cells)
            
            # Add edge if not already present
            if not visibility_graph.has_edge(cell_occluder, cell_occluded):
                visibility_graph.add_edge(cell_occluder, cell_occluded)
                
    # Step 4: Topological sort
    # Perform topological sort on the augmented graph
    sorted_cells = topological_sort(visibility_graph)
    
    return sorted_cells

def construct_projection_complex(cells):
    """
    Constructs the projection complex by projecting edges and vertices
    and computing their intersections.
    Note: Robust computation of intersections may require exact arithmetic.
    """
    # Implementation detail: Project 3D edges to 2D lines.
    # Compute all pairwise intersections of these 2D lines.
    # Store vertices and intersection points.
    # [G] This function assumes a robust 2D line intersection routine is available.
    pass

def ray_shoot(origin, direction, cells):
    """
    Performs a ray shooting query.
    Returns a list of exterior faces intersected by the ray, sorted by distance from origin.
    """
    # [G] This function performs a standard ray-casting against the mesh geometry.
    # It must identify only exterior faces and sort them by intersection distance.
    pass

def get_cell_containing_face(face, cells):
    """
    Returns the cell that contains the given face.
    """
    # [G] Simple lookup or search to find the cell owning the face.
    pass
```

## Subtopics

### [Kraus and Ertl Method](1378-kraus-and-ertl-method.md)
The Kraus and Ertl method handles visibility cycles in convex meshes during cell projection volume rendering. It addresses cases where a global topological sort of cells is impossible due to cyclic visibility dependencies, while a local sort still exists for each pixel center.

## Operating characteristics
*   **Time Complexity:** $O(n + b^2)$, where $n$ is the number of cells and $b$ is the number of exterior faces [S1][S2, S3, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Improves runtime over the $O(n^2)$ Stein et al. method by reducing the number of necessary visibility checks to only those involving exterior faces [S1][S2, S3, unofficial].
*   **Limitations:**
    *   Performs ray shooting queries in object-precision, which is expensive [S1][S2, S3, unofficial].
    *   Hard to avoid redundant queries due to object-precision [S1][S2, S3, unofficial].
    *   Robustly computing intersections of projected edges is difficult and may require exact arithmetic packages, slowing performance [S1][S2, S3, unofficial].
*   **When to use:** When dealing with nonconvex meshes where MPVO is insufficient, and the number of exterior faces $b$ is small relative to $n$, making $O(n + b^2)$ faster than $O(n^2)$.
*   **When not to use:** When high precision is required for edge intersections, or when the overhead of exact arithmetic is prohibitive. In such cases, image-space methods like SXMPVO may be preferred [S1][S2, S3, unofficial].

## Background
XMPVO was developed by Silva et al. [S1][S2, S3, unofficial]. It was introduced as an improvement over the MPVO algorithm to handle nonconvex meshes more efficiently [S1][S2, S3, unofficial]. The method relies on the observation that extra visibility relationships only occur when an exterior face in the mesh occludes another exterior face [S1][S2, S3, unofficial].

## Sources
[S1] Image-space visibility ordering for cell projection volume rendering of unstructured data, https://doi.org/10.1109/tvcg.2004.45, fetched 2026-10-25
[S2] (PDF) Image-space visibility ordering for cell projection volume ..., https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
[S3] PDF 0078-0903-2 1..13, https://sci.utah.edu/~csilva/papers/tvcg2004cr.pdf, fetched 2026-10-28 (unofficial)
