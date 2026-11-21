# Residency Octree

Also called: Residency-Octree Hierarchy, hybrid data structure for out-of-core volume rendering, multi-resolution page tables combined with a special octree data structure

## Overview
The Residency Octree is a hybrid data structure for out-of-core direct volume rendering (DVR) of multi-volume datasets that decouples the cache residency of multi-resolution data from the resolution-independent spatial subdivision determined by an octree [S1, unofficial][S2]. Unlike standard octrees where each node represents exactly one brick (one-to-one node-to-brick correspondence), each residency octree node represents a geometric spatial region and references all resolution levels in a bricked volume hierarchy [S1, unofficial][S2]. This decoupling enables efficient mixing of channel resolutions, cache-miss compensation by substituting missing bricks with available resolutions, and fine-grained empty-space skipping independent of the underlying bricking granularity [S1, unofficial][S2]. It combines the flexibility of page-table hierarchies with the traversal efficiency of octrees [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A multi-channel bricked volume hierarchy with $n$ total channels, a brick cache, and a user-defined approximate leaf-node size in voxels [S1, unofficial][S2].
*   **Output:** A fully pointerless octree structure stored in a GPU buffer, enabling ray-guided traversal for rendering $m \le n$ channels [S1, unofficial][S2].

**Data Structures:**
The residency octree is implemented as a full, pointerless octree where all nodes are pre-allocated with fixed locations in a storage buffer to remove indirection [S1, unofficial].

*   **Node Layout:** Each node consists of $m$ unsigned 32-bit integers, where $m$ is the number of simultaneously visualized channels [S1, unofficial].
    *   **Per Channel (32 bits):**
        *   **16 bits:** Bitmap tracking which resolution levels the node is at least *partially mapped* in [S1, unofficial].
        *   **8 bits:** Minimum scalar value ($min$) in the spatial region [S1, unofficial].
        *   **8 bits:** Maximum scalar value ($max$) in the spatial region [S1, unofficial].
    *   **Multi-Channel Extension:** For channels where information is not yet initialized, the node stores a special `INVALID` value [S1, unofficial]. If `INVALID`, the 32-bit slot stores a pointer to the next lower subdivision level node with valid information for that channel, or `UNKNOWN` if none exists. This pointer replaces the metadata to reduce memory usage [S1, unofficial].
    *   **Memory Scaling:** Memory usage scales linearly with the number of simultaneously visualized channels $m$ [S1, unofficial].

**Definitions of Mapping States:**
For a given octree node and resolution level:
1.  **Fully Mapped:** Cache-resident bricks of that level fully cover the node's spatial extent [S1, unofficial][S2].
2.  **Partially Mapped:** At least one brick of that level spatially overlaps with the node's extent and is resident in the cache [S1, unofficial][S2].
3.  **Not Mapped:** No resident bricks overlap the node [S1, unofficial][S2].
*   If a node is fully mapped, all its children are also fully mapped in that level [S1, unofficial][S2].
*   If a node is partially mapped, its parent is also partially mapped [S1, unofficial][S2].
*   Empty or homogeneous nodes are implicitly considered fully mapped in all resolution levels [S1, unofficial][S2].

**Parameters:**
*   **Leaf-node size:** Defined indirectly by the user at initialization as an approximate leaf-node size in voxels [S1, unofficial].
*   **Brick size:** Fixed and identical across all resolution levels in the bricked volume hierarchy (e.g., $32^3$ voxels) [S2].
*   **Minimum voxels:** A parameter specifying the minimum number of voxels to consider when computing metadata to avoid inaccuracies from coarse resolutions [S1, unofficial].
*   **Channel importance:** User-defined weights to guide resolution range selection [S2].

**Algorithm Steps:**

1.  **Initialization:**
    *   Determine the number of subdivision levels by iteratively subdividing the volume until the node size in each dimension is $\le$ the user-defined approximate leaf-node size in voxels [S1, unofficial].
    *   Pre-allocate the pointerless octree buffer [S1, unofficial].
    *   Initialize the multi-channel page table hierarchy and brick cache [S1, unofficial][S2].

2.  **Incremental Construction & Update (Server/Client Sync):**
    *   **Metadata Update:** When new metadata for a volume region is received:
        *   Create the node if it doesn't exist, or update it if it does [S2].
        *   Initialize metadata as `INVALID` for all channels except the one for which metadata was received [S2].
        *   For the received channel, check if the node is partially mapped in each resolution level and update the bitmap accordingly [S2].
        *   Propagate metadata up the tree: Parent $min = \min(child_1.min, \dots, child_8.min)$; Parent $max = \max(child_1.max, \dots, child_8.max)$ [S1, unofficial].
    *   **Brick Residency Update:** When a new brick is received:
        *   Select an available slot in the brick cache [S2].
        *   Identify all residency octree leaf nodes whose spatial extent overlaps the brick's spatial extent [S2].
        *   Mark these leaf nodes as *partially mapped* in the brick's resolution level and channel [S2].
        *   Propagate this update up the tree to the root node [S2].
    *   **Metadata Computation (Fallback):** If the server does not support metadata queries, the client must fetch bricks at a resolution sufficient to meet the *minimum number of voxels* requirement to compute accurate $min/max$ values. These bricks are uploaded to the GPU and residency info is updated [S1, unofficial].

3.  **Ray Traversal:**
    *   Start traversal at the third level (or a user-defined level) instead of the root, and resume from the parent node of the last visited node rather than restarting per sample [S2].
    *   For each node visited:
        *   Check transfer-function independent metadata ($min, max$) [S1, unofficial][S2]. If the isosurface threshold $T$ lies outside $[min, max]$, prune the subtree [G].
        *   Check the residency bitmap. If the node is empty or homogeneous, treat it as fully mapped and skip loading bricks [S1, unofficial].
        *   If data is missing for a node, generate a ray-guided request to the server [S2].
        *   If a brick is missing but a lower resolution is available, substitute it (cache-miss compensation) [S1, unofficial][S2].
    *   For multi-channel rendering: Evaluate each visible channel at every sample along the ray, even if channels have different resolutions, using accumulation-level intermixing [S1, unofficial][S2].

4.  **Channel Replacement:**
    *   When a channel is replaced by another, all nodes storing valid data for the old channel are marked `INVALID` for the new channel. The pointer mechanism is used to find valid data in lower subdivision levels if available [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class OctreeNode:
    def __init__(self):
        self.min_val: int = 0
        self.max_val: int = 0
        self.partial_map_bitmap: int = 0  # 16 bits for resolution levels
        self.channel_data: list = []      # List of m channel states

def traverse_ray(ray_origin, ray_direction, node_index, last_visited_parent):
    current_node = get_node(node_index)
    
    # 1. Empty-space skipping
    if ray_is_outside_threshold(current_node.min_val, current_node.max_val):
        return  # Prune subtree
    
    # 2. Check residency
    if current_node.is_empty_or_homogeneous():
        return
    
    # 3. Check if we have data for the current resolution/channel
    if not has_resident_brick(current_node, current_resolution):
        lower_res_brick = find_available_lower_resolution_brick(current_node)
        if lower_res_brick:
            sample_with_brick(ray, lower_res_brick)
            return
        else:
            request_brick_from_server(current_node.spatial_extent, current_resolution)
            return

    # 4. If leaf or further subdivision needed
    if is_leaf(current_node):
        sample_with_brick(ray, get_resident_brick(current_node, current_resolution))
    else:
        for child_index in get_children_indices(current_node):
            traverse_ray(ray_origin, ray_direction, child_index, current_node)

def update_residency_on_brick_load(brick, resolution_level, channel):
    overlapping_leaves = find_overlapping_leaf_nodes(brick.spatial_extent)
    for leaf in overlapping_leaves:
        leaf.partial_map_bitmap |= (1 << resolution_level)
        propagate_partial_map_up(leaf)

def propagate_partial_map_up(node):
    if node is root:
        return
    parent = get_parent(node)
    if node.is_partially_mapped():
        parent.partial_map_bitmap |= node.partial_map_bitmap
    propagate_partial_map_up(parent)
```

## Subtopics

### [Octree-based memory virtualization](5140-octree-based-memory-virtualization.md)
Octree-based memory virtualization is an out-of-core rendering technique for multi-channel volume datasets that utilizes operating system memory mapping to load octree nodes and bricks on demand. It distinguishes itself from the Residency Octree by enforcing full subtree residency in the cache for high-resolution rendering, whereas the Residency Octree allows partial subtree residency. Unlike Virtual Memory Volume Rendering, which uses a multi-resolution page table hierarchy to avoid pre-computed 3D hierarchies, this method relies on an explicit octree structure.

### [Residency Octree Update](5181-residency-octree-update.md)
Residency Octree Update is a fully GPU-implemented parallel method for maintaining the spatial residency state of volumetric data bricks within a hierarchical octree structure. It solves the problem of keeping the residency octree consistent with the dynamic brick cache during both data loading and cache eviction, enabling efficient culling of non-resident volume regions during rendering. It differs from static octree construction by being fully parallel and event-driven, updating nodes only when bricks are added or removed from the cache. It differs from simple flat caches by maintaining a hierarchical bitmap representation of partial residency across multiple resolution levels and channels.

### [Mixed-Resolution Multi-Volume Renderer](5215-mixed-resolution-multi-volume-renderer.md)
A ray-guided renderer that accesses metadata and cached volume data through the Residency Octree to enable efficient mixing of resolutions across different channels in multi-volume datasets. It is implemented as a WebGPU compute shader that evaluates each visible channel at every sample along a ray, even when channels have different resolutions, to reduce shader divergence.

### Residency Octree with Channel-Agnostic Brick Cache
Uses accumulation-level intermixing to combine different channels [S1, unofficial]. The residency octree is backed by a channel-agnostic brick cache where each cache entry only contains data belonging to a single channel. This allows the number of channels to not be limited by the texture format used by the brick cache implementation, unlike Brix et al.'s approach [S1, unofficial]. Residency octrees support changes to the selection of visible channels at run-time by replacing only the parts of their nodes related to the affected channels [S1, unofficial].

### Residency-Octree Node Classification
Classification of octree nodes based on cache residency of bricks covering their spatial extent at a given resolution level [S1, unofficial]. A node is *fully mapped* if cache-resident bricks of that level fully cover the node's spatial extent. A node is *partially mapped* if at least one brick of that level overlaps with the node's spatial extent and is resident in the cache. Fully mapped implies partially mapped, but not vice versa. If a node is fully mapped in a resolution level, all child nodes are also fully mapped in that level. If a node is partially mapped in a resolution level, its parent node is also partially mapped [S1, unofficial].

### Multi-Channel Residency Octree
An extension where all channels share the same octree structure, but each node stores culling and residency information for up to $m \le n$ channels. This allows using the same spatial subdivision for empty-space skipping while rendering each channel in a different resolution [S1, unofficial]. When a channel is replaced by another one, all nodes storing valid data for the old channel are marked as `INVALID` for the new channel. If invalid, the node stores a pointer to the node in the next lower subdivision level that has valid information for that channel, or a special `UNKNOWN` value if no such node exists. The pointer replaces the metadata for that channel to reduce memory [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Ray traversal is performed for spatial regions corresponding to octree nodes instead of bricks, and this traversal is independent of the number of channels [S2].
*   **Memory Complexity:** Memory usage scales linearly with the number of simultaneously visualized channels $m$ [S1, unofficial]. This can be mitigated by combining channels via dimensionality reduction, interleaving channels in four-component textures, or using compressed texture formats [S2]. The bit-width of the brick ID data type (e.g., 32-bit or 64-bit) constrains the maximum volume size and the maximum number of channels representable by the multi-channel page table hierarchy [S2].
*   **Performance:** Implemented as a web-based, pure client-side renderer using WebGPU and WebAssembly, it is faster than prior approaches and efficient for many data channels with a flexible and adaptive choice of data resolution [S2].
*   **Strengths:** Decouples cache residency from spatial subdivision, allowing adaptive resolution mixing and cache-miss compensation without requiring all lower resolutions to be resident [S1, unofficial][S2]. Supports fine-grained empty-space skipping independent of data subdivision [S2].
*   **Limitations:** Accumulation-level intermixing (evaluating illumination per channel per sample) is computationally more expensive than image-level intermixing [S1, unofficial]. If the server does not support metadata queries, metadata must be computed on the fly, requiring careful resolution selection to meet the minimum voxel count [S1, unofficial].
*   **When to use:** Out-of-core DVR of large-scale volumetric data sets with a large number of channels, such as Immunofluorescence (IF) imaging data [S2].
*   **When not to use:** Not given in the sources.

## Background
The Residency Octree was introduced as a hybrid approach for scalable web-based volume rendering of large biomedicinal datasets [S1, unofficial][S2]. It addresses the limitations of previous octree-based out-of-core methods (e.g., [CNLE09], [BPH14]) which used a one-to-one mapping between bricks and octree nodes, intrinsically coupling resolution levels to spatial subdivision [S1, unofficial][S2]. It also improves upon page-table-based approaches which lacked efficient strategies for substituting missing high-resolution data with lower-resolution data [S2]. The method is designed for multi-channel immunoﬂuorescence data, using minima/maxima for empty-space skipping instead of histograms to maintain precision for unimodal transfer functions [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
