# Octree Empty Block Culling

Also called: Octree Leaf Culling, Recursive Status Check

## Overview
Octree Empty Block Culling is a visibility culling technique that recursively traverses an octree to determine if a node contains any non-empty data blocks, returning a status of `EMPTY` or `NONEMPTY` to enable early ray termination or skipping [S1, unofficial]. It builds on [Empty Space Skipping with Octree](1610-empty-space-skipping-with-octree.md) by providing the specific recursive logic required to classify nodes based on the presence of underlying data blocks [S1, unofficial]. Unlike [Octree Traversal with Min/Max Pruning](1739-octree-traversal-with-min-max-pruning.md) which uses bounding volume overlaps, this method relies on explicit block existence checks [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A pointer to the current octree node (`node`), the maximum tree depth (`depth`), and a memory management unit or block loader interface (`mmu`) capable of retrieving data blocks [S1, unofficial].
*   **Output:** A boolean or status enum indicating `NONEMPTY` (true) or `EMPTY` (false) [S1, unofficial].

**Data Structures:**
*   **Octree Node:** Contains a `depth` field indicating the current level in the hierarchy [S1, unofficial].
*   **Block Loader (`mmu`):** Provides a `getBlock` method that returns a pointer to the data block associated with a leaf node, or `nullptr` if the block is not loaded or does not exist [S1, unofficial].

**Algorithm Steps:**
1.  Initialize the current node's status to `EMPTY` [S1, unofficial].
2.  Check if the current node is a leaf node by comparing `node->depth` to the global `depth` [S1, unofficial].
    *   If it is a leaf:
        *   Attempt to load the block using `mmu->getBlock` [S1, unofficial].
        *   If the returned block pointer is `nullptr`, set `node->status` to `EMPTY` and return `false` [S1, unofficial].
        *   Otherwise, set `node->status` to `NONEMPTY` and return `true` [S1, unofficial].
    *   If it is an internal node:
        *   Iterate through all 8 children of the node [S1, unofficial].
        *   Calculate the linear index of each child using the formula `(index << 3) + i + 1`, where `index` is the linear index of the current node and `i` is the child index (0-7) [S1, unofficial].
        *   Recursively call the status check function on each child [S1, unofficial].
        *   If any child returns `true` (indicating `NONEMPTY`), set `node->status` to `NONEMPTY` [S1, unofficial].
        *   Return `node->status == NONEMPTY` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def check_node_status(node, max_depth, mmu):
    """
    Recursively determines if an octree node contains non-empty data blocks.
    
    Args:
        node: The current octree node object.
        max_depth: The maximum depth of the octree.
        mmu: Memory management unit with a getBlock method.
        
    Returns:
        bool: True if the node or any of its descendants contain non-empty blocks, False otherwise.
    """
    # Initialize status to EMPTY
    node.status = "EMPTY"
    
    # Check if leaf node
    if node.depth == max_depth:
        # Attempt to load the block
        block = mmu.getBlock(node)
        
        # If block is nullptr, it is empty
        if block is None:
            node.status = "EMPTY"
            return False
        
        # Block exists, mark as non-empty
        node.status = "NONEMPTY"
        return True
    
    # Internal node: recursively check all 8 children
    # Linear indexing formula: (index << 3) + i + 1
    # Note: 'node.index' is assumed to be the linear index of the current node
    for i in range(8):
        child_index = (node.index << 3) + i + 1
        child_node = get_node_by_index(child_index) # Implementation detail: retrieve child node
        
        if check_node_status(child_node, max_depth, mmu):
            node.status = "NONEMPTY"
            return True
            
    return node.status == "NONEMPTY"
```

**Parameters:**
*   `max_depth`: The depth at which nodes are considered leaves. Typical values depend on the resolution of the volume data [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Depth Initialization:** The leaf check relies on comparing `node->depth` to a class member `depth`. If the tree depth is not set correctly during initialization, all nodes may be incorrectly treated as internal or leaf nodes, leading to incorrect culling results [S1, unofficial].

## Subtopics

### [View-Dependent Visibility Culling](11833-view-dependent-visibility-culling.md)
View-dependent visibility culling is an acceleration technique that discards voxels or nodes occluded by previously traversed geometry closer to the camera. It extends empty-space skipping by ordering traversal based on camera position, allowing the renderer to skip nodes hidden behind already-rendered surfaces. Unlike static culling, it leverages the multiresolution nature of hierarchical structures (like octrees or quadtrees) to adaptively decide extraction or rendering resolution based on visibility.

### Octree Range Culling
In this variant, shells' ranges over a node are computed once per pass for up to 4 tiles of 64 functions (each lane owns a function of every tile) instead of once per tile, only for shells reaching the node [S2, unofficial]. A dropped shell has every AO below $10^{-10}$ over the node; `fn_tail` ($2 \times 10^{-10}$ times the column sum of $|C|$) is added to every function's bound for them [S2, unofficial].

### Octree Node Collision Test
This method tests whether an octree node collides with obstacles to classify it as Empty or Mixed, avoiding explicit Blocked detection [S3, unofficial]. It runs a collision test using the node size; it returns Empty if no collision is detected, and Mixed if a collision is detected [S3, unofficial]. A pitfall of this single-pass size test is that it may miss internal full occlusion; this is resolved by recursive subdivision in `BuildNodeRecursive` [S3, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The time complexity is proportional to the number of nodes visited. In the worst case (fully occupied volume), it visits all nodes. In the best case (empty volume), it visits only the root and immediate children if they are marked empty. Memory complexity is $O(1)$ auxiliary space per recursive call stack frame, proportional to the tree depth [S1, unofficial].

**Measured Performance:**
Not given in the sources.

**Scaling:**
Performance scales with the sparsity of the volume data. Highly sparse volumes benefit significantly from early termination [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Simple to implement; effective for sparse volumes; enables early ray termination [S1, unofficial].
*   **Limitations:** Requires explicit block existence checks; sensitive to correct depth initialization [S1, unofficial].

**When to use it and when not:**
Use it when the volume data is sparse and stored in an octree with explicit block pointers. Do not use it when the volume is dense or when the octree structure does not support efficient block retrieval [S1, unofficial].

## Background
The method originates from implementations of parallel volume rendering systems using octrees, such as those found in open-source repositories like `hopexn/VolumeRenderingParallel` [S1, unofficial]. It is used in GPU and CPU-based volume renderers that require efficient culling of empty regions to improve rendering performance [S1, unofficial].

## Sources
[S1] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
[S2] md_gto_iso_gpu: optimized GPU isosurface extraction (non-blocking jobs, flood, one measured path), https://github.com/scanberg/mdlib/pull/15, fetched 2026-10-28 (unofficial)
[S3] UE5 Tutorial | Simple SVO Pathfinder for 3D Pathfinding, https://www.youtube.com/watch?v=QmDbX_gZSLA, fetched 2026-10-27 (unofficial)
