# SVDAG Parameter Configurations

Also called: SVDAG Configurations, SVDAG1-6

## Overview
SVDAG Parameter Configurations define the binary layout of nodes in a Sparse Voxel Directed Acyclic Graph (SVDAG) by specifying the bit-lengths of the Child Node Mask (CHNM), child node pointers (PT), and the addressing word size [S1]. These configurations determine the trade-off between memory footprint and addressable space, allowing the data structure to be tailored to the specific density and resolution of the voxelized scene [S1]. Unlike fixed-size node layouts, these variants enable variable-length pointer encoding to minimize binary overhead, with SVDAG2 through SVDAG6 offering compressed representations compared to the baseline SVO and SVDAG1 [S1].

## How it works
The method defines six distinct node layouts (SVO, SVDAG1–SVDAG6) used to store hierarchical volume data [S1]. The layout is determined by three parameters: the Child Node Mask length ($L_{CHNM}$), the lengths of up to three distinct child pointer types ($L_{PT1}, L_{PT2}, L_{PT3}$), and the addressing word length ($L_{addr}$) [S1]. The addressing word size dictates the granularity of pointer offsets and the maximum addressable space [S1].

### Data Structures
Each node configuration is defined by the following fields [S1]:
1. **Child Node Mask (CHNM)**: A bitmask indicating which of the 8 suboctants are active.
    * **SVO/SVDAG1**: 8 bits, aligned to 32 bits using 24 reserved bits.
    * **SVDAG2–SVDAG6**: 16 bits.
2. **Child Pointers (PT)**: Variable-length pointers to child nodes. The number of types and their lengths vary by configuration.
    * **SVO/SVDAG1**: Single type, 32 bits.
    * **SVDAG2**: Two types: PT1 (8 bits), PT2 (8 bits).
    * **SVDAG3**: Three types: PT1 (8 bits), PT2 (16 bits), PT3 (32 bits).
    * **SVDAG4**: Three types: PT1 (16 bits), PT2 (16 bits), PT3 (32 bits).
    * **SVDAG5**: Three types: PT1 (16 bits), PT2 (16 bits), PT3 (32 bits).
    * **SVDAG6**: Three types: PT1 (16 bits), PT2 (16 bits), PT3 (32 bits).
3. **Addressing Word**: The unit size used for pointer arithmetic and alignment.
    * **SVO/SVDAG1**: 32 bits.
    * **SVDAG2–SVDAG4**: 8 bits.
    * **SVDAG5**: 16 bits.
    * **SVDAG6**: 32 bits.

### Addressable Space
The maximum addressable space is determined by the addressing word length and the pointer length [S1]. The relationship is defined by the table below [S1]:

| Pointer Length ($L_{PT}$) | Addressing Word ($L_{addr}$) | Addressable Space |
| :--- | :--- | :--- |
| 8 bits | 8 bits | 256 Bytes |
| 8 bits | 16 bits | 64 KB |
| 8 bits | 32 bits | 4 GB |
| 16 bits | 8 bits | 512 Bytes |
| 16 bits | 16 bits | 128 KB |
| 16 bits | 32 bits | 8 GB |
| 32 bits | 8 bits | 1024 Bytes |
| 32 bits | 16 bits | 256 KB |
| 32 bits | 32 bits | 16 GB |

### Configuration Details
The specific parameters for each configuration are as follows [S1]:
* **SVO**: $L_{CHNM}=8$ (aligned to 32), $L_{PT1}=32$, $L_{addr}=32$.
* **SVDAG1**: Identical to SVO. Based on reference [17].
* **SVDAG2**: $L_{CHNM}=16$, $L_{PT1}=8$, $L_{PT2}=8$, $L_{addr}=8$.
* **SVDAG3**: $L_{CHNM}=16$, $L_{PT1}=8$, $L_{PT2}=16$, $L_{PT3}=32$, $L_{addr}=8$.
* **SVDAG4**: $L_{CHNM}=16$, $L_{PT1}=16$, $L_{PT2}=16$, $L_{PT3}=32$, $L_{addr}=8$.
* **SVDAG5**: $L_{CHNM}=16$, $L_{PT1}=16$, $L_{PT2}=16$, $L_{PT3}=32$, $L_{addr}=16$.
* **SVDAG6**: $L_{CHNM}=16$, $L_{PT1}=16$, $L_{PT2}=16$, $L_{PT3}=32$, $L_{addr}=32$.

### Implementation Notes
* **Alignment**: For SVO and SVDAG1, the 8-bit CHNM is padded with 24 reserved bits to achieve 32-bit alignment [S1].
* **Pointer Types**: SVDAG2 through SVDAG6 utilize a 16-bit CHNM to permit three distinct child node pointer types and varying addressing word lengths [S1].
* **Edge Cases**: An 8-bit pointer length is considered impractical for very small 3D scenes due to limited addressable space (256B to 1024B depending on addressing word) [S1].

## Subtopics

### [Variable-Length Child Node Pointers](5948-variable-length-child-node-pointers.md)
Uses varying bit lengths (8b, 16b, 32b) for child pointers to optimize compression ratios based on scene size, at the cost of increased traversal complexity due to mixed alignment.

### [SVDAG with 2b Header Tags](5971-svdag-with-2b-header-tags.md)
Replaces the 1-bit Child Node Mask with 2-bit Header Tags to distinguish between passive suboctants and three active pointer types, enabling variable-length encoding.

### [Extra Child Pointer Encoding](15412-extra-child-pointer-encoding.md)
A steganographic technique that hides data in unused slots of the child pointer array (where the mask count $N < 8$) without affecting traversal logic.

## Operating characteristics
Performance was measured on an AMD Ryzen 7 7435HS (3.1 GHz), nVidia GeForce RTX3060, 12 GB RAM, Win 11 Home [S1]. Three datasets were used: Angel Lucy (489k triangles), Porsche (22k triangles), and Skull (80k triangles), voxelized at resolutions $128^3$ to $4096^3$ [S1].

### Memory Footprint
For the Angel Lucy model at $4096^3$ resolution, the binary sizes (in KB) are [S1]:
* **SVO**: 59,581.0 KB
* **SVDAG1**: 14,928.3 KB
* **SVDAG2**: 8,731.5 KB
* **SVDAG3**: 7,637.6 KB
* **SVDAG4**: 8,934.0 KB
* **SVDAG5**: 8,864.9 KB
* **SVDAG6**: 9,138.0 KB

### Compression Ratio
Compression ratio is defined relative to a 1-bit-per-voxel grid [S1]. For Angel Lucy at $4096^3$:
* **SVO**: 140.8
* **SVDAG1–SVDAG6**: 16.1 (all variants achieve the same ratio at this resolution) [S1].

### Compression Success Rate
This metric compares the compression success rate of each variant against SVDAG1 [S1]. For Angel Lucy at $4096^3$:
* **SVDAG2**: 1.71
* **SVDAG3**: 1.95
* **SVDAG4**: 1.67
* **SVDAG5**: 1.68
* **SVDAG6**: 1.63

### Pointer Type Proportions
For Angel Lucy at $4096^3$ using SVDAG2, the proportion of pointers by type is [S1]:
* **PT1 (8b)**: 51.13%
* **PT2 (8b)**: 77.99%

### Strengths and Limitations
* **Strengths**: SVDAG2–SVDAG6 significantly reduce memory footprint compared to SVO and SVDAG1. SVDAG3 achieves the smallest size (7,637.6 KB) for Angel Lucy $4096^3$ [S1].
* **Limitations**: Smaller addressing words (8b) limit the addressable space, which may be insufficient for very large scenes or deep hierarchies [S1].

## Background
The SVDAG parameter configurations were introduced to optimize the memory representation of voxelized 3D scenes using Sparse Voxel Directed Acyclic Graphs [S1]. The baseline SVO and SVDAG1 configurations follow the layout defined in reference [17] [S1]. The subsequent variants (SVDAG2–SVDAG6) were developed to explore the trade-offs between pointer length, addressing word size, and compression efficiency [S1].

## Sources
[S1] Parameter Optimization in Sparse Voxel DAGs for Efficient Geometry Representation of Voxelized 3D Scenes, https://doi.org/10.12700/aph.22.7.2025.7.11, fetched 2026-10-27
