# Sparse Voxel Octree (SVO) Builder

Also called: SVO Builder, BuildSVO, SVO construction, Sparse Voxel Octree (SVO) Construction, Voxel-to-SVO Builder

## Overview
The Sparse Voxel Octree (SVO) Builder constructs a hierarchical Sparse Voxel Octree from a flat list of voxelized geometry fragments. It differs from its parent [Sparse Voxel Octree Construction (Basic Version)](1694-sparse-voxel-octree-construction-basic-version.md) by explicitly detailing the GPU compute shader pipeline for level-by-level node allocation, attribute injection, and mipmap generation, whereas the parent focuses on the high-level logical steps of flagging and memory allocation [S1, S5, unofficial]. It addresses the problem of efficiently converting triangle meshes into a compact hierarchical voxel representation for real-time global illumination and pathfinding [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of voxelized meshes (fragments), typically generated via conservative rasterization or ray marching. Each fragment is defined by its 3D integer coordinates $(x, y, z)$ and optional attributes (albedo, normal, density) [S1, S2, S5, unofficial].
*   **Parameters:**
    *   `voxel_size`: Physical size of a single voxel in world space [S2, unofficial].
    *   `maxDepth`: Maximum subdivision depth of the octree. Typical values are not specified in the sources, but it determines when leaf nodes are created instead of children [S3, S4, unofficial].
    *   `bounds`: User-specified world-space bounding box. If input meshes fall outside these bounds, the builder automatically expands `min`/`max` to include them, and the final bounds are extended to a power of two [S2, unofficial].
*   **Output:** A Sparse Voxel Octree data structure containing nodes with parent pointers, neighbor pointers, and attribute data [S2, S5, unofficial].

**Data Structures:**
The tree is stored in a flat array `_tree` of `uint32_t` values [S3, unofficial]. Each node entry uses the following bit layout:
*   **Bits 0–16 (17 bits):** Index of the first child node in the `_tree` array.
*   **Bit 17 (1 bit):** Leaf flag. If set, the node is a leaf; if clear, it is an internal node with children.
*   **Value 0x0000FFFF:** Represents an empty node (child index 0, not a leaf) [S3, unofficial].
*   **Value 0x00004200:** Represents a solid leaf node [S3, unofficial].

*Note: Some implementations (e.g., UE5 pathfinding) use a 1-byte metadata struct where bits 0–6 store Depth and bit 7 stores IsLeaf, alongside separate fields for position, size, parent index, and children indices [S4, unofficial]. The GPU builder described below uses the flat `uint32_t` array layout.*

**Algorithm Steps:**
The construction is performed using GPU compute shaders in the following sequence [S1, S5, unofficial]:

1.  **Build Voxel Fragment List:** Collect all voxels from input meshes into a flat list of 3D integer coordinates (`Vec<IVec3>`) [S2, S5, unofficial]. Compute the bounding box in voxel coordinates:
    $$ \text{min} = \lfloor \text{world\_min} / \text{voxel\_size} \rfloor $$
    $$ \text{max} = \lceil \text{world\_max} / \text{voxel\_size} \rceil $$
    Calculate the origin (min) and size (max dimension rounded up to the next power of two) [S2, unofficial].

2.  **Initialize Root:** Clear memory for the first 8 nodes (start=0, count=8) using a `node_init` shader. Initialize the root node (level 0) [S5, unofficial].

3.  **Iterative Level Construction:** Iterate from level $L=1$ to `resolution - 1` (where `resolution` is $\lceil \log_2(\text{size}) \rceil$) [S5, unofficial]:
    a.  **Node Flagging:** Execute the `node_flag` shader. It binds the octree buffer and the voxel list to determine which nodes at the current level are internal (have children) or leaves. Nodes that need further division are tagged [S5, unofficial].
    b.  **Memory Allocation:** Execute the `node_alloc` shader. It uses an atomic counter to track the number of allocated nodes. The output start address is `alloc_start`, and the count of new nodes is `alloc_count * 8` [S5, unofficial].
    c.  **Initialization:** Clear the memory for the newly allocated nodes using the `node_init` shader [S5, unofficial].
    d.  **State Update:** Update state variables for the next iteration: `start` becomes `alloc_start`, `count` becomes `alloc_count * 8`, and `alloc_start` becomes `start + count` [S5, unofficial].

4.  **Leaf Storage:** After the loop, execute the `store_leaf` shader. It binds the octree buffer and voxel list, and dispatches compute work based on the voxel list size to store leaf node data [S5, unofficial].

5.  **Attribute Injection:** Inject voxel attributes (albedo, normal, etc.) into the leaf nodes. If multiple voxel fragments fall into the same leaf node, their attribute values are averaged [S1, unofficial].

6.  **Mipmap Generation:** Mipmap node values from bottom to top, level by level, into the interior nodes [S1, unofficial].

**Pseudocode:**
The following pseudocode combines the high-level logic from [S1, S5, unofficial] with the structural details from [S2, unofficial]. It assumes a GPU compute shader environment where `atomicAdd`, `node_init`, `node_flag`, `node_alloc`, and `store_leaf` are shader invocations.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_svo(voxel_list, voxel_size, bounds_min, bounds_max):
    # 1. Compute Bounding Box
    min_voxel = floor(bounds_min / voxel_size)
    max_voxel = ceil(bounds_max / voxel_size)
    
    # Handle empty input
    if min_voxel == max_voxel:
        origin = min_voxel
        size = 1
    else:
        origin = min_voxel
        # Round up to next power of two
        max_dim = max(max_voxel.x - min_voxel.x, 
                      max_voxel.y - min_voxel.y, 
                      max_voxel.z - min_voxel.z)
        size = next_power_of_two(max_dim)

    # 2. Initialize Root
    # Allocate space for root's 8 children
    alloc_start = 0
    alloc_count = 8
    node_init(alloc_start, alloc_count, value=0x0000FFFF) # Empty nodes

    current_start = 0
    current_count = 8
    resolution = log2(size)

    # 3. Iterative Level Construction
    for level in range(1, resolution):
        # Tag nodes that need further division
        # node_flag reads from current_start, writes flags to a temporary buffer
        node_flag(current_start, current_count, voxel_list)
        
        # Allocate memory for children of internal nodes
        # Returns new start address and count of allocated nodes
        alloc_start, alloc_count = node_alloc(current_start, current_count)
        
        # Initialize allocated memory
        node_init(alloc_start, alloc_count, value=0x0000FFFF)
        
        # Update state for next level
        current_start = alloc_start
        current_count = alloc_count * 8 # Each internal node has 8 children

    # 4. Store Leaves and Inject Attributes
    # Store leaf data from voxel_list into the tree
    store_leaf(current_start, current_count, voxel_list)
    
    # Average attributes if multiple fragments map to same leaf
    # This is handled within store_leaf or a post-process step
    average_leaf_attributes(current_start, current_count)

    # 5. Mipmap Values
    # Propagate values from leaves up to root
    mipmap_upwards(resolution)

    return SparseVoxelOctree(origin=origin, size=size)
```

**Edge Cases and Pitfalls:**
*   **Empty Input:** If the voxel list is empty, `min` and `max` are both zero. The builder handles this by returning a tree with `min == max` and `size = 1` [S2, unofficial].
*   **Bounds Expansion:** If input meshes fall outside user-specified bounds, the builder automatically expands `min`/`max` to include them. The final bounds are extended to a power of two [S2, unofficial].
*   **Layer Validation:** The builder validates that each layer's node count is a multiple of 8 and that positions follow `OFFSETS_IN_MORTON_CODE_ORDER` relative to the first child. This fails if grouping is incorrect [S2, unofficial].
*   **Missing Neighbors:** If a neighbor lacks children, the current node's children link directly to the neighbor instead of its children [S2, unofficial].
*   **Work Imbalance:** In parallel implementations, SVO sparsity can cause work imbalance. This is mitigated by using unbalanced scheduling [S4, unofficial].

## Subtopics

### [GPU-based Octree Generation](5332-gpu-based-octree-generation.md)
This method generates a hierarchical octree data structure directly on the GPU by partitioning the input volume into fixed-size cubes of $32 \times 32 \times 32$ voxels. Unlike standard Sparse Voxel Octrees that only subdivide non-empty regions, this approach uniformly subdivides the entire volume space down to the minimum node size, which simplifies compute shader logic by avoiding dynamic branching for empty regions. It constructs a pre-allocated array-based octree by mapping every non-empty voxel to a hierarchical path.

### [Sparse Octree Voxelization Performance](9137-sparse-octree-voxelization-performance.md)
This article documents the performance characteristics and implementation details of the Sparse Voxel Octree (SVO) construction method used for GPU-based voxelization. It addresses the problem of efficiently converting triangle meshes into a compact hierarchical voxel representation for real-time global illumination. Unlike in-core CPU methods or grid-based GPU approaches, this method generates the SVO directly on the GPU, optimizing for parallel triangle processing and minimizing atomic contention. It is distinct from [Grid-Free Out-Of-Core GPU Voxelization](11494-grid-free-out-of-core-gpu-voxelization.md) by its reliance on preallocated buffers and specific atomic merging strategies for fragment consolidation.

### [GPU-based SVO Construction and Mipmap Generation](16487-gpu-based-svo-construction-and-mipmap-generation.md)
A GPU compute shader pipeline that constructs a Sparse Voxel Octree (SVO) by allocating nodes per level, writing leaf nodes from a fragment list, and generating mipmaps level-by-level from leaves up to the root. It differs from its parent [Sparse Voxel Octree (SVO) Builder](2617-sparse-voxel-octree-svo-builder.md) by explicitly detailing the dispatch dimensions, constant buffer bindings, and the specific loop structure for mipmap generation, whereas the parent focuses on the high-level logical steps of flagging and memory allocation.

### [FlagOctree Construction](16509-flagoctree-construction.md)
FlagOctree Construction is a GPU-based algorithm for building a sparse voxel octree from a list of fragment positions using atomic operations to mark nodes in a shared pool as occupied. It builds the tree structure implicitly via pointers stored in a single `childPtr` field, distinguishing itself from sibling methods by its specific bit-packing strategy for node flags and pointer extraction.

## Operating characteristics
*   **Time Complexity:** Linear in the number of input voxels for collection and leaf grouping. Iterative layer construction scales with tree depth ($\log_2(\text{size})$). Parent/neighbor filling scales linearly with total node count [S2, unofficial].
*   **Memory Complexity:** The flat array `_tree` stores one `uint32_t` per node. The number of nodes depends on the sparsity of the input and the `maxDepth`.
*   **Performance:** The method is designed for GPU execution using compute shaders, leveraging parallel processing for voxelization and tree construction [S1, S5, unofficial].
*   **Strengths:** Reduces memory consumption compared to dense voxel grids. Efficiently handles large scenes by only storing non-empty regions [S1, unofficial].
*   **Limitations:** Requires careful management of atomic operations and memory allocation on the GPU. Sparsity can lead to work imbalance in parallel implementations [S4, unofficial].
*   **When to Use:** Real-time global illumination, pathfinding in large 3D environments, and scenarios requiring efficient spatial queries on voxelized data [S1, S2, unofficial].
*   **When Not to Use:** When the scene is extremely dense (approaching a full grid), the overhead of the octree structure may outweigh the benefits. When dynamic updates are frequent and the cost of rebuilding the tree is prohibitive.

## Background
The method is based on the algorithm described in Chapter 22 of *OpenGL Insights* by Cyril Crassin and Simon Green [S1, unofficial]. It is also referenced in the context of 3D flight navigation using sparse voxel octrees [S2, unofficial]. The implementation is used in the `svo-rs` Rust crate for Bevy engine integration [S2, unofficial].

## Sources
[S1] otaku690/SparseVoxelOctree, https://github.com/otaku690/SparseVoxelOctree, fetched 2026-10-26 (unofficial)
[S2] mkmarek/svo-rs, https://github.com/mkmarek/svo-rs, fetched 2026-10-26 (unofficial)
[S3] cianjinks/Sparse-Voxel-Octree, https://github.com/cianjinks/Sparse-Voxel-Octree, fetched 2026-10-26 (unofficial)
[S4] UE5 Tutorial | Simple SVO Pathfinder for 3D Pathfinding, https://www.youtube.com/watch?v=QmDbX_gZSLA, fetched 2026-10-27 (unofficial)
[S5] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
