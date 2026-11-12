# Multi-Level Array Grid

Also called: Classic multi-level array grid

## Overview
The Multi-Level Array Grid is a hierarchical acceleration structure for ray tracing that recursively subdivides the scene bounding box into uniform grid cells based on item density [S1, unofficial][S2, unofficial]. It solves the problem of efficient spatial traversal by replacing large empty regions with null pointers and refining occupied regions into finer grids [S1, unofficial][S2, unofficial]. Unlike sibling methods that may use hashing or fixed-depth macrocells, this method uses a fixed maximum depth of 2 and a specific heuristic for grid dimensions to balance memory usage and traversal speed [S1, unofficial][S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   A set of scene primitives (items).
*   The scene bounding box, defined by minimum and maximum coordinates in 3D space.

**Outputs:**
*   A root `GridNode` representing the hierarchical grid structure.

### Data Structures
The structure consists of two main node types, `InnerNode` and `GridNode`, defined as follows [S1, unofficial][S2, unofficial]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```c
struct InnerNode {
    float min[3];       // Minimum extents of the node's bounding box
    float max[3];       // Maximum extents of the node's bounding box
    int dim[3];         // Grid dimensions (Mx, My, Mz)
    float delta[3];     // Cell size in each dimension (redundant, cached for performance)
    GridNode** cells;   // Array of pointers to child GridNodes
};

struct GridNode {
    int index;          // Low bit: 1=Inner, 0=Leaf. High bits: Item count (for leaves)
    union {
        struct {
            int* items; // Pointer to array of primitive indices (Leaf)
        } leaf;
        struct {
            InnerNode* interior; // Pointer to child InnerNode (Inner)
        } inner;
    };
};
```
*Note: The `index` field encodes the node type in the least significant bit. For leaf nodes, the remaining high bits store the number of items [S1, unofficial][S2, unofficial].*

### Construction Algorithm
The construction proceeds recursively [S1, unofficial][S2, unofficial]:

1.  **Compute Bounding Box:** Calculate the scene's global bounding box. Let $S_i$ be the size of the bounding box in dimension $i \in \{x, y, z\}$. Let $N$ be the total number of primitives in the current node.
2.  **Initialize Root:** Create the first node as a leaf containing all $N$ scene items.
3.  **Recursive Processing:** For each node, apply the following logic:
    *   **Termination Conditions:** If the number of items in the node is less than 8, or if the current grid depth is greater than 2, the node remains a leaf. Recursion stops for this branch [S1, unofficial][S2, unofficial].
    *   **Expansion:** Otherwise, expand the node to an inner node.
        1.  **Compute Dimensions:** Calculate the grid dimensions $M_i$ for each axis using the heuristic attributed to Woo (1992) [S1, unofficial][S2, unofficial]:
            $$M_i = \frac{S_i}{\max\{S_j\}} \sqrt[3]{\rho N} \quad (i \in \{x, y, z\})$$
            Where:
            *   $S_i$ is the bounding box size in dimension $i$.
            *   $\max\{S_j\}$ is the maximum dimension size among $x, y, z$.
            *   $\rho$ is a density factor, typically 4 [S1, unofficial][S2, unofficial].
            *   $N$ is the number of items in the current node.
        2.  **Compute Delta:** Calculate cell size $\delta_i = S_i / M_i$ for each dimension.
        3.  **Allocate Cells:** Allocate an array `cells` of size $M_x \times M_y \times M_z$.
        4.  **Populate Cells:**
            *   Compute the list of items falling into each cell.
            *   If a cell's item list is empty, set the corresponding pointer in `cells` to `null`.
            *   If a cell's item list is not empty, initialize a new `GridNode` as a leaf containing those items.
        5.  **Recurse:** For each non-null child cell, recursively process the new leaf node.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_multi_level_array_grid(items, depth=0):
    """
    Recursively builds the multi-level array grid.
    items: list of primitive indices
    depth: current recursion depth
    """
    n = len(items)
    
    # Termination conditions
    if n < 8 or depth > 2:
        # Create leaf node
        # Store item count in high bits of index, 0 in low bit
        index = (n << 1) | 0
        return GridNode(index=index, leaf_items=items)

    # Compute bounding box of current items
    bbox_min, bbox_max = compute_bounding_box(items)
    S = [bbox_max[i] - bbox_min[i] for i in range(3)]
    max_S = max(S)
    
    # Heuristic for grid dimensions
    rho = 4
    M = [int((S[i] / max_S) * (rho * n) ** (1/3)) for i in range(3)]
    
    # Compute delta (cell size)
    delta = [S[i] / M[i] for i in range(3)]
    
    # Allocate cells array
    # Size is M[0] * M[1] * M[2]
    cells = [None] * (M[0] * M[1] * M[2])
    
    # Assign items to cells
    for item_idx in items:
        pos = get_item_position(item_idx) # Helper to get center or representative point
        cell_coords = [0, 0, 0]
        for i in range(3):
            # Clamp to ensure valid index
            cell_coords[i] = min(int((pos[i] - bbox_min[i]) / delta[i]), M[i] - 1)
        
        cell_idx = (cell_coords[0] * M[1] + cell_coords[1]) * M[2] + cell_coords[2]
        
        if cells[cell_idx] is None:
            cells[cell_idx] = []
        cells[cell_idx].append(item_idx)
    
    # Recurse and build inner node
    inner_cells = []
    for cell_items in cells:
        if cell_items is None:
            inner_cells.append(None)
        else:
            # Recursively build child
            child_node = build_multi_level_array_grid(cell_items, depth + 1)
            inner_cells.append(child_node)
            
    inner_node = InnerNode(
        min=bbox_min,
        max=bbox_max,
        dim=M,
        delta=delta,
        cells=inner_cells
    )
    
    # Create root inner node wrapper
    # Index low bit 1 indicates inner node
    index = 1 
    return GridNode(index=index, inner=inner_node)
```

### Parameters
*   **Leaf Threshold:** 8 items. Nodes with fewer items remain leaves [S1, unofficial][S2, unofficial].
*   **Max Depth:** 2. Recursion stops if depth exceeds 2 [S1, unofficial][S2, unofficial].
*   **Density Factor ($\rho$):** Typically 4 [S1, unofficial][S2, unofficial].

### Edge Cases and Pitfalls
*   **Empty Cells:** Represented as `null` pointers to save memory. Traversal must check for `null` before dereferencing [S1, unofficial][S2, unofficial].
*   **Redundant Data:** `delta` is stored in `InnerNode` but can be computed from `min`, `max`, and `dim`. It is cached for performance [S1, unofficial][S2, unofficial].
*   **One-Level Grid:** Setting the grid depth limit to 1 simulates a single-level grid [S1, unofficial][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Construction is $O(N)$ assuming uniform distribution, but depends on item distribution. Traversal is faster than one-level grids because ray-triangle intersections are reduced by skipping empty subtrees [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** Memory efficient for sparse scenes due to null pointers for empty cells. However, the fixed array structure for inner nodes can waste space if grids are small or irregularly populated compared to hashed variants [S1, unofficial][S2, unofficial].
*   **Performance:** Profiling indicates that grid traversal dominates render time. Multi-level grids spend significantly less time on ray-triangle intersections compared to one-level grids [S1, unofficial][S2, unofficial].
*   **Strengths:** Simple implementation, good memory conservation for sparse data, effective reduction of intersection tests.
*   **Limitations:** Fixed depth limit (2) may not be optimal for all scenes. The heuristic for grid dimensions may not always produce optimal load balancing.

## Background
The Multi-Level Array Grid is described in the paper "Multi-level Grid Strategies for Ray Tracing" (2010) as a classic implementation used for performance comparison against newer strategies like the Multi-Level Hashed Grid [S1, unofficial][S2, unofficial]. The dimension heuristic used is attributed to Woo (1992) [S1, unofficial][S2, unofficial]. It serves as a baseline for evaluating more advanced hierarchical grid structures.

## Sources
[S1] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2010/28153/, fetched 2026-10-24 (unofficial)
[S2] PDF Multi-level Grid Strategies for Ray Tracing, https://pdfs.semanticscholar.org/f922/40132d0d7340ff97ac4074b0a6906fb44c44.pdf, fetched 2026-10-25 (unofficial)
