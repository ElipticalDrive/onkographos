# Residency-Octree Updates


## Overview
Residency-Octree Updates is a method for maintaining the spatial residency state of volumetric data bricks within a hierarchical octree structure, ensuring that the tree accurately reflects which data is currently resident in the GPU cache. It solves the problem of keeping the residency octree consistent with the dynamic brick cache during both data loading and cache eviction, enabling efficient culling of non-resident volume regions during rendering. It differs from static octree construction by being fully parallel and event-driven, updating nodes only when bricks are added or removed from the cache, and from simple flat caches by maintaining a hierarchical bitmap representation of partial residency across multiple resolution levels and channels.

## How it works
The method maintains a multi-channel residency octree where each node stores metadata and residency bitmaps. The updates are triggered by two events: receiving new metadata for a volume region and receiving a new brick (or evicting one).

### Data Structures
*   **Residency Octree Node**:
    *   `metadata`: Channel-specific status (e.g., `INVALID`, `VALID`).
    *   `residency_bitmap`: A bitfield where each bit corresponds to a resolution level. A bit is set if the node is partially mapped (i.e., at least one brick at that resolution level is resident in the cache).
    *   `children`: Pointers to child nodes (for non-leaves).
*   **Brick Cache**:
    *   Fixed-size LRU cache storing resident bricks.
    *   Each brick has a spatial extent and a resolution level.

### Inputs and Outputs
*   **Inputs**:
    *   New metadata for a volume region (channel, resolution, spatial bounds).
    *   New brick data (spatial bounds, resolution level, channel).
    *   Eviction request (brick to remove).
*   **Outputs**:
    *   Updated residency octree with correct `metadata` and `residency_bitmap` values.

### Algorithm Steps

#### 1. Metadata Update
When new metadata for a volume region is received:
1.  **Check Node Existence**: Determine if a node corresponding to the region's spatial extent exists in the octree.
2.  **If New Node**:
    *   Create the node and add it to the octree.
    *   Initialize `metadata` as `INVALID` for all channels.
    *   Set `metadata` for the received channel to `VALID` (or appropriate state).
    *   Initialize `residency_bitmap` by checking each resolution level to see if the node is partially mapped. This involves checking if any brick at that resolution level overlapping the node's extent is currently in the cache.
3.  **If Existing Node**:
    *   Update the `metadata` for the specific channel.
    *   If this is a multi-channel volume and the node already holds metadata for another channel, re-initialize the `residency_bitmap` for the new channel's resolution levels by checking cache residency.

#### 2. Brick Addition
When a new brick is received and stored in the cache:
1.  **Select Slot**: Choose an available slot in the brick cache. If none is available, trigger LRU Eviction (see below).
2.  **Identify Leaf Nodes**: Find all residency-octree leaf nodes whose spatial extent overlaps with the brick's spatial extent.
3.  **Mark Leaves**: For each identified leaf node, mark it as partially mapped in the brick's resolution level and channel. This is done by setting the corresponding bit in the node's `residency_bitmap`.
4.  **Propagate Upwards**:
    *   Traverse up the tree from the affected leaf nodes to the root.
    *   For each non-leaf node, update its `residency_bitmap` by performing a bit-wise OR of its children's bitmaps.
    *   **Termination Condition**: If a node's `residency_bitmap` remains unchanged after the OR operation, stop propagating updates up the tree from that branch.

#### 3. LRU Cache Eviction
When a brick is evicted from the cache:
1.  **Identify Affected Leaves**: Find all leaf nodes whose spatial extent overlaps with the evicted brick's spatial extent.
2.  **Check Residency**: For each affected leaf node and for the resolution level of the evicted brick:
    *   Check if any *other* brick at that resolution level overlapping the node's extent is still resident in the cache.
    *   If no other brick is resident, the node is no longer partially mapped in that resolution level.
    *   Set the corresponding bit in the node's `residency_bitmap` to zero.
3.  **Propagate Upwards**:
    *   Traverse up the tree from the affected leaf nodes to the root.
    *   For each non-leaf node, update its `residency_bitmap` by setting it to the bit-wise OR of its children's bitmaps.
    *   Terminate propagation if the bitmap is unchanged.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_residency_on_metadata(node_id, channel, resolution_levels):
    """
    Called when new metadata for a volume region is received.
    node_id: Identifier for the octree node.
    channel: The channel index.
    resolution_levels: List of resolution levels to check for partial mapping.
    """
    node = get_or_create_node(node_id)
    
    # Initialize metadata
    if node.is_new:
        node.metadata = {ch: INVALID for ch in all_channels}
        node.metadata[channel] = VALID
    else:
        node.metadata[channel] = VALID
        
    # Initialize residency bitmap for the channel's resolution levels
    # This checks if any brick at these levels is currently in cache
    for res_level in resolution_levels:
        if is_partially_mapped(node, res_level, channel):
            set_bit(node.residency_bitmap, res_level)
        else:
            clear_bit(node.residency_bitmap, res_level)

def update_residency_on_brick_add(brick):
    """
    Called when a new brick is stored in the cache.
    brick: Object with spatial_extent, resolution_level, channel.
    """
    # 1. Select slot (handled by cache manager, assume slot is now occupied)
    
    # 2. Find overlapping leaf nodes
    leaf_nodes = find_leaf_nodes_overlapping(brick.spatial_extent)
    
    # 3. Mark leaves
    for node in leaf_nodes:
        set_bit(node.residency_bitmap, brick.resolution_level)
        
    # 4. Propagate up
    for node in leaf_nodes:
        current = node
        while current is not None:
            parent = current.parent
            if parent is None:
                break
            
            # Bit-wise OR of children
            new_bitmap = 0
            for child in parent.children:
                new_bitmap |= child.residency_bitmap
            
            if parent.residency_bitmap == new_bitmap:
                # Unchanged, terminate propagation for this branch
                break
            else:
                parent.residency_bitmap = new_bitmap
                current = parent

def update_residency_on_eviction(evicted_brick):
    """
    Called when a brick is evicted from the cache.
    evicted_brick: Object with spatial_extent, resolution_level.
    """
    # 1. Find overlapping leaf nodes
    leaf_nodes = find_leaf_nodes_overlapping(evicted_brick.spatial_extent)
    
    # 2. Check and update leaves
    for node in leaf_nodes:
        # Check if any other brick at this resolution level is resident
        is_still_mapped = is_any_brick_resident(node, evicted_brick.resolution_level)
        
        if not is_still_mapped:
            clear_bit(node.residency_bitmap, evicted_brick.resolution_level)
            
            # 3. Propagate up
            current = node
            while current is not None:
                parent = current.parent
                if parent is None:
                    break
                
                new_bitmap = 0
                for child in parent.children:
                    new_bitmap |= child.residency_bitmap
                
                if parent.residency_bitmap == new_bitmap:
                    break
                else:
                    parent.residency_bitmap = new_bitmap
                    current = parent

def is_partially_mapped(node, resolution_level, channel):
    """
    Check if any brick at the given resolution level and channel
    overlapping the node's extent is in the cache.
    """
    # Implementation depends on cache lookup efficiency
    # [G] In practice, this might use a spatial index or hash map for O(1) or O(log N) lookup
    return cache.contains_brick_overlapping(node.spatial_extent, resolution_level, channel)

def is_any_brick_resident(node, resolution_level):
    """
    Check if any brick at the given resolution level overlapping
    the node's extent is in the cache.
    """
    return cache.contains_brick_overlapping(node.spatial_extent, resolution_level)
```

### Parameters and Edge Cases
*   **Bitmap Size**: The number of bits in `residency_bitmap` corresponds to the number of resolution levels. [S1, unofficial]
*   **Multi-channel Volumes**: Residency is tracked per channel. A node may be partially mapped for one channel but not another. [S1, unofficial]
*   **Precision**: Bit-wise operations are exact. No floating-point precision issues. [S1, unofficial]
*   **Boundaries**: Spatial overlap checks must be robust to handle bricks that exactly match node boundaries or are slightly larger/smaller due to discretization. [S1, unofficial]
*   **Degenerate Input**: If a brick has no overlap with any node, no updates occur. [S1, unofficial]

## Subtopics

### Dynamic Octree Generation
A technique used in Mega-NeRF to manage octree size by bounding it to available GPU memory and pruning nodes that do not contribute to rendering. It differs from static residency updates by actively growing and shrinking the tree based on rendering visibility rather than just cache residency. [S2]
*   **Definition**: Track the number of pixels visible from each node during rendering. Subdivide the top $k$ nodes with the most visible pixels. Prune entries that have not recently contributed to reclaim space when capacity is hit. [S2]
*   **Parameters**:
    *   Maximum tree size: 20M elements. [S2]
    *   Top $k$ nodes to subdivide: 16,384. [S2]
    *   Maximum tree depth observed: roughly 12. [S2]
*   **When to choose**: Use when memory is constrained and the scene has varying levels of detail that can be dynamically adjusted based on camera view, rather than when managing a fixed set of pre-computed volume bricks. [S2]

## Operating characteristics
*   **Time Complexity**:
    *   Metadata Update: $O(1)$ for node creation/update, $O(R)$ for bitmap initialization where $R$ is the number of resolution levels. [S1, unofficial]
    *   Brick Addition/Eviction: $O(L \cdot D)$ where $L$ is the number of leaf nodes overlapping the brick and $D$ is the tree depth. In practice, propagation terminates early if bitmaps are unchanged. [S1, unofficial]
*   **Memory Complexity**: $O(N)$ where $N$ is the number of nodes in the octree. Each node stores a small bitmap. [S1, unofficial]
*   **Strengths**:
    *   Efficiently maintains hierarchical residency information. [S1, unofficial]
    *   Early termination in propagation reduces update cost. [S1, unofficial]
    *   Supports multi-channel volumes. [S1, unofficial]
*   **Limitations**:
    *   Requires frequent cache lookups to check partial mapping. [S1, unofficial]
    *   Bitmap updates are not parallelized in the description, though the parent article mentions fully GPU-implemented parallel construction. [S1, unofficial]
*   **When to use**: When rendering large volumetric datasets with dynamic caching and hierarchical culling is required. [S1, unofficial]
*   **When not to use**: For static volumes where the entire dataset is resident, or for scenes where dynamic LOD based on visibility is more important than cache residency. [S2]

## Background
The Residency-Octree Update method was introduced in the context of scalable web-based volume rendering of large biomedicine datasets. It was designed to efficiently manage the residency of volume bricks in a GPU cache while maintaining a hierarchical octree structure for culling. The method builds on earlier work in spatial culling and LRU cache eviction. [S1, unofficial]

Dynamic Octree Generation was introduced in the context of Mega-NeRF for scalable construction of large-scale NeRFs. It addresses the challenge of managing octree size in memory-constrained environments by dynamically adjusting the tree based on rendering visibility. [S2]

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly-Throughs, https://doi.org/10.1109/cvpr52688.2022.01258, fetched 2026-10-26
