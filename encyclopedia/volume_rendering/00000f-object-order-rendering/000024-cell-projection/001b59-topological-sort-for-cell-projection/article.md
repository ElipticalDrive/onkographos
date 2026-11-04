# Topological Sort for Cell Projection

Also called: MPVO, MPVO algorithm, Meshed Polyhedra Visibility Ordering, Maximal Polyhedra Visibility Ordering, Exact Meshed Polyhedra Visibility Ordering, Exact Visibility Ordering for Cell Projection

## Overview
Topological Sort for Cell Projection (MPVO) is an $O(n)$ algorithm for computing a back-to-front visibility ordering of convex polyhedral cells in a convex grid [S1][S3, book]. It solves the occlusion ordering problem required for correct alpha-blending in object-order volume rendering by constructing a directed graph of pairwise occlusion relations between adjacent cells and performing a topological sort [S1][S3, book]. Unlike [Tetrahedral Volume Rendering](820-tetrahedral-volume-rendering.md) or [Projected Tetrahedra](877-projected-tetrahedra.md) which may rely on simpler centroid or distance-based heuristics, MPVO guarantees a correct visibility order for convex meshes by respecting the transitive occlusion chain along viewing rays [S1][S3, book]. It fails on non-convex meshes because it ignores occlusion across empty space between non-adjacent cells [S1][S3, book].

## How it works
**Inputs:** A convex grid of convex polyhedral cells; a viewpoint $V$ (for perspective) or a projection direction vector $\vec{d}$ (for orthogonal) [S1][S3, book].
**Outputs:** A list of cells sorted in back-to-front order, where an occluded cell precedes an occluder [S1][S3, book].

**Data Structures:**
1. **Cell Node:** Contains `incount` (integer, number of incoming edges), `outgoing` (list of cell references it occludes), and `incoming` (list of cell references that occlude it) [S1][S3, book].
2. **Directed Graph:** Implicitly defined by adjacency, where nodes are cells and edges represent occlusion relations [S1][S3, book].
3. **Queue:** A FIFO queue for cells with `incount == 0` [S1][S3, book].

**Definitions:**
*   Relation $B <_{vp} A$: Cell $B$ must be rendered before cell $A$ [S1]. This holds if the viewpoint $V$ is on the same side of the shared face $F$ between $A$ and $B$ as cell $A$ is [S1]. This implies $A$ occludes $B$, but $B$ cannot occlude $A$ [S1].
*   Arrow Direction: The direction is calculated using the plane equation for the face and the coordinates of the viewpoint (for perspective views) or a vector along the direction of projection (for orthogonal views) [S4, unofficial].

**Algorithm Steps:**
1. **Phase I: Construct Partial Ordering (Graph Construction)**
    *   Initialize all cells $C$: set $C.incount = 0$ [S1].
    *   For each interior face $f$ shared by cells $c_1$ and $c_2$:
        *   Determine the direction of occlusion using the face plane equation and viewpoint/projection vector [S4, unofficial].
        *   **Perspective View:** If $V$ and $c_1$ are on the same side of the plane of $f$, then $c_1$ occludes $c_2$ [S1][S3, book].
        *   **Orthogonal View:** If the projection direction $\vec{d}$ aligns with the outward normal of $c_1$ relative to $f$, $c_1$ is in front of $c_2$ [S4, unofficial].
        *   Add a directed edge from the occluded cell to the occluder [S1].
        *   Increment the occluder's `incount` [S1].
    *   Identify **source cells**: Cells with $incount == 0$ [S1]. Add these to the `queue` [S1].
2. **Phase II: Topological Sort (Kahn's Algorithm)**
    *   While `queue` is not empty:
        *   Dequeue a cell $C$ [S1].
        *   Add $C$ to the sorted output list [S1].
        *   For each neighbor $B$ in $C.outgoing$: decrement $B.incount$ [S1]. If $B.incount == 0$, enqueue $B$ [S1].
    *   If the queue empties before all cells are output, a visibility cycle exists [S1][S3, book].
    *   *Note:* An alternative Phase II uses Depth-First Search with `visited` and `cycle_detection` flags to output cells [S4, S5, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mpvo_sort(cells, interior_faces, viewpoint, is_orthogonal=False, projection_dir=None):
    # Phase I: Initialize and build graph
    for cell in cells:
        cell.incount = 0
        cell.outgoing = []
        cell.incoming = []
        
    for c1, c2, face in interior_faces:
        # Determine occlusion direction
        if is_orthogonal:
            n = face.normal
            if dot(projection_dir, n) > 0:
                occluder, occluded = c1, c2
            else:
                occluder, occluded = c2, c1
        else:
            # Perspective: V and c1 on same side of face plane -> c1 occludes c2
            n = face.normal
            P = face.center
            d_v = dot(n, viewpoint - P)
            d_c1 = dot(n, c1.center - P)
            if (d_v > 0 and d_c1 > 0) or (d_v < 0 and d_c1 < 0):
                occluder, occluded = c1, c2
            else:
                occluder, occluded = c2, c1
                
        # Add edge: occluded -> occluder
        occluded.outgoing.append(occluder)
        occluder.incoming.append(occluded)
        occluder.incount += 1
        
    # Identify source cells (incount == 0)
    queue = [c for c in cells if c.incount == 0]
    
    # Phase II: Topological Sort (Kahn's algorithm)
    sorted_output = []
    head = 0
    while head < len(queue):
        current_cell = queue[head]
        head += 1
        sorted_output.append(current_cell)
        
        for neighbor in current_cell.outgoing:
            neighbor.incount -= 1
            if neighbor.incount == 0:
                queue.append(neighbor)
                
    if len(sorted_output) < len(cells):
        raise ValueError("Visibility cycle detected")
        
    return sorted_output
```

**Edge Cases and Pitfalls:**
*   **Visibility Cycles:** If the graph contains a cycle, the topological sort fails [S3, book]. In the queue-based variant, this is detected if the queue empties before all cells are output [S1]. Cycles imply no global visibility order exists; cells must be subdivided or per-pixel sorting used [S1][S3, book].
*   **Non-Convex Grids:** If the grid is non-convex, viewing rays may leave the volume through an external face, cross empty space, and re-enter another cell [S1]. MPVO only considers shared faces, so it misses these cross-gap occlusions [S1][S3, book].
*   **Complexity Accounting:** Each cell is treated twice during initialization and once in the loop; each edge is treated once in initialization and once in the loop [S1].

## Subtopics

### [A-buffer visibility sort for non-convex grids](1057-a-buffer-visibility-sort-for-non-convex-grids.md)
Manages pixel-level intersection lists for exterior faces on non-convex grids, avoiding true visibility sort complexities while ensuring correct hardware rendering. Choose when exact ordering is needed but the mesh has concavities.

### [Power Distance Sort](1060-power-distance-sort.md)
Sorts based on power distance with small constant computations. Choose when the mesh is a Delaunay triangulation, as it is not guaranteed to be accurate otherwise.

### [Per-Pixel Sorting (General Variant)](1061-per-pixel-sorting-general-variant.md)
Sorts all faces projecting onto a pixel at render time. Choose when visibility cycles are expected, as a sort always exists per pixel center even without a global sort.

### [XMPVO](1363-xmpvo.md)
Augments MPVO by discovering exterior face relations using ray shooting queries at edge intersections and vertices of the projection complex. Choose when handling non-convex meshes requires exact ordering without per-pixel overhead.

### [MPVONC](1377-mpvonc.md)
Heuristic extension to nonconvex meshes sorting exterior cells by centroid distance. Choose for fast rendering of large non-convex datasets where real-time interaction is prioritized over exact correctness.

## Operating characteristics
*   **Time Complexity:** $O(n)$ for convex grids, where $n$ is the number of cells [S1][S3, book].
*   **Memory Complexity:** Linear space $O(n)$ for data structures [S3, book].
*   **Measured performance with hardware, resolution, and scene:** Not given in the sources.
*   **Scaling:** Linear scaling with cell count $n$ [S1][S3, book].
*   **Strengths:** Very fast, accurate for convex meshes, low computational overhead, detects visibility cycles [S3, book].
*   **Limitations:** Fails for non-convex meshes (requires extra gap relations or approximate methods) [S1][S3, book]. Requires convexity of cells and grid [S1][S3, book].
*   **When to use:** Convex grids of convex polyhedral cells where exact visibility ordering is required and performance is critical [S1][S3, book].
*   **When not to use:** Non-convex meshes, non-planar faces, or when real-time interaction with very large non-convex datasets is needed (use Approximate MPVONC or Per-Pixel Sorting) [S2, unofficial][S3, book].

## Background
The MPVO algorithm was described by Williams [27] and similarly by Max et al. [15] [S3, book][S4, unofficial]. It is based on the observation that for convex meshes, local occlusion relations across shared faces define a partial order that can be topologically sorted to yield a correct global visibility order [S3, book]. It is a foundational method for cell projection volume rendering [S3, book]. It is used in direct volume rendering and scientific visualization [S2, unofficial].

## Sources
[S1] Volume Rendering for Curvilinear and Unstructured Grids, https://www.sci.utah.edu/~csilva/papers/cgi2003.pdf, fetched 2026-10-27
[S2] Tetrahedral projection using vertex shaders, https://www.researchgate.net/publication/221247894_Tetrahedral_projection_using_vertex_shaders, fetched 2026-10-28 (unofficial)
[S3] Cell projection of meshes with non-planar faces, https://digital.library.unt.edu/ark:/67531/metadc1408947/?q=projected tetrahedra, fetched 2026-10-28 (book)
[S4] Image-space visibility ordering for cell projection volume rendering of unstructured data, https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
[S5] PDF 0078-0903-2 1..13, https://sci.utah.edu/~csilva/papers/tvcg2004cr.pdf, fetched 2026-10-28 (unofficial)
