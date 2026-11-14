# Brick ID Page Table Layout

Also called: Address Translation, Brick ID structure, Page Table Address Translation

## Overview
The Brick ID Page Table Layout is a sparse memory management technique that maps virtual volume addresses to physical brick locations using a page table structure [S1, unofficial]. It solves the problem of efficiently accessing large, sparse volumetric datasets by decoupling the logical coordinate space from the physical storage layout [S1, unofficial]. Unlike sibling methods such as the ExaBricks Hierarchy or Sparse Page Table and Brick Pool, this specific layout encodes spatial offsets and page table identifiers directly into a fixed-width integer structure, enabling direct arithmetic-based address translation without tree traversal [S2].

## How it works
**Inputs and Outputs:**
*   Input: A virtual address ($virtAddx$) within the volume space, the brick size ($brickSize$), and the page table data structure.
*   Output: The corresponding physical address ($physAddx$) where the brick data resides in memory [S1, unofficial].

**Data Structures:**
The core structure is the **Brick ID**, a 32-bit unsigned integer used in a multi-channel page table hierarchy [S2]. Its layout is defined as follows:
*   **Bits 0–23 (24 bits):** Spatial offset. Divided into three 8-bit fields representing the x, y, and z coordinates of the brick relative to the origin [S2].
*   **Bits 24–31 (8 bits):** Page Table ID. This uniquely identifies one of $(m \times k) \le 256$ page tables, where $m$ is the number of data channels and $k$ is the number of resolution levels representable on the GPU [S2].

**Algorithm:**
Address translation is performed via direct indexing and arithmetic [S1, unofficial]:

1.  Calculate the page table entry index by dividing the virtual address by the brick size:
    $$index = \lfloor virtAddx / brickSize \rfloor$$
2.  Retrieve the page table entry:
    $$pt\_entry = pageTable[index]$$
3.  Calculate the physical address by adding the offset within the brick to the base physical address stored in the entry:
    $$physAddx = pt\_entry.physAddx + (virtAddx \pmod{brickSize})$$

**Parameters:**
*   $brickSize$: The linear dimension of a brick in voxels. Chosen by the user at initialization [S2].
*   $m$: Number of channels.
*   $k$: Number of resolution levels.
*   Constraint: The product $m \times k$ must be $\le 256$ to fit within the 8-bit Page Table ID field [S2].

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on integer division and modulo. Ensure $brickSize$ is a power of two for efficient bitwise operations if performance is critical, though the formula holds generally [S1, unofficial].
*   **Boundaries:** If $virtAddx$ falls outside the mapped range, the page table entry may be invalid or point to unmapped memory. The implementation must check for valid entries before accessing $pt\_entry.physAddx$ [G].
*   **Degenerate Input:** If $brickSize$ is zero, division by zero occurs. $brickSize$ must be $> 0$ [G].

## Subtopics

### Patch Metadata Layout
A variant used in terrain rendering contexts where patch metadata is stored in main memory at runtime [S3]. It totals 40 bytes per patch and includes: 1) xy coordinates of the first triangle, 2) offset into the compressed bitstream, 3) a single quantization parameter, 4) object-space error term, and 5) bounding sphere data [S3]. This layout supports view-dependent mesh generation via recursive traversal of a binary tree indexing scheme [S3].

## Operating characteristics
*   **Time Complexity:** Address translation is $O(1)$, involving only array indexing and arithmetic operations [S1, unofficial].
*   **Memory Complexity:** The page table size depends on the volume resolution and brick size. The Brick ID structure itself is fixed at 32 bits per entry [S2].
*   **Strengths:** Fast, deterministic address lookup without recursion or stack usage, suitable for GPU execution [S1, unofficial]. Simple implementation using standard textures (e.g., r8unorm) for the brick cache [S2].
*   **Limitations:** The 8-bit Page Table ID limits the number of channels and resolution levels to a product of 256, restricting scalability for very high-resolution multi-channel data [S2]. Spatial hashing variants (mentioned as a related concept) require collision handling, but the direct page table approach avoids this at the cost of potentially sparse table usage [S1, unofficial].
*   **When to use:** Ideal for large-scale scientific visualization where direct memory access speed is critical and the number of channels/resolution levels is constrained [S1, unofficial][S2].

## Background
The concept of address translation using page tables for sparse textures was introduced in the context of GPU architectures supporting sparse residency, such as Vulkan's `VK_IMAGE_CREATE_SPARSE_RESIDENCY_BIT` and CUDA unified memory [S1, unofficial]. The specific Brick ID structure with 24-bit spatial offsets and 8-bit page table IDs was detailed in the "Residency Octree" paper for scalable web-based volume rendering [S2]. The method builds on general sparse texture techniques [S1, unofficial].

## Sources
[S1] GPU-Based Large-Scale Scientific Visualization, https://johanna-b.github.io/LargeSciVis2018/slides/LargeScaleVisualization_Architectures.pdf, fetched 2026-10-29 (unofficial)
[S2] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
[S3] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
