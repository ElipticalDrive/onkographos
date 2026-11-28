# Residency Octree Update

Also called: Incremental Construction

## Overview
Residency Octree Update is a fully GPU-implemented parallel method for maintaining the spatial residency state of volumetric data bricks within a hierarchical octree structure [S1, unofficial]. It solves the problem of keeping the residency octree consistent with the dynamic brick cache during both data loading and cache eviction, enabling efficient culling of non-resident volume regions during rendering [S2]. It differs from static octree construction by being fully parallel and event-driven, updating nodes only when bricks are added or removed from the cache [S1, unofficial]. It differs from simple flat caches by maintaining a hierarchical bitmap representation of partial residency across multiple resolution levels and channels [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of added brick IDs, a list of removed brick IDs, and the current state of the brick cache (residency status of bricks) [S1, unofficial][S2].
*   **Output:** Updated culling metadata (min/max values) and partially mapped resolution bitmasks for all affected nodes in the residency octree, up to the root [S1, unofficial].

**Data Structures:**
*   **Residency Octree Node:**
    *   `culling_min`: Scalar minimum value of the volume data within this node's spatial extent [S1, unofficial].
    *   `culling_max`: Scalar maximum value of the volume data within this node's spatial extent [S1, unofficial].
    *   `residency_bitmap`: A bitmask where each bit represents a specific resolution level and/or channel [S2]. A bit is set if at least one brick at that resolution/channel is resident in the cache and overlaps the node's spatial extent [S2].
*   **Update-Candidate Buffer:** A dynamic buffer storing indices of nodes that require recomputation in the next level up [S1, unofficial].

**Parameters:**
*   **Brick Size:** Typically $32^3$ voxels [S1, unofficial].
*   **Leaf-Node Size:** The approximate spatial extent of leaf nodes in the residency octree [S1, unofficial]. Can be decoupled from brick size (e.g., $32^3, 16^3, 8^3$, or $8 \times 8 \times 4$) [S1, unofficial].

**Algorithm Steps:**
The update process is initiated by the application immediately after the frame's brick-cache update is completed [S1, unofficial]. It proceeds in a breadth-first manner from leaf nodes up to the root, using indirect compute passes to manage workgroup dispatch and enable early termination [S1, unofficial].

1.  **Process Removed Bricks:** For each brick ID in the "removed" list, determine the corresponding leaf nodes in the residency octree whose spatial extent overlaps the brick's extent [S2]. For each such leaf node, check if it is still partially resident in the resolution level of the removed brick [S1, unofficial]. This is done by verifying if any other brick at that resolution level overlapping the node is still resident in the cache [S2]. If the node is no longer partially resident in that resolution, update its `residency_bitmap` (clear the corresponding bit) [S1, unofficial]. Mark the parent node of the updated leaf node as an update candidate in the `update-candidate buffer` [S1, unofficial][S2].
2.  **Compute Culling Metadata for New Bricks:** For each scalar value in each newly added brick, determine the corresponding leaf node in the residency octree [S1, unofficial]. If the scalar value is lower than the node's current `culling_min` or greater than its `culling_max`, update the node's metadata [S1, unofficial]. Use atomic operations (`atomicMin`, `atomicMax`) to ensure concurrent updates from multiple threads are safe [S1, unofficial]. Mark the parent node of the updated leaf node as an update candidate in the `update-candidate buffer` [S1, unofficial].
3.  **Process Added Bricks:** For each brick ID in the "added" list, determine all leaf nodes in the residency octree whose spatial extent overlaps the brick's extent [S2]. Mark these leaf nodes as partially mapped in the brick's resolution level and channel by setting the corresponding bits in their `residency_bitmap` [S2]. If a node's `residency_bitmap` changed due to this operation, mark its parent node as an update candidate in the `update-candidate buffer` [S1, unofficial][S2].
4.  **Compile Update Candidates:** Process the `update-candidate buffer` to compile a unique list of all nodes marked as candidates for the next level up [S1, unofficial]. Clear the `update-candidate buffer` for subsequent iterations [S1, unofficial]. Use the count of candidates to control the number of workgroups dispatched for the next indirect compute pass [S1, unofficial]. If no candidates remain, terminate the update process [S1, unofficial].
5.  **Process Update Candidates:** For each candidate node, recompute its `culling_min`, `culling_max`, and `residency_bitmap` from its child nodes [S1, unofficial]. The `residency_bitmap` is updated via bitwise OR of the child node bitmasks: $B_{parent} = B_{child1} \lor B_{child2} \lor \dots \lor B_{child8}$, where $B$ is the bitmask [S2]. The `culling_min` and `culling_max` are recomputed as the minimum and maximum of the respective values from all child nodes [S1, unofficial]. If any of the node's values (`culling_min`, `culling_max`, or `residency_bitmap`) changed compared to the previous state, mark the node's parent as an update candidate in the `update-candidate buffer` [S1, unofficial].
6.  **Repeat:** Repeat Steps 4 and 5 until the root node is reached or no update candidates remain [S1, unofficial]. Early termination occurs if a node's bitmask remains unchanged by the OR operation during Step 5 [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def residency_octree_update(added_bricks, removed_bricks, brick_cache, octree_nodes):
    # Step 1: Process Removed Bricks
    for brick_id in removed_bricks:
        if brick_cache[brick_id].is_resident:
            continue
        leaf_nodes = get_leaf_nodes_overlapping_extent(brick_cache[brick_id].spatial_extent)
        for node_idx in leaf_nodes:
            node = octree_nodes[node_idx]
            res = brick_cache[brick_id].resolution
            if not check_partial_residency(node_idx, res, brick_cache):
                old_bitmap = node.residency_bitmap
                node.residency_bitmap = clear_bit(node.residency_bitmap, res)
                if old_bitmap != node.residency_bitmap:
                    add_to_candidates(node.parent_id)

    # Step 2: Compute Culling Metadata for New Bricks
    for brick_id in added_bricks:
        if not brick_cache[brick_id].is_resident:
            continue
        leaf_nodes = get_leaf_nodes_overlapping_extent(brick_cache[brick_id].spatial_extent)
        for node_idx in leaf_nodes:
            node = octree_nodes[node_idx]
            val_min = brick_cache[brick_id].min_value
            val_max = brick_cache[brick_id].max_value
            old_min = node.culling_min
            old_max = node.culling_max
            node.culling_min = min(node.culling_min, val_min)
            node.culling_max = max(node.culling_max, val_max)
            if node.culling_min != old_min or node.culling_max != old_max:
                add_to_candidates(node.parent_id)

    # Step 3: Process Added Bricks (Residency Bitmap)
    for brick_id in added_bricks:
        if not brick_cache[brick_id].is_resident:
            continue
        leaf_nodes = get_leaf_nodes_overlapping_extent(brick_cache[brick_id].spatial_extent)
        for node_idx in leaf_nodes:
            node = octree_nodes[node_idx]
            res = brick_cache[brick_id].resolution
            old_bitmap = node.residency_bitmap
            node.residency_bitmap = set_bit(node.residency_bitmap, res)
            if old_bitmap != node.residency_bitmap:
                add_to_candidates(node.parent_id)

    # Step 4 & 5: Iterative Upward Propagation
    candidates = list(update_candidates)
    while candidates:
        next_candidates = []
        for node_idx in candidates:
            node = octree_nodes[node_idx]
            new_min = float('inf')
            new_max = float('-inf')
            new_bitmap = 0
            for child_idx in node.child_ids:
                child = octree_nodes[child_idx]
                new_min = min(new_min, child.culling_min)
                new_max = max(new_max, child.culling_max)
                new_bitmap = new_bitmap | child.residency_bitmap

            changed = False
            if new_min != node.culling_min:
                node.culling_min = new_min
                changed = True
            if new_max != node.culling_max:
                node.culling_max = new_max
                changed = True
            if new_bitmap != node.residency_bitmap:
                node.residency_bitmap = new_bitmap
                changed = True

            if changed:
                add_to_candidates(node.parent_id, next_candidates)

        candidates = next_candidates
        # Step 4: Buffer cleared by reassigning candidates list
        # Dispatch controlled by len(candidates)

def set_bit(bitmap, bit_index):
    return bitmap | (1 << bit_index)

def clear_bit(bitmap, bit_index):
    return bitmap & ~(1 << bit_index)

def add_to_candidates(node_idx, target_list=None):
    if node_idx is None:
        return
    if target_list is None:
        target_list = update_candidates
    if node_idx not in target_list:
        target_list.append(node_idx)

def check_partial_residency(node_idx, resolution, brick_cache):
    # Verify if any other brick at the given resolution overlapping the node is resident [G]
    bricks = get_bricks_overlapping_extent(get_node_extent(node_idx))
    for brick in bricks:
        if brick.is_resident and brick.resolution == resolution:
            return True
    return False

def get_leaf_nodes_overlapping_extent(extent):
    # Traverse octree to find leaves intersecting extent [G]
    pass

def get_node_extent(node_idx):
    # Derived from octree structure [G]
    pass

def get_bricks_overlapping_extent(extent):
    # Spatial index lookup [G]
    pass
```

## Subtopics

### Decoupling Resolution Levels from Spatial Subdivision
An optimization technique where the spatial subdivision granularity (leaf-node size) of the residency octree is decoupled from the resolution levels of the bricked volume hierarchy [S1, unofficial]. This allows finer empty-space skipping than the bricking granularity [S1, unofficial]. It is beneficial for anisotropic volumes and can achieve higher frame rates [S1, unofficial]. Typical brick size is $32^3$ voxels, while leaf-node sizes can be $32^3, 16^3, 8^3$, or even $8 \times 8 \times 4$ [S1, unofficial].

### GPU-Resident Octree Editing
An approach where the octree is edited by a compute pass on the GPU, rather than on the CPU with deltas uploaded every frame [S3, unofficial]. This is required to satisfy the constraint of no CPU readback per frame in the hot loop [S3, unofficial]. Discrete events (player digs, meteor lands, explosion) may synchronise, but terrain streaming, LOD selection, or acceleration structure refits doing readback per frame remain forbidden [S3, unofficial]. An octree edited on the CPU and re-uploaded every frame violates the constraint of no CPU readback per frame in the hot loop [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** The update process is proportional to the number of changed bricks and the height of the octree [S1, unofficial]. The use of indirect compute passes and early termination (when bitmasks do not change) optimizes performance by avoiding unnecessary work [S1, unofficial][S2].
*   **Memory Complexity:** Requires storage for the octree nodes (culling metadata and bitmasks) and the update-candidate buffer [S1, unofficial]. The buffer size is bounded by the number of affected nodes per frame [S1, unofficial].
*   **Performance:** Fully GPU-implemented, allowing parallel construction and updates [S1, unofficial]. Decoupling leaf-node size from brick size can improve performance by enabling finer empty-space skipping [S1, unofficial].
*   **Strengths:** Efficient handling of dynamic data loading and eviction; parallel execution on GPU; early termination reduces overhead [S1, unofficial][S2].
*   **Limitations:** Requires atomic operations for culling metadata updates, which can introduce contention [S1, unofficial]. Performance depends on the spatial distribution of changes and the octree structure [S1, unofficial].

## Background
The method is implemented and evaluated within a scalable web-based volume rendering framework for large biomedical datasets [S1, unofficial]. It is part of the Residency Octree framework, which decouples cache residency from spatial subdivision [S1, unofficial]. The method was evaluated on datasets such as CyCIF Small and CyCIF Medium [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
[S3] research(render,world): LOD strategy for terraformable planet terrain, https://github.com/lobinuxsoft/kooch/issues/614, fetched 2026-10-26 (unofficial)
