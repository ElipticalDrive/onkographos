# SVDAG Parameter Configurations

Also called: SVDAG Configurations, SVDAG1-6

## Overview
SVDAG Parameter Configurations define the specific bit-widths for Child Node Masks (CHNM), Child Node Pointers (PT), and Addressing Words within a Symmetry-aware Sparse Voxel DAG (SSVDAG) node layout [S1]. They optimize the trade-off between memory footprint and addressable space in hierarchical data structures by allowing variable-length pointers [S1]. Unlike the base SVO configuration, which uses a single fixed 32-bit pointer type, configurations SVDAG2–SVDAG6 introduce multiple pointer types (PT1, PT2, PT3) with lengths of 8, 16, or 32 bits to reduce memory consumption for large scenes while maintaining addressability [S1]. SVDAG1 remains identical to the base SVO configuration [S1].

## How it works
The method selects a configuration based on scene resolution and the target balance between memory size and addressable space [S1]. The output is a defined layout specifying the bit-lengths of the CHNM, PT types, and addressing word [S1].

### Data Structures and Layouts
Each configuration defines the size of three key fields within the node header and pointer structure [S1]:
1. **CHNM (Child Node Mask):** A bitmask indicating which of the 8 suboctants are active.
2. **PT (Child Node Pointer):** The pointer to the child node. Configurations SVDAG2–SVDAG6 support up to three distinct pointer types (PT1, PT2, PT3) with varying bit lengths [S1].
3. **Addressing Word:** The word size used for memory addressing calculations [S1].

The exact bit layouts for each configuration are [S1]:
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

### Formulas and Addressable Space
The choice of Addressing Word and Pointer Length determines the maximum addressable memory space [S1]. The relationship is defined by the formula:
$$ \text{Addressable Space} = W \times 2^P $$
where $W$ is the addressing word size in bytes, and $P$ is the pointer length in bits [S1]. The specific addressable spaces for the tested configurations are [S1]:
| Pointer Length ($P$) | Addressing Word 8b ($W=1$) | Addressing Word 16b ($W=2$) | Addressing Word 32b ($W=4$) |
| :--- | :--- | :--- | :--- |
| **8b** | 256 B | 64 KB | 4 GB |
| **16b** | 512 B | 128 KB | 8 GB |
| **32b** | 1024 B | 256 KB | 16 GB |

### Implementation Steps
1. Select a configuration (SVO, SVDAG1–6) based on the scene's voxel resolution and the desired compression ratio [S1].
2. Allocate node memory according to the CHNM size, followed by the variable-length PT fields for each active suboctant [S1].
3. Align the node layout to the addressing word boundary using reserved bits if necessary [S1].
4. Encode child node offsets using the selected PT type that fits within the addressable space defined by $W$ and $P$ [S1]. (The exact runtime selection algorithm for PT1/PT2/PT3 is not detailed in the sources [G].)

### Edge Cases and Pitfalls
*   **Alignment Padding:** In SVO and SVDAG1, the 8-bit CHNM must be padded with 24 reserved bits to align to a 32-bit boundary [S1]. Failure to apply this padding causes misalignment in subsequent fields [S1].
*   **Pointer Type Differentiation:** In configurations like SVDAG2, PT1 and PT2 share the same 8-bit length [S1]. The distinction between them is handled by the Header Tags (HT) in the parent node's mask [S1].
*   **Addressing Word Limitation:** The addressing word size strictly caps the addressable space regardless of pointer length [S1]. For example, an 8-bit addressing word limits the maximum addressable space to 1024 B (1 KB), even when 32-bit pointers are used [S1].

## Subtopics

### [Variable-Length Child Node Pointers](5948-variable-length-child-node-pointers.md)
A technique in Sparse Voxel DAGs (SVDAG) where child node pointers (PT) have varying bit lengths (8b, 16b, or 32b) to optimize address space usage based on scene size and detail requirements.

### [SVDAG with 2b Header Tags](5971-svdag-with-2b-header-tags.md)
An optimization where the Child Node Mask (CHNM) Header Tags (HT) are 2 bits long instead of 1 bit. This allows four distinct codes (00, 01, 10, 11) to distinguish between passive suboctants and three different types of active child node pointers (PT1, PT2, PT3), enabling compression by using shorter pointers for closer nodes.

## Operating characteristics
*   **Memory Efficiency:** SVDAG configurations significantly reduce memory usage compared to SVO and SVDAG1 [S1]. For the Angel Lucy model at 4096³ resolution, SVO size is 59,581.0 KB, while SVDAG3 achieves the lowest size at 7,637.6 KB [S1].
*   **Compression Ratio:** Compared to a 1-bit-per-voxel grid, SVDAG1–SVDAG6 achieve a compression ratio of 16.1 for Angel Lucy at 4096³ [S1].
*   **Success Rate:** SVDAG3 shows the highest compression success rate (1.95x better than SVDAG1) for Angel Lucy at 4096³ [S1]. SVDAG2, SVDAG4, SVDAG5, and SVDAG6 achieve success rates of 1.71, 1.67, 1.68, and 1.63 respectively [S1].
*   **Pointer Distribution:** In SVDAG2 at 4096³, PT1 (8b) accounts for 51.13% of pointers [S1].
*   **Hardware:** Tests were conducted on an AMD Ryzen 7 7435HS (3.1 GHz, octacore), nVidia GeForce RTX3060, 12 GB RAM DDR6, 512 GB SSD, running Win 11 Home [S1].

## Background
The SVDAG parameter configurations were introduced to optimize Sparse Voxel Directed Acyclic Graphs for efficient geometry representation of voxelized 3D scenes [S1]. The configurations SVDAG1–6 were tested on three polygonal surface models: Angel Lucy (489k triangles), Porsche (22k triangles), and Skull (80k triangles), voxelized at resolutions from 128³ to 4096³ [S1]. The study aimed to determine the optimal balance between pointer length, addressing word size, and memory footprint across varying scene densities [S1].

## Sources
[S1] Parameter Optimization in Sparse Voxel DAGs for Efficient Geometry Representation of Voxelized 3D Scenes, https://doi.org/10.12700/aph.22.7.2025.7.11, fetched 2026-10-27
