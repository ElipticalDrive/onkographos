# Naive Octree Node Representation

Also called: Branch Tree Node, Direct Memory Octree Node, Octree Voxel Storage, Pointer-based Octree Node

## Overview
The Naive Octree Node Representation is a direct memory layout for sparse voxel octrees where each non-leaf node explicitly stores an array of 8 child pointers, and leaf nodes store a single data value (typically a color or density index) [S1, S3, unofficial]. It solves the problem of hierarchical spatial partitioning by providing a straightforward, pointer-chased traversal mechanism for voxel data [S2, unofficial]. It is distinguished from brick-based or rasterized variants by its lack of structural compression; every internal node allocates memory for all 8 children, leading to high memory overhead for sparse or deep trees [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel position $(x, y, z)$, a depth level, and a value (e.g., color index) to insert or retrieve.
*   **Output:** The octree structure with updated nodes, or the value associated with the voxel.

**Data Structures:**
The core structure is the `OctreeNode`. Two primary layouts are cited:

1.  **Standard Naive Layout [S1, S3, unofficial]:**
    *   `children`: An array of 8 pointers to `OctreeNode` (or `Node`).
    *   `isLeaf`: A boolean flag indicating if the node is a leaf.
    *   `value` (or `voxelColorIndex`): If `isLeaf` is true, this stores the voxel data (e.g., `uint32_t` or `int8_t` index). If `isLeaf` is false, this field is unused or repurposed depending on implementation [S1, S3, unofficial].
    *   `centerX`, `centerY`, `centerZ`: Coordinates defining the node's spatial extent [S3, unofficial].
    *   `depth`: The current depth in the octree hierarchy [S3, unofficial].

2.  **Template-Based Layout [S2, unofficial]:**
    *   Inherits from a base `tree_node`.
    *   Contains `std::array<base_type*, 8> children_`.

**Memory Cost:**
*   On a 64-bit processor: 72 Bytes per node (8 pointers $\times$ 8 bytes + 1 byte bool + 1 byte index + padding) [S1, unofficial].
*   On a 32-bit processor: 40 Bytes per node (8 pointers $\times$ 4 bytes + 1 byte bool + 1 byte index + padding) [S1, unofficial].

**Algorithm Steps:**

1.  **Root Initialization:**
    *   Set `depth = 0`.
    *   Set `isLeaf = true`.
    *   Set `value = 0`.
    *   Set `centerX = centerY = centerZ = 0` [S3, unofficial].

2.  **Non-Root Initialization:**
    *   Set `isLeaf = true`.
    *   Set `value = 0`.
    *   Set `childDepth = depth - 1`.
    *   Set `childSize = 1 << childDepth` (i.e., $2^{depth-1}$).
    *   Set `centerX = x + childSize`, `centerY = y + childSize`, `centerZ = z + childSize` [S3, unofficial].

3.  **Insertion (`SetVoxel`):**
    *   **Base Case:** If `depth == 0`, set `this->value = value` and return [S3, unofficial].
    *   **Leaf Optimization:** If `isLeaf` is true and `this->value == value`, return (no change needed) [S3, unofficial].
    *   **Split:** If `isLeaf` is true but the value differs (or needs expansion), call `initChildren()` [S3, unofficial].
        *   Copy `this->value` to all 8 new children.
        *   Set `this->isLeaf = false`.
    *   **Child Selection:** Compute `childIndex` using `GetChildIndex(x, y, z)` [S3, unofficial].
    *   **Recurse:** Call `children[childIndex].SetVoxel(x, y, z, value)` [S3, unofficial].
    *   **Collapse:** Call `TryCollapse()` to merge identical children back into a leaf if possible [S3, unofficial].

4.  **Retrieval (`GetValue`):**
    *   If `isLeaf`, return `value` [S3, unofficial].
    *   Else, compute `childIndex` via `GetChildIndex(x, y, z)` and return `children[childIndex].GetValue(x, y, z)` [S3, unofficial].

5.  **Child Initialization (`initChildren`):**
    *   Set `childDepth = depth - 1`.
    *   Set `childSize = 1 << childDepth`.
    *   Allocate `children = new OctreeNode[8]` [S3, unofficial].
    *   Compute base coordinates: `baseX = centerX - childSize`, `baseY = centerY - childSize`, `baseZ = centerZ - childSize` [S3, unofficial].
    *   Initialize 8 children with offsets corresponding to the 8 octants:
        *   Indices 0-7 correspond to combinations of $(0,0,0)$, $(childSize,0,0)$, $(0,childSize,0)$, $(childSize,childSize,0)$, $(0,0,childSize)$, $(childSize,0,childSize)$, $(0,childSize,childSize)$, $(childSize,childSize,childSize)$ relative to the base [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class OctreeNode:
    def __init__(self, depth, x, y, z):
        self.depth = depth
        self.isLeaf = True
        self.value = 0
        self.centerX = x
        self.centerY = y
        self.centerZ = z
        self.children = None

    def init_children(self):
        # Calculate child dimensions
        child_depth = self.depth - 1
        child_size = 1 << child_depth
        
        # Allocate 8 children
        self.children = [None] * 8
        self.isLeaf = False
        
        # Base coordinates for the current node's bounding box
        base_x = self.centerX - child_size
        base_y = self.centerY - child_size
        base_z = self.centerZ - child_size
        
        # Initialize all 8 children
        # Order: x, y, z increments (0 to 7)
        for i in range(8):
            # Determine offset for this child index
            dx = (i & 1) * child_size
            dy = (i & 2) * child_size
            dz = (i & 4) * child_size
            
            child = OctreeNode(child_depth, base_x + dx, base_y + dy, base_z + dz)
            child.value = self.value  # Inherit current leaf value
            self.children[i] = child

    def get_child_index(self, x, y, z):
        # Determine which octant (x,y,z) falls into relative to center
        # Assuming x, y, z are absolute coordinates and centerX/Y/Z is the midpoint
        # This logic depends on the specific coordinate system; 
        # standard octree indexing usually checks against the center.
        idx = 0
        if x >= self.centerX: idx |= 1
        if y >= self.centerY: idx |= 2
        if z >= self.centerZ: idx |= 4
        return idx

    def set_voxel(self, x, y, z, value):
        # Base case: depth 0
        if self.depth == 0:
            self.value = value
            return

        # If leaf and value is same, do nothing
        if self.isLeaf and self.value == value:
            return

        # If leaf and value differs, split
        if self.isLeaf:
            self.init_children()

        # Recurse into child
        child_idx = self.get_child_index(x, y, z)
        self.children[child_idx].set_voxel(x, y, z, value)
        
        # Try to collapse children back into a leaf if they are all identical
        self.try_collapse()

    def get_value(self, x, y, z):
        if self.isLeaf:
            return self.value
        
        child_idx = self.get_child_index(x, y, z)
        return self.children[child_idx].get_value(x, y, z)

    def try_collapse(self):
        # Check if all 8 children are leaves with the same value
        if self.children is None:
            return
            
        first_child = self.children[0]
        if not first_child.isLeaf:
            return
            
        common_value = first_child.value
        all_same = True
        
        for child in self.children[1:]:
            if not child.isLeaf or child.value != common_value:
                all_same = False
                break
        
        if all_same:
            self.value = common_value
            self.isLeaf = True
            self.children = None
```

**Parameters:**
*   **Bit Mapping for Leaf Compression (Variant):** In compressed variants, the highest position child represents the leftmost 8-bit strip, and the lowest position child represents the rightmost strip [S1, unofficial].
*   **Palette:** Colors are often stored as indices into a palette of 256 entries, where 0 represents absence [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Depth 0:** This is the base case. Attempting to recurse below depth 0 is prevented by the explicit `depth == 0` check in `SetVoxel` [S3, unofficial].
*   **Undefined Helpers:** `GetChildIndex` and `TryCollapse` are referenced but not fully defined in source snippets; implementation requires a specific child index mapping function and a post-insertion collapse check [S3, unofficial].
*   **Memory Overhead:** For deep trees, the memory cost factor ($mc$) approaches 8 compared to a dense grid, as every internal node stores 8 pointers even if most children are null or identical [S1, unofficial].

## Subtopics

### Bitmask Leaf Compression
A compression technique that eliminates leaf nodes at the last level by packing their information (8 voxels) into a 64-bit integer in their parent node [S1, unofficial]. It stores the color of the 8 voxels as an index into a palette of 256 entries, using 8 bits per voxel (where 0 represents absence) [S1, unofficial]. The 8 voxels are packed into a single `int64_t` field (`leafs`) in the parent node [S1, unofficial]. This reduces memory overhead by avoiding the allocation of leaf nodes, preventing the memory cost factor from approaching 8 for deep trees [S1, unofficial]. The layout is `struct Node { Node * children[8]; int64_t leafs; }` [S1, unofficial].

### Octree Data Structure Layout
A specific implementation detail where the root node is stored at the end of a contiguous `nodes` vector, accessed via `&nodes.back()` [S4, unofficial]. The `Octree` class stores global bounds (`min`, `max`, `size`, `gridlength`) and statistics (`n_nodes`, `n_data`) alongside a pointer to voxel data and the vector of nodes [S4, unofficial]. The default constructor initializes `min` to $(0,0,0)$, `max` to $(1,1,1)$, `size` to $(1,1,1)$, and `gridlength` to 128 [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Insertion and retrieval are $O(\log N)$ where $N$ is the number of voxels, assuming a balanced tree. Traversal requires pointer chasing for every level of depth.
*   **Memory Complexity:** $O(N)$ nodes, but with a high constant factor. Each internal node consumes 72 bytes (64-bit) or 40 bytes (32-bit) [S1, unofficial]. For sparse data, this is significantly less than a dense grid, but for dense data, the overhead of pointers makes it less efficient than brick-based or compressed representations [S1, unofficial].
*   **Strengths:** Simple implementation; direct mapping of spatial hierarchy; easy to debug and visualize [S1, S3, unofficial].
*   **Limitations:** High memory overhead due to explicit child pointers; poor cache locality due to pointer chasing; significant waste if many children are null or identical [S1, unofficial].
*   **When to Use:** Small to medium-sized sparse voxel datasets where implementation simplicity is prioritized over memory efficiency [S1, S3, unofficial].
*   **When Not to Use:** Large-scale scenes requiring real-time performance with limited memory, or dense voxel data where brick-based or compressed representations are superior [S1, unofficial].

## Background
The naive octree node representation is a foundational concept in spatial data structures, originating from general computer science literature on octrees [S2, unofficial]. It is widely used in early implementations of sparse voxel octrees for global illumination and terrain generation [S1, S3, unofficial]. The specific memory cost calculations (72/40 bytes) and the bitmask compression variant are detailed in an unofficial study on SVO compression [S1, unofficial]. The template-based C++ implementation is found in the `planaria/dmc` repository [S2, unofficial]. The recursive insertion/retrieval logic is exemplified in terrain generation projects [S3, unofficial]. The contiguous vector storage variant is seen in CPU voxel raycasting implementations [S4, unofficial].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S2] planaria/dmc, https://github.com/planaria/dmc, fetched 2026-10-26 (unofficial)
[S3] Maxsimuss/3dTerrainGeneration, https://github.com/Maxsimuss/3dTerrainGeneration, fetched 2026-10-27 (unofficial)
[S4] Forceflow/cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
