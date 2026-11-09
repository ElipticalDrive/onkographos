# Visibility Ordering for Curvilinear Grids

Also called: BFS/DFS Adjacency Graph Sort for Curvilinear Grids, Curvilinear Grid Visibility Ordering

## Overview
This method generates a back-to-front visibility ordering for cells in a curvilinear grid (specifically hexahedra) to support semi-transparent volume rendering [S1, unofficial]. It solves the problem of determining the correct draw order for non-convex or complex connected volumes by combining an undirected breadth-first search (BFS) with directed depth-first searches (DFS) on the grid's adjacency graph [S1, unofficial]. Unlike its parent method, which assumes convexity, this approach handles potentially non-convex volumes by deriving a directed "immediately occludes" relation from screen-space geometry, allowing it to work on general curvilinear grids without decomposing cells into simpler primitives [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A curvilinear grid defined by hexahedral cells. The grid topology is implicit; edges and faces are determined by arithmetic on cell indices rather than explicit adjacency structures [S1, unofficial]. The viewpoint position and projection matrix are required to compute screen-space coordinates.
*   **Output:** A `visNumber` assigned to each cell, representing its position in the back-to-front rendering order.

**Data Structures:**
*   **Cell State:** Each cell maintains a `marked` boolean flag and an optional `visNumber` integer. Initially, all cells are `marked = false` and `visNumber` is undefined.
*   **FIFO Queue:** A first-in-first-out queue storing cell indices, used for the BFS phase.
*   **Recursion Stack:** Used implicitly by the DFS function.

**Definitions:**
*   **Immediately Occludes:** Cell $A$ immediately occludes Cell $B$ if they share a face and $A$ is closer to the viewer than $B$ in screen space. This is determined by the $z$-component of the shared face normal in screen space [S1, unofficial]. Specifically, if the normal points towards the viewer relative to the shared face, the cell on the "back" side occludes the cell on the "front" side.
*   **Uphill Neighbor:** A neighbor that does *not* immediately occlude the current cell (i.e., it is further back or laterally adjacent in a way that doesn't create a direct occlusion edge from current to neighbor).

**Algorithm Steps:**
1.  **Initialization:**
    *   Find the cell $C_{start}$ that contains the vertex farthest from the viewpoint (minimum $z$ in screen space) [S1, unofficial].
    *   Initialize an empty FIFO queue.
    *   Enqueue $C_{start}$ into the FIFO queue.
    *   Set `nextNum = 0`.
    *   Mark all cells as unmarked.

2.  **Main Loop:**
    *   While the FIFO queue is not empty:
        *   Dequeue the front cell, denoted as `nextCell`.
        *   If `nextCell` is not marked:
            *   Call `depthFirstSearch(nextCell, nextNum)`, which returns a new number `newNum`.
            *   Update `nextNum = newNum`.

3.  **Depth-First Search (DFS) Function:**
    *   Input: `cell` (current cell index), `nextNum` (current counter).
    *   Mark `cell` as marked.
    *   For each `neighbor` adjacent to `cell` (sharing a face):
        *   If `neighbor` is not marked:
            *   Check if `neighbor` immediately occludes `cell`.
            *   If **yes** (`neighbor` is in front of `cell`):
                *   Enqueue `neighbor` into the FIFO queue.
                *   Do **not** recurse into `neighbor` from this DFS call.
            *   If **no** (`neighbor` is behind or not directly occluding):
                *   Recursively call `depthFirstSearch(neighbor, nextNum)`, yielding `newNum`.
                *   Update `nextNum = newNum`.
        *   Else if `neighbor` is marked but has no `visNumber`:
            *   Raise Error: Cycle in visibility order. (Note: This case has never been observed in practice [S1, unofficial]).
        *   Else if `neighbor` is marked and has a `visNumber`:
            *   Do not visit (already processed).
    *   Assign `visNumber[cell] = nextNum`.
    *   Return `nextNum + 1`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def visibility_ordering(curvilinear_grid, viewpoint, projection_matrix):
    """
    Generates visibility ordering for hexahedral cells in a curvilinear grid.
    """
    num_cells = curvilinear_grid.num_cells()
    vis_numbers = [-1] * num_cells
    marked = [False] * num_cells
    fifo_queue = []
    
    # Step 1: Find starting cell (farthest vertex)
    # Note: Requires iterating all vertices of all cells to find min screen-space z
    start_cell = find_farthest_vertex_cell(curvilinear_grid, viewpoint, projection_matrix)
    
    fifo_queue.append(start_cell)
    next_num = 0
    
    while fifo_queue:
        next_cell = fifo_queue.pop(0)  # Dequeue front
        
        if not marked[next_cell]:
            new_num = depth_first_search(next_cell, next_num, curvilinear_grid, marked, fifo_queue)
            next_num = new_num
            
    return vis_numbers

def depth_first_search(cell, next_num, curvilinear_grid, marked, fifo_queue):
    """
    Recursive DFS to assign visibility numbers.
    """
    marked[cell] = True
    
    # Get adjacent cells sharing a face
    neighbors = get_adjacent_cells(cell, curvilinear_grid)
    
    for neighbor in neighbors:
        if not marked[neighbor]:
            # Determine occlusion direction
            # Compute shared face normal in screen space
            # If neighbor immediately occludes cell (neighbor is closer to viewer)
            if neighbor_immediately_occludes(cell, neighbor, curvilinear_grid):
                # Push uphill neighbor to FIFO for later processing
                fifo_queue.append(neighbor)
            else:
                # Neighbor is behind or not directly occluding; recurse
                new_num = depth_first_search(neighbor, next_num, curvilinear_grid, marked, fifo_queue)
                next_num = new_num
        else:
            if vis_numbers[neighbor] == -1:
                # Cycle detected
                raise ValueError("Cycle in visibility order")
            # Else: already processed, skip
            
    vis_numbers[cell] = next_num
    return next_num + 1

def neighbor_immediately_occludes(cell_a, cell_b, curvilinear_grid):
    """
    Returns True if cell_b immediately occludes cell_a.
    Determined by the z-component of the shared face normal in screen space.
    """
    # 1. Identify shared face between cell_a and cell_b
    # In curvilinear hex grids, this is determined by index arithmetic
    shared_face = get_shared_face(cell_a, cell_b, curvilinear_grid)
    
    # 2. Compute face normal in world space
    normal_world = compute_face_normal(shared_face, curvilinear_grid)
    
    # 3. Transform normal to screen space
    # Note: Normals transform via the inverse-transpose of the model-view matrix,
    # then projected. For simple orthographic or perspective, check z-component
    # of the transformed normal relative to the view direction.
    # Simplified: If the normal points towards the viewer, the cell on the
    # "back" side of the face occludes the cell on the "front" side.
    # Specifically, if the z-component of the screen-space normal indicates
    # cell_b is in front of cell_a.
    
    # Implementation detail: Compute the vector from cell_a centroid to cell_b centroid
    # in screen space. If this vector points in the same general direction as
    # the view vector (or specifically, if the shared face normal's z-component
    # implies cell_b is closer), then cell_b occludes cell_a.
    
    # Based on source: "The z component in screen space of the shared face normal
    # determines which cell occludes the other."
    screen_normal_z = transform_normal_to_screen_z(normal_world)
    
    # If the normal points towards the viewer (positive z in some conventions,
    # or negative depending on coordinate system), the cell on the opposite
    # side of the normal is the "back" cell.
    # Standard convention: If normal points out of cell_a, and z > 0,
    # then cell_b is in front of cell_a?
    # Let's stick to the source's logic: "neighbor immediately occludes cell"
    # if the relation holds.
    
    # Heuristic implementation based on source description:
    # If the shared face normal's screen-space z-component indicates that
    # cell_b is closer to the camera than cell_a.
    return screen_normal_z > 0  # Placeholder for exact sign convention
```

**Parameters and Edge Cases:**
*   **Starting Cell:** The choice of the farthest vertex cell is critical for the BFS initialization. If multiple cells share the farthest vertex, any one suffices.
*   **Cycles:** The algorithm detects cycles if a marked neighbor lacks a `visNumber`. The source notes this has never happened in practice [S1, unofficial].
*   **Non-Convexity:** The algorithm is designed for connected, possibly non-convex volumes [S1, unofficial]. It relies on the local "immediately occludes" relation, which may not capture global occlusions, but the BFS/DFS combination propagates the ordering correctly for rendering purposes.

## Subtopics

## Operating characteristics
*   **Time Complexity:** Linear, $O(N)$, where $N$ is the number of cells. The algorithm performs a single pass over the adjacency graph using BFS and DFS [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing cell states (`marked`, `visNumber`) and the FIFO queue. No auxiliary adjacency structures are needed for curvilinear grids, as topology is derived from index arithmetic [S1, unofficial].
*   **Performance:** On an SGI 4D/VGX, the algorithm orders approximately 60,000 hexahedra per second, comparable to the 60,000 tetrahedra per second reported for tetrahedral decomposition [S1, unofficial].
*   **Strengths:**
    *   Avoids the 6x cell multiplication cost of tetrahedral decomposition, reducing ordering overhead by a factor of ~5 [S1, unofficial].
    *   Leverages implicit topology of curvilinear grids, eliminating the need for explicit adjacency graph construction [S1, unofficial].
*   **Limitations:**
    *   Requires screen-space normal computation for every shared face.
    *   Relies on the "immediately occludes" heuristic; while robust in practice, it lacks a formal theoretical guarantee for all non-convex cases [S1, unofficial].
    *   Hexahedra are more complex to render than tetrahedra, trading ordering efficiency for rendering complexity [S1, unofficial].

## Background
The method was described in a 1993 NASA technical report by Williams, focusing on the rapid exploration of curvilinear grids using direct volume rendering [S1, unofficial]. It builds on the well-known linear-time topological sorting algorithm for directed acyclic graphs [MHC90, Wil92b]. The approach was developed as a simpler alternative to heuristics for non-convex volumes, taking advantage of the undirected adjacency graph of hexahedral cells and directing it based on screen-space occlusion relations [S1, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19930015755: Rapid exploration of curvilinear grids using direct volume rendering, https://archive.org/details/NASA_NTRS_Archive_19930015755, fetched 2026-10-26 (unofficial)
