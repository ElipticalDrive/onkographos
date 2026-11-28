# Rendered Feedback Information

Also called: Feedback-based LRU Cache Management, Lazy Evaluation Scheme for Node Lists

## Overview
Rendered Feedback Information is a lazy evaluation scheme for managing the GPU-to-CPU feedback of traversed octree nodes in volume rendering [S1]. It solves the problem of keeping necessary volume data in the CPU cache pool by collecting node indices only when they are actively accessed or require subdivision, relying on an LRU mechanism to evict unused data automatically [S1]. It differs from active tree maintenance methods by eliminating the need to explicitly send "collapse" or removal information; if a node is no longer traversed, it simply stops appearing in the feedback list, allowing the cache to replace it naturally [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Traversal state of the octree, current level-of-detail (LOD) requirements, and camera position.
*   **Output:** A set of node indices and subdivision tags written to render targets (node-list textures), which the CPU reads to manage its cache pool.

**Data Structures:**
1.  **Node Index:** 30 bits.
2.  **Subdivision Tag:** 1 bit. Indicates if the node needs further subdivision because the required LOD is missing or if it is a terminal node [S1].
3.  **Node-List Textures:** Multiple Multi-Render Targets (MRTs).
    *   The first MRT stores the output color.
    *   Remaining MRTs store node information.
    *   Each texture provides space for 4 encountered nodes (one per RGBA channel).
    *   Each channel stores one node index (30 bits) and its subdivision tag (1 bit) [S1].

**Parameters:**
*   **MRT Count:** Three MRTs are used as the optimal choice for balancing capacity and performance [S1].
*   **Capacity per Pixel:** $3 \times 4 = 12$ nodes per pixel per frame [S1].
*   **Spatial Grouping:** Rays are grouped in $2 \times 2$ packages [S1].
*   **Temporal FIFO:** A First-In-First-Out buffer for node indices.

**Steps:**
1.  **Traversal and Collection:** During ray traversal, stop at the node corresponding to the needed LOD.
    *   If the required data is present, traverse the mipmapped brick.
    *   If the LOD is missing or the node is terminal, add the node index to the node-list.
    *   If the node needs further subdivision, add it to the node-list with the subdivision tag set to 1. Limit subdivisions to one request per ray to potentially change opacity and allow early ray termination in the next frame [S1].
2.  **Spatial Coherence Exploitation:** Group rays into $2 \times 2$ packages. Each ray in the package stores a different set of traversed nodes. For example, the upper-left ray stores the first 12 nodes, the upper-right the second 12, etc. This totals 48 node indices per package [S1].
3.  **Temporal Coherence Exploitation:** Push rendered nodes into a FIFO buffer.
    *   In the first frame, stop pushing after 48 elements.
    *   In the second frame, stop pushing after 96 elements.
    *   Shift the 48-element window over a small set of frames. This increases the number of retrievable nodes and ensures information output in each frame, even in regions with fewer nodes [S1].
4.  **Lazy Evaluation:** Do not send collapse information. If rendering no longer descends into a node, its index is not put into the list. The LRU mechanism on the CPU side replaces the data automatically when the timestamp is not reset [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_ray_package(rays, octree, cache_pool):
    """
    Process a 2x2 package of rays to generate feedback for the CPU cache.
    """
    # Initialize FIFO for temporal coherence
    fifo = []
    max_fifo_size = 48 # Base limit, can be 96 in subsequent frames
    
    # Each ray in the 2x2 package will store a different set of nodes
    # Total capacity per package = 4 rays * 12 nodes/ray = 48 nodes
    package_nodes = [] 
    
    for ray_index, ray in enumerate(rays):
        ray_nodes = []
        
        # Traverse the octree for this ray
        node = octree.root
        while node is not None:
            # Check if required LOD data is present
            if octree.has_data(node, ray.required_lod):
                # Traverse the mipmapped brick
                traverse_brick(node, ray)
                
                # Add node index to list (lazy evaluation: no collapse info needed)
                ray_nodes.append({
                    'index': node.index,
                    'subdivision_tag': 0
                })
                
                # If we have reached the limit for this ray's slot in the package
                if len(ray_nodes) >= 12:
                    break
            else:
                # LOD missing or terminal node
                ray_nodes.append({
                    'index': node.index,
                    'subdivision_tag': 1 # Needs subdivision or is terminal
                })
                
                # Limit subdivisions to one request per ray
                if ray.subdivision_requested:
                    break
                else:
                    ray.subdivision_requested = True
                    # Move to child for further traversal if needed
                    node = get_child(node, ray.direction)
                    continue
            
            # Move to next node in traversal path
            node = get_next_node(node, ray.direction)
            
        # Add this ray's nodes to the package list
        package_nodes.extend(ray_nodes)
        
        # If package is full, we can stop processing other rays in this package
        # for this frame's feedback, but typically all 4 rays contribute their
        # specific 12 nodes.
        
    # Apply temporal coherence via FIFO
    # In practice, this involves shifting the window and pushing new nodes
    for node_info in package_nodes:
        if len(fifo) < max_fifo_size:
            fifo.append(node_info)
        else:
            # FIFO is full, oldest nodes will be evicted
            fifo.pop(0)
            fifo.append(node_info)
            
    # Write to MRTs
    # Each MRT stores 4 nodes per pixel (one per channel)
    # 3 MRTs -> 12 nodes per pixel
    write_to_mrt(rays, fifo)
    
    return fifo

def write_to_mrt(rays, fifo):
    """
    Write the FIFO contents to the node-list textures.
    """
    # Each pixel corresponds to a ray (or ray package center)
    # We need to map the FIFO nodes to the 3 MRTs * 4 channels
    # This is a simplified representation of the packing logic
    mrt_count = 3
    nodes_per_mrt_channel = 4
    
    # Flatten FIFO to a list of node indices/tags
    nodes = [n['index'] for n in fifo]
    tags = [n['subdivision_tag'] for n in fifo]
    
    # Distribute nodes across MRTs and channels
    # This is a high-level description; actual implementation requires
    # careful packing of 30-bit indices and 1-bit tags into 32-bit floats/ints
    for mrt_idx in range(mrt_count):
        for channel_idx in range(4):
            if len(nodes) > 0:
                node_idx = nodes.pop(0)
                tag = tags.pop(0)
                # Pack node_idx (30 bits) and tag (1 bit) into a 32-bit value
                packed_value = (node_idx << 1) | tag
                write_channel(mrt_idx, channel_idx, packed_value)
```

**Edge Cases and Pitfalls:**
*   **Insufficient Capacity:** Using only 3 MRTs provides 12 nodes per pixel, which may be insufficient for complex scenes. The spatial and temporal coherence strategies are required to mitigate this [S1].
*   **GPU Limitations:** Arbitrarily sized node-lists cannot be efficiently output due to GPU limitations (e.g., only eight RGBA32 render targets are typically possible) [S1].
*   **Precision:** The 30-bit node index must be carefully packed into the 32-bit render target channels along with the 1-bit subdivision tag [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays and the depth of the octree traversal, but reduced by early termination and caching.
*   **Memory Complexity:** Depends on the number of MRTs and the FIFO size. With 3 MRTs and a FIFO of 48-96 elements, the memory footprint is small and fixed per frame [S1].
*   **Performance:** The lazy evaluation scheme simplifies previous solutions that needed to maintain the tree actively, reducing CPU-GPU synchronization overhead [S1].
*   **Strengths:** Eliminates the need for explicit collapse information, reducing feedback data size and complexity. Exploits both spatial and temporal coherence to increase effective cache capacity [S1].
*   **Limitations:** Requires careful management of the FIFO and MRT packing. May miss nodes in very complex scenes if coherence assumptions break down [S1].

## Background
Rendered Feedback Information is part of the GigaVoxels framework, introduced by Lapouge et al. in 2009 [S1]. It was developed to address the challenges of rendering large-scale voxel data by efficiently managing the transfer of node information between the GPU and CPU. The method leverages the inherent coherence in volume rendering to minimize the amount of data that needs to be explicitly managed by the CPU cache replacement policy.

## Sources
[S1] GigaVoxels, https://doi.org/10.1145/1507149.1507152, fetched 2026-10-24
