# Sparse Voxel Octree (SVO) Builder

Also called: BuildNodeRecursive, BuildSVO, SVO Build, SVO construction, Sparse Voxel Octree (SVO) Construction

## Overview
An algorithm to construct a Sparse Voxel Octree from a stream of voxelized geometry or a voxel list by maintaining a flat array of nodes and expanding them level-by-level or recursively [S1, unofficial][S4, unofficial]. It maps occupied voxels to hierarchical paths in the octree, pre-allocating memory for internal and leaf nodes up to a specified maximum depth [S2, unofficial]. Unlike queue-based methods that merge nodes dynamically, this approach follows a strict depth-first or level-order expansion pattern to ensure a predictable memory layout [S1, unofficial].

## How it works
**Inputs:**
- `maxDepth`: The maximum depth of the octree. Determines when leaf nodes are created instead of children [S2, unofficial].
- `voxel_list`: A list of occupied voxels used to flag internal nodes and leaves during GPU-based iterative builds [S4, unofficial].
- `collision_trace` (optional): A custom collision trace channel (e.g., `SVOObstacle`) used exclusively for occupancy testing, with a default response of Ignore [S3, unofficial].

**Outputs:**
- `_tree`: A flat array of `uint32_t` representing the SVO nodes [S2, unofficial].

**Data Structures:**
The `_tree` array stores node metadata as `uint32_t` [S2, unofficial]:
- **Bits 0–16 (17 bits):** Index of the first child node [S2, unofficial].
- **Bit 17 (1 bit):** Leaf flag. If set, the node is a leaf [S2, unofficial].
- **Value `0x0000FFFF`:** Empty node. Child index is 0, leaf flag is 0 [S2, unofficial].
- **Value `0x00004200`:** Solid leaf node. Indicates an occupied leaf [S2, unofficial].
*Note: Alternative implementations may use a 1-byte struct with 7 bits for Depth and 1 bit for IsLeaf, plus fields for position, size, parent index, and children indices [S3, unofficial]. The following pseudocode follows the dense flat array layout [S2, unofficial].*

**Algorithm Steps (Recursive/Stack-based Dense Build):**
1. Initialize `_tree` with a single root node `0x0000FFFF` [S2, unofficial].
2. Initialize a stack with `StackEntry{index=0, depth=0}` [S2, unofficial].
3. Loop while stack is not empty:
    a. Pop top of stack into `current`.
    b. Set `index = current.index`, `depth = current.depth`.
    c. Calculate `offset = _tree.size() - index`.
    d. Update `_tree[index]` with child offset: `_tree[index] |= offset << 17` [S2, unofficial].
    e. Increment `index` by `offset`.
    f. If `depth < maxDepth`:
        i. Increment `depth`.
        ii. Loop 8 times (for each octant):
            1. Push `0x0000FFFF` to `_tree`.
            2. Push `StackEntry{index + (8 - i), depth}` to stack [S2, unofficial].
    g. Else (leaf case):
        i. Loop 8 times:
            1. Push `0x00004200` to `_tree` [S2, unofficial].
4. Return `_tree` [S2, unofficial].

**Algorithm Steps (GPU Iterative Build):**
1. Initialize root node (level 0) by clearing memory for the first 8 nodes using a node initialization routine [S4, unofficial].
2. Iterate from level 1 to `resolution - 1`:
    a. Execute a node flagging routine to determine which nodes at the current level are internal (have children) or leaves, binding the octree buffer and voxel list [S4, unofficial].
    b. Execute a node allocation routine to allocate memory for children of internal nodes using an atomic counter. The output start address is `alloc_start`, and the count of new nodes is `alloc_count * 8` [S4, unofficial].
    c. Update state variables: `start` becomes `alloc_start`, `count` becomes `alloc_count * 8`, and `alloc_start` becomes `start + count` [S4, unofficial].
    d. Clear memory for newly allocated nodes using a node initialization routine [S4, unofficial].
3. Execute a leaf storage routine to store leaf node data, binding the octree buffer and voxel list [S4, unofficial].

**Pseudocode (Dense Recursive/Stack-based):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_svo_dense(max_depth):
    EMPTY_NODE = 0x0000FFFF
    LEAF_NODE = 0x00004200
    tree = [EMPTY_NODE]
    stack = [(0, 0)]

    while stack:
        idx, depth = stack.pop()
        offset = len(tree) - idx
        tree[idx] = (tree[idx] & 0x1FFFF) | (offset << 17)
        idx += offset

        if depth < max_depth:
            depth += 1
            for i in range(8):
                tree.append(EMPTY_NODE)
                stack.append((idx + (8 - i), depth))
        else:
            for i in range(8):
                tree.append(LEAF_NODE)

    return tree
```

**Pseudocode (GPU Iterative):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_svo_gpu(voxel_list, resolution):
    # Pre-allocate octree buffer
    octree_buffer = [0] * (8 * resolution)
    start = 0
    count = 8
    alloc_start = 0

    for level in range(1, resolution):
        # node_flag: determine which nodes at current level are internal or leaves
        # [G] Binds octree buffer and voxel list to compute shader dispatch
        internal_mask = [False] * count
        
        # node_alloc: allocate memory for children of internal nodes
        # [G] Uses atomic counter to track allocated nodes
        alloc_count = sum(internal_mask)
        alloc_start = len(octree_buffer)
        octree_buffer.extend([0] * (alloc_count * 8))
        
        # Update state variables
        new_start = alloc_start
        new_count = alloc_count * 8
        next_alloc_start = new_start + new_count
        
        # node_init: clear memory for newly allocated nodes
        for i in range(new_start, new_start + new_count):
            octree_buffer[i] = 0x0000FFFF
            
        start = new_start
        count = new_count
        alloc_start = next_alloc_start

    # store_leaf: store leaf node data
    # [G] Binds octree buffer and voxel list, dispatches compute work based on voxel list size
    for i in range(start, start + count):
        octree_buffer[i] = 0x00004200

    return octree_buffer
```

**Parameters:**
- `maxDepth`: Maximum octree subdivision depth. Root starts at Depth 0. Typical value is not specified in the sources, but determines when leaf nodes are created [S2, unofficial].
- `SVOObstacle`: Custom collision trace channel used exclusively for occupancy testing. Default response is Ignore [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Work Imbalance:** In parallel implementations, SVO sparsity can cause work imbalance, where one task takes significantly longer. Mitigated by using "Unbalanced" scheduling [S3, unofficial].
- **Root Collision:** If the root node collides with an obstacle, it must be recursively subdivided to `MaxDepth` [S3, unofficial].
- **Invalid Child Roots:** Child roots that are invalid (e.g., blocked) should be treated as non-existent and skipped during merge [S3, unofficial].
- **Buffer Preallocation:** The approach requires preallocating the octree buffer with a fixed size, which may limit dynamic scene updates [S5, unofficial].

## Subtopics

### [GPU-based Octree Generation](5332-gpu-based-octree-generation.md)
Generates a hierarchical octree data structure directly on the GPU by partitioning the input volume into fixed-size cubes of $32 \times 32 \times 32$ voxels. Unlike standard Sparse Voxel Octrees that only subdivide non-empty regions, this approach uniformly subdivides the entire volume space down to the minimum node size, which simplifies compute shader logic by avoiding dynamic branching for empty regions. It constructs a pre-allocated array-based octree by mapping every non-empty voxel to a hierarchical path.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** On a Kepler-based NVIDIA GTX680, with max resolution $512^3$ (9 octree levels) and RGBA32F values: Hand scene total time 3.04ms (with atomic merging), Dragon 12.98ms, Sponza 15.44ms [S5, unofficial]. Octree construction (flagging) is the most time-consuming step [S5, unofficial].
- **Scaling:** Performance is 30% to 58% faster than on a Fermi-based GTX480. Atomic fragment merging is up to 80% faster [S5, unofficial].
- **Strengths:** Reduces memory consumption compared to regular voxel grids [S5, unofficial]. Enables fast prevoxelization of static environments and real-time updates for dynamic objects [S5, unofficial].
- **Limitations:** Requires preallocating the octree buffer with a fixed size [S5, unofficial]. Work imbalance can occur in parallel implementations due to sparsity [S3, unofficial].
- **When to use:** When a dense, flat array representation is required for efficient GPU raycasting or voxel-based global illumination [S5, unofficial].
- **When not to use:** When memory efficiency is critical for highly sparse volumes [S5, unofficial].

## Background
The Sparse Voxel Octree (SVO) Builder is a method for constructing hierarchical acceleration structures for volume rendering and raycasting [S5, unofficial]. It is based on the method described in Chapter 22 of OpenGL Insight by Cyril Crassin and Simon Green [S5, unofficial]. The algorithm is used inside voxel-based global illumination techniques for prevoxelizing static environments and updating dynamic objects in real time [S5, unofficial]. Dense recursive builds are documented in unofficial GitHub repositories [S1, unofficial]. GPU-based iterative builds are documented in unofficial GitHub repositories [S4, unofficial]. Pathfinding-specific SVO constructions are documented in unofficial video tutorials [S3, unofficial].

## Sources
[S1] otaku690/SparseVoxelOctree, https://github.com/otaku690/SparseVoxelOctree, fetched 2026-10-26 (unofficial)
[S2] cianjinks/Sparse-Voxel-Octree, https://github.com/cianjinks/Sparse-Voxel-Octree, fetched 2026-10-26 (unofficial)
[S3] UE5 Tutorial | Simple SVO Pathfinder for 3D Pathfinding, https://www.youtube.com/watch?v=QmDbX_gZSLA, fetched 2026-10-27 (unofficial)
[S4] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S5] PDF K14190_text - Icare3D, https://icare3d.org/research/OpenGLInsights-SparseVoxelization.pdf, fetched 2026-10-29 (unofficial)
