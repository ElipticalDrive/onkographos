# SSVDAG Node Layout and Tagged Pointer Encoding

Also called: Compact representation, Tagged pointer encoding

## Overview
This method defines the memory layout of inner nodes in a Symmetry-aware Sparse Voxel Directed Acyclic Graph (SSVDAG) to minimize memory consumption while supporting GPU-friendly traversal [S1, unofficial]. It replaces the fixed-size child pointers of standard Sparse Voxel DAGs (SVDAGs) with variable-length tagged pointers (2 or 4 bytes) that embed transformation codes, allowing the representation of symmetry-related subtrees with reduced overhead [S1, unofficial]. Unlike standard SVDAGs which use only 0 or 4-byte pointers, this layout introduces 2-byte pointers for frequent, close-range references, significantly compressing the data structure for high-resolution scenes [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Input:** An SSVDAG structure containing an array of inner-level nodes, each represented by a 16-bit header followed by variable-length child pointer data, along with an indexing structure providing the base address of the next level's array and the level-word size [S1, unofficial].
*   **Output:** The absolute memory address of the requested child node and its associated 3-bit transformation tag [S1, unofficial].

### Data Structures

#### Inner-Level Node Header
Each inner node begins with a 16-bit header containing 8 entries, one for each potential child (octant 0–7) [S1, unofficial]. Each entry is a 2-bit code [S1, unofficial]:
*   `00`: Null pointer (child does not exist). No data follows for this child [S1, unofficial].
*   `01`: 16-bit child pointer [S1, unofficial].
*   `10`: 32-bit child pointer [S1, unofficial].
*   `11`: 32-bit child pointer [S1, unofficial].

#### Child Pointer Encoding
The format of the child pointer data immediately following the header depends on the header tag [S1, unofficial]. All pointers encode a 3-bit transformation code (reflection tag) and an offset [S1, unofficial].

**Tag 01 (16-bit Pointer):**
*   **Size:** 2 bytes (16 bits) [S1, unofficial].
*   **Layout:**
    *   Bits 15–13 (3 MSBs): Transformation code (reflection tag) [S1, unofficial].
    *   Bits 12–0 (13 LSBs): Offset in number of level-words from the start of the next level's array [S1, unofficial].
*   **Addressing Range:**
    *   For inner levels: Level-word size is 2 bytes. Max offset $2^{13}$ words $\times$ 2 bytes/word = $2^{14}$ bytes (16 KB) [S1, unofficial].
    *   For leaf levels: Level-word size is 8 bytes. Max offset $2^{13}$ words $\times$ 8 bytes/word = $2^{16}$ bytes (64 KB) [S1, unofficial].

**Tags 10 and 11 (32-bit Pointer):**
*   **Size:** 4 bytes (32 bits) [S1, unofficial].
*   **Layout:**
    *   Bits 31–29 (3 MSBs): Transformation code (reflection tag) [S1, unofficial].
    *   Bits 28–0 (29 LSBs): Lowest 29 bits of the offset [S1, unofficial].
    *   Bit 29 (Highest bit of offset): Set to the value of the rightmost bit of the header tag (bit 1 of the 2-bit header code) [S1, unofficial].
        *   If header tag is `10` (binary 2), rightmost bit is 0. Offset MSB = 0 [S1, unofficial].
        *   If header tag is `11` (binary 3), rightmost bit is 1. Offset MSB = 1 [S1, unofficial].
*   **Addressing Range:**
    *   For inner levels: Max offset $\approx 2^{30}$ words $\times$ 2 bytes/word > 2 GB [S1, unofficial].
    *   For leaf levels: Max offset $\approx 2^{30}$ words $\times$ 8 bytes/word > 8 GB [S1, unofficial].

### Algorithm: Accessing the i-th Child Pointer
To access the $i$-th child pointer (where $i \in [0, 7]$) of a node stored at memory address `node_addr`:
1.  **Read Header:** Load the 16-bit header from `node_addr` [S1, unofficial].
2.  **Compute Offset:** Calculate the byte offset `ptr_offset` within the node data (immediately following the header) where the $i$-th pointer begins [S1, unofficial]. This is the sum of the sizes of all pointers for children $j < i$ [S1, unofficial].
    *   For each child $j < i$, read the 2-bit code from the header [S1, unofficial].
    *   If code is `00`, size is 0 [S1, unofficial].
    *   If code is `01`, size is 2 [S1, unofficial].
    *   If code is `10` or `11`, size is 4 [S1, unofficial].
    *   `ptr_offset = sum(sizes for j in 0..i-1)` [S1, unofficial].
3.  **Read Pointer Data:** Read `size` bytes from `node_addr + 16 bits + ptr_offset` [S1, unofficial].
4.  **Decode Transformation:** Extract the 3 MSBs of the pointer data [S1, unofficial]. This is the reflection tag for the child subtree [S1, unofficial].
5.  **Decode Offset:**
    *   If tag is `01`: Extract bits 12–0. This is the word offset [S1, unofficial].
    *   If tag is `10` or `11`: Extract bits 28–0. Combine with the MSB (derived from header tag) to form the full 30-bit offset [S1, unofficial].
6.  **Calculate Child Address:**
    *   Determine the base address of the next level's array (provided by the indexing structure) [S1, unofficial].
    *   Convert word offset to byte offset: `byte_offset = word_offset * level_word_size` [S1, unofficial].
        *   `level_word_size` is 2 bytes for inner levels, 8 bytes for leaf levels [S1, unofficial].
    *   `child_addr = next_level_base + byte_offset` [S1, unofficial].

### Traversal State Management
During ray traversal, the transformation status must be maintained to correctly interpret indices and voxel contents [S1, unofficial]:
1.  **Initialization:** Set `transform_status = 0` at the root [S1, unofficial].
2.  **Descent:** When moving to a child node:
    *   Extract the reflection tag `t` from the child pointer [S1, unofficial].
    *   Update `transform_status = transform_status XOR t` [S1, unofficial].
    *   Push `transform_status` to the traversal stack together with the current node context to be able to restore it upon backtracking [S1, unofficial].
3.  **Application:**
    *   For inner nodes: Apply `transform_status` to the indices used to access child pointers if the traversal logic requires coordinate transformation [S1, unofficial].
    *   For leaf nodes: Apply `transform_status` to the voxel content indices or attributes [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_child_address(header, node_data, child_idx, next_level_base, level_word_size):
    """
    Extracts the child pointer and transformation tag for a specific child index.
    
    Args:
        header (int): 16-bit integer containing 8 x 2-bit codes.
        node_data (list[int]): Byte array representing the variable-length pointer data following the header.
        child_idx (int): Index of the child (0-7).
        next_level_base (int): Base address of the next level's array.
        level_word_size (int): 2 for inner levels, 8 for leaf levels.
        
    Returns:
        tuple: (child_address, transform_tag)
    """
    # Extract 2-bit codes for children 0 to 7 from the 16-bit header
    # Assuming standard packing: Child 0 is bits 15-14, Child 1 is 13-12, etc.
    codes = []
    for i in range(8):
        shift = 2 * (7 - i)
        code = (header >> shift) & 0x3
        codes.append(code)
        
    # Calculate byte offset to the start of the child_idx pointer
    # Sum sizes of all preceding children (0 to child_idx-1)
    ptr_offset = 0
    for j in range(child_idx):
        if codes[j] == 0:
            ptr_offset += 0
        elif codes[j] == 1:
            ptr_offset += 2
        else: # codes[j] is 2 or 3
            ptr_offset += 4
            
    # Read pointer bytes from node_data
    ptr_size = 2 if codes[child_idx] == 1 else 4
    ptr_bytes = node_data[ptr_offset : ptr_offset + ptr_size]
    
    # Convert bytes to integer (big-endian)
    ptr_val = 0
    for b in ptr_bytes:
        ptr_val = (ptr_val << 8) | b
        
    # Extract transformation tag (3 MSBs)
    transform_tag = (ptr_val >> (8 * ptr_size - 3)) & 0x7
    
    # Decode offset
    if codes[child_idx] == 1: # 16-bit pointer
        word_offset = ptr_val & 0x1FFF # Lower 13 bits
    else: # 32-bit pointer (codes 2 or 3)
        low_offset = ptr_val & 0x1FFFFFF # Lower 29 bits
        # MSB of offset is the rightmost bit of the header tag (bit 1 of the 2-bit code)
        msb = codes[child_idx] & 0x1
        word_offset = (msb << 29) | low_offset
        
    # Compute absolute address
    byte_offset = word_offset * level_word_size
    return next_level_base + byte_offset, transform_tag
```

### Edge Cases and Pitfalls
*   **Variable Size Access:** Accessing the $i$-th child requires summing the sizes of all preceding children ($0$ to $i-1$) [S1, unofficial]. Since sizes can be 0, 2, or 4 bytes, this differs from standard SVDAGs where sizes are only 0 or 4 [S1, unofficial]. This computation must be performed in the shader, typically via a manually unrolled loop [S1, unofficial].
*   **Offset Reconstruction:** For 32-bit pointers, the most significant bit of the offset is not stored in the pointer data but is derived from the header tag [S1, unofficial]. Failure to correctly extract and apply this bit will result in incorrect address calculation [S1, unofficial].
*   **Transformation Propagation:** The transformation tag extracted from a pointer applies to the entire subtree [S1, unofficial]. The traversal state must be correctly maintained (XOR update and stack push/pop) to ensure correct coordinate mapping for nested symmetric structures [S1, unofficial].

## Subtopics

### [Compact SSVDAG Encoding](5227-compact-ssvdag-encoding.md)
Compact SSVDAG Encoding is a GPU-friendly memory layout for Symmetry-aware Sparse Voxel Directed Acyclic Graphs that minimizes pointer overhead while enabling fast, decompression-free ray tracing. It distinguishes itself from standard SVDAGs by combining leaf grouping, frequency-based pointer compaction, and half-word memory alignment to reduce the memory footprint of high-resolution scenes. The technique relies on highly skewed reference distributions, where in a 64K³ Powerplant dataset, the most common 10% of nodes account for nearly 90% of pointers at level 14 and nearly 50% at level 11.

### [SVDAG Parameter Configurations](5945-svdag-parameter-configurations.md)
SVDAG Parameter Configurations define the specific bit-widths for Child Node Masks (CHNM), Child Node Pointers (PT), and Addressing Words (addr. word) within a Symmetry-aware Sparse Voxel DAG (SSVDAG) node layout. They solve the problem of optimizing memory footprint versus addressable space in hierarchical data structures by allowing variable-length pointers. Unlike the base SVO configuration which uses fixed 32-bit pointers, these configurations (SVDAG1–6) enable the use of shorter pointers (8-bit or 16-bit) for nodes located closer to the root, significantly reducing memory consumption for large scenes while maintaining the ability to address distant nodes via longer pointers.

## Operating characteristics

*   **Memory Footprint:** Significantly reduced compared to fixed-pointer SVDAGs [S1, unofficial]. By using 2-byte pointers for frequent, close-range references (Tag 01), the average pointer size is lowered [S1, unofficial].
*   **Addressable Space:**
    *   Inner levels: >2 GB via 32-bit pointers [S1, unofficial].
    *   Leaf levels: >8 GB via 32-bit pointers [S1, unofficial].
    *   Close-range inner/leaf nodes: Addressable within 16 KB / 64 KB respectively using 16-bit pointers [S1, unofficial].
*   **Performance:** Traversal requires computing variable offsets for child pointers, which is handled via a manually unrolled loop in the shader [S1, unofficial]. This adds minor computational overhead compared to fixed-offset access but is necessary for the compact layout [S1, unofficial].
*   **Strengths:** Lossless compression, GPU-friendly traversal (no decompression), efficient handling of symmetric structures [S1, unofficial].
*   **Limitations:** Complex pointer decoding logic; requires careful management of transformation states during traversal [S1, unofficial].

## Background

The SSVDAG node layout and tagged pointer encoding was introduced in the paper "SSVDAGs: Symmetry-aware Sparse Voxel DAGs" by CRS4 [S1, unofficial]. It builds upon the memory access patterns established in standard SVDAGs by Kämpe et al. (2013) [S1, unofficial]. The method addresses memory bottlenecks in high-resolution volume rendering by compressing pointer data through variable-length encoding and symmetry-aware transformation tags [S1, unofficial].

## Sources

[S1] PDF SSVDAGs: Symmetry-aware Sparse Voxel DAGs - CRS4, https://www.crs4.it/vic/data/papers/i3d2016-symmetry-dags.pdf, fetched 2026-10-27 (unofficial)
