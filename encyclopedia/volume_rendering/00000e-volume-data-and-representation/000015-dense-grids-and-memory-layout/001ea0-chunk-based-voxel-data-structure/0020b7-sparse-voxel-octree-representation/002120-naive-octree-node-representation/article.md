# Naive Octree Node Representation

Also called: Branch Tree Node, Direct Memory Octree Node, Pointer-based Octree Node

## Overview
The Naive Octree Node Representation is a direct memory layout for sparse voxel octrees where each non-leaf node explicitly stores eight child pointers and leaf nodes store a single voxel value [S1, unofficial]. It solves the problem of hierarchical spatial partitioning by providing a straightforward, pointer-based traversal mechanism without complex bit-packing or index compression [S2, unofficial]. It differs from [Sparse Voxel Octree with Bricks](2354-sparse-voxel-octree-with-bricks.md) by using individual node allocations rather than fixed-size blocks, and from [Bitmask Leaf Compression](3603-bitmask-leaf-compression.md) by retaining explicit leaf node structures, which results in higher memory overhead but simpler implementation logic [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A voxel position $(x, y, z)$ and a voxel value (e.g., color or density).
*   **Output:** An updated octree structure reflecting the insertion or query of the voxel.

**Data Structures**
The core structure is the `OctreeNode`. According to [S3, unofficial], the fields are declared in memory order as:
1.  `isLeaf`: `bool`
2.  `value`: `uint32_t` (stores the voxel value if leaf, or acts as a placeholder if not)
3.  `children`: `OctreeNode*` (allocated as an array of 8 pointers if not a leaf)
4.  `centerX`, `centerY`, `centerZ`: `uint8_t`
5.  `depth`: `uint8_t`

Alternatively, [S1, unofficial] defines a struct `Node` with the following layout:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct Node {
    Node * children[8];
    int8_t voxelColorIndex;
    bool isLeaf;
};
```
This variant uses a 64-bit processor memory cost of 72 Bytes and a 32-bit processor cost of 40 Bytes [S1, unofficial]. The pointer array dominates the size.

**Numbered Steps**

1.  **Root Initialization**: Create the root node with `depth = 0`, `isLeaf = true`, `value = 0`, and `centerX = centerY = centerZ = 0` [S3, unofficial].
2.  **Non-Root Initialization**: For a child node at depth $d$, set `isLeaf = true`, `value = 0`. Calculate `childDepth = d - 1`, `childSize = 1 << childDepth`. Set `centerX = x + childSize`, `centerY = y + childSize`, `centerZ = z + childSize` [S3, unofficial].
3.  **SetVoxel(x, y, z, value)**:
    *   If `depth == 0`, set `this->value = value` and return [S3, unofficial].
    *   If `isLeaf` and `this->value == value`, return (no change needed) [S3, unofficial].
    *   If `isLeaf` and `this->value != value`, call `initChildren()` and copy `this->value` to all 8 children [S3, unofficial].
    *   Compute `childIndex` using `GetChildIndex(x, y, z)` [S3, unofficial].
    *   Recurse: `children[childIndex].SetVoxel(x, y, z, value)` [S3, unofficial].
    *   Call `TryCollapse()` to merge empty children back into a leaf if possible [S3, unofficial].
4.  **GetValue(x, y, z)**:
    *   If `isLeaf`, return `value` [S3, unofficial].
    *   Else, compute `childIndex` via `GetChildIndex(x, y, z)` and return `children[childIndex].GetValue(x, y, z)` [S3, unofficial].
5.  **initChildren()**:
    *   Set `childDepth = depth - 1`, `childSize = 1 << childDepth` [S3, unofficial].
    *   Allocate `children = new OctreeNode[8]` [S3, unofficial].
    *   Compute base coordinates: `baseX = centerX - childSize`, `baseY = centerY - childSize`, `baseZ = centerZ - childSize` [S3, unofficial].
    *   Initialize 8 children with offsets relative to the base: $(0,0,0)$, $(childSize,0,0)$, $(0,childSize,0)$, $(childSize,childSize,0)$, $(0,0,childSize)$, $(childSize,0,childSize)$, $(0,childSize,childSize)$, $(childSize,childSize,childSize)$ [S3, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class OctreeNode:
    def __init__(self, depth=0, x=0, y=0, z=0):
        self.depth = depth
        self.is_leaf = True
        self.value = 0
        self.children = None
        self.center_x = x
        self.center_y = y
        self.center_z = z
        self.child_size = 1 << depth if depth > 0 else 0

    def set_voxel(self, x, y, z, value):
        # Base case: leaf node at depth 0
        if self.depth == 0:
            self.value = value
            return

        # If leaf and value is same, no update needed
        if self.is_leaf and self.value == value:
            return

        # If leaf but value differs, expand to children
        if self.is_leaf:
            self.init_children()
            # Copy current leaf value to all new children
            for child in self.children:
                child.value = self.value
                child.is_leaf = True

        # Determine which child to recurse into
        child_index = self.get_child_index(x, y, z)
        
        # Ensure child exists (should be created in init_children)
        if self.children[child_index] is None:
            # This case implies the tree structure wasn't fully initialized
            # In a naive representation, children are allocated on expansion.
            # If we are here, init_children was called, so children exist.
            pass
            
        self.children[child_index].set_voxel(x, y, z, value)
        
        # Optional: Try to collapse empty children back into a leaf
        self.try_collapse()

    def get_value(self, x, y, z):
        if self.is_leaf:
            return self.value
        
        child_index = self.get_child_index(x, y, z)
        return self.children[child_index].get_value(x, y, z)

    def init_children(self):
        self.is_leaf = False
        self.children = [None] * 8
        
        child_depth = self.depth - 1
        child_size = 1 << child_depth
        
        base_x = self.center_x - child_size
        base_y = self.center_y - child_size
        base_z = self.center_z - child_size
        
        # Offsets for the 8 children
        offsets = [
            (0, 0, 0), (child_size, 0, 0), (0, child_size, 0), (child_size, child_size, 0),
            (0, 0, child_size), (child_size, 0, child_size), (0, child_size, child_size), (child_size, child_size, child_size)
        ]
        
        for i, (ox, oy, oz) in enumerate(offsets):
            self.children[i] = OctreeNode(
                depth=child_depth,
                x=base_x + ox,
                y=base_y + oy,
                z=base_z + oz
            )

    def get_child_index(self, x, y, z):
        # Assuming x, y, z are relative to the node's center or base
        # Standard octree indexing:
        # bit 0: x, bit 1: y, bit 2: z
        # This depends on coordinate system. A common mapping:
        # index = (x_bit << 2) | (y_bit << 1) | z_bit
        # However, without specific coordinate normalization, we assume
        # the caller provides coordinates relative to the current node's space.
        # For implementation completeness, we assume a helper that maps
        # global/local coords to 0-7 index.
        # [G] Placeholder logic for index calculation based on relative position
        # In a real implementation, this checks which octant (x,y,z) falls into.
        # Example: if x > center, set bit 0, etc.
        return 0 # Placeholder: Implementation requires specific coordinate mapping

    def try_collapse(self):
        # Check if all children are leaves with the same value
        # If so, merge them back into this node
        pass # Implementation details for collapse not provided in sources
```

**Parameters**
*   **Memory Cost**: 72 Bytes on a 64-bit processor; 40 Bytes on a 32-bit processor [S1, unofficial].
*   **Depth**: The maximum recursion depth determines the resolution. Depth 0 is the base case [S3, unofficial].

**Edge Cases and Pitfalls**
*   **Depth 0 Recursion**: Attempting to recurse below depth 0 is prevented by the explicit `depth == 0` check in `SetVoxel` [S3, unofficial].
*   **Undefined Helpers**: `GetChildIndex` and `TryCollapse` are referenced but not defined in the source snippets [S3, unofficial]. Implementation requires a child index mapping function and a post-insertion collapse check.
*   **Memory Overhead**: Using explicit leaf nodes causes memory usage to approach 8 times that of a full octree of depth $d-1$ if the tree is dense [S1, unofficial].

## Subtopics

### [Bitmask Leaf Compression](3603-bitmask-leaf-compression.md)
Eliminates explicit leaf nodes by packing voxel data into a bitmask within the parent node, reducing memory cost factor $mc$ from 8 to 1 for the deepest level. Choose this when memory is constrained and the tree is dense at the leaves.

### [Octree Data Structure Layout](4518-octree-data-structure-layout.md)
Defines the contiguous array storage for pointers and data, trading $O(1)$ indexing for logarithmic traversal. Choose this when cache locality is prioritized over pointer-chasing overhead.

## Operating characteristics
*   **Time Complexity**: Traversal is $O(\log N)$ where $N$ is the number of voxels, due to the hierarchical depth [S1, unofficial].
*   **Memory Complexity**: High overhead due to explicit pointers. Each non-leaf node consumes 8 pointers plus metadata. For a full octree, memory usage approaches $8 \times$ the size of a full octree of depth $d-1$ [S1, unofficial].
*   **Strengths**: Simple implementation; direct pointer access allows easy debugging and modification.
*   **Limitations**: High memory consumption for dense scenes; pointer chasing can cause cache misses.
*   **When to Use**: Small to medium sparse scenes where implementation simplicity is valued over memory efficiency.
*   **When Not to Use**: Large-scale dense scenes or memory-constrained environments (e.g., mobile GPUs) where [Bitmask Leaf Compression](3603-bitmask-leaf-compression.md) or [Sparse Voxel Octree with Bricks](2354-sparse-voxel-octree-with-bricks.md) are preferred.

## Background
The naive representation is the foundational structure for sparse voxel octrees, described in early works on voxel-based rendering and spatial partitioning [S1, unofficial]. It serves as the baseline against which compressed variants like bitmask leaf compression are compared [S1, unofficial]. The structure is widely used in educational contexts and simple terrain generation systems [S3, unofficial].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S2] planaria/dmc, https://github.com/planaria/dmc, fetched 2026-10-26 (unofficial)
[S3] Maxsimuss/3dTerrainGeneration, https://github.com/Maxsimuss/3dTerrainGeneration, fetched 2026-10-27 (unofficial)
