# Compaction of Update Information

Also called: HistoPyramids Reduction, Node-List Compaction

## Overview
This method reduces the bandwidth of node update information sent from the GPU to the CPU by removing duplicate node references using spatial coherence and HistoPyramids [S1, unofficial]. It solves the problem of limited bandwidth when transferring sparse voxel tree navigation data for adaptive rendering [S1, unofficial]. It differs from sibling methods by using a 2D-locality-optimized prefix-sum scan (HistoPyramids) on selection masks derived from local list comparisons, rather than global sorting or hash-based deduplication [S1, unofficial].

## How it works
**Inputs:**
- Node lists generated during ray traversal. Each ray produces a list of encountered nodes.
- A set of node-list textures (MRTs).

**Outputs:**
- A compact node index texture containing unique node indices and their associated ray pixel positions.
- A bitmask indicating which nodes were kept.

**Data Structures:**
1.  **Node List Entry:** A 32-bit integer per node.
    -   Bits 0–29: Node index (30 bits).
    -   Bit 30: Subdivision tag (1 bit). Indicates if the CPU should reset the timestamp (node present) or induce subdivision (node missing/terminal) [S1, unofficial].
    -   Bit 31: Unused in the entry itself, but used in the final compacted texture for pixel position encoding [S1, unofficial].
2.  **Node-List Textures:** Multiple render targets (MRTs).
    -   Each texture provides room for four encountered nodes (one per RGBA channel).
    -   The subdivision tag is stored alongside the node index in the same 32-bit value [S1, unofficial].
    -   The method uses seven node-list textures, allowing for a $7 \times 4 = 28$-node output per ray [S1, unofficial].
3.  **Selection Mask Texture:** Stores a bitvector per pixel. The $i$-th bit indicates if the $i$-th element in the node list should be kept [S1, unofficial].
4.  **Compact Node Index Texture:** A single RGBA32 texture storing indices of up to four nodes per pixel, along with ray pixel position [S1, unofficial].

**Steps:**
1.  **Render Target Setup:** Reserve the first render target for output color. Use the remaining buffers as node-list textures [S1, unofficial].
2.  **Node List Generation:** During rendering, traverse the tree. Stop at the required level-of-detail. If data is present, add the current node index to the node-list. If the level-of-detail is missing or the node is terminal, add it to the node-list. The CPU manages the pool and timestamps [S1, unofficial].
3.  **Selection Mask Derivation (First Pass):**
    -   Compare the center node list against surrounding lists (2D locality pattern) [S1, unofficial].
    -   An entry is kept only if it does not appear in the surrounding lists [S1, unofficial].
    -   **Approximate Comparison:** For efficiency, the $i$-th list element is tested only against the $(i-1)$-th, $i$-th, and $(i+1)$-th elements of neighboring lists [S1, unofficial]. This is conservative and may induce "false positives" (keeping duplicates), but avoids costly full comparisons [S1, unofficial].
    -   Output a bitvector to the selection mask texture [S1, unofficial].
4.  **HistoPyramids Reduction:**
    -   Apply HistoPyramids (a GPU-optimized prefix-sum scan exploiting 2D locality) to the selection mask [S1, unofficial].
    -   This reduces the selection mask to determine the final compacted indices [S1, unofficial].
5.  **Index Recovery and Encoding:**
    -   During reduction, original pixel positions are lost. To recover node indices later, encode the ray's pixel position in the remaining 20 bits (over 32) of the 12 node entries per list [S1, unofficial].
    -   After reduction, each pixel contains a ray pixel-position and a bitmask [S1, unofficial].
6.  **Compact Texture Creation:**
    -   Create a compact node index texture from the bitmask and position.
    -   Typically, the bitmask contains 2–3 non-zero entries. Store indices of four nodes in a single RGBA32 texture [S1, unofficial].
    -   If the bitmask exceeds the limit (more than 4 nodes), postpone requests to the next frame [S1, unofficial].
7.  **CPU Transfer:** Transfer the compacted texture to the CPU [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Setup
color_buffer = allocate_render_target()
node_list_textures = allocate_mrt_textures(count=7) # 7 textures for 28 nodes

# Step 2: Generate Node Lists (GPU Kernel)
def generate_node_lists(rays):
    for ray in rays:
        node_list = []
        node = traverse_tree(ray)
        while node is not None:
            if data_present(node):
                node_list.append((node.index, 0)) # 0: no subdivision needed
            else:
                node_list.append((node.index, 1)) # 1: subdivision needed
                break
            node = node.child
        # Store in node_list_textures, 4 nodes per texture/channel
        store_to_mrt(node_list, node_list_textures)

# Step 3: Selection Mask Derivation
def derive_selection_mask(node_list_textures):
    selection_mask = allocate_texture()
    for pixel in selection_mask:
        center_list = get_list_from_mrt(pixel, node_list_textures)
        surrounding_lists = get_neighbor_lists(pixel, node_list_textures)
        mask = 0
        for i, (idx, tag) in enumerate(center_list):
            # Approximate comparison: check against i-1, i, i+1 of neighbors
            is_unique = True
            for neighbor_list in surrounding_lists:
                for j in range(max(0, i-1), min(len(neighbor_list), i+2)):
                    if neighbor_list[j].index == idx:
                        is_unique = False
                        break
                if not is_unique:
                    break
            if is_unique:
                mask |= (1 << i)
        selection_mask[pixel] = mask
    return selection_mask

# Step 4: HistoPyramids Reduction
def reduce_selection_mask(selection_mask):
    # HistoPyramids implementation (prefix-sum scan)
    # Returns reduced indices and bitmask
    return histo_pyramids_scan(selection_mask)

# Step 5: Index Recovery and Encoding
def recover_indices(reduced_data, original_node_lists):
    compact_texture = allocate_rgba32_texture()
    for pixel in reduced_data:
        bitmask = pixel.bitmask
        pixel_pos = pixel.pixel_position # Encoded in remaining bits
        nodes_to_keep = []
        for i in range(len(bitmask)):
            if bitmask & (1 << i):
                # Retrieve original node index and tag
                node = original_node_lists[pixel][i]
                nodes_to_keep.append(node)
        
        # Store up to 4 nodes in RGBA channels
        for c in range(4):
            if c < len(nodes_to_keep):
                compact_texture[pixel][c] = encode_node(nodes_to_keep[c], pixel_pos)
            else:
                compact_texture[pixel][c] = 0
    return compact_texture

# Step 6: Transfer
def transfer_to_cpu(compact_texture):
    if compact_texture.is_full():
        postpone_to_next_frame()
    else:
        send_to_cpu(compact_texture)
```

**Parameters:**
-   Node index bits: 30 [S1, unofficial].
-   Subdivision tag bits: 1 [S1, unofficial].
-   Number of node-list textures: 7 [S1, unofficial].
-   Nodes per texture: 4 (one per channel) [S1, unofficial].
-   Total nodes per ray: 28 [S1, unofficial].
-   Pixel position encoding bits: 20 (remaining bits in 32-bit entry) [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Bandwidth Limit:** If the bitmask exceeds the capacity of the compact texture (more than 4 nodes), requests are postponed to the next frame [S1, unofficial].
-   **False Positives:** The approximate comparison may keep duplicate nodes, increasing the size of the compacted texture slightly, but this is conservative and acceptable [S1, unofficial].
-   **Precision:** The 30-bit node index limit may restrict the maximum tree depth or node count [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** The selection mask derivation is $O(N \cdot K)$ where $N$ is the number of rays and $K$ is the number of nodes per list, but the approximate comparison reduces the constant factor. HistoPyramids is $O(N)$ with high GPU efficiency [S1, unofficial].
-   **Memory Complexity:** Requires 7 node-list textures plus the compact texture. The compact texture is small (4 nodes per pixel) [S1, unofficial].
-   **Performance:** Reduces bandwidth significantly by removing duplicates. The method allows for high-quality rendering (e.g., 20 FPS for a $1024^3$ volume with complex shading) [S1, unofficial].
-   **Strengths:** Efficient GPU implementation, low bandwidth usage, lazy evaluation (no active tree maintenance) [S1, unofficial].
-   **Limitations:** Limited by 30-bit node index, requires postponement if too many nodes are encountered, approximate comparison may keep some duplicates [S1, unofficial].

## Background
This method was introduced in "Ray-Guided Streaming for Efficient and Detailed Voxel Rendering" [S1, unofficial]. It builds on the concept of HistoPyramids [ZTTS06] for prefix-sum scans. It was used in applications such as cumulus cloud rendering with multiple scattering [S1, unofficial].

## Sources
[S1] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
