# Tree64
Also called: Sparse 64-trees, contree, wide 4³ trees

## Overview
Tree64 is a sparse voxel tree with a $4^3$ branching factor, also known as a contree [S1, unofficial][S2]. Its main advantages over standard octrees are shallower trees and lower overhead per voxel [S1, unofficial]. Traversal is fast because each level skips $4^3$ cells and the tree is half as deep as an octree [S2]. It is primarily used for real-time ray tracing of large voxel scenes [S1, unofficial].

## How it works
Tree64 represents a 3D volume as a tree where each internal node covers a $4 \times 4 \times 4$ sub-volume [S1, unofficial]. Unlike standard octrees that branch by 8, Tree64 nodes branch by 64, allowing for shallower hierarchies and fewer traversal steps [S1, unofficial][S2].

### Inputs and Outputs
- **Input:** A 3D voxel grid (implicitly or explicitly defined).
- **Output:** A memory layout of Tree64 nodes representing the sparse volume.

### Data Structures
Each Tree64 node has a fixed size of 12 bytes [S2]. The layout is:
1. **Child Mask (64 bits):** A bitmask indicating which of the 64 children exist. If bit $i$ is set, child $i$ exists; otherwise, it is empty [S2].
2. **Leaf Flag (1 bit):** Indicates if the node is a leaf. If set, the node contains voxel data; otherwise, it contains child pointers [S2].
3. **Child Pointer (31 bits):** An offset or pointer to the first child node. Only children that exist are stored [S2]. Relative pointers are "backwards" to allow for trees to be built without intermediate buffers (i.e., parent nodes appear after their children), and far pointers are signaled by a reserved upper range rather than a dedicated bit [S1, unofficial].

The total size is $64 + 1 + 31 = 96$ bits = 12 bytes [S2].

### Traversal Algorithm
Traversal is typically performed using a top-down DDA over bitmasks, which was optimized to a ray marching algorithm that benefits from bitmasks for longer ray steps and is less prone to divergence [S1, unofficial].

1. **Coordinate Mirroring:** The coordinate system is mirrored to the negative ray octant to simplify intersection calculations and node bounds clamping [S1, unofficial].
2. **Stack Management:** An ancestor node stack is stored in group-shared memory on GPU to reduce latency [S1, unofficial].
3. **Ray Marching:**
    - Start at the root node.
    - Initialize a 3D DDA state for the $4 \times 4 \times 4$ child grid.
    - Step through the grid, checking the child mask against the current cell index.
    - Jump to the child node indicated by the active bitmask entry.
    - Repeat until a leaf node is reached.
    - In the leaf, perform standard ray-marching through the contained voxels [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_tree64(ray_origin, ray_direction, root_node):
    BLOCK_SIZE = 4
    # [G] Initialize 3D DDA state for a 4x4x4 grid
    inv_dir = [1.0 / d if d != 0.0 else float('inf') for d in ray_direction]
    # Assume ray_origin is transformed to the mirrored negative octant [S1, unofficial]
    x, y, z = int(ray_origin[0]), int(ray_origin[1]), int(ray_origin[2])
    t_max = [inv_dir[i] * (next_boundary - ray_origin[i]) for i, next_boundary in enumerate([x + 1, y + 1, z + 1])]
    step = [1 if d > 0 else -1 for d in ray_direction]
    
    current_node = root_node
    while current_node is not None:
        # Check leaf flag (bit 0 of the 12-byte node)
        if current_node.flags & 1:
            return march_leaf_voxels(ray_origin, ray_direction, current_node.data)
        
        # Map 3D grid position to 4x4x4 child index
        child_idx = x + 4 * y + 16 * z
        if not (current_node.mask >> child_idx) & 1:
            # Advance DDA to next cell boundary
            if t_max[0] < t_max[1]:
                if t_max[0] < t_max[2]:
                    x += step[0]; t_max[0] += inv_dir[0]
                else:
                    z += step[2]; t_max[2] += inv_dir[2]
            else:
                if t_max[1] < t_max[2]:
                    y += step[1]; t_max[1] += inv_dir[1]
                else:
                    z += step[2]; t_max[2] += inv_dir[2]
            child_idx = x + 4 * y + 16 * z
            if not (current_node.mask >> child_idx) & 1:
                return None
        
        # Jump to child node using relative pointer arithmetic
        current_node = current_node.children[child_idx]
    return None

def march_leaf_voxels(ray_origin, ray_direction, voxel_data):
    # [G] Standard ray-marching through the 4x4x4 voxel grid
    # Returns hit distance or None
    ...
```

### Edge Cases and Pitfalls
- **Editing Complexity:** Edits with compressed nodes are complicated and require a pool allocator [S2]. Editing cost is approximately O(log m) due to path copying capabilities [S1, unofficial].
- **Rebuilding:** Trees within chunks are fully rebuilt upon changes, using a traditional chunk hierarchy for bookkeeping [S1, unofficial].
- **Deduplication:** Tree64 deduplicates less effectively than a Sparse Voxel DAG (SVDAG) because its node layout cannot represent complete leaf paths [S2].

## Subtopics

### [Voxtree](4994-voxtree.md)
A Rust implementation of Tree64 optimized for CPU performance, using address-shifting for dynamic updates.

## Operating characteristics
- **Memory:** Approximately 0.19 bytes per voxel when fully populated, nearly 3x less than Sparse Voxel Octrees (SVOs) [S2]. In the Bistro scene at 8K resolution, it used 224 MB (0.62 bytes/voxel) compared to 368 MB for ESVO [S2].
- **Performance:** On the Sponza 2k scene, Tree64 achieved 182.6 Mrays/s for primary rays and 124.2 Mrays/s for primary + 1 diffuse bounce [S1, unofficial]. On the Eco House 2k scene, it achieved 107.5 Mrays/s [S1, unofficial]. On the Bistro 4k scene, it achieved 109.7 Mrays/s [S1, unofficial]. Speed claims are scene- and implementation-dependent [S2].
- **Depth:** The tree is half as deep as an octree, reducing traversal overhead [S2].
- **Limitations:** Editing is more complex than SVOs due to the need for pool allocators and full chunk rebuilding [S1, unofficial][S2]. It does not support efficient subtree merging like DAGs [S2].

## Background
Tree64 was introduced as a sparse voxel tree with a $4^3$ branching factor, also known as a contree [S1, unofficial]. It was implemented in the VoxelRT project by dubiousconst282 [S1, unofficial]. The structure aims to balance the shallow depth of wide trees with the memory efficiency of sparse representations [S1, unofficial][S2].

## Sources
[S1] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
[S2] https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, fetched 2026-10-28
