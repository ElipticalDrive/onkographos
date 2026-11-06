# Convex Grid Topological Visibility Sorting

Also called: Adjacency graph sort, Convex Grid Topological, Graph-based visibility sort, Topological Sort for Back-to-Front Visibility, Visibility Sorting (Convex Grid Topological)

## Overview
Convex Grid Topological Visibility Sorting is an $O(n)$ algorithm for determining the back-to-front rendering order of cells in a convex grid composed of convex polyhedral cells [S1][S2]. It solves the visibility sorting problem by constructing a directed graph of pairwise occlusion relations between adjacent cells and performing a topological sort [S1][S2]. Unlike [Exact Visibility Ordering for Cell Projection Volume Rendering (SXMPVO)](944-exact-visibility-ordering-for-cell-projection-volume-renderi.md) which may require more complex geometric checks, this method relies solely on the convexity of the grid volume and the cells to guarantee that local face-sharing relations imply global visibility order [S1][S2].

## How it works

### Inputs and Outputs
**Inputs:**
1. A set of $n$ convex cells forming a convex grid volume [S1][S2].
2. The 3D position of the viewpoint $V$ [S1][S2].
3. An adjacency structure providing, for each face of each cell, the identity of the neighboring cell sharing that face [S1].

**Outputs:**
1. A list of cells sorted in back-to-front order (from farthest to nearest relative to $V$) [S1][S2].

### Data Structures
*   **Cell Object:** Represents a single cell. Contains:
    *   `id`: Unique integer identifier.
    *   `incount`: Integer, initialized to 0. Counts the number of direct predecessors in the occlusion graph (cells that must be rendered before this cell) [S2].
    *   `faces`: List of face descriptors. Each face descriptor contains `neighbor_id` (integer ID of the adjacent cell, or `None` for boundary faces), `normal` (outward-pointing unit normal vector), and `point` (a known point on the face plane) [S1][S2].
*   **Queue:** A FIFO queue containing cell IDs with `incount` equal to 0 [S2].
*   **Output List:** A list to store the final sorted cell IDs [S2].

### Partial Order Relation
The core relation is $B <_{vp} A$, meaning cell $B$ must precede cell $A$ in the visibility sort [S1][S2].

**Definition:** Let $A$ and $B$ be two cells sharing a common face $F$ [S1][S2]. The relation $B <_{vp} A$ holds if and only if the viewpoint $V$ and cell $A$ lie on the same side of the plane defined by face $F$ [S1][S2].

**Geometric Test:**
Let $N$ be the outward-pointing normal vector of face $F$ (pointing from $A$ into $B$) [S2]. Let $P_A$ be any point inside cell $A$ (e.g., the centroid) and $P_V$ be the viewpoint [S2].
1. Calculate $d_A = (P_A - P_{face}) \cdot N$ [S2].
2. Calculate $d_V = (P_V - P_{face}) \cdot N$ [S2].
3. If $d_A$ and $d_V$ have the same sign (i.e., $V$ is on the same side of the face as $A$), then $A$ occludes $B$ from the perspective of $V$ [S1][S2]. Thus, $B <_{vp} A$ [S2].

### Algorithm Steps
1. **Initialize Counts:** For each cell $C$ in the grid, set $C.incount = 0$ [S2].
2. **Build Occlusion Graph:** For each cell $A$ in the grid, iterate over all its faces. For each face shared with neighbor $B$, evaluate the geometric test. If $B <_{vp} A$ holds, increment $A.incount$ by 1 [S2].
3. **Initialize Queue:** For each cell $C$ in the grid, if $C.incount == 0$, add $C$ to the Queue [S2].
4. **Topological Sort:** While the Queue is not empty, remove a cell $C$ from the Queue, append $C$ to the Output List, and iterate over all neighbors $B$ of $C$. If the relation $C <_{vp} B$ holds, decrement $B.incount$ by 1. If $B.incount$ reaches 0, add $B$ to the Queue [S2].
5. **Cycle Detection:** If the Queue becomes empty and the Output List contains fewer than $n$ cells, a visibility cycle exists [S1][S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convex_grid_topological_sort(cells, viewpoint):
    """
    Sorts convex cells in a convex grid back-to-front.
    
    Args:
        cells: List of Cell objects. Each Cell has:
            - id: int
            - faces: List of dicts with keys 'neighbor_id', 'normal', 'point'
        viewpoint: tuple (x, y, z) representing the camera position.
        
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
    for cell_A in cells:
        for face in cell_A.faces:
            neighbor_B_id = face['neighbor_id']
            if neighbor_B_id is None:
                continue
            # Find neighbor_B object
            neighbor_B = next(c for c in cells if c.id == neighbor_B_id)
            
            # Geometric test: V and A on same side of face plane
            # N points out of A (into B)
            N = face['normal']
            P_face = face['point']
            P_A = cell_A.center  # Assumes cell has a centroid attribute
            P_V = viewpoint
            
            # Dot products to determine half-space
            side_A = dot(P_A - P_face, N)
            side_V = dot(P_V - P_face, N)
            
            # Check if same sign (both positive or both negative)
            if (side_A > 0 and side_V > 0) or (side_A < 0 and side_V < 0):
                # V and A are on the same side -> A occludes B -> B <vp A
                # Edge in graph: B -> A. Increment incount of A.
                cell_A.incount += 1
                
    # Step 3: Initialize Queue
    for cell in cells:
        if cell.incount == 0:
            queue.append(cell)
            
    # Step 4: Process Queue
    while queue:
        cell_C = queue.pop(0)
        output_list.append(cell_C)
        
        # Iterate over neighbors of C
        for face in cell_C.faces:
            neighbor_B_id = face['neighbor_id']
            if neighbor_B_id is None:
                continue
            neighbor_B = next(c for c in cells if c.id == neighbor_B_id)
            
            # Check if C <vp B
            N = face['normal']
            P_face = face['point']
            P_C = cell_C.center
            P_V = viewpoint
            
            side_C = dot(P_C - P_face, N)
            side_V = dot(P_V - P_face, N)
            
            if (side_C > 0 and side_V > 0) or (side_C < 0 and side_V < 0):
                # C and V are on the same side -> C occludes B -> C <vp B
                # Edge C -> B. Decrement incount of B.
                neighbor_B.incount -= 1
                if neighbor_B.incount == 0:
                    queue.append(neighbor_B)
                    
    # Step 5: Check for cycles
    if len(output_list) < n:
        raise ValueError("Visibility cycle detected. Grid may not be convex or requires subdivision.")
        
    return output_list

def dot(u, v):
    """Standard dot product for 3D vectors."""
    return u[0]*v[0] + u[1]*v[1] + u[2]*v[2]
```

### Parameters and Edge Cases
*   **Convexity:** The algorithm assumes the entire grid volume is convex [S1][S2]. If the grid is non-convex, the algorithm may fail because a viewing ray might exit the grid and re-enter it [S1][S2].
*   **Boundary Cells:** Cells with external faces are called boundary cells [S2]. The basic algorithm does not account for rays crossing external gaps [S1][S2]. For non-convex grids, additional $O(b^2)$ tests between boundary cells are required to add directed edges across gaps [S1][S2].
*   **Visibility Cycles:** If the queue empties before all cells are processed, a cycle exists [S1][S2]. In a strictly convex grid with a valid viewpoint, this should not occur [S1][S2].
*   **Precision:** Geometric tests involving dot products and point-in-halfspace checks are sensitive to floating-point precision [G]. Degenerate cases (viewpoint exactly on a face plane) require epsilon-based tolerance checks [G].
*   **Parameters:** Typical values for parameters are Not given in the sources.

## Subtopics

### [Visibility Ordering for Curvilinear Grids](6700-visibility-ordering-for-curvilinear-grids.md)
An algorithm to generate a visibility ordering for cells in a curvilinear grid (hexahedra) to support semi-transparent rendering, using a combination of undirected breadth-first search and directed depth-first searches on the adjacency graph.

## Operating characteristics
*   **Time Complexity:** $O(n)$ for convex grids with adjacency structures, where $n$ is the number of cells [S1][S2]. The initialization and graph building steps process each cell twice and each directed edge once [S2]. The while loop treats each cell and each edge once [S2]. Since each cell has a maximum of six faces, the total number of edges is proportional to $n$ [S1][S2].
*   **Extended Complexity:** For non-convex grids, the extended topological sort complexity is $O(n + b^2)$, where $b$ is the number of boundary cells [S1]. An optimized implementation by Silva et al. (1998) reduces this to $O((b + I)\log_2 b + n)$, where $I$ is the number of crossings among exterior face edges [S1].
*   **Space Complexity:** $O(n)$ to store the graph (incounts) and the queue/output list [S2].
*   **Strengths:** Extremely fast for convex grids [S1][S2]. No complex geometric intersection tests are needed during the sort phase, only local face tests [S1]. If the geometry does not change between views, the adjacency graph and plane equations can be reused, requiring only edge direction updates per viewpoint [S1].
*   **Limitations:** Only applicable to convex grids of convex cells [S1][S2]. Fails or requires augmentation for non-convex grids [S1][S2]. Does not handle self-occlusion or complex internal structures that break convexity [S1].
*   **When to Use:** Ideal for structured grids (e.g., regular hexahedral meshes) where the domain is convex and the viewpoint is external or the grid is treated as a convex hull [S1][S2].
*   **When Not to Use:** Non-convex domains, unstructured grids with non-convex cells, or when high precision is required for complex occlusion scenarios not covered by local face relations [S1][S2].

## Background
*   **Origin:** The algorithm is described in the paper "Volume rendering for curvilinear and unstructured grids" by Comba et al. [S1][S2].
*   **History:** It was developed as a faster alternative to $O(n^2)$ pairwise occlusion tests for volume rendering on curvilinear and unstructured grids [S1][S2]. The method leverages the topological properties of convex grids to reduce the sorting problem to a graph topological sort [S1][S2]. Edelsbrunner (1989) proved that for a Delaunay tetrahedralization of scattered data points, no visibility cycles are possible [S1].
*   **Usage:** Used in volume rendering pipelines for scientific visualization where the data domain is convex and structured or curvilinear grids are used [S1][S2].

## Sources
[S1] Approximate volume rendering for curvilinear and unstructured grids by hardware-assisted polyhedron projection, https://doi.org/10.1002/(sici)1098-1098(2000)11:1<53::aid-ima6>3.0.co;2-7, fetched 2026-10-25
[S2] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30
