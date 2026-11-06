# Residency-Octree Updates


## Overview
This method manages the parallel construction and maintenance of a multi-channel residency octree on the GPU, ensuring that spatial culling information accurately reflects the current state of the brick cache. It solves the problem of keeping hierarchical occupancy data consistent with dynamic cache additions and evictions without CPU intervention. Unlike static octree construction, this approach updates nodes only when bricks are added or removed, using bitmap propagation to minimize redundant writes.

## How it works
The method maintains a residency octree where each node stores metadata and residency bitmaps. The core logic handles two events: receiving new metadata/bricks and evicting bricks due to cache pressure.

### Inputs and Outputs
*   **Inputs**: New volume metadata, new brick data, brick cache state, LRU access history.
*   **Outputs**: Updated residency octree nodes (metadata and bitmaps), updated brick cache.

### Data Structures
*   **Residency Octree Node**:
    *   `metadata`: Channel-specific status (e.g., `VALID`, `INVALID`).
    *   `residency_bitmap`: A bitfield where each bit corresponds to a resolution level. A bit is set if the node is partially mapped at that level.
    *   `children`: Pointers to child nodes (for non-leaves).
*   **Brick Cache**:
    *   Fixed-size array of slots.
    *   Each slot holds a brick identifier, its spatial extent, resolution level, and channel.
    *   LRU tracking for eviction.

### Numbered Steps

#### 1. Metadata Reception
When new metadata for a volume region is received:
1.  Check if a node exists for the region's spatial extent in the octree.
2.  **If new**:
    *   Create the node and add it to the octree.
    *   Initialize `metadata` as `INVALID` for all channels except the received one.
    *   Initialize `residency_bitmap` by checking if the node is partially mapped in each resolution level. For the received channel, mark the relevant bits based on current cache state. For other channels, initialize residency info similarly if it is a multi-channel volume and the node already holds metadata for another channel [S1, unofficial].
3.  **If exists**:
    *   Update the channel-specific `metadata`.
    *   Initialize residency info for the new channel if the node already holds metadata for another channel (multi-channel case) [S1, unofficial].

#### 2. Brick Addition
When a new brick is received:
1.  Select an available slot in the brick cache. If none is available, trigger LRU Eviction (see below).
2.  Determine all residency-octree leaf nodes whose spatial extent overlaps the brick's spatial extent.
3.  Mark these leaf nodes as partially mapped in the brick's resolution level and channel. This involves setting the corresponding bit in their `residency_bitmap`.
4.  Propagate the update up the tree to the root:
    *   For each non-leaf node on the path, update its `residency_bitmap` by computing the bit-wise OR of its children's bitmaps.
    *   Terminate upward propagation as soon as a node's bitmap remains unchanged by this operation [S1, unofficial].

#### 3. LRU Cache Eviction
If no slot is available for a new brick:
1.  Evict the least recently accessed brick from the cache.
2.  Identify all leaf nodes whose spatial extent overlaps the evicted brick's spatial extent.
3.  For each such leaf node:
    *   Check if any other brick in the same resolution level is resident in the cache.
    *   If no other brick is resident, the node is no longer partially mapped in that resolution level. Set the corresponding bit in the node's `residency_bitmap` to zero.
4.  Update non-leaf nodes by setting their bitmaps to the bit-wise OR of their child nodes' bitmaps, propagating up to the root [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_residency_octree(new_brick, cache, octree_root):
    """
    Handles addition of a new brick and cache eviction if necessary.
    """
    # Step 1: Find or evict slot
    slot = cache.find_available_slot()
    if slot is None:
        evicted_brick = cache.evict_lru()
        handle_eviction(evicted_brick, cache, octree_root)
        slot = cache.find_available_slot() # Should now be available

    # Step 2: Store brick
    cache.store(slot, new_brick)

    # Step 3: Update leaf nodes
    leaf_nodes = octree_root.find_leaves_overlapping(new_brick.spatial_extent)
    for node in leaf_nodes:
        node.residency_bitmap.set_bit(new_brick.resolution_level, new_brick.channel)

    # Step 4: Propagate up
    propagate_update(leaf_nodes, octree_root)

def handle_eviction(evicted_brick, cache, octree_root):
    """
    Handles the removal of a brick due to LRU eviction.
    """
    leaf_nodes = octree_root.find_leaves_overlapping(evicted_brick.spatial_extent)
    for node in leaf_nodes:
        # Check if any other brick in the same resolution level is resident
        is_still_mapped = False
        for brick in cache.get_bricks_at_resolution(evicted_brick.resolution_level):
            if brick.overlaps(node.spatial_extent):
                is_still_mapped = True
                break
        
        if not is_still_mapped:
            node.residency_bitmap.clear_bit(evicted_brick.resolution_level, evicted_brick.channel)

    # Propagate up from affected leaves
    # Note: In a parallel implementation, this might be done via atomic operations or reduction
    propagate_update(leaf_nodes, octree_root)

def propagate_update(affected_leaves, root):
    """
    Propagates bitmap updates up the tree.
    Terminates early if a node's bitmap does not change.
    """
    # Collect unique paths to root from affected leaves
    paths = collect_paths_to_root(affected_leaves)
    
    for path in paths:
        for node in path:
            if node.is_leaf:
                continue
            
            new_bitmap = 0
            for child in node.children:
                new_bitmap |= child.residency_bitmap
            
            if new_bitmap == node.residency_bitmap:
                # No change, terminate this path
                break
            else:
                node.residency_bitmap = new_bitmap
```

### Parameters and Edge Cases
*   **Bitmap Size**: The number of bits in `residency_bitmap` equals the number of resolution levels.
*   **Multi-channel Volumes**: Residency info is initialized per channel. If a node exists for one channel, adding metadata for another channel requires re-evaluating residency for the new channel [S1, unofficial].
*   **Precision**: Bit-wise operations are exact. No floating-point precision issues.
*   **Boundaries**: Leaf nodes must correctly handle spatial overlaps. If a brick is exactly on a boundary, it should be included in the overlap check.
*   **Degenerate Input**: Empty cache or empty octree should be handled gracefully (no-op).

## Operating characteristics
*   **Time Complexity**: $O(N \cdot H)$ where $N$ is the number of leaf nodes overlapping the brick and $H$ is the height of the octree. Early termination reduces average case.
*   **Memory Complexity**: $O(M)$ where $M$ is the number of nodes in the octree. Each node stores a small bitmap.
*   **Strengths**: Fully GPU-implemented, parallel construction, minimal updates (only when bricks change), efficient bit-wise operations.
*   **Limitations**: Requires careful synchronization in parallel updates. LRU eviction can cause cascading updates if many bricks are evicted simultaneously.
*   **When to use**: Dynamic volume rendering with streaming data, where cache state changes frequently.
*   **When not to use**: Static volumes with no cache updates, or when CPU-side management is preferred.

## Background
This method is described in the context of scalable web-based volume rendering of large biomedical datasets. It builds on earlier work using LRU cache eviction [SCRL20] (cited in [S1, unofficial]). The approach is designed for GPU implementation to avoid CPU-GPU synchronization bottlenecks.

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
