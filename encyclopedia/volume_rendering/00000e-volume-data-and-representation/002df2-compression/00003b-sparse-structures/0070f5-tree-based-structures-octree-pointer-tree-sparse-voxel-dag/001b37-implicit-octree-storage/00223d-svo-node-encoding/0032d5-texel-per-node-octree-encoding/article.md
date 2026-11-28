# Texel-per-Node Octree Encoding

Also called: Texel-based SVO, Texture-based Octree Node Encoding

## Overview
Texel-per-Node Octree Encoding is a storage optimization for Sparse Voxel Octrees that maps each octree node to exactly one texture texel to maximize tree depth within hardware texture size limits [S1, unofficial]. It differs from general SVO Node Encoding [SVO Node Encoding](2705-svo-node-encoding.md) by packing node type and child pointers directly into 2D texture channels rather than using raw integer arrays [S1, unofficial].

## How it works
**Inputs:** An octree structure where nodes represent spatial subdivisions [S1, unofficial].
**Outputs:** A 2D texture containing the encoded octree [S1, unofficial].

**Data Structures:**
The core data structure is a 2D texture where each texel $(x, y)$ corresponds to one octree node [S1, unofficial]. The channel layout is defined as follows:
- **Alpha Channel (.a):** Indicates the node type [S1, unofficial].
  - Value $> 0$: Inner node (has children) [S1, unofficial].
  - Value $= 0$: Leaf node (contains data) or Empty node [S1, unofficial].
- **Red Channel (.r):** For inner nodes, stores the offset to the first child node in the texture array [S1, unofficial].
- **Green/Blue Channels (.g, .b):** Unused for node metadata in this encoding [S1, unofficial].

**Numbered Steps:**
1. **Initialize Root:** Assume the root node represents a unit cube [S1, unofficial]. Assign it to a specific texel index in the texture [S1, unofficial].
2. **Traverse and Encode:** For each node in the octree:
   a. Determine if the node is empty, leaf, or inner [S1, unofficial].
   b. Write the node type to the Alpha channel (.a) [S1, unofficial].
   c. If the node is inner, calculate the index of its first child in the texture array and write this integer offset to the Red channel (.r) [S1, unofficial].
3. **Child Ordering:** Store child nodes in a specific, deterministic order to allow reconstruction of spatial relationships without storing explicit coordinates [S1, unofficial].
4. **Bounding Box Implied:** Do not store bounding box coordinates for each node; take advantage of implied bounding boxes [S1, unofficial].

**Formulas:**
- **Node Type Check:**
  $$ \text{is\_inner}(texel) = texel.a > 0 $$
- **Child Offset Retrieval:**
  $$ \text{first\_child\_index} = texel.r $$
- **Bounding Box Derivation:**
  Given a parent bounding box and child index $c \in [0, 7]$, the child's bounding box is derived by splitting the parent's dimensions along the x, y, and z axes according to the binary representation of $c$ [G].

**Parameters:**
- **Max Texture Size:** 4096x4096 (4K) for DirectX 9 compatible hardware [S1, unofficial].
- **Texture Size Limits:** The maximum depth of the octree is limited by the number of texels in the texture [S1, unofficial]. A 4Kx4K texture holds $16,777,216$ nodes [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Overflow:** If the octree exceeds the texture capacity, overflow occurs [S1, unofficial].
- **Precision:** Not given in the sources.
- **Empty Nodes:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_octree_to_texture(root_node, max_texels=4096*4096):
    # Allocate flat texture buffer: W x H x 4 channels (RGBA)
    texture = [0] * (max_texels * 4)
    index = 0
    # Stack for iterative DFS traversal
    stack = [(root_node, 0)]
    
    while stack:
        node, parent_idx = stack.pop()
        if index >= max_texels:
            raise OverflowError("Octree exceeds 4Kx4K texture limit")
            
        texel_idx = index
        index += 1
        
        # Alpha channel: 0 for empty, 1 for leaf/inner
        texture[texel_idx * 4 + 3] = 0 if node.is_empty else 1
        
        if not node.is_empty and not node.is_leaf:
            # Red channel: offset to first child in the packed array
            texture[texel_idx * 4 + 0] = index
            # Push children in fixed order (e.g., 0 to 7) to maintain implied bounding boxes
            for child in reversed(node.children):
                stack.append((child, texel_idx))
                
    return texture
```

## Subtopics

### Texture-Based Octree Rendering
A rendering technique that uses an octree built around bricks of a certain size [S2, unofficial]. Cuts of the octree are reconstructed into textures and sent to the GPU for volume rendering [S2, unofficial]. The octree lookup process was a bottleneck in early implementations, resulting in sub-interactive frame rates for medium-sized (256^3) volumes [S2, unofficial]. This method differs from Texel-per-Node Encoding by focusing on the rendering pipeline's use of brick-based textures rather than the storage format of the octree nodes themselves [S2, unofficial]. It is chosen when out-of-core GPU paging is required to bypass limited GPU memory [S2, unofficial].

## Operating characteristics
- **Memory Complexity:** Memory usage is strictly bounded by the texture size ($W \times H \times 4$ bytes) [S1, unofficial].
- **Strengths:** Maximizes tree depth within hardware limits [S1, unofficial].
- **Limitations:** Limited by maximum texture dimensions (e.g., 4K) [S1, unofficial]. Requires specific child ordering to reconstruct spatial hierarchy [S1, unofficial].

## Background
The technique was proposed in a Stack Overflow discussion by user 3Dave in 2012, building on the concept of efficient sparse octree storage [S1, unofficial]. The approach uses a 4K by 4K 2D texture in DX9 to store octree nodes (one texel per node) [S1, unofficial].

## Sources
[S1] Efficient storage for a sparse octree?, https://stackoverflow.com/questions/6085209/efficient-storage-for-a-sparse-octree, fetched 2026-10-26 (unofficial)
[S2] PDF A Survey of Octree Volume Rendering Methods, https://www.sci.utah.edu/~knolla/octsurvey.pdf, fetched 2026-10-29 (unofficial)
