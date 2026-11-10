# Sparse Voxel Octree (SVO) Builder

Also called: BuildNodeRecursive, BuildSVO, SVO Build, SVO construction, Sparse Voxel Octree (SVO) Construction

## Overview
The Sparse Voxel Octree (SVO) Builder is a GPU-based algorithm that constructs a hierarchical Sparse Voxel Octree from a list of voxelized geometry fragments using compute shaders [S1, unofficial]. It solves the problem of efficiently converting triangle meshes into a compact, hierarchical voxel representation suitable for real-time global illumination techniques like Voxel Cone Tracing [S1, unofficial]. Unlike in-core CPU methods or grid-based GPU approaches, this method generates the SVO directly on the GPU by processing the voxel list level-by-level, flagging nodes, allocating memory for internal nodes, and storing leaf nodes [S4, unofficial]. It is distinct from [Brute Force SVO Construction](1704-brute-force-svo-construction.md) by its use of compute shaders for parallel level-by-level construction, and from [Out-of-Core SVO Construction via Morton Order](1885-out-of-core-svo-construction-via-morton-order.md) by its reliance on pre-allocated buffers and specific atomic merging strategies for fragment consolidation [S1, unofficial].

## How it works
The method constructs the SVO in a series of compute shader passes, building the tree level-by-level from the root down to the maximum depth [S1, unofficial].

### Inputs and Outputs
*   **Input:** A list of voxel fragments generated from conservative rasterization of triangle geometry [S1, unofficial]. Each fragment represents a voxel occupied by the mesh.
*   **Output:** A flat array `_tree` storing the octree nodes, where each node is a 32-bit integer [S2, unofficial].

### Data Structures
The core data structure is the `_tree` array, which stores node metadata as `uint32_t` values [S2, unofficial]. The layout of a node is as follows:
*   **Bits 0–16 (17 bits):** Index of the first child node in the `_tree` array.
*   **Bit 17 (1 bit):** Leaf flag. If set, the node is a leaf; if clear, it is an internal node.
*   **Value `0x0000FFFF`:** Represents an empty node (child index 0, leaf flag clear) [S2, unofficial].
*   **Value `0x00004200`:** Represents a solid leaf node (child index 0, leaf flag set) [S2, unofficial].

### Parameters
*   **maxDepth:** The maximum depth of the octree. The root is at depth 0. When the current depth equals `maxDepth`, leaf nodes are created instead of children [S2, unofficial][S3, unofficial].

### Steps
The construction proceeds in the following phases:

1.  **Build Voxel Fragment List:** Generate a list of voxel fragments from the input geometry using conservative rasterization [S1, unofficial].
2.  **Initialize Root:** Initialize the root node (level 0) by clearing memory for the first 8 nodes (indices 0–7) using a `node_init` shader [S4, unofficial]. The root is pushed to a processing stack with initial state [S2, unofficial].
3.  **Level-by-Level Construction:** Iterate from level 1 to `resolution - 1` [S4, unofficial]. For each level:
    *   **Tag Nodes:** Execute a `node_flag` shader to determine which nodes at the current level are internal (have children) or leaves based on the voxel list [S4, unofficial].
    *   **Allocate Memory:** Execute a `node_alloc` shader to allocate memory for the children of the identified internal nodes. This uses an atomic counter to track the number of allocated nodes. The output start address is `alloc_start`, and the count of new nodes is `alloc_count * 8` [S4, unofficial].
    *   **Initialize Buffer:** Clear the memory for the newly allocated nodes using the `node_init` shader [S4, unofficial].
    *   **Update State:** Update state variables for the next iteration: `start` becomes `alloc_start`, `count` becomes `alloc_count * 8`, and `alloc_start` becomes `start + count` [S4, unofficial].
4.  **Inject Attributes:** After the octree structure is built, inject voxel attributes (albedo, normal, etc.) into the leaf nodes. If multiple voxel fragments fall into the same leaf node, their attribute values are averaged [S1, unofficial]. This is done using a `store_leaf` shader that binds the octree buffer and voxel list [S4, unofficial].
5.  **Mipmapping:** Mipmap node values from bottom to top, level by level, into the interior nodes [S1, unofficial].

### Pseudocode
The following pseudocode describes the recursive construction logic found in [S2, unofficial], adapted to be self-contained. It assumes a stack-based traversal to build the tree structure.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
EMPTY_NODE = 0x0000FFFF
LEAF_NODE = 0x00004200
MAX_DEPTH = 10  # Example value, typically determined by resolution

# Data structures
class StackEntry:
    index: int
    depth: int

def build_svo(max_depth: int) -> list:
    """
    Builds the SVO structure in a flat array.
    Returns the _tree array.
    """
    _tree = []
    stack = []
    
    # Initialize root
    # Push initial root node 0x0000FFFF to _tree
    _tree.append(EMPTY_NODE)
    
    # Push StackEntry{0, 0} to stack
    stack.append(StackEntry(index=0, depth=0))
    
    index = 0
    
    while stack:
        current = stack.pop()
        depth = current.depth
        index = current.index
        
        # Calculate offset for child pointer
        # offset = _tree.size() - index
        offset = len(_tree) - index
        
        # Set _tree[index] |= offset << 17
        # Note: This logic in [S2, unofficial] seems to update the parent's pointer
        # to the start of the children block.
        # We must ensure _tree[index] exists.
        if index < len(_tree):
            _tree[index] = _tree[index] | (offset << 17)
        
        # Increment index by offset (for next sibling/child calculation in some variants)
        # However, in the stack-based approach, index is managed by the stack entry.
        # The [S2, unofficial] snippet increments index locally. We follow the snippet's flow.
        index += offset
        
        if depth < max_depth:
            # Increment depth for children
            depth += 1
            
            # Loop 8 times to create children
            for i in range(8):
                # Push 0x0000FFFF to _tree
                _tree.append(EMPTY_NODE)
                
                # Push StackEntry{index + (8 - i), depth} to stack
                # Note: The index calculation in [S2, unofficial] is specific to its linear allocation scheme.
                # Here we push the index where the child's data will start or be referenced.
                child_index = len(_tree)
                stack.append(StackEntry(index=child_index, depth=depth))
        else:
            # Max depth reached, create leaf nodes
            for i in range(8):
                # Push 0x00004200 to _tree
                _tree.append(LEAF_NODE)
                
                # Push StackEntry for leaf (though no further expansion needed)
                child_index = len(_tree)
                stack.append(StackEntry(index=child_index, depth=depth))
                
    return _tree
```

### Edge Cases and Pitfalls
*   **Work Imbalance:** Due to the sparsity of the SVO, work distribution in parallel compute shaders can be uneven, causing bottlenecks. This is mitigated by using unbalanced scheduling for parallel tasks [S3, unofficial].
*   **Root Collision:** If the root node collides with an obstacle, it must be recursively subdivided to `MaxDepth` [S3, unofficial].
*   **Invalid Child Roots:** During merging, child roots that are invalid (treated as non-existent or blocked) should be skipped and not collected into the node array [S3, unofficial].

## Subtopics

### [GPU-based Octree Generation](5332-gpu-based-octree-generation.md)
This method generates a hierarchical octree data structure directly on the GPU by partitioning the input volume into fixed-size cubes of $32 \times 32 \times 32$ voxels. Unlike standard Sparse Voxel Octrees that only subdivide non-empty regions, this approach uniformly subdivides the entire volume space down to the minimum node size, which simplifies compute shader logic by avoiding dynamic branching for empty regions. It constructs a pre-allocated array-based octree by mapping every non-empty voxel to a hierarchical path.

### [Sparse Octree Voxelization Performance](9137-sparse-octree-voxelization-performance.md)
This article documents the performance characteristics and implementation details of the Sparse Voxel Octree (SVO) construction method used for GPU-based voxelization. It addresses the problem of efficiently converting triangle meshes into a compact hierarchical voxel representation for real-time global illumination. Unlike in-core CPU methods or grid-based GPU approaches, this method generates the SVO directly on the GPU, optimizing for parallel triangle processing and minimizing atomic contention. It is distinct from [Grid-Free Out-Of-Core GPU Voxelization](11494-grid-free-out-of-core-gpu-voxelization.md) by its reliance on preallocated buffers and specific atomic merging strategies for fragment consolidation.

## Operating characteristics
*   **Time Complexity:** The construction is performed level-by-level, with each level processed in parallel using compute shaders. The complexity is generally linear with respect to the number of nodes and fragments, but depends heavily on the sparsity and distribution of the geometry.
*   **Memory Complexity:** The SVO stores only non-empty nodes, leading to significant memory savings compared to dense voxel grids. The flat array `_tree` grows dynamically based on the number of internal and leaf nodes required.
*   **Performance:** The method is optimized for GPU execution, leveraging compute shaders for parallel processing. Performance is affected by the resolution of the voxelization and the complexity of the input geometry.
*   **Strengths:** Efficient memory usage, suitable for real-time applications, parallelizable on GPU.
*   **Limitations:** Requires careful handling of work imbalance due to sparsity. The level-by-level construction may involve multiple shader passes and atomic operations, which can introduce overhead.
*   **When to Use:** Real-time global illumination, voxel cone tracing, applications requiring efficient hierarchical voxel data on the GPU.
*   **When Not to Use:** Applications requiring strict CPU-side processing, or where the overhead of GPU compute shaders is prohibitive.

## Background
The method is based on the approach described in Chapter 22 of *OpenGL Insights* by Cyril Crassin and Simon Green [S1, unofficial]. It is widely used in real-time rendering for techniques like Voxel Cone Tracing, where a hierarchical voxel representation is needed for efficient light propagation calculations [S1, unofficial]. The implementation details provided in the sources are from unofficial repositories and tutorials, indicating its adoption in community-driven graphics projects [S1, unofficial][S4, unofficial].

## Sources
[S1] otaku690/SparseVoxelOctree, https://github.com/otaku690/SparseVoxelOctree, fetched 2026-10-26 (unofficial)
[S2] cianjinks/Sparse-Voxel-Octree, https://github.com/cianjinks/Sparse-Voxel-Octree, fetched 2026-10-26 (unofficial)
[S3] UE5 Tutorial | Simple SVO Pathfinder for 3D Pathfinding, https://www.youtube.com/watch?v=QmDbX_gZSLA, fetched 2026-10-27 (unofficial)
[S4] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
