# Residency Octree Update

Also called: Incremental Construction

## Overview
Residency Octree Update is the incremental, fully GPU-based method for maintaining the multi-channel residency octree data structure during out-of-core direct volume rendering. It solves the problem of keeping the spatial subdivision tree consistent with the dynamic state of the brick cache (added and evicted bricks) without rebuilding the entire tree from scratch. What sets it apart from its siblings is its breadth-first, candidate-driven propagation of metadata updates from leaf nodes to the root, using indirect compute passes to minimize GPU work by terminating early when no changes occur [S1, unofficial].

## How it works
The method updates the residency octree to reflect changes in the brick cache. It handles metadata reception, brick addition, and LRU cache eviction by marking leaf nodes as partially mapped and propagating bitmap updates up the tree [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   List of added brick IDs and their spatial extents.
    *   List of removed (evicted) brick IDs and their spatial extents.
    *   Scalar values for newly added bricks (for culling metadata).
    *   Current state of the residency octree (bitmasks, min/max values).
*   **Outputs:**
    *   Updated residency octree nodes (bitmasks, culling metadata).
    *   Indication of completion (root reached or no candidates).

### Data Structures
*   **Residency Octree Node:**
    *   `bitmask`: A bitmask representing which resolution levels/channels are partially resident in this node's spatial extent.
    *   `min_value`: Minimum scalar value in the node's extent.
    *   `max_value`: Maximum scalar value in the node's extent.
    *   `children`: Pointers or indices to 8 child nodes (for non-leaf nodes).
*   **Update-Candidate Buffer:**
    *   A buffer storing indices of nodes that require recomputation in the next level up.
    *   Cleared after each iteration.

### Numbered Steps
1.  **Initialization:** Start timing. Application triggers updates after the frame's brick-cache update is completed [S1, unofficial].
2.  **Process Removed Bricks:**
    *   For each non-resident brick, determine its corresponding leaf nodes in the residency octree.
    *   For each leaf node, check if it is still partially resident in the removed brick's resolution level.
    *   If not partially resident, update the node's partially mapped resolution bitmap.
    *   Mark the parent node as an update candidate in the update-candidate buffer [S1, unofficial].
3.  **Compute Culling Metadata for New Bricks:**
    *   For each scalar value in each newly added brick, determine the corresponding leaf node.
    *   If the value is lower than the node's current minimum or greater than its maximum, update the node's culling metadata.
    *   Use atomic operations (`atomicMin`, `atomicMax`) for concurrent updates to min/max values.
    *   Mark the parent node as an update candidate in the update-candidate buffer [S1, unofficial].
4.  **Process Added Bricks:**
    *   For each added brick, determine all corresponding leaf nodes whose spatial extent overlaps the brick.
    *   Mark these leaf nodes as partially mapped in the brick's resolution level and channel.
    *   If the node's bitmap of partially mapped resolutions changes, mark the parent node as an update candidate in the update-candidate buffer [S1, unofficial].
5.  **Compile Candidates:**
    *   Process the update-candidate buffer to compile a list of all nodes marked as candidates for the next step.
    *   Clear the update-candidate buffer for subsequent iterations.
    *   Use the candidate count to control the number of workgroups dispatched for the next indirect compute pass.
    *   If no candidates remain, terminate the update process [S1, unofficial].
6.  **Process Update Candidates:**
    *   For each candidate node, recompute its culling metadata and bitmap of partially mapped resolutions from its child nodes.
    *   If at least one of the node's values (bitmap or min/max) changes, mark the node's parent as an update candidate in the update-candidate buffer.
    *   For non-leaf nodes, update residency via bitwise OR of child node bitmasks:
        $$B_{parent} = B_{child1} \lor B_{child2} \lor ... \lor B_{child8}$$
        where $B$ is the bitmask [S2].
7.  **Repeat:** Repeat Steps 5 and 6 until the root node is reached or no update candidate remains [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def residency_octree_update(added_bricks, removed_bricks, octree_root):
    """
    Incrementally update the residency octree based on brick cache changes.
    
    Args:
        added_bricks: List of (brick_id, spatial_extent, scalar_values)
        removed_bricks: List of (brick_id, spatial_extent)
        octree_root: Root node of the residency octree
    
    Returns:
        None (updates octree in place)
    """
    candidate_buffer = []
    
    # Step 2: Process Removed Bricks
    for brick_id, extent in removed_bricks:
        leaf_nodes = get_leaf_nodes_for_extent(extent)
        for leaf in leaf_nodes:
            if not is_partially_resident(leaf, brick_id.resolution):
                update_bitmask(leaf, brick_id.resolution, False)
                candidate_buffer.append(leaf.parent)
    
    # Step 3: Compute Culling Metadata for New Bricks
    for brick_id, extent, scalar_values in added_bricks:
        leaf_nodes = get_leaf_nodes_for_extent(extent)
        for leaf in leaf_nodes:
            for value in scalar_values:
                atomic_min(leaf.min_value, value)
                atomic_max(leaf.max_value, value)
            candidate_buffer.append(leaf.parent)
    
    # Step 4: Process Added Bricks
    for brick_id, extent, _ in added_bricks:
        leaf_nodes = get_leaf_nodes_for_extent(extent)
        for leaf in leaf_nodes:
            old_bitmask = leaf.bitmask
            update_bitmask(leaf, brick_id.resolution, True)
            if leaf.bitmask != old_bitmask:
                candidate_buffer.append(leaf.parent)
    
    # Step 5: Compile Candidates
    unique_candidates = list(set(candidate_buffer))
    candidate_buffer.clear()
    
    if not unique_candidates:
        return
    
    # Step 6 & 7: Process and Repeat
    current_candidates = unique_candidates
    while current_candidates:
        next_candidates = []
        
        for node in current_candidates:
            if node is None:
                continue
                
            old_bitmask = node.bitmask
            old_min = node.min_value
            old_max = node.max_value
            
            # Recompute from children
            if node.is_leaf():
                # Leaf nodes are updated directly in steps 2-4
                continue
            else:
                # Update bitmask via bitwise OR of children
                new_bitmask = 0
                for child in node.children:
                    new_bitmask |= child.bitmask
                node.bitmask = new_bitmask
                
                # Recompute min/max from children
                new_min = infinity
                new_max = -infinity
                for child in node.children:
                    new_min = min(new_min, child.min_value)
                    new_max = max(new_max, child.max_value)
                node.min_value = new_min
                node.max_value = new_max
            
            # Check if values changed
            if (node.bitmask != old_bitmask or 
                node.min_value != old_min or 
                node.max_value != old_max):
                next_candidates.append(node.parent)
        
        # Clear duplicates for next iteration
        current_candidates = list(set(next_candidates))
```

### Parameters
*   **Brick Size:** 32^3 voxels (typical, used in evaluations) [S1, unofficial].
*   **Leaf-Node Sizes:** 32^3, 16^3, 8^3, or 8x8x4 (decoupled from brick size) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Concurrent Updates:** Min/max values must be updated using atomic operations (`atomicMin`, `atomicMax`) because multiple threads may update the same node concurrently [S1, unofficial].
*   **Early Termination:** Upwards propagation terminates early if a node's bitmask remains unchanged by the OR operation [S2].
*   **Boundary Conditions:** Ensure leaf nodes are correctly identified for all brick extents, including those at the volume boundaries.
*   **Precision:** Floating-point comparisons for min/max should handle NaNs or infinities appropriately if present in the data.

## Subtopics

### [Residency-Octree Updates](5552-residency-octree-updates.md)
This method updates a residency octree to reflect changes in a brick cache during volume rendering. It handles metadata reception, brick addition, and LRU cache eviction by marking leaf nodes as partially mapped and propagating bitmap updates up the tree.

### Decoupling Resolution Levels from Spatial Subdivision
An optimization technique where the spatial subdivision granularity (leaf-node size) of the residency octree is decoupled from the resolution levels of the bricked volume hierarchy, allowing finer empty-space skipping than the bricking granularity [S1, unofficial].
*   **Definition:** The residency octree's leaf-node size is independent of the brick size.
*   **Steps:**
    1.  Use a brick size of 32^3 voxels for all methods [S1, unofficial].
    2.  Use different approximate leaf-node sizes for the residency octree: 32^3, 16^3, 8^3 [S1, unofficial].
    3.  Evaluate the method with an approximate leaf-node size of 8x8x4 to demonstrate arbitrary spatial subdivision [S1, unofficial].
*   **Parameters:** Leaf-node sizes tested: 32^3, 16^3, 8^3, and 8x8x4 [S1, unofficial].
*   **Comparison:** Allows efficient empty-space skipping with finer granularity than bricking, potentially achieving higher frame rates. Arbitrary spatial subdivision can be beneficial for anisotropic volumes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Proportional to the number of changed bricks and the height of the octree. Early termination reduces work when changes are localized.
*   **Memory Complexity:** O(N) for the octree nodes, where N is the number of nodes. The candidate buffer size is proportional to the number of changed nodes per level.
*   **Measured Performance:** Higher frame rates achieved with decoupled leaf-node sizes (e.g., 8x8x4) compared to coupled sizes, due to finer empty-space skipping [S1, unofficial].
*   **Strengths:** Fully GPU-implemented, parallel, incremental (avoids full rebuild), early termination.
*   **Limitations:** Requires atomic operations for concurrent updates, which may cause contention. Performance depends on the spatial distribution of changes.
*   **When to Use:** Out-of-core DVR with dynamic brick caches.
*   **When Not to Use:** Static volumes with no cache updates, or when CPU-side tree maintenance is preferred.

## Background
The Residency Octree Update method was introduced in the context of scalable web-based volume rendering of large biomedical datasets [S1, unofficial]. It builds on the concept of the Residency Octree, which decouples cache residency from spatial subdivision [Parent]. The method was evaluated on CyCIF Small and Medium datasets, demonstrating scalability with the number of visible channels [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
