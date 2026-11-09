# Tree64
Also called: Sparse 64-trees, contree, wide 4³ trees

## Overview
Tree64 is a sparse voxel tree structure with a 4³ branching factor, designed to reduce tree depth and memory overhead compared to standard octrees [S1, unofficial][S2]. It achieves higher traversal throughput by skipping 64 voxels per node level, resulting in a tree approximately half as deep as an octree for the same resolution [S2]. It is primarily used for real-time ray tracing of large voxel scenes [S1, unofficial].

## How it works
Tree64 represents a 3D volume as a tree where each internal node covers a $4 \times 4 \times 4$ sub-volume. Unlike standard octrees that branch by 8, Tree64 nodes branch by 64, allowing for shallower hierarchies and fewer traversal steps [S1, unofficial][S2].

### Inputs and Outputs
- **Input:** A 3D voxel grid (implicitly or explicitly defined).
- **Output:** A memory layout of Tree64 nodes representing the sparse volume.

### Data Structures
Each Tree64 node has a fixed size of 12 bytes [S2]. The layout is:
1.  **Child Mask (64 bits):** A bitmask indicating which of the 64 children exist. If bit $i$ is set, child $i$ exists; otherwise, it is empty [S2].
2.  **Leaf Flag (1 bit):** Indicates if the node is a leaf. If set, the node contains voxel data; otherwise, it contains child pointers [S2].
3.  **Child Pointer (31 bits):** An offset or pointer to the first child node. Only children that exist are stored contiguously in memory following the parent [S2].

The total size is $64 + 1 + 31 = 96$ bits = 12 bytes [S2].

### Traversal Algorithm
Traversal is typically performed using a ray marching algorithm optimized for the bitmask structure [S1, unofficial].

1.  **Coordinate Mirroring:** The coordinate system is mirrored to the negative ray octant to simplify intersection calculations and node bounds clamping [S1, unofficial].
2.  **Stack Management:** An ancestor node stack is stored in group-shared memory on GPU to reduce latency [S1, unofficial].
3.  **Ray Marching:**
    -   Start at the root node.
    -   For each node, check the child mask against the ray's current position relative to the node's bounds.
    -   Identify the first active child (bit set in mask) that the ray intersects.
    -   Jump to the child node.
    -   Repeat until a leaf node is reached.
    -   In the leaf, perform standard ray-marching through the $4 \times 4 \times 4$ voxels [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Tree64 Ray Traversal
# Assumes node layout: [mask: 64b, leaf: 1b, ptr: 31b]
# Ray: origin, direction, t_max

def traverse_tree64(ray, root_node):
    current_node = root_node
    stack = [] # Group-shared memory on GPU
    
    while current_node is not None:
        # Check if leaf
        if is_leaf(current_node):
            # March through 4x4x4 voxels in the leaf
            return march_leaf_voxels(ray, current_node)
        
        # Determine which children are intersected
        # Calculate relative position of ray within current node's bounds
        # This is simplified; actual implementation uses mirrored coordinates
        relative_pos = ray.origin - current_node.bounds.min
        
        # Find first active child in bitmask that ray enters
        # The bitmask order corresponds to spatial layout of 4x4x4 children
        child_index = find_first_active_child(current_node.mask, relative_pos, ray.direction)
        
        if child_index is None:
            return None # Ray missed this subtree
        
        # Push current node to stack for backtracking if needed (though DDA usually goes deep)
        stack.append(current_node)
        
        # Jump to child
        # Children are stored contiguously after the parent
        current_node = get_child_at_offset(current_node, child_index)
        
    return None
```

### Edge Cases and Pitfalls
-   **Editing Complexity:** Edits with compressed nodes are complicated and require a pool allocator [S2].
-   **Rebuilding:** Trees within chunks are fully rebuilt upon changes, using a traditional chunk hierarchy for bookkeeping [S1, unofficial].
-   **Deduplication:** Tree64 deduplicates less effectively than a Sparse Voxel DAG (SVDAG) because its node layout cannot represent complete leaf paths [S2].

## Subtopics

### [Voxtree](4994-voxtree.md)
A Rust implementation of Tree64 optimized for CPU performance, using address-shifting for dynamic updates.

## Operating characteristics
-   **Memory:** Approximately 0.19 bytes per voxel when fully populated, nearly 3x less than Sparse Voxel Octrees (SVOs) [S2]. In the Bistro scene at 8K resolution, it used 224 MB (0.62 bytes/voxel) compared to 368 MB for ESVO [S2].
-   **Performance:** On the Sponza 2k scene, Tree64 achieved 182.6 Mrays/s for primary rays and 124.2 Mrays/s for primary + 1 diffuse bounce [S1, unofficial]. On the Eco House 2k scene, it achieved 107.5 Mrays/s [S1, unofficial]. On the Bistro 4k scene, it achieved 109.7 Mrays/s [S1, unofficial].
-   **Depth:** The tree is half as deep as an octree, reducing traversal overhead [S2].
-   **Limitations:** Editing is more complex than SVOs due to the need for pool allocators and full chunk rebuilding [S1, unofficial][S2]. It does not support efficient subtree merging like DAGs [S2].

## Background
Tree64 was introduced as a sparse voxel tree with a 4³ branching factor, also known as a contree [S1, unofficial]. It was implemented in the VoxelRT project by dubiousconst282 [S1, unofficial]. The structure aims to balance the shallow depth of wide trees with the memory efficiency of sparse representations [S1, unofficial][S2].

## Sources
[S1] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
[S2] https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, fetched 2026-10-28
