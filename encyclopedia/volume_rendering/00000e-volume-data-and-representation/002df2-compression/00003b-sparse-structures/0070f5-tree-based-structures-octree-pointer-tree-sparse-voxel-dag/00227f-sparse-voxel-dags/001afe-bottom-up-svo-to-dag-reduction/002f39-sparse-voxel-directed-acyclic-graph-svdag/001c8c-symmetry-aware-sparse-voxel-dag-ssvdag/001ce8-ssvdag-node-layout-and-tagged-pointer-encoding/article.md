# SSVDAG Node Layout and Tagged Pointer Encoding

Also called: Compact representation, Tagged pointer encoding

## Overview
This method defines the memory layout and pointer encoding scheme for Symmetry-aware Sparse Voxel Directed Acyclic Graphs (SSVDAGs) to minimize memory consumption while maintaining GPU-friendly traversal performance [S1, unofficial]. It replaces fixed-size child pointers with variable-length tagged pointers (2 or 4 bytes) that embed transformation codes, addressing a key limitation of standard SVDAGs where all non-null pointers consume 4 bytes [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Input:** A Sparse Voxel DAG structure where nodes may share subtrees via symmetry transforms (planar reflections) [S1, unofficial].
*   **Output:** A compact binary layout for inner-level nodes consisting of a 16-bit header followed by variable-length child pointer data [S1, unofficial].

### Data Structures

#### Inner-Level Node Header
Each inner-level node begins with a 16-bit header [S1, unofficial]. This header contains 8 entries, one for each potential child (corresponding to the 8 octants) [S1, unofficial]. Each entry is a 2-bit code:
*   `00`: Null pointer (child does not exist) [S1, unofficial].
*   `01`: Child pointer is 16 bits (2 bytes) [S1, unofficial].
*   `10`: Child pointer is 32 bits (4 bytes) [S1, unofficial].
*   `11`: Child pointer is 32 bits (4 bytes) [S1, unofficial].

#### Child Pointer Encoding
The format of the child pointer data immediately following the header depends on the tag in the header [S1, unofficial]. The pointer encodes both the offset to the child node and a 3-bit transformation code [S1, unofficial].

**1. 16-bit Pointer (Tag `01`)**
*   **Size:** 2 bytes [S1, unofficial].
*   **Layout:**
    *   Bits [15:13] (3 bits): Transformation code (reflection tag) [S1, unofficial].
    *   Bits [12:0] (13 bits): Offset in number of *level-words* from the start of the next level's array [S1, unofficial].
*   **Addressing Range:**
    *   For inner levels: Level-word size is 2 bytes [S1, unofficial]. Max offset $2^{13}$ words $\times$ 2 bytes/word = $2^{14}$ bytes [S1, unofficial].
    *   For leaf levels: Level-word size is 8 bytes [S1, unofficial]. Max offset $2^{13}$ words $\times$ 8 bytes/word = $2^{16}$ bytes [S1, unofficial].

**2. 32-bit Pointer (Tags `10` and `11`)**
*   **Size:** 4 bytes [S1, unofficial].
*   **Layout:**
    *   Bits [31:29] (3 bits): Transformation code (reflection tag) [S1, unofficial].
    *   Bits [28:0] (29 bits): Lowest 29 bits of the offset [S1, unofficial].
    *   Bit [29] of the offset (the 30th bit overall) is set to the rightmost bit of the header tag [S1, unofficial].
*   **Addressing Range:**
    *   For inner levels: Max offset $\approx 2^{30}$ words $\times$ 2 bytes/word = $2^{31}$ bytes (>2 GB) [S1, unofficial].
    *   For leaf levels: Max offset $\approx 2^{30}$ words $\times$ 8 bytes/word = $2^{33}$ bytes (>8 GB) [S1, unofficial].

### Algorithm: Traversal with Tagged Pointers

To traverse the SSVDAG, the shader must compute the byte offset to the $i$-th child pointer within the node's header data [S1, unofficial]. Because pointers have variable sizes (0, 2, or 4 bytes), this requires summing the sizes of all preceding pointers [S1, unofficial]. The source notes this computation is performed in the shader using a manually unrolled loop [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_byte_offset(header, child_index):
    """
    Computes the byte offset from the start of the child pointer data 
    to the beginning of the i-th child's pointer.
    """
    offset = 0
    for i in range(child_index):
        tag = (header >> (i * 2)) & 0x3
        if tag == 0x0:
            size = 0
        elif tag == 0x1:
            size = 2
        else: # 10 or 11
            size = 4
        offset += size
    return offset

def extract_tagged_pointer(header, child_index, raw_ptr_data):
    """
    Extracts the transformation code and offset from the raw pointer data.
    """
    tag = (header >> (child_index * 2)) & 0x3
    if tag == 0x0:
        return 0, 0, 0 # Null pointer
    
    if tag == 0x1: # 16-bit pointer
        transformation = (raw_ptr_data >> 13) & 0x7
        offset = raw_ptr_data & 0x1FFF
        return offset, transformation, 2
    else: # 32-bit pointer (tags 10 or 11)
        transformation = (raw_ptr_data >> 29) & 0x7
        offset_low = raw_ptr_data & 0x1FFFFFF
        header_lsb = tag & 0x1
        offset_high = header_lsb << 29
        offset = offset_low | offset_high
        return offset, transformation, 4

# Traversal state management
transformation_status = 0 # Initialized at 0 [S1, unofficial]
# When descending to child i:
#   byte_offset = compute_byte_offset(header, i)
#   raw_ptr_data = read_memory(base_address + byte_offset, ptr_size) # [G]
#   child_offset, child_transform, ptr_size = extract_tagged_pointer(header, i, raw_ptr_data)
#   transformation_status ^= child_transform # Updated by XOR-ing with the pointer's reflection tag [S1, unofficial]
#   push(transformation_status) to stack with current node to restore on backtracking [S1, unofficial]
```

### Parameters and Edge Cases
*   **Level-Word Size:** Inner levels use 2-byte words; leaf levels use 8-byte words [S1, unofficial]. This distinction is critical for interpreting the offset in 16-bit pointers [S1, unofficial].
*   **Precision/Boundaries:** The 16-bit pointer is limited to the first 16 KB (inner) or 64 KB (leaf) of the next level [S1, unofficial]. If a shared node lies beyond this range, the 32-bit pointer must be used [S1, unofficial].
*   **Pitfall:** Accessing the $i$-th child requires computing the sum of sizes of pointers $0$ to $i-1$ [S1, unofficial]. Since sizes are 0, 2, or 4 bytes, this differs from SVDAG which only had sizes 0 and 4 [S1, unofficial].

## Subtopics

### [Compact SSVDAG Encoding](5227-compact-ssvdag-encoding.md)
A GPU-friendly memory layout that combines leaf grouping, frequency-based pointer compaction, and half-word memory alignment to minimize pointer overhead while enabling fast, decompression-free ray tracing. It distinguishes itself from standard SVDAGs by leveraging highly skewed reference distributions to reduce the memory footprint of high-resolution scenes.

### [SVDAG Parameter Configurations](5945-svdag-parameter-configurations.md)
Defines the binary layout of nodes in a Symmetry-aware Sparse Voxel Directed Acyclic Graph (SSVDAG) by specifying the bit-lengths of the Child Node Mask (CHNM), child node pointers (PT), and the addressing word size. These configurations determine the trade-off between memory footprint and addressable space, allowing the data structure to be tailored to the specific density and resolution of the voxelized scene. Unlike fixed-size node layouts, these variants enable variable-length pointer encoding to minimize binary overhead, with SVDAG2 through SVDAG6 offering compressed representations compared to the baseline SVO and SVDAG1.

## Operating characteristics
*   **Memory Footprint:** Significantly reduced compared to standard SVDAGs due to variable-length pointer encoding [S1, unofficial].
*   **Complexity:** Traversal complexity increases slightly due to the need to compute variable byte offsets for child pointers [S1, unofficial]. This is mitigated by using a manually unrolled loop in the shader [S1, unofficial].
*   **Addressing Range:** Supports addressing >2GB into inner levels and >8GB into leaves, sufficient for very large scenes [S1, unofficial].
*   **Strengths:** Lossless compression, GPU-friendly (no decompression needed), handles symmetry transforms natively [S1, unofficial].
*   **Limitations:** Limited addressing range for 16-bit pointers; requires careful handling of variable-length offsets in shaders [S1, unofficial].

## Background
The SSVDAG structure builds on the Sparse Voxel DAG (SVDAG) [Kämpe et al. 2013] and introduces variable-length tagged pointers to reduce memory consumption by encoding child pointers with 2 or 4 bytes instead of a fixed size, while storing transformation codes in the header [S1, unofficial]. The compact node layout was developed to address the memory overhead of storing transformation codes and child pointers separately [S1, unofficial].

## Sources
[S1] SSVDAGs: Symmetry-aware Sparse Voxel DAGs - CRS4, https://www.crs4.it/vic/data/papers/i3d2016-symmetry-dags.pdf, fetched 2026-10-27 (unofficial)
