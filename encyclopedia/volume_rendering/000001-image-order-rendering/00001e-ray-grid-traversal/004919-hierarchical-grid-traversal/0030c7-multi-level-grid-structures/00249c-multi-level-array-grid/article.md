# Multi-Level Array Grid

Also called: Classic multi-level array grid

## Overview
The Multi-Level Array Grid is a hierarchical spatial acceleration structure for ray tracing that recursively subdivides the scene bounding box into uniform grids based on item density [S1, unofficial][S2, unofficial]. It solves the problem of efficient ray-primitive intersection by skipping empty regions of space, using a fixed-depth recursion (typically 2 levels) to balance traversal overhead against intersection cost [S1, unofficial][S2, unofficial]. It is distinguished from sibling methods like the Multi-Level Hashed Grid by its use of dense array indexing for child nodes rather than hash tables, making it memory-conservative but potentially sparse in empty areas [S1, unofficial][S2, unofficial].

## How it works
**Inputs:** A set of scene primitives (items) and their bounding boxes [S1, unofficial][S2, unofficial].
**Outputs:** A root `GridNode` representing the hierarchical grid structure [S1, unofficial][S2, unofficial].

**Data Structures:**
The structure consists of two main types of nodes, defined in C-like pseudocode below [S1, unofficial][S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```c
// Inner node: represents a subdivided grid cell
struct InnerNode {
    float min[3];       // Bounding box minimum
    float max[3];       // Bounding box maximum
    int dim[3];         // Grid dimensions (Mi for each axis)
    float delta[3];     // Cell size (width/height/depth)
    GridNode** cells;   // Array of pointers to child nodes (size = dim[0]*dim[1]*dim[2])
};

// Union node: either a leaf containing items or a pointer to an inner node
struct GridNode {
    int index;          // High bits: item count (if leaf); Low bit: type flag
                        // Low bit 1 = Inner Node, Low bit 0 = Leaf Node
    union {
        struct {
            int* items; // Array of primitive indices
        } leaf;
        struct {
            InnerNode* interior; // Pointer to child grid
        } inner;
    };
};
```

**Construction Algorithm:**
1. **Compute Scene Bounding Box:** Determine the global minimum and maximum extents of all scene primitives [S1, unofficial][S2, unofficial]. Let $S_i$ be the size of the bounding box in dimension $i \in \{x, y, z\}$ [S1, unofficial][S2, unofficial].
2. **Initialize Root:** Create a root `GridNode` as a leaf node containing all scene primitives [S1, unofficial][S2, unofficial].
3. **Recursive Construction:** Process each node recursively [S1, unofficial][S2, unofficial].
    * **Termination Conditions:** If the number of items in the current node is less than 8, or if the current grid depth is greater than 2, the node remains a leaf. Recursion stops for this branch [S1, unofficial][S2, unofficial].
    * **Expansion:** If termination conditions are not met, expand the node to an `InnerNode` [S1, unofficial][S2, unofficial].
    * **Dimension Calculation:** Calculate the grid dimensions $M_i$ for each axis using the heuristic attributed to Woo (1992) [S1, unofficial][S2, unofficial]:
        $$M_i = \frac{S_i}{\max\{S_i\}} \sqrt[3]{\rho N} \quad (i \in \{x, y, z\})$$
        Where:
        * $S_i$ is the bounding box size in dimension $i$ [S1, unofficial][S2, unofficial].
        * $\max\{S_i\}$ is the maximum dimension size among $x, y, z$ [S1, unofficial][S2, unofficial].
        * $\rho$ is a density factor, typically 4 [S1, unofficial][S2, unofficial].
        * $N$ is the number of primitives in the current node [S1, unofficial][S2, unofficial].
    * **Cell Allocation:** Compute `delta` (cell size) as $S_i / M_i$. Allocate the `cells` array of size $\prod M_i$ [S1, unofficial][S2, unofficial].
    * **Item Assignment:** Distribute primitives into the grid cells [S1, unofficial][S2, unofficial].
        * If a cell contains no primitives, set its pointer to `null` [S1, unofficial][S2, unofficial].
        * If a cell contains primitives, initialize a new `GridNode` as a leaf, storing the primitive indices in `items` and the count in the high bits of `index` [S1, unofficial][S2, unofficial].
    * **Recurse:** For each non-null child leaf node, repeat the construction process (incrementing depth) [S1, unofficial][S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class GridNode:
    def __init__(self):
        self.index = 0
        self.leaf_items = None
        self.inner_node = None

class InnerNode:
    def __init__(self, min_val, max_val, dim, delta, cells):
        self.min = min_val
        self.max = max_val
        self.dim = dim
        self.delta = delta
        self.cells = cells

def compute_bounding_box(primitive_centers):
    min_val = [float('inf')] * 3
    max_val = [float('-inf')] * 3
    for c in primitive_centers:
        for i in range(3):
            if c[i] < min_val[i]: min_val[i] = c[i]
            if c[i] > max_val[i]: max_val[i] = c[i]
    return min_val, max_val

def build_grid(primitive_centers, depth=0, max_depth=2, leaf_threshold=8, rho=4):
    if not primitive_centers:
        return None

    bbox_min, bbox_max = compute_bounding_box(primitive_centers)
    S = [bbox_max[i] - bbox_min[i] for i in range(3)]
    max_S = max(S)
    N = len(primitive_centers)

    if depth > max_depth or N < leaf_threshold:
        node = GridNode()
        node.index = (N << 1) | 0  # High bits: count, low bit: 0 (leaf)
        node.leaf_items = primitive_centers
        return node

    dim = []
    for i in range(3):
        m_i = (S[i] / max_S) * (rho * N) ** (1/3)
        dim.append(int(round(m_i)))

    delta = [S[i] / dim[i] for i in range(3)]
    total_cells = dim[0] * dim[1] * dim[2]
    cells = [None] * total_cells

    cell_items = [[] for _ in range(total_cells)]
    for c in primitive_centers:
        cx = int((c[0] - bbox_min[0]) / delta[0])
        cy = int((c[1] - bbox_min[1]) / delta[1])
        cz = int((c[2] - bbox_min[2]) / delta[2])
        cx = min(max(cx, 0), dim[0] - 1)
        cy = min(max(cy, 0), dim[1] - 1)
        cz = min(max(cz, 0), dim[2] - 1)
        c_idx = cx + cy * dim[0] + cz * dim[0] * dim[1]
        cell_items[c_idx].append(c)

    for c_idx in range(total_cells):
        items = cell_items[c_idx]
        if not items:
            cells[c_idx] = None
        else:
            leaf = GridNode()
            leaf.index = (len(items) << 1) | 0
            leaf.leaf_items = items
            cells[c_idx] = leaf

    for c_idx in range(total_cells):
        child = cells[c_idx]
        if child is not None:
            cells[c_idx] = build_grid(child.leaf_items, depth + 1, max_depth, leaf_threshold, rho)

    inner = InnerNode(bbox_min, bbox_max, dim, delta, cells)
    node = GridNode()
    node.index = 1  # Low bit: 1 (inner)
    node.inner_node = inner
    return node
```

**Parameters:**
* **Leaf Threshold:** 8 items. Nodes with fewer items remain leaves [S1, unofficial][S2, unofficial].
* **Max Depth:** 2. Recursion stops if depth exceeds 2 [S1, unofficial][S2, unofficial].
* **Density Factor ($\rho$):** 4. Used in the dimension calculation heuristic [S1, unofficial][S2, unofficial].

**Edge Cases and Pitfalls:**
* **Empty Cells:** Represented as `null` pointers to save memory [S1, unofficial][S2, unofficial].
* **Precision:** Cell indices are computed using floating-point division. Clamping is required to prevent out-of-bounds access due to floating-point errors at boundaries [G].
* **Redundant Data:** `delta` is stored in `InnerNode` but can be computed from `max`, `min`, and `dim`. It is cached for performance [S1, unofficial][S2, unofficial].

## Operating characteristics
* **Time Complexity:** Construction is $O(N)$ per level using direct assignment [S1, unofficial][S2, unofficial]. Grid traversal dominates render time, and one-level grids spend twice the time doing ray-triangle intersections than multi-level grids [S1, unofficial][S2, unofficial].
* **Memory Complexity:** Memory usage is proportional to the number of non-empty cells. Empty leaf cells are represented as null pointers to reduce memory requirements [S1, unofficial][S2, unofficial].
* **Strengths:** The construction uses a memory conservative heuristic [S1, unofficial][S2, unofficial].
* **Limitations:** The fixed depth limit (2) may not be optimal for all scenes [S1, unofficial][S2, unofficial].
* **When to Use:** It serves as a baseline for comparison with more advanced structures like hashed grids [S1, unofficial][S2, unofficial].

## Background
The Multi-Level Array Grid was described in a 2010 paper by SCITEPRESS, used as a baseline for comparison against a Multi-Level Hashed Grid implementation [S1, unofficial][S2, unofficial]. The dimension calculation heuristic is attributed to Woo (1992) [S1, unofficial][S2, unofficial]. It is part of a family of hierarchical grid structures designed to optimize ray tracing performance by reducing the number of intersection tests [S1, unofficial][S2, unofficial].

## Sources
[S1] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2010/28153/, fetched 2026-10-24 (unofficial)
[S2] PDF Multi-level Grid Strategies for Ray Tracing, https://pdfs.semanticscholar.org/f922/40132d0d7340ff97ac4074b0a6906fb44c44.pdf, fetched 2026-10-25 (unofficial)
