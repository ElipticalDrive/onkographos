# Compaction of Update Information

Also called: HistoPyramids Reduction, Node-List Compaction

## Overview
This method reduces the bandwidth of node update information sent from the GPU to the CPU during ray-guided volume rendering by eliminating duplicate node references using spatial coherence [S1, unofficial]. It differs from sibling methods like [Stack-Based Representation](1081-stack-based-representation-sbr.md) by explicitly addressing the bandwidth bottleneck of feedback data through a GPU-optimized prefix-sum reduction (HistoPyramids) rather than relying solely on CPU-side data structures or simple list truncation [S1, unofficial].

## How it works
**Inputs:**
*   A list of node indices encountered by each ray during traversal, stored in node-list textures.
*   The current frame's pixel coordinates.

**Outputs:**
*   A compacted node index texture containing unique node indices and their associated ray pixel positions, ready for CPU consumption.

**Data Structures:**
*   **Node-List Textures:** Multiple render targets (MRTs) used to store intermediate node lists. Each texture stores four nodes per texel (one per RGBA channel). Each node entry is packed into a 32-bit integer: 30 bits for the node index and 1 bit for a subdivision tag, with 1 bit unused [S1, unofficial].
*   **Selection Mask:** A texture where each pixel contains a bitmask. The $i$-th bit indicates whether the $i$-th element in the corresponding node-list texel should be kept.
*   **Compact Node Index Texture:** A single RGBA32 texture storing the final reduced node indices.

**Algorithm Steps:**

1.  **Buffer Reservation:** Reserve the first render target for output color. Use the remaining buffers as node-list textures [S1, unofficial].
2.  **Selection Mask Derivation:** In a first pass, derive a selection mask based on local spatial coherence.
    *   Compare the center node list against surrounding lists.
    *   An entry is kept only if it does not appear in the surrounding lists [S1, unofficial].
    *   To optimize, perform an approximate comparison: test the $i$-th list element only against the $(i-1)$-th, $i$-th, and $(i+1)$-th elements of neighboring lists [S1, unofficial]. This is conservative and may induce "false positives" (keeping duplicates), but avoids costly full comparisons [S1, unofficial].
3.  **HistoPyramids Reduction:** Apply HistoPyramids to reduce the selection mask. HistoPyramids is a special case of the prefix-sum scan and binary search algorithm, optimized for 2D locality and GPU texture processing units [S1, unofficial].
4.  **Index Recovery and Packing:**
    *   Recover the actual corresponding node indices from the reduced mask.
    *   Encode the pixel position of the ray in the remaining 20 bits (over 32) of the 12 node entries to allow index recovery after reduction, as the original pixel position is lost during the rearrangement [S1, unofficial].
    *   After reduction, each pixel contains a ray pixel-position and a bitmask.
5.  **Final Compaction:** Create a compact node index texture from the bitmask and position. Typically, store indices of four nodes in a single RGBA32 texture [S1, unofficial].
6.  **Postponement:** If the bitmask exceeds the limit (i.e., more than 4 unique nodes per ray), postpone requests to the next frame [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compact_node_updates(node_list_textures, pixel_positions):
    """
    Compacts node update information for GPU-to-CPU transfer.
    
    Args:
        node_list_textures: List of textures, each storing 4 nodes per texel.
                            Each node is a 32-bit int: [30-bit index, 1-bit tag, 1-bit unused].
        pixel_positions: Array of (x, y) coordinates for each ray/pixel.
        
    Returns:
        compact_texture: Single RGBA32 texture with reduced node indices and positions.
    """
    
    # Step 1: Derive selection mask via local comparison
    # For each pixel (p), compare its node list against neighbors
    selection_mask = create_empty_texture_like(node_list_textures[0])
    
    for each pixel p in image:
        center_list = get_nodes_from_textures(p, node_list_textures)
        mask_bits = []
        
        for i in range(len(center_list)):
            node = center_list[i]
            is_unique = True
            
            # Approximate comparison: check against neighbors at indices i-1, i, i+1
            for neighbor_pixel in get_surrounding_pixels(p):
                neighbor_list = get_nodes_from_textures(neighbor_pixel, node_list_textures)
                
                # Check range [max(0, i-1), min(len, i+2)]
                start_idx = max(0, i - 1)
                end_idx = min(len(neighbor_list), i + 2)
                
                for j in range(start_idx, end_idx):
                    if neighbor_list[j] == node:
                        is_unique = False
                        break
                if not is_unique:
                    break
            
            mask_bits.append(is_unique)
        
        # Pack bits into the selection mask pixel
        set_selection_mask_pixel(p, mask_bits)

    # Step 2: Reduce selection mask using HistoPyramids
    # HistoPyramids performs a prefix-sum scan optimized for 2D locality
    reduced_mask = apply_histo_pyramids_reduction(selection_mask)
    
    # Step 3: Recover indices and pack into final texture
    compact_texture = create_rgba32_texture()
    
    for each pixel p in image:
        # Retrieve reduced bitmask and original pixel position
        bitmask = get_reduced_bitmask(p, reduced_mask)
        pos = pixel_positions[p]
        
        # Extract kept nodes from original lists using bitmask
        kept_nodes = []
        for i in range(len(bitmask)):
            if bitmask[i]:
                # Retrieve node from original node_list_textures
                node = get_node_from_original_lists(p, i, node_list_textures)
                kept_nodes.append(node)
        
        # Pack up to 4 nodes into the RGBA32 texture
        # Each channel stores: [30-bit node index, 1-bit tag, 1-bit unused]
        # Note: The source mentions encoding pixel position in remaining bits 
        # of the 12 node entries to recover indices. 
        # Implementation detail: Store pos in unused bits or separate field if needed.
        # Here we assume standard packing into 4 channels.
        
        for c in range(4):
            if c < len(kept_nodes):
                compact_texture[p].channels[c] = pack_node(kept_nodes[c])
            else:
                compact_texture[p].channels[c] = 0 # Empty slot

    return compact_texture

def pack_node(node_data):
    """
    Packs node index and tag into 32 bits.
    Format: [30 bits index, 1 bit tag, 1 bit unused]
    """
    index = node_data['index']
    tag = node_data['subdivision_tag']
    return (index << 2) | (tag << 1)
```

**Parameters:**
*   **Node Index Bits:** 30 bits [S1, unofficial].
*   **Subdivision Tag Bits:** 1 bit [S1, unofficial].
*   **Unused Bits:** 1 bit per node entry [S1, unofficial].
*   **Node-List Textures:** 7 textures, allowing for a $7 \times 4 = 28$-node output per ray [S1, unofficial].
*   **Final Texture Capacity:** 4 nodes per pixel (RGBA32) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **False Positives:** The approximate comparison may keep duplicate nodes, increasing the size of the final texture. However, this is conservative and simplifies logic [S1, unofficial].
*   **Bandwidth Exceedance:** If the bitmask exceeds the 4-node limit per pixel, requests are postponed to the next frame [S1, unofficial].
*   **Precision:** The 30-bit index limit may constrain the maximum number of nodes in the tree [S1, unofficial].

## Subtopics

### Histogram Packing
A compression technique applied to images with sparse histograms of intensity levels to improve compression ratios, specifically in volumetric medical images [S2]. It addresses sparse histogram issues that cause bitrate deterioration in entropy estimation heuristics [S2]. It is compliant with JP3D via the Non-Linear Transformation (NLT) defined in JPEG 2000 Annex K, allowing the original histogram to be restored in the decoder pipeline [S2]. Performance: HP alone (HP+DWT) yields an average bitrate improvement of 5.34% across the test set. For sparse histogram images (MRI1, MRI3), it improves compression ratio by over 20%. HP alone improves the compression ratio of sparse histogram images by almost 29% on average [S2]. Pitfall: Sparse histograms make standard entropy estimation ineffective without HP [S2]. Parameter: The decoder must support the Non-Linear Transformation (NLT) extension defined in JPEG 2000 Part 2 (Annex K) to restore the original histogram [S2].

## Operating characteristics
*   **Time Complexity:** The selection mask derivation is $O(N \cdot k)$ where $N$ is the number of pixels and $k$ is the local comparison window size (typically 3). The HistoPyramids reduction is highly optimized for GPU texture processing units, leveraging 2D locality [S1, unofficial].
*   **Memory Complexity:** Requires 7 node-list textures plus the final compact texture. Each node-list texture stores 4 nodes per texel [S1, unofficial].
*   **Measured Performance:** The method allows for efficient feedback in ray-guided streaming, enabling high-quality rendering (e.g., 20 FPS for a $1024^3$ volume with complex shading) by reducing CPU-GPU bandwidth bottlenecks [S1, unofficial].
*   **Strengths:** Reduces bandwidth significantly; lazy evaluation scheme simplifies tree management; GPU-optimized reduction [S1, unofficial].
*   **Limitations:** Approximate comparison may lead to false positives; requires postponement if limits are exceeded; limited by 30-bit node index [S1, unofficial].
*   **When to Use:** When rendering large sparse volumes with ray-guided streaming where CPU-GPU feedback bandwidth is a bottleneck [S1, unofficial].
*   **When Not to Use:** When the number of unique nodes per ray consistently exceeds 4, leading to frequent postponement [S1, unofficial].

## Background
Originated in the context of Ray-Guided Streaming for Efficient and Detailed Voxel Rendering [S1, unofficial]. The method was developed to address the limitations of ad-hoc solutions (like storing only the first node) and to efficiently handle the feedback of node information from GPU to CPU in dynamic volume rendering scenarios [S1, unofficial].

## Sources
[S1] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
[S2] Employing New Hybrid Adaptive Wavelet-Based Transform and Histogram Packing to Improve JP3D Compression of Volumetric Medical Images, https://doi.org/10.3390/e22121385, fetched 2026-10-26
