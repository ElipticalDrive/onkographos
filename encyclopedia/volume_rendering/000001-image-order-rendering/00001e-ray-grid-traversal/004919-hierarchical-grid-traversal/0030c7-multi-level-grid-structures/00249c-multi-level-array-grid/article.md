# Multi-Level Array Grid

Also called: Classic multi-level array grid

## Overview
The Multi-Level Array Grid is a hierarchical acceleration structure for ray tracing that recursively subdivides the scene bounding box into a nested grid of nodes [S1, S2, unofficial]. It solves the problem of efficient spatial traversal by replacing pointer-chasing trees with contiguous array-based grids, optimizing for memory locality and reduced ray-primitive intersection counts [S1, S2, unofficial]. Unlike flat grids, it adapts resolution based on object density, and unlike tree structures, it uses fixed-size grid arrays at each level, allowing for faster traversal via integer arithmetic [S1, S2, unofficial]. It builds on the shared principles of hierarchical decomposition and ray-grid traversal defined in [Multi-level Grid Structures](6354-multi-level-grid-structures.md) [G].

## How it works
### Inputs and Outputs
- **Input**: A set of scene primitives and their bounding box [S1, S2, unofficial].
- **Output**: A root `GridNode` representing the hierarchical grid structure [S1, S2, unofficial].

### Data Structures
The structure consists of two main types of nodes: `InnerNode` and `GridNode` [S2, unofficial].

**InnerNode**: Represents a non-leaf grid level [S2, unofficial].
- `min[3]`: `float` array, minimum extents of the node's bounding box [S2, unofficial].
- `max[3]`: `float` array, maximum extents of the node's bounding box [S2, unofficial].
- `dim[3]`: `int` array, grid dimensions $(M_x, M_y, M_z)$ for this level [S2, unofficial].
- `delta[3]`: `float` array, size of each cell in the grid. Cached for performance; computable from `min`, `max`, and `dim` [S1, S2, unofficial].
- `cells`: `GridNode**`, a contiguous array of pointers to child nodes. The source specifies allocating this array with size $N$ [S1, S2, unofficial]. Empty cells are represented by `NULL` pointers to reduce memory requirements [S1, S2, unofficial].

**GridNode**: A union type representing either an inner node or a leaf [S2, unofficial].
- `index`: `int`. The low bit indicates the type: `1` for inner node, `0` for leaf [S1, S2, unofficial]. The high bits store the number of items in the leaf node [S1, S2, unofficial].
- `union`:
    - `leaf`: Contains `int* items`, a pointer to an array of primitive indices [S2, unofficial].
    - `inner`: Contains `InnerNode* interior`, a pointer to the `InnerNode` structure [S2, unofficial].

### Construction Algorithm
The construction is recursive, starting from the entire scene bounding box [S2, unofficial].

1.  **Initialize**: Compute the global scene bounding box. Create the root `GridNode` as a leaf node containing all scene primitives [S1, S2, unofficial].
2.  **Recursive Processing**: For each node, determine if it should be expanded into an inner node [S2, unofficial].
    -   **Termination Conditions**: Recursion stops if:
        -   The number of items in the node is less than 8 [S1, S2, unofficial].
        -   The grid depth exceeds 2 [S1, S2, unofficial].
    -   **Expansion**: If neither condition is met, the node becomes an inner node [S1, S2, unofficial].
3.  **Determine Grid Dimensions**: Calculate the dimensions $(M_x, M_y, M_z)$ for the new inner node using the heuristic attributed to Woo (1992) [S1, S2, unofficial]:
    $$M_i = \frac{S_i}{\max\{S_i\}} \sqrt[3]{\rho N} \quad \text{for } i \in \{x, y, z\}$$
    Where:
    -   $S_i$ is the size of the bounding box in dimension $i$ [S1, S2, unofficial].
    -   $\max\{S_i\}$ is the maximum dimension size among $x, y, z$ [S1, S2, unofficial].
    -   $\rho$ is a density factor, typically 4 [S1, S2, unofficial].
    -   $N$ is the number of primitives in the current node [S1, S2, unofficial].
4.  **Allocate and Populate**:
    -   Compute `delta` for each dimension from the bounding box size and grid dimensions [G].
    -   Allocate the `cells` array with size $N$ [S1, S2, unofficial].
    -   Distribute primitives into cells. For each primitive, determine its cell index $(c_x, c_y, c_z)$ based on its position relative to `min` and `delta`.
    -   If a cell's item list is empty, set the corresponding `cells` entry to `NULL` [S1, S2, unofficial].
    -   If a cell's item list is not empty, create a new `GridNode` leaf containing those items [S1, S2, unofficial].
    -   Recursively process each non-null child `GridNode` [S1, S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class InnerNode:
    def __init__(self, min_val, max_val, dim, delta, cells):
        self.min = min_val
        self.max = max_val
        self.dim = dim
        self.delta = delta
        self.cells = cells

class GridNode:
    def __init__(self, index, leaf_items=None, interior=None):
        self.index = index
        self.leaf_items = leaf_items
        self.interior = interior

def build_multi_level_array_grid(primitives, depth=0):
    # Base case: termination conditions
    if len(primitives) < 8 or depth > 2:
        leaf_items = list(primitives)
        index = (len(primitives) << 1) | 0  # High bits: count, Low bit: 0 (leaf)
        return GridNode(index=index, leaf_items=leaf_items)

    # Compute bounding box of current primitives
    bbox_min = [min(p.min[i] for p in primitives) for i in range(3)]
    bbox_max = [max(p.max[i] for p in primitives) for i in range(3)]
    S = [bbox_max[i] - bbox_min[i] for i in range(3)]
    max_S = max(S)
    N = len(primitives)
    rho = 4  # Density factor

    # Calculate grid dimensions
    dim_x = max(1, round((S[0] / max_S) * (rho * N) ** (1/3)))
    dim_y = max(1, round((S[1] / max_S) * (rho * N) ** (1/3)))
    dim_z = max(1, round((S[2] / max_S) * (rho * N) ** (1/3)))

    # Compute delta (cell size)
    delta_x = S[0] / dim_x
    delta_y = S[1] / dim_y
    delta_z = S[2] / dim_z

    # Initialize cell lists
    # Source specifies allocating cells array with size N [S1, S2, unofficial]
    num_cells = len(primitives)
    cell_lists = [[] for _ in range(num_cells)]

    # Distribute primitives into cells
    for p in primitives:
        center = [(p.min[i] + p.max[i]) / 2.0 for i in range(3)]
        cx = int((center[0] - bbox_min[0]) / delta_x)
        cy = int((center[1] - bbox_min[1]) / delta_y)
        cz = int((center[2] - bbox_min[2]) / delta_z)
        cx = max(0, min(cx, dim_x - 1))
        cy = max(0, min(cy, dim_y - 1))
        cz = max(0, min(cz, dim_z - 1))
        cell_idx = cx + cy * dim_x + cz * dim_x * dim_y
        cell_lists[cell_idx].append(p)

    # Create child nodes
    cells = []
    for cell_items in cell_lists:
        if len(cell_items) == 0:
            cells.append(None)
        else:
            child_node = build_multi_level_array_grid(cell_items, depth + 1)
            cells.append(child_node)

    inner_node = InnerNode(bbox_min, bbox_max, (dim_x, dim_y, dim_z), (delta_x, delta_y, delta_z), cells)
    index = 1  # Low bit: 1 (inner marker)
    return GridNode(index=index, interior=inner_node)
```

### Parameters and Edge Cases
-   **Leaf Threshold**: 8 items. Nodes with fewer items remain leaves [S1, S2, unofficial].
-   **Depth Limit**: 2. Recursion stops if depth > 2 [S1, S2, unofficial].
-   **Density Factor ($\rho$)**: Typically 4 [S1, S2, unofficial].
-   **Empty Cells**: Represented by `NULL` pointers in the `cells` array to save memory [S1, S2, unofficial].
-   **Degenerate Input**: If $S_i = 0$, the dimension calculation may fail. The pseudocode clamps dimensions to at least 1 [G].
-   **Precision**: Floating-point comparisons for bounding box and cell indexing should use epsilon guards to avoid ray leakage or double-intersection [G].

### Subtopics
-   [Two-Level Grids for Ray Tracing on GPUs](4245-two-level-grids-for-ray-tracing-on-gpus.md): Uses a fixed two-level hierarchy optimized for GPU memory coalescing. Choose when hardware constraints limit recursion depth.
-   [Macrocell Grid Traversal](5207-macrocell-grid-traversal.md): Uses a fixed macrocell size with uniform subdivision. Choose when scene distribution is uniform or when dynamic construction overhead is prohibitive.

## Operating characteristics
-   **Time Complexity**: Construction and traversal performance depend on scene distribution. Profiling indicates that grid traversal dominates render time [S2, unofficial].
-   **Memory Complexity**: $O(N)$ for primitives plus overhead for grid pointers. Empty cells are stored as null pointers, reducing memory usage compared to dense grids [S1, S2, unofficial].
-   **Performance**: Multi-level grids reduce ray-triangle intersections by a factor of two compared to one-level grids [S2, unofficial].
-   **Strengths**: Memory locality due to contiguous arrays; reduced pointer chasing; adaptive resolution [S1, S2, unofficial].
-   **Limitations**: Fixed depth limit (2) may not suffice for complex scenes; heuristic-based grid sizing may not be optimal for all distributions [S1, S2, unofficial].

## Background
The Multi-Level Array Grid was introduced as a classic implementation for performance comparison in ray tracing research [S1, S2, unofficial]. It is based on a memory-conservative heuristic attributed to Woo (1992) [S1, S2, unofficial]. The structure was used to compare against multi-level hashed grids, highlighting the trade-offs between array-based and hash-based hierarchical structures [S1, S2, unofficial].

## Sources
[S1] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2010/28153/, fetched 2026-10-24 (unofficial)
[S2] PDF Multi-level Grid Strategies for Ray Tracing, https://pdfs.semanticscholar.org/f922/40132d0d7340ff97ac4074b0a6906fb44c44.pdf, fetched 2026-10-25 (unofficial)
