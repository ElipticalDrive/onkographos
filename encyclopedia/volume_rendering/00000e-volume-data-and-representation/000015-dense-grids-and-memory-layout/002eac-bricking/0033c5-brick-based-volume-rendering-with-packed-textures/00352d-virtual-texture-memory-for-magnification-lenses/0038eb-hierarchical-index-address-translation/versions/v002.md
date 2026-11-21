# Hierarchical Index/Address Translation

Also called: Address translation texture, Layout/index texture

## Overview
Hierarchical Index/Address Translation is a method for mapping virtual volume coordinates to physical texture coordinates in GPU-based volume rendering, supporting multi-resolution rendering and memory management [S1, unofficial][S2]. It enables arbitrary coordinate transformations, such as magnification, in the fragment shader without rebinding textures by locating bricks within a single memory pool [Parent]. It sets itself apart from flat index textures by using a page table hierarchy to handle multi-resolution data and anisotropic down-sampling, which prevents storage as a regular mipmap [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Virtual volume coordinates (normalized $p$ and resolution level $l$).
*   **Output:** Physical texture coordinates or block cache indices.

**Data Structures:**
1.  **Index/Address Translation Texture (Flat/Hierarchical):** Stores metadata for bricks. Fields include:
    *   Number of bricks.
    *   Texture size of brick.
    *   Physical extent of brick [S1, unofficial].
2.  **Multi-resolution Page Directory (`texPageDir`):** A single 3D texture storing the page directory for all resolution levels $l \in [0, l_{max}]$. The origin of each level is retrieved from a small array `pageDirBase` [S2].
3.  **Page Table Cache:** Implemented as a 3D texture, managed by CPU using LRU/MRU scheme [S2].
4.  **Block Cache:** Implemented as a 3D texture, managed by CPU using LRU/MRU scheme [S2].

**Parameters:**
*   $r_{pd}(l)$: Total number of entries in the page directory for resolution level $l$ [S2].
*   $r_{pt}(l)$: Total number of entries in the virtualized page table of level $l$ [S2].
*   $r_v(l)$: Total number of voxels in the virtualized volume of level $l$ [S2].
*   $b_{pt}$: Number of entries per page table block [S2].
*   $b_{vox}$: Number of voxels per voxel block [S2].

**Steps:**
1.  **Compute Lookup Indexes:** Calculate lookup indexes from normalized coordinates $p$ using constants $r_{pd}(l)$, $r_{pt}(l)$, and $r_v(l)$ [S2].
2.  **Access Page Directory:** Retrieve the base address for resolution level $l$ from `pageDirBase` and sample `texPageDir` to find the page table location [S2].
3.  **Page Table Lookup:** Use the computed index to access the page table cache. If a cache miss occurs, report it to the CPU [S2].
4.  **Cache Management (CPU):** Upon cache miss, the CPU checks if the required block is resident in the CPU block cache. If yes, download it to the GPU block cache [S2].
5.  **Physical Address Translation:** Translate the virtual multi-resolution address $(l, p)$ to a physical address in the block cache using the page table hierarchy [S2].

**Pitfalls:**
*   **Anisotropic Down-sampling:** Prevents the multi-resolution page directory from being stored as a regular mipmap, requiring custom packing into `texPageDir` [S2].

## Subtopics

### Virtual Multi-Resolution Volume Address Translation
A method to translate a virtual multi-resolution address $(l, p)$ to a physical address in the block cache using a page table hierarchy [S2]. It handles anisotropic down-sampling by packing resolution levels into a single 3D texture `texPageDir` [S2]. Choose this when rendering multi-resolution volumes with anisotropic down-sampling where regular mipmaps are insufficient [S2].

## Operating characteristics
*   **Time Complexity:** Lookup is $O(1)$ per level in the hierarchy, but cache misses incur CPU-GPU transfer latency [S2].
*   **Memory Complexity:** Requires storage for the page directory, page table cache, and block cache, all implemented as 3D textures [S2].
*   **Performance:** Adaptive volume sampling using this method achieved a speed-up of 2.6-2.8x compared to full density rendering [S1, unofficial].
*   **Strengths:** Supports multi-resolution rendering and arbitrary coordinate transformations without rebinding textures [Parent, S1]. Efficiently handles empty space skipping [S2].
*   **Limitations:** Complex implementation due to custom packing of the page directory and CPU-GPU synchronization for cache management [S2].

## Background
The concept of using a layout/index texture for address translation was introduced in "GPU-Based Volume Ray-Casting with Advanced Illumination" [S1, unofficial]. The hierarchical multi-resolution variant was developed for "Interactive Volume Exploration of Petascale Microscopy Data Streams" to handle large datasets [S2].

## Sources
[S1] GPU-Based Volume Ray-Casting with Advanced Illumination, https://doi.org/10.2312/egt.20091069, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive Volume Exploration of Petascale Microscopy Data Streams ..., https://vccvisualization.org/publications/2012_hadwiger_petascale.pdf, fetched 2026-10-28
