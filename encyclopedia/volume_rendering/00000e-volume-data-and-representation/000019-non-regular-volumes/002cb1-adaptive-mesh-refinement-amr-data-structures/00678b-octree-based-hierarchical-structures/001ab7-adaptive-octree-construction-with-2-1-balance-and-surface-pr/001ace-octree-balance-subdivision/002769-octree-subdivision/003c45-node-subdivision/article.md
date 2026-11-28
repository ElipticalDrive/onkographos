# Node Subdivision

## Overview
Node Subdivision is the process of subdividing octree nodes into eight children by splitting the bounding box in half along each axis, redistributing existing surfels [Parent]. It executes in three distinct steps: flagging nodes, allocating subnode tiles, and initializing new nodes [S1, unofficial].
## How it works
**Inputs and Outputs:**
- **Input:** A list of voxel fragments, the current octree structure, and a target subdivision level [S1, unofficial].
- **Output:** An updated octree where flagged nodes are split into eight children, linked via a tile allocation, and initialized [S1, unofficial].

**Data Structures:**
- **Octree Node:** Contains a pointer to its children [S1, unofficial].
- **Voxel Fragment List:** An array of entries representing samples or geometric primitives to be integrated into the octree [S1, unofficial].
- **Octree Pool:** A contiguous memory block where new node tiles are allocated [S1, unofficial].
- **Atomic Counter:** A shared counter used to allocate indices within the Octree Pool [S1, unofficial].

**Steps:**
1. **Flagging:** Determine which nodes need subdivision [S1, unofficial].
2. **Allocation:** Allocate memory for the eight child nodes (a tile) for each flagged node [S1, unofficial].
3. **Initialization:** Initialize the new child nodes by setting their child node pointers to null [S1, unofficial].

**Detailed Procedure:**
1. **Flagging Nodes:** Launch one thread per entry in the voxel-fragment list [S1, unofficial]. Each thread traverses the octree from top to bottom to the current level and flags the node where it ends [S1, unofficial]. This flag is implemented by setting the most significant bit of the node's children pointer [S1, unofficial].
2. **Allocating Subnode Tiles:** Whenever a node is flagged, a set of eight subnodes (a tile) is allocated inside the octree pool [S1, unofficial]. Allocation is performed using a shared atomic counter [S1, unofficial].
3. **Initializing New Nodes:** A separate pass initializes the new nodes, primarily setting child node pointers to null [S1, unofficial]. One thread is associated with each node of the new octree level [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flag_nodes(voxel_fragments, octree_nodes, root_index, target_level):
    for fragment in voxel_fragments:
        node_idx = root_index
        for _ in range(target_level):
            child_idx = compute_child_index(octree_nodes[node_idx], fragment.position)  # [G]
            node_idx = octree_nodes[node_idx].children_ptr + child_idx
        octree_nodes[node_idx].children_ptr |= (1 << 31)

def allocate_tiles(octree_nodes, pool_counter, nodes_at_current_level):
    for node_idx in nodes_at_current_level:
        if octree_nodes[node_idx].children_ptr & (1 << 31):
            base_idx = atomic_add(pool_counter, 8)  # [G]
            octree_nodes[node_idx].children_ptr = base_idx

def initialize_nodes(octree_nodes, num_new_nodes):
    for i in range(num_new_nodes):
        octree_nodes[i].children_ptr = 0
```

**Parameters:**
- `target_level`: The depth in the octree to which fragments are projected for flagging. Not given in the sources.
- `pool_size`: Size of the Octree Pool. Not given in the sources.

**Edge Cases and Pitfalls:**
- Not given in the sources.

## Subtopics

### [Adaptive Volume Subdivision](14948-adaptive-volume-subdivision.md)
Adaptive Volume Subdivision is a method for managing the resolution of irradiance volumes by using an octree structure to store samples, subdividing only in regions where irradiance changes rapidly and keeping coarse resolution where it changes slowly. It solves the problem of uniform grids becoming unwieldy for large, complex scenes by allowing variable granularity. It differs from uniform grid approaches by dynamically adjusting voxel size based on scene complexity and irradiance gradients, and from static hierarchical methods by relying on specific heuristics (like scene depth) to drive the subdivision decisions.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** High performance on GPU when all threads operate on the same static memory region using atomic counters [S1, unofficial].
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to Use:** Not given in the sources.
- **When Not to Use:** Not given in the sources.

## Background
Node Subdivision is a component of top-down octree construction with thread scheduling [S1, unofficial]. It is used in sparse voxelization and irradiance volume rendering [S1, unofficial].

## Sources
[S1] PDF K14190_text - Icare3D, https://icare3d.org/research/OpenGLInsights-SparseVoxelization.pdf, fetched 2026-10-29 (unofficial)
