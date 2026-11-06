# Tree64
Also called: contree, wide 4³ trees

## Overview
Tree64 is a sparse voxel tree data structure with a 4³ branching factor, designed to reduce tree depth and per-voxel overhead compared to standard octrees [S1, unofficial]. It achieves higher ray traversal throughput by utilizing compact node layouts and optimized GPU traversal kernels, making it suitable for real-time rendering of large, sparse voxel scenes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A sparse 3D voxel dataset (density/opacity values).
- Output: A hierarchical tree structure optimized for ray traversal and a rendered image.

**Data Structures:**
- **Node Layout:** Each node consumes 12 bytes of memory [S1, unofficial].
- **Branching Factor:** 64 children per node (4×4×4 sub-volumes) [S1, unofficial].
- **Pointer Encoding:** Relative pointers are stored as "backwards" references to allow tree construction without intermediate buffers (parent nodes appear after children) [S1, unofficial]. Far pointers are signaled by a reserved upper range in the pointer value rather than a dedicated bit [S1, unofficial].
- **Morton Ordering:** Trees are built in depth-first order, resulting in a Z-curve (Morton) layout in memory, which improves spatial locality and memory bandwidth utilization [S1, unofficial].

**Algorithm Steps:**
1. **Tree Construction:**
   - Recursively subdivide the 3D domain into 4×4×4 blocks.
   - Store nodes in a linear array following a depth-first traversal order.
   - Encode child pointers relative to the current node, using the reserved range for far pointers [S1, unofficial].
   - *Note:* The specific bit-packing of the 12-byte node structure (type flags, child pointers, data) is not fully detailed in the sources, but Voxtree implements a similar concept by packing node type into the address field [S1, unofficial].

2. **Ray Traversal (GPU):**
   - **Coordinate System:** Mirror the coordinate system to the negative ray octant to simplify intersection calculations and node bounds clamping, removing the need for face plane offsets [S1, unofficial].
   - **Traversal Loop:**
     - Use a ray marching algorithm optimized for bitmasks to allow longer ray steps and reduce divergence [S1, unofficial].
     - Maintain an ancestor node stack in group-shared memory to achieve a ~9% speedup compared to local array variables [S1, unofficial].
     - Traverse from root to leaf, skipping empty subtrees based on node type/flags.
   - **Initial Implementation:** A top-down recursive DDA over bitmasks was used initially, performing 11-15% faster than ESVO for primary rays [S1, unofficial].

**Parameters:**
- Branching factor: 4³ (64) [S1, unofficial].
- Node size: 12 bytes [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Editing:** The structure uses a traditional chunk hierarchy for bookkeeping. Trees within each chunk are fully rebuilt upon changes, which impacts dynamic update performance [S1, unofficial].
- **Precision:** The use of "backwards" relative pointers requires careful handling of pointer arithmetic to avoid overflow or incorrect addressing, especially with far pointers [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Simplified representation of Tree64 node structure
class Tree64Node:
    # 12 bytes total layout (exact bit fields not specified in sources)
    # Assumed fields based on description:
    type_flags: int      # Indicates empty, leaf, or branch
    child_ptr_0: int     # Relative pointer to child 0
    child_ptr_1: int     # Relative pointer to child 1
    # ... up to child_ptr_63
    # Data field for leaf nodes (density/opacity)

def traverse_ray_tree64(ray_origin, ray_direction, root_node, shared_stack):
    """
    GPU kernel for traversing a Tree64 structure.
    """
    # Mirror coordinates to negative ray octant for simplified bounds
    mirrored_origin = mirror_to_negative_octant(ray_origin)
    mirrored_direction = mirror_to_negative_octant(ray_direction)
    
    current_node = root_node
    stack = shared_stack # Group-shared memory for ancestor stack
    
    while current_node is not None:
        # Check node type
        if is_empty(current_node):
            # Skip this subtree
            current_node = pop_stack(stack)
            continue
        elif is_leaf(current_node):
            # Sample density/opacity
            sample = sample_leaf(current_node, mirrored_origin, mirrored_direction)
            return sample
        else:
            # Determine which child to visit next
            # Use bitmask optimization for longer steps
            child_index = get_next_child_bitmask(current_node, mirrored_origin, mirrored_direction)
            
            # Push current node to stack if needed for backtracking
            if needs_backtracking(current_node):
                push_stack(stack, current_node)
            
            # Follow pointer
            # Handle far pointers via reserved range check
            current_node = follow_relative_pointer(current_node, child_index)
            
    return EMPTY
```

## Subtopics

### [Voxtree](4994-voxtree.md)
A Rust implementation of a 64-way sparse voxel tree that builds on Tree64 concepts, focusing on CPU performance and dynamic updates with a specific memory layout.

## Operating characteristics
**Time and Memory Complexity:**
- **Editing Cost:** Approximately $O(\log m)$ due to path copying capabilities, though full chunk rebuilds are required for changes [S1, unofficial].
- **Memory:** 12 bytes per node [S1, unofficial].

**Measured Performance:**
- **Hardware:** Tested on GPU (Vulkan SDK required) [S1, unofficial].
- **Scenes:**
  - **Eco House 2k (448k * 8³ voxels):** 107.5 Mrays/s for primary rays, 93.9 Mrays/s for primary + 1 diffuse bounce (0.87x ratio) [S1, unofficial].
  - **Bistro 4k (975k * 8³ voxels):** 109.7 Mrays/s for primary rays, 92.6 Mrays/s for primary + 1 diffuse bounce (0.84x ratio) [S1, unofficial].
  - **Forest Lake 2k (807k * 8³ voxels):** 112.1 Mrays/s for primary rays, 99.5 Mrays/s for primary + 1 diffuse bounce (0.89x ratio) [S1, unofficial].
  - **Sponza 2k:** 182.6 Mrays/s for primary rays, 124.2 Mrays/s for primary + 1 diffuse bounce [S1, unofficial].
- **Comparison:** Outperforms ESVO, PlainDDA, and MultiDDA in primary ray throughput on these scenes [S1, unofficial]. Comparable to XBrickMap and StdBVH in some metrics, but with lower CPU/GPU sync overhead than ESVO and BVH variants [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Shallower trees than octrees, lower per-voxel overhead, high throughput for coherent rays, efficient memory usage (12 bytes/node) [S1, unofficial].
- **Limitations:** Full chunk rebuilds on edits, potential divergence in ray marching if not optimized, complex pointer encoding [S1, unofficial].

## Background
Tree64 was introduced as part of the VoxelRT project, an open-source real-time voxel renderer [S1, unofficial]. It builds upon earlier work on sparse voxel octrees and wide branching factor trees, aiming to balance traversal speed and memory efficiency for real-time applications [S1, unofficial]. The structure is closely related to the work of Laine and Karras on efficient-sparse-voxel-octrees [S1, unofficial].

## Sources
[S1] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
