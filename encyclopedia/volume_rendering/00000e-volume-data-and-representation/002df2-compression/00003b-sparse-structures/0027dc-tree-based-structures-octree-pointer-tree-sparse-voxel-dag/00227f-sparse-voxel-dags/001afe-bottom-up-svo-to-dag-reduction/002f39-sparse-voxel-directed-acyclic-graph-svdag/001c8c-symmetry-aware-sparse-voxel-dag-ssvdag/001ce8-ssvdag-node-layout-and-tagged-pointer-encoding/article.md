# SSVDAG Node Layout and Tagged Pointer Encoding

Also called: Compact representation, Tagged pointer encoding

## Overview
This method defines the memory layout of inner nodes in a Symmetry-aware Sparse Voxel Directed Acyclic Graph (SSVDAG) to minimize memory consumption while supporting GPU-friendly traversal [S1, unofficial]. It replaces the fixed-size child pointers of standard SVDAGs with variable-length tagged pointers (2 or 4 bytes) that embed transformation codes, allowing the representation of symmetry-related subtrees with reduced overhead [S1, unofficial]. Unlike standard SVDAGs which use only 0 or 4-byte pointers, this layout introduces 2-byte pointers for frequent, close-range references, significantly compressing the data structure for high-resolution scenes [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Input:** A Symmetry-aware Sparse Voxel Directed Acyclic Graph (SSVDAG) structure, specifically the inner-level nodes array.
*   **Output:** A compact binary layout for inner nodes where each node consists of a 16-bit header followed by variable-length child pointer data.

### Data Structures

#### Inner-Level Node Header
Each inner node begins with a 16-bit header containing 8 entries, one for each potential child (octant 0–7). Each entry is a 2-bit code:
*   `00`: Null pointer (child does not exist). No data follows for this child.
*   `01`: 16-bit child pointer.
*   `10`: 32-bit child pointer.
*   `11`: 32-bit child pointer.

#### Child Pointer Encoding
The format of the child pointer data depends on the header tag. All pointers encode a 3-bit transformation code (reflection state) and an offset.

**Tag 01 (16-bit Pointer):**
*   **Size:** 2 bytes (16 bits).
*   **Layout:**
    *   Bits 15–13 (3 MSBs): Transformation code (reflection tag).
    *   Bits 12–0 (13 LSBs): Offset in number of level-words from the start of the next level's array.
*   **Addressing Range:**
    *   For inner levels: Level-word size is 2 bytes. Max offset $2^{13}$ words $\times$ 2 bytes/word = $2^{14}$ bytes (16 KB).
    *   For leaf levels: Level-word size is 8 bytes. Max offset $2^{13}$ words $\times$ 8 bytes/word = $2^{16}$ bytes (64 KB).

**Tags 10 and 11 (32-bit Pointer):**
*   **Size:** 4 bytes (32 bits).
*   **Layout:**
    *   Bits 31–29 (3 MSBs): Transformation code (reflection tag).
    *   Bits 28–0 (29 LSBs): Lowest 29 bits of the offset.
    *   Bit 29 (Highest bit of offset): Set to the value of the rightmost bit of the header tag (bit 1 of the 2-bit header code).
        *   If header tag is `10` (binary 2), rightmost bit is 0. Offset MSB = 0.
        *   If header tag is `11` (binary 3), rightmost bit is 1. Offset MSB = 1.
*   **Addressing Range:**
    *   For inner levels: Max offset $\approx 2^{29}$ words $\times$ 2 bytes/word > 2 GB.
    *   For leaf levels: Max offset $\approx 2^{29}$ words $\times$ 8 bytes/word > 8 GB.

### Algorithm: Accessing the i-th Child Pointer

To access the $i$-th child pointer (where $i \in [0, 7]$) of a node stored at memory address `node_addr`:

1.  **Read Header:** Load the 16-bit header from `node_addr`.
2.  **Compute Offset:** Calculate the byte offset `ptr_offset` within the node data (immediately following the header) where the $i$-th pointer begins. This is the sum of the sizes of all pointers for children $j < i$.
    *   For each child $j < i$, read the 2-bit code from the header.
    *   If code is `00`, size is 0.
    *   If code is `01`, size is 2.
    *   If code is `10` or `11`, size is 4.
    *   `ptr_offset = sum(sizes for j in 0..i-1)`
3.  **Read Pointer Data:** Read `size` bytes from `node_addr + 16 bits + ptr_offset`.
4.  **Decode Transformation:** Extract the 3 MSBs of the pointer data. This is the reflection tag for the child subtree.
5.  **Decode Offset:**
    *   If tag is `01`: Extract bits 12–0. This is the word offset.
    *   If tag is `10` or `11`: Extract bits 28–0. Combine with the MSB (derived from header tag) to form the full 30-bit offset.
6.  **Calculate Child Address:**
    *   Determine the base address of the next level's array (provided by the indexing structure).
    *   Convert word offset to byte offset: `byte_offset = word_offset * level_word_size`.
        *   `level_word_size` is 2 bytes for inner levels, 8 bytes for leaf levels.
    *   `child_addr = next_level_base + byte_offset`.

### Traversal State Management

During ray traversal, the transformation status must be maintained to correctly interpret indices and voxel contents:

1.  **Initialization:** Set `transform_status = 0` at the root.
2.  **Descent:** When moving to a child node:
    *   Extract the reflection tag `t` from the child pointer.
    *   Update `transform_status = transform_status XOR t`.
    *   Push `transform_status` to the traversal stack along with the current node context.
3.  **Backtracking:** When returning from a child, pop the stack to restore the previous `transform_status`.
4.  **Application:**
    *   For inner nodes: Apply `transform_status` to the indices used to access child pointers if the traversal logic requires coordinate transformation.
    *   For leaf nodes: Apply `transform_status` to the voxel content indices or attributes.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_child_pointer(node_header, node_data_base, child_index):
    """
    Extracts the child pointer and transformation tag for a specific child index.
    
    Args:
        node_header (int): 16-bit integer containing 8 x 2-bit codes.
        node_data_base (int): Memory address of the data following the header.
        child_index (int): Index of the child (0-7).
        
    Returns:
        tuple: (pointer_data_bytes, transformation_tag, pointer_size)
    """
    # Extract the 2-bit code for the requested child
    # Header bits: [7:6] [5:4] [3:2] [1:0] for children 0,1,2,3? 
    # Note: Source does not specify bit order (MSB/LSB) for children 0-7.
    # Assuming standard packing: Child 0 is bits 15-14, Child 1 is 13-12, etc.
    shift = 2 * (7 - child_index)
    code = (node_header >> shift) & 0x3
    
    if code == 0x0: # Null
        return (None, None, 0)
    
    # Calculate offset to this child's pointer data
    # Sum sizes of all previous children (0 to child_index-1)
    offset = 0
    for j in range(child_index):
        prev_shift = 2 * (7 - j)
        prev_code = (node_header >> prev_shift) & 0x3
        if prev_code == 0x0:
            size = 0
        elif prev_code == 0x1:
            size = 2
        else: # 0x2 or 0x3
            size = 4
        offset += size
        
    # Read pointer bytes
    ptr_addr = node_data_base + offset
    if code == 0x1: # 16-bit
        ptr_bytes = read_bytes(ptr_addr, 2)
        # Transform tag is top 3 bits of 16-bit value
        # Note: ptr_bytes is 2 bytes. Top 3 bits of the 16-bit integer.
        ptr_val = (ptr_bytes[0] << 8) | ptr_bytes[1]
        transform_tag = (ptr_val >> 13) & 0x7
        return (ptr_bytes, transform_tag, 2)
    else: # 32-bit (code 0x2 or 0x3)
        ptr_bytes = read_bytes(ptr_addr, 4)
        ptr_val = (ptr_bytes[0] << 24) | (ptr_bytes[1] << 16) | (ptr_bytes[2] << 8) | ptr_bytes[3]
        transform_tag = (ptr_val >> 29) & 0x7
        return (ptr_bytes, transform_tag, 4)

def compute_child_address(transform_tag, pointer_bytes, pointer_size, next_level_base, level_word_size):
    """
    Computes the absolute address of the child node.
    
    Args:
        transform_tag (int): 3-bit reflection tag.
        pointer_bytes (bytes): The raw bytes of the pointer.
        pointer_size (int): 2 or 4.
        next_level_base (int): Base address of the next level's array.
        level_word_size (int): 2 for inner, 8 for leaf.
        
    Returns:
        int: Absolute address of the child node.
    """
    if pointer_size == 2:
        # 16-bit pointer
        ptr_val = (pointer_bytes[0] << 8) | pointer_bytes[1]
        # Offset is lower 13 bits
        word_offset = ptr_val & 0x1FFF
    else:
        # 32-bit pointer
        ptr_val = (pointer_bytes[0] << 24) | (pointer_bytes[1] << 16) | (pointer_bytes[2] << 8) | pointer_bytes[3]
        # Lower 29 bits
        low_offset = ptr_val & 0x1FFFFFF
        # MSB of offset is the rightmost bit of the header tag.
        # However, the header tag is not passed here. 
        # The source says: "Highest bit of the offset is set to the rightmost bit of the header tag."
        # This implies the MSB (bit 29) is reconstructed from the header.
        # Since we don't have the header here, we must assume the caller provides the full offset
        # or reconstructs it. For this function, we assume the offset is fully contained in the pointer
        # if we had the MSB. But the source says the MSB is *set* to the header bit.
        # So the pointer itself only stores 29 bits. We need the header bit.
        # Let's adjust the signature to include the header bit for MSB.
        pass

def compute_child_address_with_header(transform_tag, pointer_bytes, pointer_size, next_level_base, level_word_size, header_tag_bit):
    """
    Computes the absolute address of the child node, reconstructing the full offset.
    
    Args:
        transform_tag (int): 3-bit reflection tag.
        pointer_bytes (bytes): The raw bytes of the pointer.
        pointer_size (int): 2 or 4.
        next_level_base (int): Base address of the next level's array.
        level_word_size (int): 2 for inner, 8 for leaf.
        header_tag_bit (int): The rightmost bit of the 2-bit header tag (0 or 1). Used for MSB of 32-bit offset.
        
    Returns:
        int: Absolute address of the child node.
    """
    if pointer_size == 2:
        ptr_val = (pointer_bytes[0] << 8) | pointer_bytes[1]
        word_offset = ptr_val & 0x1FFF
    else:
        ptr_val = (pointer_bytes[0] << 24) | (pointer_bytes[1] << 16) | (pointer_bytes[2] << 8) | pointer_bytes[3]
        low_offset = ptr_val & 0x1FFFFFF
        # Reconstruct MSB
        msb = header_tag_bit
        word_offset = (msb << 29) | low_offset
        
    byte_offset = word_offset * level_word_size
    return next_level_base + byte_offset
```

### Edge Cases and Pitfalls
*   **Variable Size Access:** Accessing the $i$-th child requires summing the sizes of all preceding children ($0$ to $i-1$). Since sizes can be 0, 2, or 4 bytes, this differs from standard SVDAGs where sizes are only 0 or 4. This computation must be performed in the shader, typically via a manually unrolled loop [S1, unofficial].
*   **Offset Reconstruction:** For 32-bit pointers, the most significant bit of the offset is not stored in the pointer data but is derived from the header tag. Failure to correctly extract and apply this bit will result in incorrect address calculation [S1, unofficial].
*   **Transformation Propagation:** The transformation tag extracted from a pointer applies to the entire subtree. The traversal state must be correctly maintained (XOR update and stack push/pop) to ensure correct coordinate mapping for nested symmetric structures [S1, unofficial].

## Subtopics

### [Compact SSVDAG Encoding](5227-compact-ssvdag-encoding.md)
Compact SSVDAG Encoding is a GPU-friendly memory layout for Symmetry-aware Sparse Voxel Directed Acyclic Graphs that minimizes pointer overhead while enabling fast, decompression-free ray tracing. It distinguishes itself from standard SVDAGs by combining leaf grouping, frequency-based pointer compaction, and half-word memory alignment to reduce the memory footprint of high-resolution scenes. The technique relies on highly skewed reference distributions, where in a 64K³ Powerplant dataset, the most common 10% of nodes account for nearly 90% of pointers at level 14 and nearly 50% at level 11.

### [SVDAG Parameter Configurations](5945-svdag-parameter-configurations.md)
SVDAG Parameter Configurations define the specific bit-widths for Child Node Masks (CHNM), Child Node Pointers (PT), and Addressing Words (addr. word) within a Symmetry-aware Sparse Voxel DAG (SSVDAG) node layout. They solve the problem of optimizing memory footprint versus addressable space in hierarchical data structures by allowing variable-length pointers. Unlike the base SVO configuration which uses fixed 32-bit pointers, these configurations (SVDAG1–6) enable the use of shorter pointers (8-bit or 16-bit) for nodes located closer to the root, significantly reducing memory consumption for large scenes while maintaining the ability to address distant nodes via longer pointers.

## Operating characteristics

*   **Memory Footprint:** Significantly reduced compared to fixed-pointer SVDAGs. By using 2-byte pointers for frequent, close-range references (Tag 01), the average pointer size is lowered. The distribution of pointer usage is highly skewed, with the most common 10% of nodes accounting for nearly 90% of pointers at level 14 in high-resolution scenes like the Powerplant dataset [S1, unofficial].
*   **Addressable Space:**
    *   Inner levels: >2 GB via 32-bit pointers.
    *   Leaf levels: >8 GB via 32-bit pointers.
    *   Close-range inner/leaf nodes: Addressable within 16 KB / 64 KB respectively using 16-bit pointers.
*   **Performance:** Traversal requires computing variable offsets for child pointers, which is handled via a manually unrolled loop in the shader. This adds minor computational overhead compared to fixed-offset access but is necessary for the compact layout.
*   **Strengths:** Lossless compression, GPU-friendly traversal (no decompression), efficient handling of symmetric structures.
*   **Limitations:** Complex pointer decoding logic; requires careful management of transformation states during traversal.

## Background

The SSVDAG node layout and tagged pointer encoding was introduced in the paper "SSVDAGs: Symmetry-aware Sparse Voxel DAGs" by CRS4 [S1, unofficial]. It extends the Sparse Voxel DAG (SVDAG) structure by Kämpe et al. (2013) [S1, unofficial] by incorporating symmetry awareness and variable-length pointers. The method was developed to address the memory bottleneck in high-resolution volume rendering, particularly for large datasets like the Powerplant, Boeing 777, Lucy, and David 1mm models [S1, unofficial].

## Sources

[S1] PDF SSVDAGs: Symmetry-aware Sparse Voxel DAGs - CRS4, https://www.crs4.it/vic/data/papers/i3d2016-symmetry-dags.pdf, fetched 2026-10-27 (unofficial)
