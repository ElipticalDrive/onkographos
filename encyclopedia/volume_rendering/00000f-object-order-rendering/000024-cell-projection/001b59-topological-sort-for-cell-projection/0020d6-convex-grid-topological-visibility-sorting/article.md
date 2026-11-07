# Convex Grid Topological Visibility Sorting

Also called: Adjacency graph sort, Convex Grid Topological, Graph-based visibility sort, Teller et al. Visibility Algorithm, Topological Sort for Back-to-Front Visibility, Visibility Sorting (Convex Grid Topological)

## Overview
Convex Grid Topological Visibility Sorting is an $O(n)$ algorithm that orders cells in a convex grid of convex polyhedral cells back-to-front for semi-transparent volume rendering [S2][S3, unofficial]. It constructs a directed adjacency graph where edges represent pairwise occlusion relations derived from the viewpoint position, then performs a topological sort to produce a valid compositing order [S1][S3, unofficial]. It sets itself apart from [Exact Visibility Ordering for Cell Projection Volume Rendering (SXMPVO)](944-exact-visibility-ordering-for-cell-projection-volume-renderi.md) and [Power Distance Sort](1060-power-distance-sort.md) by relying exclusively on local face-sharing topology rather than global geometric tests or distance metrics, provided the grid volume is convex [S2][S3, unofficial].

## How it works
**Inputs:**
- A grid of $n$ convex polyhedral cells.
- An adjacency structure: for each face of each cell, the index of the neighboring cell sharing that face, or a null value if the face is on the boundary [S1].
- A viewpoint position $V$ in 3D space [S2].

**Outputs:**
- A list of cell indices sorted in back-to-front order (from farthest to nearest relative to $V$) [S1][S3, unofficial].

**Data Structures:**
- **Cell Node:** Contains an integer `incount` initialized to 0, representing the number of incoming edges in the directed graph [S2][S3, unofficial].
- **Adjacency List:** For each cell, a list of neighbor cell indices corresponding to its faces [S1].
- **Queue:** A FIFO queue storing cell indices with `incount` equal to 0 [S2][S3, unofficial].
- **Output List:** A dynamic array or linked list storing the sorted cell indices [S2][S3, unofficial].

**Definitions:**
- Let $F$ be a common face shared by cells $A$ and $B$.
- Let $P_F$ be the plane equation of face $F$.
- The partial order relation $B <_{vp} A$ holds if cell $B$ must precede cell $A$ in the visibility sort [S2][S3, unofficial].
- $B <_{vp} A$ is true if the viewpoint $V$ is on the same side of the plane $P_F$ as cell $A$ [S2][S3, unofficial]. In this case, $A$ occludes $B$ across face $F$, but $B$ cannot occlude $A$ [S2][S3, unofficial].
- The directed graph has a node for each cell and a directed edge from $B$ to $A$ whenever $B <_{vp} A$ [S2][S3, unofficial].

**Algorithm Steps:**

1. **Initialize Counts:**
   For all cells $C$ in the grid, set $C.incount = 0$ [S2][S3, unofficial].

2. **Build Directed Graph:**
   For each cell $A$ and each of its faces $F$:
   - Let $B$ be the neighbor cell sharing face $F$ (from the adjacency structure) [S1].
   - If $B$ exists (i.e., $F$ is an interior face):
     - Evaluate the plane equation $P_F$ at the viewpoint $V$.
     - Evaluate the plane equation $P_F$ at the centroid (or any interior point) of cell $A$.
     - If $V$ and $A$ are on the same side of $P_F$ (i.e., the signs of the plane evaluations are identical):
       - The relation $B <_{vp} A$ holds [S2][S3, unofficial].
       - Increment $A.incount$ by 1 [S2][S3, unofficial].
       - Record a directed edge from $B$ to $A$ in the adjacency list of $B$ [S2][S3, unofficial].

3. **Initialize Queue:**
   For all cells $C$ in the grid:
   - If $C.incount == 0$, add $C$ to the Queue [S2][S3, unofficial].

4. **Topological Sort:**
   While the Queue is not empty:
   - Remove a cell $C$ from the Queue [S2][S3, unofficial].
   - Add $C$ to the end of the Output List [S2][S3, unofficial].
   - For each neighbor $B$ such that there is a directed edge from $C$ to $B$ (i.e., $C <_{vp} B$):
     - Decrement $B.incount$ by 1 [S2][S3, unofficial].
     - If $B.incount == 0$, add $B$ to the Queue [S2][S3, unofficial].

5. **Cycle Detection:**
   - If the Output List contains fewer than $n$ cells, a visibility cycle exists [S2][S3, unofficial]. The algorithm fails to produce a total order for the remaining cells [S1, S2][S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convex_grid_topological_sort(cells, adjacency, viewpoint):
    """
    Sorts cells in back-to-front order for a convex grid.
    
    Args:
        cells: List of cell objects. Each cell has a method get_centroid().
        adjacency: List of lists. adjacency[i][f] is the index of the neighbor 
                   sharing face f of cell i, or -1 if boundary.
        viewpoint: 3D vector (x, y, z).
        
    Returns:
        List of cell indices in back-to-front order.
    """
    n = len(cells)
    incount = [0] * n
    # Adjacency list for the directed graph: graph[u] contains list of v where u -> v
    graph = [[] for _ in range(n)]
    
    # Helper to determine which side of a plane a point is on
    # Plane is defined by normal N and point P on plane.
    # We assume adjacency provides face geometry or we compute it from cell vertices.
    # For simplicity, we assume a function get_face_plane(cell_idx, face_idx) returns (N, P).
    
    for cell_idx in range(n):
        cell = cells[cell_idx]
        # Iterate over all faces of the cell
        for face_idx, neighbor_idx in enumerate(adjacency[cell_idx]):
            if neighbor_idx == -1:
                continue # Boundary face, no directed edge
            
            # Get plane of the shared face
            # Note: The plane equation must be consistent. 
            # Typically, the normal points from the current cell to the neighbor.
            # However, the condition "V and A are on the same side" depends on 
            # the specific plane definition. 
            # Standard approach: Plane defined by face vertices.
            # Let's assume get_face_plane returns a plane where the normal points 
            # towards the neighbor cell (or consistently defined).
            # To be robust, we check the sign of the plane function for V and A.
            
            # Reconstruct plane from face vertices or use precomputed data.
            # For this implementation, we assume we can evaluate the plane function.
            # Let plane_func(point) return the signed distance or value.
            # The condition is: sign(plane_func(V)) == sign(plane_func(A_centroid))
            
            # We need the plane of the face shared by cell_idx and neighbor_idx.
            # The plane separates the two cells.
            # Let's assume we have a function face_plane(cell_idx, face_idx) 
            # that returns a plane object or coefficients.
            
            plane = get_face_plane(cell_idx, face_idx)
            
            val_v = plane.evaluate(viewpoint)
            val_a = plane.evaluate(cell.get_centroid())
            
            # Check if V and A are on the same side
            # If both are positive or both are negative, they are on the same side.
            # If one is zero, it's on the plane (degenerate).
            if (val_v > 0 and val_a > 0) or (val_v < 0 and val_a < 0):
                # B <_vp A holds. B is neighbor_idx, A is cell_idx.
                # Edge from B to A.
                graph[neighbor_idx].append(cell_idx)
                incount[cell_idx] += 1
            elif (val_v < 0 and val_a > 0) or (val_v > 0 and val_a < 0):
                # V and A are on opposite sides.
                # Then A <_vp B holds.
                # Edge from A to B.
                graph[cell_idx].append(neighbor_idx)
                incount[neighbor_idx] += 1
            else:
                # One of them is on the plane. 
                # This is a degenerate case. 
                # If V is on the plane, the edge direction is ambiguous.
                # If A is on the plane, the centroid is on the plane (impossible for convex cell).
                # If V is on the plane, we might skip or handle specially.
                # For robustness, if V is on the plane, the occlusion is tangential.
                # We can arbitrarily assign or skip. 
                # However, if V is exactly on the plane, the ray grazes the face.
                # Standard practice: if V is on the plane, the order is undefined.
                # We will assume general position for now.
                pass

    # Initialize Queue
    queue = []
    for i in range(n):
        if incount[i] == 0:
            queue.append(i)
            
    output_list = []
    
    # Process Queue
    head = 0
    while head < len(queue):
        c = queue[head]
        head += 1
        
        output_list.append(c)
        
        for b in graph[c]:
            incount[b] -= 1
            if incount[b] == 0:
                queue.append(b)
                
    if len(output_list) < n:
        raise ValueError("Visibility cycle detected. Grid is not convex or topology is invalid.")
        
    return output_list
```

**Edge Cases and Pitfalls:**
- **Visibility Cycles:** If the queue becomes empty before all cells are processed, a cycle exists [S1, S2][S3, unofficial]. This indicates the grid is not convex or contains cavities/concavities where viewing rays exit and re-enter the grid [S1][S3, unofficial]. The cells involved must be subdivided [S1].
- **Non-Convex Grids:** The algorithm is not correct for non-convex grids because viewing rays may leave the grid volume through an external face, cross a gap, and re-enter another cell [S3, unofficial]. Cells with external faces are called boundary cells [S3, unofficial].
- **Boundary Cells:** For non-convex grids, the algorithm can be extended by adding directed edges between boundary cells whenever a viewing ray crosses both [S1]. This requires $O(b^2)$ pairwise tests for $b$ boundary cells [S1][S3, unofficial].
- **Plane Evaluation Precision:** If the viewpoint $V$ lies exactly on a face plane, the side determination is ambiguous [G]. A small epsilon tolerance should be used when comparing plane values.

## Subtopics

### [Visibility Ordering for Curvilinear Grids](6700-visibility-ordering-for-curvilinear-grids.md)
An algorithm to generate a visibility ordering for cells in a curvilinear grid (hexahedra) to support semi-transparent rendering, using a combination of undirected breadth-first search and directed depth-first searches on the adjacency graph. It sets itself apart from [Convex Grid Topological Visibility Sorting](2370-convex-grid-topological-visibility-sorting.md) by handling non-convex volumes without requiring a convex decomposition, leveraging the implicit topology of curvilinear grids to avoid auxiliary data structures.

## Operating characteristics
- **Time Complexity:** $O(n)$ for convex grids with adjacency structures, where $n$ is the number of cells, assuming each cell has at most six faces [S1, S2][S3, unofficial]. The extended algorithm for non-convex grids with $b$ boundary cells is $O(n + b^2)$ [S1][S3, unofficial]. An optimized version by Silva et al. (1998) achieves $O((b + I)\log_2 b + n)$, where $I$ is the number of crossings among exterior face edges [S1].
- **Memory Complexity:** $O(n + e)$, where $e$ is the number of edges in the directed graph. Since each cell has a constant number of faces (e.g., 6), $e = O(n)$, so memory is $O(n)$ [S2][S3, unofficial].
- **Strengths:** Linear time complexity for convex grids; simple implementation; reuses adjacency graph and plane equations across views if geometry is static, requiring only edge direction updates per viewpoint [S1].
- **Limitations:** Fails for non-convex grids without extension; requires adjacency structure; visibility cycles may occur in complex geometries requiring subdivision [S1][S3, unofficial].
- **When to Use:** Ideal for convex grids (e.g., structured hexahedral meshes) where performance is critical and the grid topology is well-defined [S2][S3, unofficial].
- **When Not to Use:** Non-convex grids without sufficient boundary cell optimization; unstructured grids without precomputed adjacency [S1][S3, unofficial].

## Background
- **Origin:** The algorithm is based on pairwise ordering relations between cells sharing a common face [S2][S3, unofficial]. It was described in the context of volume rendering for curvilinear and unstructured grids [S2][S3, unofficial].
- **History:** Teller et al. developed a from-region visibility algorithm used by Funkhouser et al. in their walkthrough system, which requires long preprocessing times and assumes axis-aligned cells [S4, S4, unofficial]. The $O(n)$ topological sort for convex grids is a more efficient variant that does not require such strict assumptions [S2][S3, unofficial].
- **Usage:** Used in hardware-assisted polyhedron projection for volume rendering [S1].

## Sources
[S1] Approximate volume rendering for curvilinear and unstructured grids by hardware-assisted polyhedron projection, https://doi.org/10.1002/(sici)1098-1098(2000)11:1<53::aid-ima6>3.0.co;2-7, fetched 2026-10-25
[S2] Volume rendering for curvilinear and unstructured grids, https://doi.org/10.1109/cgi.2003.1214468, fetched 2026-10-24
[S3] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
[S4] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
