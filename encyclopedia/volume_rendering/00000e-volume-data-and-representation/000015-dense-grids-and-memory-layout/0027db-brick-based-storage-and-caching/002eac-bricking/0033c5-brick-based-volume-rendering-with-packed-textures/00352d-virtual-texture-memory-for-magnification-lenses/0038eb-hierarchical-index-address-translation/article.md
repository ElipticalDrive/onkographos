# Hierarchical Index/Address Translation

Also called: Address translation texture, Layout/index texture

## Overview
Hierarchical Index/Address Translation maps virtual volume coordinates to physical texture coordinates in GPU-based volume rendering, supporting multi-resolution rendering and memory management [S1, unofficial][S2]. It enables arbitrary coordinate transformations, such as magnification, in the fragment shader without rebinding textures by locating bricks within a single memory pool [Parent]. It sets itself apart from flat index textures by using a page table hierarchy to handle multi-resolution data and anisotropic down-sampling, which prevents storage as a regular mipmap [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Virtual volume coordinates (normalized spatial coordinates $p$ and resolution level $l$).
*   **Output:** Physical texture coordinates or block cache indices.

**Data Structures:**
1.  **Index/Address Translation Texture (Flat/Hierarchical):** Stores metadata for bricks, including the number of bricks, texture size of brick, and physical extent of brick [S1, unofficial].
2.  **Multi-resolution Page Directory (`texPageDir`):** A single 3D texture storing the page directory for all resolution levels $l \in [0, l_{max}]$. The origin of each level is retrieved from a small array `pageDirBase` [S2].
3.  **Page Table Cache:** Implemented as a 3D texture, managed by the CPU using an LRU/MRU scheme [S2].
4.  **Block Cache:** Implemented as a 3D texture, managed by the CPU using an LRU/MRU scheme [S2].

**Parameters:**
*   $r_{pd}(l)$: Total number of entries in the page directory for resolution level $l$ [S2].
*   $r_{pt}(l)$: Total number of entries in the virtualized page table of level $l$ [S2].
*   $r_v(l)$: Total number of voxels in the virtualized volume of level $l$ [S2].
*   $b_{pt}$: Number of entries per page table block [S2].
*   $b_{vox}$: Number of voxels per voxel block [S2].

**Steps:**
1.  Compute lookup indexes from normalized coordinates $p$ using constants $r_{pd}(l)$, $r_{pt}(l)$, and $r_v(l)$ [S2].
2.  Access the page directory by retrieving the base address for resolution level $l$ from `pageDirBase` and sampling `texPageDir` to find the page table location [S2].
3.  Perform a page table lookup using the computed index to access the page table cache. Report cache misses to the CPU [S2].
4.  Handle cache management on the CPU: upon a cache miss, check if the required block is resident in the CPU block cache. If resident, download it to the GPU block cache [S2].
5.  Translate the virtual multi-resolution address $(l, p)$ to a physical address in the block cache using the resolved page table hierarchy [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def translate_virtual_to_physical(p, l, pageDirBase, texPageDir, pageTableCache, blockCache):
    # 1. Compute lookup indexes from normalized coordinates p using constants r_pd(l), r_pt(l), r_v(l)
    # Exact index computation formula not provided in sources [G]
    lookup_idx = compute_index_from_normalized(p, l)

    # 2. Access page directory: retrieve base address for level l from pageDirBase
    base_offset = pageDirBase[l]

    # Sample texPageDir to find page table location
    page_table_entry = texPageDir[base_offset + lookup_idx]

    # 3. Page table lookup: use computed index to access page table cache
    if page_table_entry not in pageTableCache:
        # Cache miss: report to CPU for management
        # CPU checks CPU block cache and downloads to GPU if resident [S2]
        page_table_cache = load_page_table_from_cpu(page_table_entry)
    else:
        page_table_cache = pageTableCache[page_table_entry]

    # 4. Cache management handled by CPU LRU/MRU scheme [S2]

    # 5. Translate virtual multi-resolution address (l, p) to physical address in block cache
    physical_coords = page_table_cache.resolve_physical(p)
    return physical_coords
```

**Pitfalls:**
*   **Anisotropic Down-sampling:** Prevents the multi-resolution page directory from being stored as a regular mipmap, requiring custom packing into `texPageDir` [S2].

## Subtopics

### [Address Translation](20555-address-translation.md)
Address translation maps virtual volume coordinates (specifying sample locations along viewing rays) to physical addresses (coordinates in actual texture space) to access volume data stored in individual bricks, enabling out-of-core rendering. It allows the renderer to specify sample locations in a unified space covering the whole bounding box of the volume, independent of individual brick storage, and then resolve the actual texture coordinates where the brick's data resides. This method is distinct from hierarchical approaches like octree rendering, which can be viewed as address translation where the octree itself acts as a hierarchical page table and tree traversal performs the lookup.

### Virtual Multi-Resolution Volume Address Translation
A method to translate a virtual multi-resolution address $(l, p)$ to a physical address in the block cache using a page table hierarchy [S2]. It handles anisotropic down-sampling by packing resolution levels into a single 3D texture `texPageDir` [S2]. Choose this when rendering multi-resolution volumes with anisotropic down-sampling where regular mipmaps are insufficient [S2].

## Operating characteristics
*   **Time Complexity:** Lookup is $O(1)$ per hierarchy level, but cache misses incur CPU-GPU transfer latency [S2].
*   **Memory Complexity:** Requires storage for the page directory, page table cache, and block cache, all implemented as 3D textures [S2].
*   **Performance:** Adaptive volume sampling using this method achieved a speed-up of 2.6-2.8x compared to full density rendering [S1, unofficial].
*   **Strengths:** Supports multi-resolution rendering and arbitrary coordinate transformations without rebinding textures [Parent, S1]. Efficiently handles empty space skipping [S2].
*   **Limitations:** Complex implementation due to custom packing of the page directory and CPU-GPU synchronization for cache management [S2].

## Background
The concept of using a layout/index texture for address translation was introduced in "GPU-Based Volume Ray-Casting with Advanced Illumination" [S1, unofficial]. The hierarchical multi-resolution variant was developed for "Interactive Volume Exploration of Petascale Microscopy Data Streams" to handle large datasets [S2].

## Sources
[S1] GPU-Based Volume Ray-Casting with Advanced Illumination, https://doi.org/10.2312/egt.20091069, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive Volume Exploration of Petascale Microscopy Data Streams ..., https://vccvisualization.org/publications/2012_hadwiger_petascale.pdf, fetched 2026-10-28
