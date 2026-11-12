# Multi-Resolution Multi-Channel Page Table (MRMCPT)
Also called: page table approach

## Overview
Multi-Resolution Multi-Channel Page Table (MRMCPT) is an out-of-core volume rendering method that manages a multi-resolution hierarchy using page tables for spatial subdivision and cache residency [S1]. It builds on [Sparse Page Table and Brick Pool](2550-sparse-page-table-and-brick-pool.md) by organizing the mapping of logical voxel regions to resident bricks into a hierarchical structure that supports multiple resolution levels [S1]. Unlike [Sparse Brickmap Volume Rendering](2978-sparse-brickmap-volume-rendering.md) which may use different spatial indexing, MRMCPT relies on explicit page table lookups to translate virtual addresses to physical brick locations [S1]. Its primary distinction is the use of a page-table-based residency management system rather than a tree traversal or hash-based approach, which results in limited empty-space skipping capabilities compared to octree-based methods [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A multi-resolution volume dataset (typically stored as a hierarchy of resolution levels) and a transfer function defining the visible value range.
*   **Output:** Rendered image frames with bricks loaded into GPU memory (cache) as needed.

**Data Structures:**
The core data structure is the **Page Table**, which maps virtual brick coordinates to physical memory addresses.
*   **Page Table Entry (PTE):** Contains at least a physical address (`physAddx`) and potentially validity/residency flags.
*   **Brick:** A block of voxels (e.g., $8^3$ or larger) stored in the brick pool. Apron padding is used for interpolation [S1].
*   **Virtual Address Space:** The logical 3D coordinate space of the volume, divided into bricks.

**Algorithm Steps:**
1.  **Initialization:** Load the top-level page table into memory. The page table maps virtual brick indices to physical brick locations in the pool.
2.  **Ray Casting Setup:** For each pixel, cast a ray through the volume. Determine the virtual brick coordinates intersected by the ray.
3.  **Address Translation:**
    *   Compute the virtual brick index $(x, y, z)$ for the current sample position.
    *   Look up the Page Table Entry (PTE) for this virtual index.
    *   If the PTE indicates the brick is resident (valid), retrieve the physical address.
    *   If the brick is not resident, trigger a page fault: load the required brick from disk/storage into the brick pool, update the PTE, and retry.
    *   Calculate the physical sample position: $physAddx = pt\_entry.physAddx + (virtAddx \pmod{brickSize})$ [S2, unofficial].
4.  **Rendering:** Sample the brick data at the physical position and accumulate color/opacity.
5.  **Culling:** Skip bricks that are entirely empty. Note: MRMCPT only skips *entirely* empty bricks; it does not skip bricks containing values outside the current transfer function range [S1].

**Parameters:**
*   **Brick Size:** Typically $8^3$ voxels or larger, depending on the base implementation [S1].
*   **Page Table Block Size:** In multi-level variants, page tables are organized into blocks (e.g., $32^3$ entries) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Limited Culling:** MRMCPT cannot skip bricks that contain data but are outside the visible transfer function range. This leads to higher cache usage and lower performance on sparse or multi-channel datasets compared to octrees [S1].
*   **Cache Thrashing:** Because it requires lower-resolution bricks to be resident for rendering higher resolutions (in some implementations), it may suffer from cache thrashing if the working set is large [S1].

## Subtopics

### Multi-Level Page Tables
Multi-Level Page Tables is a variant introduced by Hadwiger et al. in 2012 [S2, unofficial]. It virtualizes page tables recursively, similar to CPU multi-level page tables, treating pages of page table entries like pages of voxels [S2, unofficial]. This creates a recursive page table hierarchy that is decoupled from the data resolution levels, ensuring the number of page table levels is significantly less than the number of data resolution levels [S2, unofficial].
*   **Parameters:** Voxel block size is $32^3$ voxels; Page table block size is $32^3$ page table entries [S2, unofficial].
*   **Scalability Examples:**
    *   For a $32,000 \times 32,000 \times 4,000$ volume (4 TB, 11 resolution levels): 2 page table levels with $32 \times 32 \times 4$ page table blocks [S2, unofficial].
    *   For a $128,000 \times 128,000 \times 16,000$ volume (196 TB, 13 resolution levels): 2 page table levels with $128 \times 128 \times 16$ page table blocks [S2, unofficial].
    *   For a $512,000 \times 512,000 \times 64,000$ volume (15 PB, 15 resolution levels): 3 page table levels with $16 \times 16 \times 2$ page table blocks [S2, unofficial].
    *   For a $2,000,000 \times 2,000,000 \times 250,000$ volume (888 PB, 17 resolution levels): 3 page table levels with $64 \times 64 \times 8$ page table blocks [S2, unofficial].

### Uniform Page Tables
Uniform Page Tables use page tables that are uniform in each level and can be mipmapped [S2, unofficial]. A page table can be created for each resolution level, resulting in a page table mipmap [S2, unofficial]. This approach is only feasible when the page table is not too large; for medium-sized volumes or large page/brick sizes, it requires a full-size page table, which may be impractical [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Address translation is $O(1)$ per brick lookup in a flat page table, or $O(L)$ for $L$ levels in multi-level page tables. Ray casting performance is limited by the number of bricks accessed.
*   **Memory Complexity:** Memory usage is proportional to the number of resident bricks plus the size of the page table hierarchy. Multi-level page tables reduce the page table overhead for extremely large volumes [S2, unofficial].
*   **Performance:** On sparse or multi-channel datasets, MRMCPT performs worse than octree-based approaches due to its inability to perform fine-grained empty-space skipping [S1]. It requires more bricks to be resident in the cache compared to residency octrees [S1].
*   **Strengths:** Simple address translation logic; decoupled hierarchy allows independent management of page table structure and data resolution [S2, unofficial].
*   **Limitations:** High cache usage on sparse data; cannot skip non-empty but invisible bricks [S1].

## Background
MRMCPT was introduced as a baseline out-of-core rendering method using a multi-resolution hierarchy managed via page tables [S1]. The multi-level page table variant was introduced by Hadwiger et al. in 2012 [S2, unofficial]. It is used in large-scale scientific visualization where volumes exceed GPU memory capacity [S2, unofficial].

## Sources
[S1] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
[S2] GPU-Based Large-Scale Scientific Visualization, https://johanna-b.github.io/LargeSciVis2018/slides/LargeScaleVisualization_Architectures.pdf, fetched 2026-10-29 (unofficial)
