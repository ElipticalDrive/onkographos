# Brick ID Page Table Layout

Also called: Brick ID structure

## Overview
The Brick ID Page Table Layout is a memory-efficient indexing scheme used within brick-based hierarchical volume storage to map spatial brick locations to their corresponding page tables in a multi-channel, multi-resolution environment [S1]. It solves the problem of efficiently identifying which page table contains the data for a specific brick without requiring large, dense lookup structures. It is distinguished from sibling methods like the ExaBricks Hierarchy by its specific bit-packing of spatial offsets and page table identifiers into fixed-width integers, and from the Sparse Page Table and Brick Pool by its explicit separation of spatial offset and page table ID within the brick ID structure [S1, 2550-sparse-page-table-and-brick-pool.md].

## How it works
**Inputs and Outputs:**
*   **Input:** Spatial coordinates of a brick $(x, y, z)$ relative to the volume origin, the number of channels $m$, and the number of resolution levels $k$ representable on the GPU.
*   **Output:** A 32-bit unsigned integer representing the Brick ID, which uniquely identifies the brick's location and its associated page table.

**Data Structures:**
The core data structure is the **Brick ID**, a 32-bit unsigned integer. Its layout is defined as follows:
*   **Bits 0–23 (24 bits):** Spatial offset. This field encodes the brick's position relative to the origin. It is subdivided into three 8-bit fields:
    *   Bits 0–7: $x$ coordinate offset.
    *   Bits 8–15: $y$ coordinate offset.
    *   Bits 16–23: $z$ coordinate offset.
*   **Bits 24–31 (8 bits):** Page Table ID. This field uniquely identifies one of the $(m \times k)$ page tables, where $m$ is the number of channels and $k$ is the number of resolution levels [S1].

**Formulas:**
The Brick ID $B$ for a brick at offset $(x, y, z)$ and page table index $P$ is calculated as:
$$ B = (P \ll 24) \ | \ (z \ll 16) \ | \ (y \ll 8) \ | \ x $$
Where:
*   $x, y, z$ are 8-bit unsigned integers representing the spatial offset.
*   $P$ is an 8-bit unsigned integer representing the page table ID.
*   $\ll$ denotes bitwise left shift.
*   $|$ denotes bitwise OR.

**Parameters:**
*   $m$ (number of channels): User-defined at initialization.
*   $k$ (number of resolution levels): User-defined at initialization.
*   Constraint: $m \times k \le 256$ [S1]. This ensures the Page Table ID fits within 8 bits.
*   Brick cache size: Implemented as a single `r8unorm` texture [S1].

**Steps:**
1.  Initialize the system with user-defined $m$ and $k$. Verify that $m \times k \le 256$.
2.  For each brick in the volume:
    a. Determine its spatial offset $(x, y, z)$ relative to the volume origin. Ensure $x, y, z$ fit within 8 bits (range 0–255).
    b. Determine the page table index $P$ associated with the brick's channel and resolution level. Ensure $P < 256$.
    c. Compute the Brick ID using the formula above.
    d. Store the Brick ID in the appropriate page table or residency structure (e.g., the residency octree nodes use 16 bits for partially mapped resolutions and 8 bits each for min/max scalar values, but the Brick ID itself is the 32-bit identifier) [S1].

**Edge Cases and Pitfalls:**
*   **Spatial Overflow:** If the volume dimensions exceed $256^3$ bricks, the 8-bit spatial offset fields will overflow. The sources do not specify a handling mechanism for volumes larger than $256^3$ bricks in this specific layout [S1].
*   **Page Table ID Overflow:** If $m \times k > 256$, the page table ID cannot be stored in 8 bits. The constraint $m \times k \le 256$ must be enforced at initialization [S1].
*   **Precision:** The spatial offsets are integers. Sub-brick precision is not supported within the Brick ID itself [S1].

## Subtopics

### Patch Metadata Layout
This variant is not a direct child of the Brick ID Page Table Layout but is a related metadata structure in hierarchical volume/mesh compression. It stores per-patch metadata in main memory at runtime, totaling 40 bytes. The fields are: 1) xy coordinates of the first triangle, 2) offset into the compressed bitstream, 3) a single quantization parameter, and 4) object-space error term and bounding sphere data (exact byte breakdown for error/sphere not detailed, but implied by SOAR context) [S2]. It is used for view-dependent mesh generation via a binary tree indexing scheme [S2].

## Operating characteristics
*   **Time Complexity:** O(1) for Brick ID generation and lookup, as it involves simple bitwise operations and array indexing [S1].
*   **Memory Complexity:** The Brick ID itself is fixed at 4 bytes (32 bits) per brick [S1]. The page table hierarchy size depends on $m$ and $k$, with a maximum of 256 page tables [S1].
*   **Performance:** The layout is designed for efficient GPU access, with the brick cache implemented as an `r8unorm` texture [S1]. The residency octree is pointerless, reducing memory overhead and improving cache locality [S1].
*   **Strengths:** Compact representation, efficient bitwise manipulation, suitable for GPU texture-based caching [S1].
*   **Limitations:** Limited to volumes with dimensions $\le 256$ bricks per axis and total page tables $\le 256$ [S1].

## Background
The Brick ID Page Table Layout was introduced in the context of scalable web volume rendering, specifically in the "Residency Octree" paper [S1]. It is part of a hybrid approach that combines a residency octree with a brick-based storage system to manage memory efficiently for large volumetric datasets. The method was developed to support multi-channel and multi-resolution data on GPUs, where memory bandwidth and cache efficiency are critical [S1].

## Sources
[S1] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
[S2] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
