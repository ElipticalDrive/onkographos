# LOD Tree Search
Also called: OctTree
## Overview
LOD Tree Search is a traversal algorithm for multidimensional search trees (specifically 3D binary trees keyed by minimum value, maximum value, and Level-of-Detail) to efficiently retrieve voxel cells intersecting an isosurface at a specific resolution [S2, unofficial]. It solves the problem of finding geometric primitives for isosurface extraction without processing the entire dataset, by pruning branches that cannot contain the target isovalue or do not match the desired LOD [S2, unofficial].
## How it works
**Inputs:**
*   `branch`: The root node or subtree of the LOD Tree.
*   `desired_lod`: The target Level-of-Detail integer.
*   `isovalue`: The scalar threshold defining the isosurface.
*   `center`: Coordinate center for distance calculation (default 0,0,0) [S1, unofficial].
*   `range`: Distance in metres from `center` for switching LOD (default 20) [S1, unofficial].
*   `highRes` / `lowRes`: Geometry instances for inside/outside range rendering [S1, unofficial].

**Outputs:**
*   A list of cells (nodes) that satisfy the LOD and isovalue intersection criteria [S2, unofficial].

**Data Structures:**
The tree is a 3D binary search tree. Each node `c` stores:
*   `min_val`: The minimum data value in the cell's domain.
*   `max_val`: The maximum data value in the cell's domain.
*   `lod`: The Level-of-Detail (resolution) key for this cell.
*   `left`: Pointer to the left child (smaller keys).
*   `right`: Pointer to the right child (larger keys) [S2, unofficial].

**Algorithm Steps:**
1.  **Check LOD Mismatch (Too High):** If `c.lod > desired_lod`, the current cell is too detailed. The algorithm recurses into the `left` branch, which contains cells with smaller keys (including potentially smaller LODs) [S2, unofficial].
2.  **Check LOD Mismatch (Too Low):** If `c.lod < desired_lod`, the current cell is too coarse. The algorithm recurses into the `right` branch, which contains cells with larger keys (including potentially larger LODs) [S2, unofficial].
3.  **Check LOD Match:** If `c.lod == desired_lod`:
    a.  **Check Isosurface Intersection:** Verify if the isosurface passes through the cell using the bounding box of values:
        $$ \text{min\_val} < \text{isovalue} \quad \land \quad \text{max\_val} > \text{isovalue} $$
    b.  **Add Cell:** If the condition is true, add `c` to the output list [S2, unofficial].
    c.  **Search Right:** Recursively call `search_minimum` on `c.right` to find other cells at the same LOD level [S2, unofficial].
    d.  **Search Left:** Recursively call `search_minimum` on `c.left` to find other cells at the same LOD level [S2, unofficial].
4.  **Distance-Based LOD Switching:** If the viewer distance to `center` is within `range`, render `highRes` children; otherwise render the `lowRes` object [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def search_lod(branch, desired_lod, isovalue, output_list):
    """Traverses the LOD tree to find cells intersecting the isosurface at the desired LOD."""
    if branch is None:
        return
    if branch.lod > desired_lod:
        search_minimum(branch.left, desired_lod, isovalue, output_list)
    elif branch.lod < desired_lod:
        search_minimum(branch.right, desired_lod, isovalue, output_list)
    else:
        if branch.min_val < isovalue and branch.max_val > isovalue:
            output_list.append(branch)
        search_minimum(branch.right, desired_lod, isovalue, output_list)
        search_minimum(branch.left, desired_lod, isovalue, output_list)

def search_minimum(branch, desired_lod, isovalue, output_list):
    """Handles traversal logic, pruning branches where min/max bounds cannot contain the isovalue."""
    if branch is None:
        return
    if branch.min_val >= isovalue or branch.max_val <= isovalue:
        return
    search_lod(branch, desired_lod, isovalue, output_list)
```

**Optimizations and Pitfalls:**
*   **Min Key Pruning:** If `branch.min_val < isovalue`, all cells in the left branch (which have smaller keys) will also have `min_val < isovalue`. The algorithm can skip testing the minimum key condition for those nodes, though it must still check LOD and max bounds [S2, unofficial].
*   **Max Key Pruning:** Similarly, if the maximum key condition is met for a branch, testing can stop for that branch's specific key checks [S2, unofficial].
*   **LOD Boundary Optimization:** If the `desired_lod` is the greatest or least possible value in the tree, and the search moves to a smaller (left) subtree of an already minimum LOD node, LOD key testing can stop. This is critical because the search is most time-consuming at the lowest level-of-detail [S2, unofficial].
*   **Precision:** The intersection test relies on strict inequalities (`<` and `>`). Edge cases where `min_val` or `max_val` exactly equals `isovalue` are not explicitly defined in the source as intersections, implying they are excluded [S2, unofficial].

## Subtopics

### [LODTree Construction](16369-lodtree-construction.md)
A parallel divide-and-conquer algorithm that builds the multidimensional search tree used by LOD Tree Search. It partitions data using three keys—minimum value, maximum value, and LOD—to create a 3D binary tree structure. Choose this when you need to efficiently build the underlying data structure for structured grids.

### Hierarchical LOD Navigation and Rendering
A variant that manages a hierarchy of 3D Gaussians for navigation and rendering. When navigating, it checks if a merged node's contribution exceeds $\tau_h = 1$; if true, it reloads finer Gaussians and updates coarse parent nodes, then reloads children and removes parents from the active set [S3]. For rendering, it starts at root nodes and descends until nodes are finer than $\tau = 1$ or a leaf is reached, continuously checking parents for $S_i < \tau$ to potentially switch to coarser nodes and offload Gaussians [S3]. This traversal takes 5 to 30 ms per step depending on size and is computed asynchronously with double buffering to avoid visual stutter [S3].

## Operating characteristics
*   **Time Complexity:** Depends on the distribution of data and the density of the isosurface. Pruning based on min/max bounds and LOD keys reduces traversal significantly, though worst-case performance scales with tree depth and data sparsity [S2, unofficial].
*   **Memory Complexity:** Linear with respect to the number of nodes in the LOD Tree, as each cell stores min, max, and LOD keys alongside child pointers [S2, unofficial].
*   **Strengths:** Efficiently isolates cells at a specific resolution, avoiding the cost of processing higher or lower LODs. Key boundary optimizations reduce unnecessary checks [S2, unofficial].
*   **Limitations:** Performance degrades if the isosurface intersects many cells across all LODs or if the tree is unbalanced. The algorithm assumes a pre-constructed 3D binary search tree [S2, unofficial].
*   **When to Use:** When extracting isosurfaces from large voxel datasets where only a specific level of detail is required, and the data is structured in a multikey search tree [S2, unofficial].
*   **When Not to Use:** When the isosurface is sparse and scattered across many LODs, or when the tree construction cost outweighs the search savings for a single query [S2, unofficial].

## Background
The method originates from the DYNAISOLOD system, which generates triangles on a cell-by-cell basis using the Marching Cubes algorithm for each cell found [S2, unofficial]. It was developed to handle dynamic isosurface extraction and level-of-detail in voxel space [S2, unofficial]. The X3D `OctTree` node provides a complementary distance-based LOD switching mechanism for volume rendering, allowing users to define high-resolution and low-resolution geometry instances based on viewer proximity to a coordinate center [S1, unofficial].

## Sources
[S1] DTIC ADA469447: Development of an Amendment to X3D to Create a Standard Specification of Medical Image Volume Rendering, Segmentation, and Registration, https://archive.org/details/DTIC_ADA469447, fetched 2026-10-26 (unofficial)
[S2] Dynamic Isosurface Extraction and Level-of-Detail in Voxel Space, https://digital.library.unt.edu/ark:/67531/metadc694729/?q=marching cubes, fetched 2026-10-28 (unofficial)
[S3] Immediate 3D Gaussian Splat Reconstruction of Unordered Input with Global Consistency, https://doi.org/10.1145/3799902.3811167, fetched 2026-10-26
