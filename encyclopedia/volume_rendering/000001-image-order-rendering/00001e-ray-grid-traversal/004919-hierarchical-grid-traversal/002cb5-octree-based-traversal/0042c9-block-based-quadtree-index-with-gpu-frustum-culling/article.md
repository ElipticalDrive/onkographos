# Block-based Quadtree Index with GPU Frustum Culling

Also called: Compute shader culling, GPU-side quadtree traversal

## Overview
This method accelerates large-scale terrain rendering by storing a block-based quadtree index entirely in GPU video memory and performing frustum culling and Level-of-Detail (LOD) selection via parallel compute shaders. It differs from CPU-based hierarchical traversal [Octree-based traversal](5348-octree-based-traversal.md) by eliminating CPU-GPU synchronization overhead and from standard sparse voxel approaches [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) by using fixed-size tile blocks with Z-order storage for cache-friendly access.

## How it works
**Inputs and Outputs:**
*   **Input:** Global terrain elevation data (e.g., SRTM), camera frustum parameters, and terrain resolution parameters ($L, K$).
*   **Output:** A list of visible nodes and their corresponding LOD levels, ready for rendering.

**Data Structures:**
The method relies on two primary buffers residing in GPU memory:
1.  **Quadtree Node Index Structure:** A compact structure array containing per-node metadata. Fields include:
    *   Bounding boxes (AABB).
    *   Center coordinates.
    *   Hierarchy levels (depth).
    *   Offsets within the data buffer.
    *   Storage order: Z-order curve (Morton code) to ensure spatially adjacent nodes are adjacent in memory [S1].
2.  **Tile Data Buffer:** Contains the actual elevation samples.
    *   Layout: Tiles are written in Z-order curve order.
    *   Internal Tile Layout: Within each tile, elevation values are stored contiguously in row-major order [S1].
    *   Texture Mapping: Texture atlases are aligned one-to-one with elevation data blocks, sharing deterministic buffer offsets to avoid secondary lookup tables [S1].

**Parameters:**
*   $L$: Exponent defining total terrain size as $2^L \times 2^L$ samples [S1].
*   $K$: Exponent defining leaf node (tile) size as $2^K \times 2^K$ samples. Typical value $K=9$ (512×512 samples) [S1].
*   $D$: Quadtree depth, calculated as $D = L - K$ [S1].

**Algorithm Steps:**
1.  **Index Construction:**
    *   Divide the terrain domain into fixed-size tile blocks of $2^K \times 2^K$ samples.
    *   Calculate quadtree depth $D = L - K$.
    *   Compute total number of nodes $N_{total} = \sum_{i=0}^{D} 4^i = \frac{4^{D+1}-1}{3}$ [S1].
    *   Populate the Node Index Structure with bounding boxes, centers, levels, and Z-order offsets for all $N_{total}$ nodes.
    *   Write elevation data to the Tile Data Buffer in Z-order, with row-major storage within tiles [S1].

2.  **Parallel Culling (Compute Shader):**
    *   Launch a compute shader with one thread per node in the index structure.
    *   Each thread reads its node's bounding box and center.
    *   Perform frustum culling: Test if the node's AABB intersects the camera frustum.
    *   Perform occlusion determination (if applicable): Check visibility against previously culled nodes or depth buffer (implementation detail varies, but cited as part of the parallel output) [S1].
    *   Determine LOD level based on distance or screen-space error (computed on GPU) [S1].
    *   Output a list of visible nodes and their LOD levels.

3.  **Rendering:**
    *   Use the visible node list to fetch elevation and texture data from the Tile Data Buffer and Texture Atlas, respectively, using the precomputed offsets [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader Kernel: ProcessOneNode
# Input: node_index (uint), frustum_planes (array of 6 planes), output_list (append-only buffer)
# Globals: node_indices (array of NodeInfo), tile_data_offsets (array of uint)

struct NodeInfo {
    float3 center;
    float3 bbox_min;
    float3 bbox_max;
    uint level;
    uint data_offset;
    uint z_order;
}

function is_visible(node: NodeInfo, frustum: Frustum) -> bool:
    # AABB vs Frustum intersection test
    # Returns true if any part of the AABB is inside the frustum
    for each plane in frustum:
        if all corners of node.bbox_min/max are outside plane:
            return false
    return true

function compute_lod(node: NodeInfo, camera: Camera) -> uint:
    # Example: LOD based on distance to camera center
    dist = length(node.center - camera.position)
    # Simplified LOD logic; actual implementation may use screen-space error
    if dist < 1000: return 0
    elif dist < 5000: return 1
    else: return 2

kernel main():
    node_id = dispatch_thread_id.x
    node = node_indices[node_id]
    
    if is_visible(node, camera.frustum):
        lod = compute_lod(node, camera)
        output_list.append(node_id, lod)
```

**Edge Cases and Pitfalls:**
*   **Memory Capacity:** For global data (e.g., $1.66 \times 10^{11}$ samples at 30m resolution), the number of leaf nodes ($\sim 6.3 \times 10^5$) and total nodes ($\sim 8.4 \times 10^5$) fit in GPU memory with $K=9$ [S1]. Smaller $K$ increases node count exponentially; larger $K$ reduces granularity.
*   **Z-order Cache Friendliness:** Deviating from Z-order storage may degrade performance due to poor cache locality during traversal [S1].
*   **Texture Alignment:** Mismatched offsets between elevation and texture buffers will cause rendering artifacts. The one-to-one alignment is critical [S1].

## Operating characteristics
*   **Time Complexity:** Frustum culling is $O(N)$ where $N$ is the number of nodes, but executed in parallel on the GPU. The constant factor is low due to simple AABB tests.
*   **Memory Complexity:** $O(N)$ for the index structure. For global SRTM30m data with $K=9$, $\sim 8.4 \times 10^5$ nodes fit in GPU memory [S1].
*   **Performance:** Eliminates CPU-GPU transfer overhead for culling. Verified superior rendering throughput over WebGL-based methods [S1].
*   **Strengths:** Scalable to global datasets; cache-friendly storage; parallel culling.
*   **Limitations:** Requires GPU memory for the entire index; less flexible for dynamic topology changes compared to CPU-managed trees [S1].

## Background
This method was proposed in the context of WebGPU terrain visualization, addressing the bottleneck of CPU-side traversal and data reorganization for large-scale datasets [S1]. It builds on prior work in WebGPU for terrain [1-5] and dynamic terrain tiling [7], leveraging space-filling curves for cache efficiency [9].

## Sources
[S1] Efficient Parsing and Rendering of Large-Scale Terrain Data Based on WebGPU, https://doi.org/10.1145/3844212.3844228, fetched 2026-11-02
