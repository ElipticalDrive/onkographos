# Multi-Level Array Grid

Also called: Classic multi-level array grid

## Overview
The Multi-Level Array Grid is a hierarchical acceleration structure for ray tracing that recursively subdivides the scene bounding box into a nested grid of nodes based on item density [S1, S2, unofficial]. It serves as a baseline comparison for more advanced sparse or hashed grid implementations, offering a dense, array-based representation where empty cells are explicitly nullified to save memory [S1, S2, unofficial]. Unlike sibling methods that may use hashing or fixed macrocells, this method uses a deterministic, memory-conservative heuristic to determine grid dimensions at each level, ensuring a predictable memory layout at the cost of potential fragmentation in sparse scenes [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of scene primitives (items) and their bounding boxes.
*   **Output:** A root `GridNode` representing the hierarchical structure, enabling ray traversal.

**Data Structures:**
The structure consists of two primary C-style structs, `InnerNode` and `GridNode` [S1, S2, unofficial].

1.  `InnerNode`: Represents a non-leaf node containing a grid of pointers.
    *   `float min[3]`: Minimum extent of the node's bounding box.
    *   `float max[3]`: Maximum extent of the node's bounding box.
    *   `int dim[3]`: Number of cells along each axis (x, y, z).
    *   `float delta[3]`: Size of each cell along each axis (redundant, computed from min/max and dim, but cached for performance) [S1, S2, unofficial].
    *   `GridNode** cells`: A 1D array of pointers to child nodes, indexed linearly. The size is `dim[0] * dim[1] * dim[2]` [S1, S2, unofficial].

2.  `GridNode`: A union-like structure distinguishing between leaf and inner nodes via bit-packing in the `index` field [S1, S2, unofficial].
    *   `int index`:
        *   **Low bit:** `1` indicates an inner node, `0` indicates a leaf node.
        *   **High bits:** For leaf nodes, stores the count of items contained in the leaf [S1, S2, unofficial].
    *   **Union Payload:**
        *   If leaf (low bit 0): `int* items` pointing to an array of primitive indices [S1, S2, unofficial].
        *   If inner (low bit 1): `InnerNode* interior` pointing to the `InnerNode` structure [S1, S2, unofficial].

**Construction Algorithm:**
The construction is recursive. The process determines grid dimensions using a heuristic attributed to Woo (1992) [S1, S2, unofficial].

1.  Compute the global scene bounding box $B$. Let $S_i$ be the size of $B$ in dimension $i \in \{x, y, z\}$. Let $N$ be the total number of primitives in the current node's scope.
2.  Initialize the root as a leaf node containing all $N$ primitives.
3.  Recursively process each node:
    a.  **Termination Check:** If the number of items in the node is less than 8, or the current grid depth exceeds 2, the node remains a leaf. Store the item indices in the leaf's `items` array and update the `index` high bits with the count. Stop recursion for this branch [S1, S2, unofficial].
    b.  **Expansion:** If termination conditions are not met, expand the node to an inner node.
    c.  **Dimension Calculation:** Compute the grid dimensions $M_i$ for each axis using the formula:
        $$M_i = \frac{S_i}{\max\{S_x, S_y, S_z\}} \sqrt[3]{\rho N}$$
        Where $\rho$ is a density factor, typically 4 [S1, S2, unofficial].
    d.  **Delta Calculation:** Compute cell size $\delta_i = S_i / M_i$ for each axis.
    e.  **Allocation:** Allocate the `cells` array in the `InnerNode` with size $\prod M_i$. Initialize all pointers to `null` [S1, S2, unofficial].
    f.  **Item Assignment:** Iterate through all primitives in the node. For each primitive, determine which cell $(c_x, c_y, c_z)$ it overlaps. Add the primitive's index to that cell's list.
    g.  **Child Creation:** For each cell in the grid:
        i.  If the cell's item list is empty, leave the pointer as `null`.
        ii. If the item list is not empty, create a new `GridNode` with low bit `0` (leaf). Set the `items` pointer to the list of primitives and the high bits of `index` to the list size.
    h.  **Recursion:** For each newly created leaf child (non-null), recursively apply steps 3a-3g.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_multi_level_array_grid(primitives, depth=0):
    """
    Builds the Multi-Level Array Grid recursively.
    primitives: list of primitive objects with bounding boxes
    depth: current recursion depth
    returns: GridNode
    """
    # 1. Compute bounding box of current primitives
    if not primitives:
        return None # Should not happen if called correctly

    min_x = min(p.min[0] for p in primitives)
    min_y = min(p.min[1] for p in primitives)
    min_z = min(p.min[2] for p in primitives)
    max_x = max(p.max[0] for p in primitives)
    max_y = max(p.max[1] for p in primitives)
    max_z = max(p.max[2] for p in primitives)

    S_x = max_x - min_x
    S_y = max_y - min_y
    S_z = max_z - min_z
    max_S = max(S_x, S_y, S_z)
    N = len(primitives)

    # 2. Termination Check
    # Leaf threshold: < 8 items OR depth > 2
    if N < 8 or depth > 2:
        # Create Leaf Node
        # index: high bits = N, low bit = 0
        index = (N << 1) | 0
        items_array = [p.id for p in primitives] # Assuming primitives have an ID
        return GridNode(index=index, items=items_array)

    # 3. Expansion to Inner Node
    # Calculate dimensions M_i
    # rho = 4
    rho = 4
    root_cubed = rho * N
    root_cubed_val = root_cubed ** (1/3)
    
    M_x = int(round((S_x / max_S) * root_cubed_val)) if max_S > 0 else 1
    M_y = int(round((S_y / max_S) * root_cubed_val)) if max_S > 0 else 1
    M_z = int(round((S_z / max_S) * root_cubed_val)) if max_S > 0 else 1
    
    # Ensure at least 1 dimension
    M_x = max(1, M_x)
    M_y = max(1, M_y)
    M_z = max(1, M_z)

    # Calculate delta
    delta_x = S_x / M_x
    delta_y = S_y / M_y
    delta_z = S_z / M_z

    # Allocate cells array (1D list of None)
    total_cells = M_x * M_y * M_z
    cells = [None] * total_cells

    # 4. Assign primitives to cells
    # Helper to map point to cell index
    def get_cell_index(p):
        # Clamp to bounding box to avoid precision issues at edges
        # Assuming p.bbox is (min_x, min_y, min_z, max_x, max_y, max_z)
        # We check overlap. For simplicity, assume point-in-cell logic based on center or min.
        # Standard approach: floor((p.min - node.min) / delta)
        cx = int((p.min[0] - min_x) / delta_x)
        cy = int((p.min[1] - min_y) / delta_y)
        cz = int((p.min[2] - min_z) / delta_z)
        
        # Clamp indices to valid range [0, M-1]
        cx = max(0, min(cx, M_x - 1))
        cy = max(0, min(cy, M_y - 1))
        cz = max(0, min(cz, M_z - 1))
        
        return cz * M_x * M_y + cy * M_x + cx

    # Initialize cell lists
    cell_lists = [[] for _ in range(total_cells)]
    
    for p in primitives:
        idx = get_cell_index(p)
        cell_lists[idx].append(p.id)

    # 5. Create child nodes and recurse
    inner_node = InnerNode(
        min=[min_x, min_y, min_z],
        max=[max_x, max_y, max_z],
        dim=[M_x, M_y, M_z],
        delta=[delta_x, delta_y, delta_z],
        cells=cells
    )

    for i in range(total_cells):
        if cell_lists[i]:
            # Recurse for non-empty cells
            # Note: We need the actual primitive objects for the child, not just IDs.
            # In a real implementation, primitives would be passed by reference or ID lookup.
            # Here we assume we can retrieve them or they are passed in context.
            # For this pseudocode, we assume `primitives` is a lookupable list or we pass the subset.
            # To keep it self-contained, we assume we have access to the original list or a map.
            # Let's assume we filter the original primitives list for this cell.
            # This is a simplification; in C, pointers are used.
            
            # Re-fetch primitives for this cell (inefficient for pure Python, but illustrative)
            # In C, we would have passed the list of pointers.
            # We will assume a helper `get_primitives_by_ids(ids)` exists or we pass the subset.
            # To be strictly self-contained without external state, we must pass the subset.
            # However, the algorithm description says "item lists... initialized as leaf".
            # So we create a leaf with the items.
            
            # Create Leaf Node for this cell
            # The leaf contains the items. It does NOT recurse further if it's a leaf.
            # Wait, the algorithm says "Recurse".
            # "If the item list for that cell is not empty, the cell is initialized as a leaf node containing the items in question. Recurse."
            # This implies the child node is a GridNode which is a LEAF, but then we recurse ON THAT LEAF?
            # No, the recursion happens on the *node*. If the node is expanded, its children are processed.
            # The children are created as leaves first. Then we recurse on them.
            
            child_node = build_multi_level_array_grid_subset(cell_lists[i], depth + 1)
            inner_node.cells[i] = child_node
        else:
            inner_node.cells[i] = None

    # Create Inner GridNode
    # index: low bit = 1 (inner), high bits = 0 (or unused)
    index = 1 
    return GridNode(index=index, interior=inner_node)

def build_multi_level_array_grid_subset(primitive_ids, depth):
    """Helper to build subtree for a specific set of primitives."""
    # Retrieve primitive objects from global scope or passed context
    # For this self-contained example, we assume `all_primitives` is available
    # or we pass the objects directly.
    # Let's assume we pass the objects.
    pass

# Corrected recursive call structure to match C logic exactly:
def build_grid_recursive(primitive_objects, depth):
    N = len(primitive_objects)
    if N < 8 or depth > 2:
        index = (N << 1) | 0
        items = [p.id for p in primitive_objects]
        return GridNode(index=index, items=items)
    
    # ... compute M, delta, allocate cells ...
    # ... assign to cell_lists ...
    
    inner_node = InnerNode(...)
    
    for i in range(total_cells):
        if cell_lists[i]:
            # Recurse
            child = build_grid_recursive(cell_lists[i], depth + 1)
            inner_node.cells[i] = child
        else:
            inner_node.cells[i] = None
            
    index = 1
    return GridNode(index=index, interior=inner_node)
```

**Parameters:**
*   **Leaf Threshold:** 8 items. Nodes with fewer items remain leaves [S1, S2, unofficial].
*   **Depth Limit:** 2. Recursion stops if depth exceeds 2 [S1, S2, unofficial].
*   **Density Factor ($\rho$):** Typically 4 [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Cells:** Represented as `null` pointers in the `cells` array. Traversal must check for `null` before dereferencing [S1, S2, unofficial].
*   **Precision:** Cell indices are computed using floating-point division. Clamping indices to `[0, M_i - 1]` is necessary to handle primitives on the boundary of the bounding box [G].
*   **Redundant Data:** `delta` is stored in `InnerNode` but can be computed from `min`, `max`, and `dim`. It is cached for performance [S1, S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Construction is $O(N \cdot D)$ where $N$ is the number of primitives and $D$ is the depth (limited to 2). Traversal time is dominated by ray-grid intersection and ray-triangle intersection. Profiling indicates grid traversal dominates render time, and one-level grids spend twice as much time on ray-triangle intersections as multi-level grids [S2, unofficial].
*   **Memory Complexity:** Memory usage is proportional to the number of cells in the grid. Empty cells are nullified, reducing memory overhead compared to a fully dense grid. However, in sparse scenes, the fixed array structure of inner nodes can still be inefficient compared to hashed grids [S1, S2, unofficial].
*   **Strengths:** Simple implementation; predictable memory layout; fast traversal due to array indexing; effective reduction in ray-triangle intersections compared to flat grids [S1, S2, unofficial].
*   **Limitations:** Memory inefficiency in highly sparse scenes due to array allocation for inner nodes; fixed depth limit may not adapt to complex scene structures; heuristic dimensions may not be optimal for all distributions [S1, S2, unofficial].
*   **When to use:** As a baseline for comparison; scenes with moderate density where array access speed is prioritized over memory efficiency [S1, S2, unofficial].
*   **When not to use:** Highly sparse scenes where hashed grids or other sparse structures are preferred; scenes requiring deep recursion beyond depth 2 [S1, S2, unofficial].

## Background
The Multi-Level Array Grid was described in a 2010 paper by SCITEPRESS, used as a classic implementation for performance comparison against multi-level hashed grids [S1, S2, unofficial]. The dimension calculation heuristic is attributed to Woo (1992) [S1, S2, unofficial]. It is part of the "Multi-level Grid Structures" family, specifically the array-based variant, contrasting with the hashed variant [S1, S2, unofficial].

## Sources
[S1] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2010/28153/, fetched 2026-10-24 (unofficial)
[S2] PDF Multi-level Grid Strategies for Ray Tracing, https://pdfs.semanticscholar.org/f922/40132d0d7340ff97ac4074b0a6906fb44c44.pdf, fetched 2026-10-25 (unofficial)
