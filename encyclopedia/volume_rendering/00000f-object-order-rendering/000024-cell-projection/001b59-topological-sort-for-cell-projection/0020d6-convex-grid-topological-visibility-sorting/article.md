# Convex Grid Topological Visibility Sorting

Also called: Convex Grid Topological, Visibility Sorting (Convex Grid Topological)

## Overview
Convex Grid Topological Visibility Sorting is an $O(n)$ algorithm for determining the back-to-front rendering order of cells in a convex grid composed of convex polyhedral cells [S1][S2, unofficial]. It solves the visibility sorting problem by constructing a directed graph of pairwise occlusion relations between adjacent cells and performing a topological sort [S1][S2, unofficial]. Unlike [Exact Visibility Ordering for Cell Projection Volume Rendering (SXMPVO)](944-exact-visibility-ordering-for-cell-projection-volume-renderi.md) which may require more complex geometric checks, this method relies solely on the convexity of the grid volume and the cells to guarantee that local face-sharing relations imply global visibility order [S1][S2, unofficial]. It is a specific instance of [Topological Sort for Cell Projection](999-topological-sort-for-cell-projection.md) optimized for convex topologies.

## How it works

### Inputs and Outputs
**Inputs:**
1.  A set of $n$ convex cells forming a convex grid volume.
2.  The 3D position of the viewpoint $V$.
3.  The geometric definition of each cell, specifically its faces and neighbors. For each cell $C$, the set of adjacent cells sharing a common face is known.

**Outputs:**
1.  A list of cells sorted in back-to-front order (from farthest to nearest relative to $V$).

### Data Structures
*   **Cell Object:** Represents a single cell. Contains:
    *   `id`: Unique identifier.
    *   `incount`: Integer, initialized to 0. Counts the number of direct predecessors in the occlusion graph (cells that must be rendered before this cell).
    *   `neighbors`: List of cell IDs sharing a face with this cell.
    *   `face_normals`: List of outward-pointing normal vectors for each face, or a method to determine the side of a point relative to a face.
*   **Queue:** A FIFO queue (or stack, as any valid topological order is acceptable) containing cell IDs with `incount` equal to 0.
*   **Output List:** A list to store the final sorted cell IDs.

### Partial Order Relation
The core relation is $B <_{vp} A$, meaning cell $B$ must precede cell $A$ in the visibility sort [S1][S2, unofficial].

**Definition:** Let $A$ and $B$ be two cells sharing a common face $F$. Let $V$ be the viewpoint. The relation $B <_{vp} A$ holds if and only if the viewpoint $V$ and cell $A$ lie on the same side of the plane defined by face $F$ [S1][S2, unofficial].

**Geometric Test:**
Let $N$ be the outward-pointing normal vector of face $F$ (pointing away from cell $A$ and into cell $B$, or vice versa; consistency is key). Let $P_A$ be any point inside cell $A$ and $P_V$ be the viewpoint.
If the normal $N$ points from $A$ to $B$:
1.  Calculate $d_A = (P_A - P_{face}) \cdot N$. By definition, $d_A < 0$ (or $>0$ depending on normal direction convention, but $A$ is on the "negative" side if $N$ points out).
2.  Calculate $d_V = (P_V - P_{face}) \cdot N$.
3.  If $d_V$ has the same sign as $d_A$ (i.e., $V$ is on the same side of the face as $A$), then $A$ occludes $B$ from the perspective of $V$. Thus, $B <_{vp} A$ [S1][S2, unofficial].

*Note: The sources state "If the viewpoint V is on the same side of the face F between cells A and B as cell A is, then cell A occludes cell B" [S1][S2, unofficial]. This implies that the ray from $V$ to $B$ must pass through $A$'s volume or the shared face is oriented such that $A$ is "in front" of $B$ relative to $V$.*

### Algorithm Steps

1.  **Initialize Counts:**
    For each cell $C$ in the grid:
    *   Set $C.incount = 0$.

2.  **Build Occlusion Graph:**
    For each cell $A$ in the grid:
    *   For each neighbor $B$ of $A$ (sharing a face $F$):
        *   Determine if $B <_{vp} A$ using the geometric test above.
        *   If $B <_{vp} A$:
            *   Increment $A.incount$ by 1.
            *   (Implicitly, add a directed edge from $B$ to $A$ in the graph).

3.  **Initialize Queue:**
    For each cell $C$ in the grid:
    *   If $C.incount == 0$:
        *   Add $C$ to the Queue.

4.  **Topological Sort:**
    While the Queue is not empty:
    *   Remove a cell $C$ from the Queue.
    *   Append $C$ to the Output List.
    *   For each neighbor $B$ of $C$:
        *   Check if there is an occlusion relation $C <_{vp} B$ (i.e., $C$ must precede $B$).
        *   *Note: The sources describe iterating over "directed edges from C to a cell B". In the graph construction, an edge $B \to A$ exists if $B <_{vp} A$. Therefore, if we process $C$, we look for cells $B$ such that $C <_{vp} B$. This is equivalent to checking neighbors $B$ where the relation $C <_{vp} B$ holds.*
        *   If $C <_{vp} B$:
            *   Decrement $B.incount$ by 1.
            *   If $B.incount == 0$:
                *   Add $B$ to the Queue.

5.  **Cycle Detection:**
    *   If the Queue becomes empty and the Output List contains fewer than $n$ cells, a visibility cycle exists. This indicates the algorithm cannot produce a valid linear order for the current configuration (though for convex grids, this should not happen if the grid is truly convex and the viewpoint is outside or properly handled) [S1][S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convex_grid_topological_sort(cells, viewpoint):
    """
    Sorts convex cells in a convex grid back-to-front.
    
    Args:
        cells: List of Cell objects. Each Cell has:
            - id: int
            - neighbors: List of Cell objects sharing a face
            - get_face_normal(face_index): Returns outward normal vector
            - get_center(): Returns centroid point
        viewpoint: Point3D representing the camera position.
        
    Returns:
        List of Cell objects sorted back-to-front.
    """
    n = len(cells)
    output_list = []
    queue = []
    
    # Step 1: Initialize incounts
    for cell in cells:
        cell.incount = 0
        
    # Step 2: Build graph and compute incounts
    # We iterate over all directed edges. An edge B->A exists if B <vp A.
    # B <vp A means B must be rendered before A.
    # This happens if V and A are on the same side of the shared face F.
    
    for cell_A in cells:
        for neighbor_B in cell_A.neighbors:
            # Check if neighbor_B <vp cell_A
            # Find the shared face between cell_A and neighbor_B
            # Note: Implementation detail: finding the shared face geometry
            # is assumed to be available or computable.
            face = get_shared_face(cell_A, neighbor_B)
            if face is None:
                continue
                
            # Determine if B <vp A
            # Condition: Viewpoint V and Cell A are on the same side of Face F.
            # Let N be the normal of F pointing OUT of A (into B).
            # Point A_center is inside A. Point V is viewpoint.
            # Vector from Face to A_center is roughly opposite to N.
            # Vector from Face to V is V - Face_Point.
            
            # Let's define the test precisely based on sources:
            # "If the viewpoint V is on the same side of the face F between cells A and B as cell A is"
            
            # Get a point on the face
            face_point = face.center
            
            # Get normal pointing out of A
            normal = face.normal_out_of_A 
            
            # Vector from face to A's center
            vec_A = cell_A.center - face_point
            # Vector from face to Viewpoint
            vec_V = viewpoint - face_point
            
            # Dot products to determine side
            # If normal points out of A, vec_A . normal should be negative (or positive depending on convention)
            # Let's assume standard outward normal: vec_A . normal < 0
            side_A = dot(vec_A, normal)
            side_V = dot(vec_V, normal)
            
            # Check if they are on the same side
            # Same side means (side_A > 0 and side_V > 0) or (side_A < 0 and side_V < 0)
            if (side_A > 0 and side_V > 0) or (side_A < 0 and side_V < 0):
                # V and A are on the same side.
                # Source: "cell A occludes cell B... B must come before A"
                # So B <vp A.
                # Edge in graph: B -> A.
                # Increment incount of A.
                cell_A.incount += 1
                
    # Step 3: Initialize Queue
    for cell in cells:
        if cell.incount == 0:
            queue.append(cell)
            
    # Step 4: Process Queue
    while queue:
        cell_C = queue.pop(0) # Remove from front
        output_list.append(cell_C)
        
        # For all directed edges from C to a cell B
        # Edge C -> B exists if C <vp B
        for neighbor_B in cell_C.neighbors:
            # Check if C <vp B
            face = get_shared_face(cell_C, neighbor_B)
            if face is None:
                continue
                
            face_point = face.center
            # Normal pointing out of C
            normal = face.normal_out_of_C
            
            vec_C = cell_C.center - face_point
            vec_V = viewpoint - face_point
            
            side_C = dot(vec_C, normal)
            side_V = dot(vec_V, normal)
            
            if (side_C > 0 and side_V > 0) or (side_C < 0 and side_V < 0):
                # C and V are on the same side of the face.
                # C occludes B. C <vp B.
                # Edge C -> B.
                # Decrement incount of B.
                neighbor_B.incount -= 1
                if neighbor_B.incount == 0:
                    queue.append(neighbor_B)
                    
    # Step 5: Check for cycles
    if len(output_list) < n:
        raise Exception("Visibility cycle detected. Grid may not be convex or viewpoint handling is incorrect.")
        
    return output_list
```

### Parameters and Edge Cases
*   **Convexity:** The algorithm assumes the entire grid volume is convex. If the grid is non-convex, the algorithm may fail or produce incorrect results because a viewing ray might exit the grid and re-enter it [S1][S2, unofficial].
*   **Boundary Cells:** Cells with external faces (boundary cells) are handled, but the simple topological sort does not account for rays crossing external gaps [S1][S2, unofficial]. For non-convex grids, additional $O(b^2)$ tests between boundary cells are required [S2, unofficial].
*   **Visibility Cycles:** If the queue empties before all cells are processed, a cycle exists. In a strictly convex grid with a valid viewpoint, this should not occur [S1][S2, unofficial].
*   **Precision:** Geometric tests involving dot products and point-in-halfspace checks are sensitive to floating-point precision. Degenerate cases (viewpoint exactly on a face plane) require careful handling [G].

## Operating characteristics

*   **Time Complexity:** $O(n)$, where $n$ is the number of cells. The initialization and graph building steps process each cell and each edge (face sharing) a constant number of times. Since each cell has a maximum of 6 faces (in a hexahedral grid), the number of edges is proportional to $n$ [S1][S2, unofficial].
*   **Space Complexity:** $O(n)$ to store the graph (incounts) and the queue/output list.
*   **Strengths:** Extremely fast for convex grids. No complex geometric intersection tests are needed during the sort phase, only local face tests.
*   **Limitations:** Only applicable to convex grids of convex cells. Fails or requires augmentation for non-convex grids. Does not handle self-occlusion or complex internal structures that break convexity.
*   **When to Use:** Ideal for structured grids (e.g., regular hexahedral meshes) where the domain is convex and the viewpoint is external or the grid is treated as a convex hull.
*   **When Not to Use:** Non-convex domains, unstructured grids with non-convex cells, or when high precision is required for complex occlusion scenarios not covered by local face relations.

## Background

*   **Origin:** The algorithm is described in the paper "Volume rendering for curvilinear and unstructured grids" by Comba et al. [S1][S2, unofficial].
*   **History:** It was developed as a faster alternative to $O(n^2)$ pairwise occlusion tests for volume rendering on curvilinear and unstructured grids [S1][S2, unofficial]. The method leverages the topological properties of convex grids to reduce the sorting problem to a graph topological sort.
*   **Usage:** Used in volume rendering pipelines for scientific visualization where the data domain is convex and structured or curvilinear grids are used.

## Sources

[S1] Volume rendering for curvilinear and unstructured grids, https://doi.org/10.1109/cgi.2003.1214468, fetched 2026-10-24
[S2] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
