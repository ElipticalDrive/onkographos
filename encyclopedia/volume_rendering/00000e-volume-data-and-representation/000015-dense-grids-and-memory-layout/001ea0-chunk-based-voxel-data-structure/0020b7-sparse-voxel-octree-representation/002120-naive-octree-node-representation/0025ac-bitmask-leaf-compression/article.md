# Bitmask Leaf Compression

Also called: Voxels como mascara de bits

## Overview
Bitmask Leaf Compression is a memory optimization technique for Sparse Voxel Octrees that eliminates leaf nodes at the deepest level by packing their voxel data into a single integer field within their parent node [S1, unofficial]. It solves the excessive memory overhead of naive octree representations, where storing explicit leaf nodes causes memory usage to approach 8 times that of a full octree of depth $d-1$ [S1, unofficial]. By removing the leaf node structure entirely, it reduces the memory cost factor $mc$ from 8 to 1 for the deepest level [S1, unofficial].

## How it works
**Inputs:** A Sparse Voxel Octree with a defined maximum depth $d$.
**Outputs:** An optimized octree structure where the penultimate level nodes contain packed leaf data.

**Data Structures:**
The node structure is defined as [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct Node {
    Node * children[8]; // Pointers to 8 children
    int64_t leafs;      // 64-bit integer packing leaf voxel data
};
```
*   `children`: Array of 8 pointers. If a child is null, it indicates no subtree exists.
*   `leafs`: A 64-bit signed integer (`int64_t`) used only by nodes at depth $d-1$ (the penultimate level).

**Algorithm Steps:**
1.  **Identify Penultimate Level Nodes:** Locate all nodes at depth $d-1$ in the octree. These nodes would normally have 8 leaf children at depth $d$.
2.  **Color Quantization:** For each of the 8 potential leaf voxels, determine its color. Instead of storing a full RGBA vector, map the color to an index in a palette of 256 entries [S1, unofficial].
    *   The palette has 256 entries.
    *   Index 0 (`00000000_2`) represents the absence of a voxel (empty space) [S1, unofficial].
    *   Indices 1–255 represent existing voxels with specific colors [S1, unofficial].
3.  **Bit Packing:** Pack the 8 color indices (8 bits each) into the `leafs` field of the parent node.
    *   Total bits: $8 \times 8 = 64$ bits [S1, unofficial].
    *   **Bit Mapping:** The highest position child (index 7, typically corresponding to specific spatial coordinates depending on Morton order) represents the leftmost 8-bit strip. The lowest position child (index 0) represents the rightmost strip [S1, unofficial].
    *   Specifically, if the children are indexed $0$ to $7$, the bits for child $i$ occupy bits $8i$ to $8i+7$ in the `leafs` integer, or vice versa depending on endianness and specific implementation of "leftmost/rightmost". The source states: "the highest position child represents the leftmost 8-bit strip, and the lowest position child represents the rightmost strip" [S1, unofficial]. This implies a mapping where Child 7 maps to the most significant bits (leftmost in visual representation) and Child 0 maps to the least significant bits (rightmost) [S1, unofficial].
4.  **Eliminate Leaf Nodes:** Remove the explicit leaf node structures at depth $d$. The parent node at depth $d-1$ now holds the data directly.
5.  **Handle Non-Leaf Parents:** Nodes at depth $< d-1$ continue to use the `children` array to point to their subtrees. Nodes at depth $d-1$ that do not have all 8 children present (sparse regions) still use the `leafs` field, but the corresponding 8-bit segments will contain the index 0 (empty) [S1, unofficial].

**Parameters:**
*   **Palette Size:** 256 entries [S1, unofficial].
*   **Bits per Voxel:** 8 bits [S1, unofficial].
*   **Empty Voxel Index:** 0 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The method limits color resolution to 256 distinct colors per voxel. This is a trade-off for memory savings [S1, unofficial].
*   **Boundaries:** The compression only applies to the *last* level. If the octree depth is not fixed or if the maximum depth is not reached, the optimization does not apply to shallower levels [S1, unofficial].
*   **Degenerate Input:** If a node at depth $d-1$ has fewer than 8 children, the missing children are treated as empty (index 0) in the bitmask [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_leafs(node, palette):
    """
    Compresses the leaf voxels of a node at depth d-1 into its 'leafs' field.
    Assumes node is at the penultimate level of the octree.
    """
    # Initialize leafs field to 0 (all empty)
    node.leafs = 0
    
    # Iterate through the 8 potential children positions
    for i in range(8):
        # Determine if child i exists in the naive structure
        # In a naive structure, we would check node.children[i]
        # Here, we assume we are processing a node that *would* have had children
        # or we are converting from a structure where we know the voxel status.
        
        # Check if voxel i is present
        is_present = check_voxel_presence(i) # [G] Helper to determine if voxel i has data
        
        if is_present:
            # Get color and map to palette index
            color = get_voxel_color(i) # [G] Helper to retrieve color
            index = palette.find_or_add(color)
        else:
            index = 0 # 0 represents absence
        
        # Pack index into the 64-bit integer
        # Source [S1, unofficial]: Highest position child (7) -> leftmost strip (MSB)
        # Lowest position child (0) -> rightmost strip (LSB)
        # Assuming standard bit numbering where bit 0 is LSB:
        # Child 0 -> bits 0-7
        # Child 7 -> bits 56-63
        
        # Shift index to correct position
        shifted_index = index << (8 * i)
        
        # OR into leafs
        node.leafs |= shifted_index

def check_voxel_presence(child_index):
    """
    [G] Placeholder for logic determining if a voxel exists.
    In a naive octree, this would check if node.children[child_index] is not null.
    """
    pass

def get_voxel_color(child_index):
    """
    [G] Placeholder for logic retrieving color.
    In a naive octree, this would access node.children[child_index].color.
    """
    pass
```

## Subtopics

### Leaf Array Compression
A variant that stores all leaf voxels in a single flat array in Morton order to improve cache efficiency, particularly for LOD-optimized hierarchies where finest level voxels are reduced [S2, unofficial]. It starts with a copy of finest level voxels, replaces removed voxels with the simplified voxel from the first finest child, marks removed voxels with a bit, and compresses the array using parallel reduction based on these bits [S2, unofficial]. The new index to the compressed leaf array is stored in parent nodes as the child index [S2, unofficial]. This alleviates cache misses caused by sparse distribution of finest level voxels in LOD hierarchies [S2, unofficial].

## Operating characteristics
**Time Complexity:**
*   **Compression:** $O(N)$ where $N$ is the number of nodes at depth $d-1$, as each node processes 8 voxels.
*   **Decompression:** $O(1)$ per voxel access, as the data is directly in the parent node.

**Memory Complexity:**
*   **Naive Representation:** Memory cost factor $mc$ approaches 8 for deep trees [S1, unofficial].
*   **Bitmask Compression:** Reduces memory overhead by eliminating leaf nodes. The memory cost for the deepest level is reduced by a factor of 8 compared to naive representation [S1, unofficial]. The `leafs` field adds 8 bytes per penultimate node, but saves the size of 8 leaf nodes.

**Strengths:**
*   Significant memory reduction for deep octrees [S1, unofficial].
*   Simple implementation [S1, unofficial].

**Limitations:**
*   Limited to 256 colors per voxel [S1, unofficial].
*   Only applies to the deepest level [S1, unofficial].
*   Requires a fixed maximum depth [S1, unofficial].

**When to Use:**
*   When memory is constrained and the octree depth is known and fixed [S1, unofficial].
*   When color precision can be limited to 256 levels [S1, unofficial].

**When Not to Use:**
*   When higher color precision is required [S1, unofficial].
*   When the octree depth is dynamic or shallow [S1, unofficial].

## Background
Originated in the context of optimizing Sparse Voxel Octrees for Global Illumination calculations [S1, unofficial]. The technique addresses the inefficiency of using explicit leaf nodes in naive octree representations, where the memory overhead becomes prohibitive for deep trees [S1, unofficial]. The term "Voxels como mascara de bits" is used in the source [S1, unofficial].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S2] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
