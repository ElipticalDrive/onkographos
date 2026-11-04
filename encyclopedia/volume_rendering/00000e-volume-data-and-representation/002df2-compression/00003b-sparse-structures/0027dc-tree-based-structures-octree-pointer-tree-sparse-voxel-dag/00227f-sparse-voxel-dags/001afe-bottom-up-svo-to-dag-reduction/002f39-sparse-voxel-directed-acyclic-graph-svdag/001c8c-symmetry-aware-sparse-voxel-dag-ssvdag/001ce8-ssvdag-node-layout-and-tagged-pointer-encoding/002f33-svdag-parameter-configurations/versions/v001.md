# SVDAG Parameter Configurations

Also called: SVDAG Configurations, SVDAG1-6

## Overview
SVDAG Parameter Configurations define the specific bit-widths for Child Node Masks (CHNM), Child Node Pointers (PT), and Addressing Words (addr. word) within a Symmetry-aware Sparse Voxel DAG (SSVDAG) node layout [S1]. They solve the problem of optimizing memory footprint versus addressable space in hierarchical data structures by allowing variable-length pointers [S1]. Unlike the base SVO configuration which uses fixed 32-bit pointers, these configurations (SVDAG1–6) enable the use of shorter pointers (8-bit or 16-bit) for nodes located closer to the root, significantly reducing memory consumption for large scenes while maintaining the ability to address distant nodes via longer pointers [S1].

## How it works
The method configures the binary layout of SSVDAG nodes. The core inputs are the desired scene resolution and the target balance between memory size and addressable space. The output is a specific configuration (SVDAG1–6) that dictates the bit-lengths of the node components.

### Data Structures and Layouts
Each configuration defines the size of three key fields within the node header and pointer structure [S1]:
1.  **CHNM (Child Node Mask):** A bitmask indicating which of the 8 suboctants are active.
2.  **PT (Child Node Pointer):** The pointer to the child node. In configurations with multiple pointer types, different bits are used for different types.
3.  **Addressing Word:** The word size used for memory addressing calculations.

The following table lists the exact bit layouts for each configuration [S1]:

| Configuration | CHNM [bits] | PT1 [bits] | PT2 [bits] | PT3 [bits] | Addressing Word [bits] | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **SVO** | 8 (aligned to 32) | 32 | - | - | 32 | Baseline SVO layout |
| **SVDAG1** | 8 (aligned to 32) | 32 | - | - | 32 | Identical to SVO [S1] |
| **SVDAG2** | 16 | 8 | 8 | - | 8 | Two 8-bit pointer types |
| **SVDAG3** | 16 | 8 | 16 | 32 | 8 | Three pointer types |
| **SVDAG4** | 16 | 16 | 16 | 32 | 8 | Three pointer types |
| **SVDAG5** | 16 | 16 | 16 | 32 | 16 | Three pointer types |
| **SVDAG6** | 16 | 16 | 16 | 32 | 32 | Three pointer types |

*Note: For SVO and SVDAG1, the 8-bit CHNM is aligned to 32 bits using 24 reserved bits [S1].*

### Addressable Space
The choice of Addressing Word and Pointer Length determines the maximum addressable memory space [S1]. The relationship is defined by the formula:
$$ \text{Addressable Space} = \text{Addressing Word Size} \times 2^{\text{Pointer Length}} $$

The specific addressable spaces for the tested configurations are [S1]:

| Pointer Length | Addressing Word 8b | Addressing Word 16b | Addressing Word 32b |
| :--- | :--- | :--- | :--- |
| **8b** | 256 B | 64 KB | 4 GB |
| **16b** | 512 B | 128 KB | 8 GB |
| **32b** | 1024 B | 256 KB | 16 GB |

### Implementation Logic
When encoding a node, the system selects the shortest pointer type (PT1, PT2, or PT3) that can address the target child node's location within the addressable space defined by the configuration's addressing word [S1].

1.  **Calculate Offset:** Determine the byte offset of the child node from the current node's base address.
2.  **Check PT1:** If the offset fits within the addressable space of PT1 (e.g., 256 B for 8b pointer/8b word), use PT1.
3.  **Check PT2:** If not, check if it fits within PT2's space (e.g., 64 KB for 16b pointer/8b word). If yes, use PT2.
4.  **Use PT3:** If neither, use PT3 (32b), which typically addresses the largest space (e.g., 4 GB or 16 GB depending on the addressing word).

### Edge Cases and Pitfalls
*   **Alignment:** In SVO and SVDAG1, the 8-bit CHNM must be padded with 24 reserved bits to align to a 32-bit boundary [S1]. Failure to do so causes misalignment in subsequent fields.
*   **Pointer Type Mismatch:** In configurations like SVDAG2, PT1 and PT2 are both 8 bits. The distinction between them is handled by the Header Tags (HT) in the parent node's mask or a separate tag field, not by pointer length [S1].
*   **Addressing Word Limitation:** Using an 8-bit addressing word limits the total addressable space significantly compared to 32-bit addressing, even if 32-bit pointers are used. For example, SVDAG6 uses 32-bit pointers but an 8-bit addressing word, limiting its addressable space to 4 GB, whereas SVDAG6 with a 32-bit addressing word would address 16 GB [S1].

## Subtopics

### [Variable-Length Child Node Pointers](5948-variable-length-child-node-pointers.md)
A technique in Sparse Voxel DAGs (SVDAG) where child node pointers (PT) have varying bit lengths (8b, 16b, or 32b) to optimize address space usage based on scene size and detail requirements.

### [SVDAG with 2b Header Tags](5971-svdag-with-2b-header-tags.md)
An optimization where the Child Node Mask (CHNM) Header Tags (HT) are 2 bits long instead of 1 bit. This allows four distinct codes (00, 01, 10, 11) to distinguish between passive suboctants and three different types of active child node pointers (PT1, PT2, PT3), enabling compression by using shorter pointers for closer nodes.

## Operating characteristics
*   **Memory Efficiency:** SVDAG configurations significantly reduce memory usage compared to SVO and SVDAG1. For the Angel Lucy model at 4096³ resolution, SVO size is 59,581.0 KB, while SVDAG3 achieves the lowest size at 7,637.6 KB [S1].
*   **Compression Ratio:** Compared to a 1-bit-per-voxel grid, SVDAG configurations achieve a compression ratio of approximately 16.1 for Angel Lucy at 4096³ [S1].
*   **Success Rate:** SVDAG3 shows the highest compression success rate (1.95x better than SVDAG1) for Angel Lucy at 4096³ [S1].
*   **Pointer Distribution:** In SVDAG2 at 4096³, PT1 (8b) accounts for 51.13% of pointers, and PT2 (8b) accounts for 77.99% (Note: The source table lists PT2 count for SVDAG2 as 77.99%, but this seems to refer to the cumulative or specific type distribution in SVDAG3; for SVDAG2, PT1 is 51.13% and PT2 is the remaining active pointers, though the exact PT2 percentage for SVDAG2 is not explicitly separated from PT3 in the provided text snippet, SVDAG3 PT1 is 51.11% and PT2 is 20.53% [S1]).
*   **Hardware:** Tests were conducted on an AMD Ryzen 7 7435HS, nVidia GeForce RTX3060, 12 GB RAM [S1].

## Background
The SVDAG parameter configurations were introduced in the context of optimizing Sparse Voxel Directed Acyclic Graphs for efficient geometry representation of voxelized 3D scenes [S1]. The configurations SVDAG1–6 were tested on three polygonal surface models: Angel Lucy (489k triangles), Porsche (22k triangles), and Skull (80k triangles), voxelized at resolutions from 128³ to 4096³ [S1]. The study aimed to determine the optimal balance between pointer length, addressing word size, and memory footprint [S1].

## Sources
[S1] Parameter Optimization in Sparse Voxel DAGs for Efficient Geometry Representation of Voxelized 3D Scenes, https://doi.org/10.12700/aph.22.7.2025.7.11, fetched 2026-10-27
