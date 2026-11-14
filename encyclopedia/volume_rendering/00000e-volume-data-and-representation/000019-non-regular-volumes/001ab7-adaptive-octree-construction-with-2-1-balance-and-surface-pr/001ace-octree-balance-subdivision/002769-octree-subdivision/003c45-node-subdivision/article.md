# Node Subdivision
Also called: Octree Node Subdivision, Adaptive Octree Node Subdivision

## Overview
Node Subdivision is the core mechanism for refining an adaptive octree during top-down construction, transforming coarse nodes into finer 2×2×2 tiles to resolve high-frequency geometry or irradiance gradients. It differs from static subdivision by dynamically determining which nodes require refinement based on a pre-computed list of voxel fragments, and from uniform grid approaches by allowing variable granularity [S1, unofficial]. This method enables efficient memory usage by only allocating child nodes where necessary, addressing the unwieldiness of uniform grids in large, complex scenes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A list of voxel fragments (each containing position and data), the current octree structure, and a target subdivision level.
- **Output:** An updated octree where flagged nodes are split into 8 children, linked via a tile allocation, and initialized.

**Data Structures:**
- **Octree Node:** Contains a pointer to its children. The children pointer is a 32-bit or 64-bit integer. The most significant bit (MSB) is used as a flag.
- **Voxel Fragment List:** An array of structures, each representing a sample or geometric primitive to be integrated into the octree.
- **Octree Pool:** A contiguous memory block (or pool) where new node tiles are allocated.
- **Atomic Counter:** A shared counter used to allocate indices within the Octree Pool.

**Steps:**
1. **Flagging:** Determine which nodes need subdivision.
2. **Allocation:** Allocate memory for the 8 child nodes (a tile) for each flagged node.
3. **Initialization:** Initialize the new child nodes (e.g., setting pointers to null).

**Detailed Procedure:**

1. **Flagging Nodes:**
   - Launch one thread per entry in the voxel-fragment list.
   - Each thread traverses the octree from the root down to the target level.
   - The traversal follows the path defined by the fragment's position.
   - At the target level, the thread identifies the node that contains the fragment.
   - The thread sets the most significant bit (MSB) of that node's children pointer to indicate it must be subdivided.
   - Since multiple threads may target the same node, this operation is idempotent regarding the flag state (once flagged, it stays flagged).

2. **Allocating Subnode Tiles:**
   - Iterate through all nodes at the current level.
   - For each node where the MSB of the children pointer is set:
     - Use a shared atomic counter to allocate a contiguous block of memory for 8 new nodes (a tile) from the Octree Pool.
     - Update the node's children pointer to point to the base address of this new tile.
     - Clear the MSB flag in the children pointer to indicate the node is now subdivided and the pointer is valid.

3. **Initializing New Nodes:**
   - Launch one thread per newly created node in the new octree level.
   - Each thread initializes its assigned node.
   - Primary initialization: Set the node's children pointer to `null` (or 0).
   - This ensures that leaf nodes or nodes not flagged for further subdivision in subsequent passes start with a clean state.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global state
octree_nodes: Array[OctreeNode]
voxel_fragments: Array[VoxelFragment]
octree_pool: Array[OctreeNode]
pool_counter: AtomicInt = 0

def flag_nodes():
    # Launch one thread per fragment
    parallel_for i in range(len(voxel_fragments)):
        fragment = voxel_fragments[i]
        node = root_node
        # Traverse to the target level
        for level in range(target_level):
            child_index = get_child_index(node, fragment.position)
            node = octree_nodes[node.children_ptr + child_index]
        # Flag the node by setting MSB of children pointer
        # Assuming children_ptr is an integer index into octree_nodes
        node.children_ptr |= (1 << 31) # Set MSB

def allocate_tiles():
    # Iterate over all nodes at the current level
    # Note: In practice, this might be done via a traversal or by iterating
    # a list of nodes at the specific level.
    for node in nodes_at_current_level:
        if node.children_ptr & (1 << 31): # Check MSB
            # Allocate tile of 8 nodes
            base_index = atomic_add(pool_counter, 8)
            # Update children pointer to point to the new tile
            # Clear the MSB flag
            node.children_ptr = base_index

def initialize_nodes():
    # Launch one thread per new node
    # The number of new nodes is 8 * number_of_flagged_nodes
    parallel_for i in range(num_new_nodes):
        node_index = i
        octree_nodes[node_index].children_ptr = 0 # Null pointer
```

**Parameters:**
- `target_level`: The depth in the octree to which fragments are projected for flagging. Typical values depend on scene scale and desired resolution.
- `pool_size`: Size of the Octree Pool. Must be large enough to hold all potential subdivisions.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point position precision may cause fragments to be assigned to incorrect leaf nodes if not handled carefully.
- **Boundary Conditions:** Fragments on node boundaries must have a deterministic assignment rule (e.g., based on coordinate comparison).
- **Memory Exhaustion:** If the Octree Pool is too small, allocation will fail. Dynamic resizing or a larger initial pool is required.
- **Race Conditions:** The atomic counter for pool allocation must be thread-safe. The flagging step is safe because setting the MSB is idempotent.

## Subtopics

### [Adaptive Volume Subdivision](14948-adaptive-volume-subdivision.md)
Adaptive Volume Subdivision is a method for managing the resolution of irradiance volumes by using an octree structure to store samples, subdividing only in regions where irradiance changes rapidly and keeping coarse resolution where it changes slowly. It solves the problem of uniform grids becoming unwieldy for large, complex scenes by allowing variable granularity. It differs from uniform grid approaches by dynamically adjusting voxel size based on scene complexity and irradiance gradients, and from static hierarchical methods by relying on specific heuristics (like scene depth) to drive the subdivision decisions.

## Operating characteristics
- **Time Complexity:** $O(N \cdot L)$ for flagging, where $N$ is the number of voxel fragments and $L$ is the depth of the octree. Allocation and initialization are $O(M)$, where $M$ is the number of newly created nodes.
- **Memory Complexity:** $O(M)$ for the new nodes, where $M$ is the number of subdivided nodes. The octree pool must be pre-allocated or dynamically managed.
- **Performance:** High performance on GPU due to parallel flagging and atomic operations. The use of atomic counters for pool allocation is efficient for static memory regions [S1, unofficial].
- **Strengths:** Efficient memory usage for sparse data; adaptive resolution; parallelizable.
- **Limitations:** Requires pre-computation of voxel fragments; atomic operations can become a bottleneck if contention is high; memory fragmentation if pool is not managed well.
- **When to Use:** When dealing with large, sparse scenes where uniform grids are too memory-intensive; when high-frequency details are localized.
- **When Not to Use:** When the scene is dense and uniform, or when memory allocation overhead is prohibitive.

## Background
Node Subdivision is a key component of adaptive octree construction methods, particularly in the context of sparse voxelization and irradiance volume rendering. It was described in the context of the Icare3D project, which focuses on real-time rendering techniques using OpenGL and GPU acceleration [S1, unofficial]. The method leverages GPU parallelism to efficiently manage the hierarchical structure of the octree, enabling real-time adaptive refinement.

## Sources
[S1] PDF K14190_text - Icare3D, https://icare3d.org/research/OpenGLInsights-SparseVoxelization.pdf, fetched 2026-10-29 (unofficial)
