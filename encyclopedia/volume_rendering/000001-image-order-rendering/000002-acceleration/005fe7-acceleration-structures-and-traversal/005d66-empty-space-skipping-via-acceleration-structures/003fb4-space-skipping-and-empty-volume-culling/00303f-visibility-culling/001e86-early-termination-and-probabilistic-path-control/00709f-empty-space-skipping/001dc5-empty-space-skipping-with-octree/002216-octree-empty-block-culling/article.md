# Octree Empty Block Culling

Also called: Octree Leaf Culling, Recursive Status Check

## Overview
Octree Empty Block Culling is a visibility culling technique that recursively traverses an octree to determine if a node contains any non-empty data blocks, returning a status of `EMPTY` or `NONEMPTY` to enable early ray termination or skipping [S1, unofficial]. It builds on [Empty Space Skipping with Octree](1610-empty-space-skipping-with-octree.md) by providing the specific recursive logic required to classify nodes based on the presence of underlying data blocks [S1, unofficial]. Unlike [Octree Traversal with Min/Max Pruning](1739-octree-traversal-with-min-max-pruning.md) which uses bounding volume overlaps, this method relies on explicit block existence checks [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A linear index to the current octree node (`idx`), the maximum tree depth (`max_depth`), a flat array of octree nodes (`nodes`), and a memory management unit or block loader interface (`mmu`) [S1, unofficial].
*   **Output:** A boolean indicating `NONEMPTY` (`true`) or `EMPTY` (`false`) [S1, unofficial].

**Data Structures:**
*   **Octree Node:** Contains a `depth` field indicating the current hierarchy level and a `status` field storing `EMPTY` or `NONEMPTY` [S1, unofficial].
*   **Block Loader (`mmu`):** Provides a `getBlock` method that returns a pointer to the data block associated with a leaf node, or `nullptr` if the block is not loaded or does not exist [S1, unofficial].

**Algorithm Steps:**
1. Initialize `nodes[idx].status` to `EMPTY` [S1, unofficial].
2. If `nodes[idx].depth == max_depth` (leaf node):
    *   Call `mmu.getBlock(idx)` [S1, unofficial].
    *   If the result is `nullptr`, set `nodes[idx].status = EMPTY` and return `false` [S1, unofficial].
    *   Otherwise, set `nodes[idx].status = NONEMPTY` and return `true` [S1, unofficial].
3. If internal node:
    *   Iterate `i` from 0 to 7 [S1, unofficial].
    *   Compute child linear index: `child_idx = (idx << 3) + i + 1` [S1, unofficial].
    *   Recursively call the status check on `child_idx` [S1, unofficial].
    *   If any child returns `true`, set `nodes[idx].status = NONEMPTY` [S1, unofficial].
    *   Return `nodes[idx].status == NONEMPTY` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def check_node_status(idx, max_depth, nodes, mmu):
    """
    Recursively determines if an octree node contains non-empty data blocks.
    
    Args:
        idx: Linear index of the current octree node.
        max_depth: The maximum depth of the octree.
        nodes: Flat array of octree node objects.
        mmu: Memory management unit with a getBlock method.
        
    Returns:
        bool: True if the node or any of its descendants contain non-empty blocks, False otherwise.
    """
    nodes[idx].status = "EMPTY"
    
    # Leaf check
    if nodes[idx].depth == max_depth:
        block = mmu.getBlock(idx)
        if block is None:
            nodes[idx].status = "EMPTY"
            return False
        nodes[idx].status = "NONEMPTY"
        return True
    
    # Internal node: recursively check all 8 children
    for i in range(8):
        child_idx = (idx << 3) + i + 1
        if check_node_status(child_idx, max_depth, nodes, mmu):
            nodes[idx].status = "NONEMPTY"
            return True
            
    return nodes[idx].status == "NONEMPTY"
```

**Parameters:**
*   `max_depth`: The depth at which nodes are considered leaves. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Depth Initialization:** The leaf check compares `node->depth` to the maximum depth parameter. If the tree depth is not set correctly during initialization, all nodes may be incorrectly treated as internal or leaf nodes, leading to incorrect culling results [S1, unofficial].

## Subtopics

### [View-Dependent Visibility Culling](11833-view-dependent-visibility-culling.md)
View-dependent visibility culling is an acceleration technique that discards voxels or nodes occluded by previously traversed geometry closer to the camera. It extends empty-space skipping by ordering traversal based on camera position, allowing the renderer to skip nodes hidden behind already-rendered surfaces. Unlike static culling, it leverages the multiresolution nature of hierarchical structures (like octrees or quadtrees) to adaptively decide extraction or rendering resolution based on visibility.

### Octree Range Culling
In this variant, shells' ranges over a node are computed once per pass for up to 4 tiles of 64 functions (each lane owns a function of every tile) instead of once per tile, only for shells reaching the node [S2, unofficial]. A dropped shell has every AO below $10^{-10}$ over the node; `fn_tail` ($2 \times 10^{-10}$ times the column sum of $|C|$) is added to every function's bound for them [S2, unofficial].

### Octree Node Collision Test
This method tests whether an octree node collides with obstacles to classify it as Empty or Mixed, avoiding explicit Blocked detection [S3, unofficial]. It runs a collision test using the node size; it returns Empty if no collision is detected, and Mixed if a collision is detected [S3, unofficial]. A pitfall of this single-pass size test is that it may miss internal full occlusion; this is resolved by recursive subdivision in `BuildNodeRecursive` [S3, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The time complexity is proportional to the number of nodes visited during the recursive traversal [S1, unofficial]. In the worst case, it visits all nodes; in the best case, it visits only the root and immediate children if they are marked empty [G]. Memory complexity is $O(D)$ auxiliary space per recursive call stack frame, where $D$ is the tree depth [G].

**Measured Performance:**
Not given in the sources.

**Scaling:**
Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Simple to implement; enables early ray termination [S1, unofficial].
*   **Limitations:** Requires explicit block existence checks; sensitive to correct depth initialization [S1, unofficial].

**When to use it and when not:**
Use it when the volume data is sparse and stored in an octree with explicit block pointers [S1, unofficial]. Not given in the sources.

## Background
The method originates from implementations of parallel volume rendering systems using octrees, such as those found in open-source repositories like `hopexn/VolumeRenderingParallel` [S1, unofficial]. It is used in GPU and CPU-based volume renderers that require efficient culling of empty regions to improve rendering performance [S1, unofficial].

## Sources
[S1] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
[S2] md_gto_iso_gpu: optimized GPU isosurface extraction (non-blocking jobs, flood, one measured path), https://github.com/scanberg/mdlib/pull/15, fetched 2026-10-28 (unofficial)
[S3] UE5 Tutorial | Simple SVO Pathfinder for 3D Pathfinding, https://www.youtube.com/watch?v=QmDbX_gZSLA, fetched 2026-10-27 (unofficial)
