# Virtual Multi-Resolution Volume Address Translation

Also called: Page Table Hierarchy Address Translation

## Overview
Virtual Multi-Resolution Volume Address Translation is a method that maps virtual multi-resolution volume coordinates to physical addresses in a GPU block cache using a page table hierarchy [S1]. It solves the problem of addressing anisotropic down-sampled volume data by packing multiple resolution levels into a single 3D texture, enabling efficient memory management for petascale microscopy data streams [S1]. It builds on [Hierarchical Index/Address Translation](8371-hierarchical-index-address-translation.md) by providing the specific translation logic and data layout for multi-resolution page directories.

## How it works
**Inputs:** Virtual multi-resolution address $(l, p)$, where $l$ is the resolution level and $p$ is the normalized coordinate.
**Outputs:** Physical address in the block cache.

**Data Structures:**
1.  **texPageDir:** A single 3D texture storing the multi-resolution page directory. All resolution levels $l \in [0, l_{max}]$ are packed into this texture [S1].
2.  **pageDirBase:** A small array storing the origin (offset) of each resolution level within `texPageDir` [S1].
3.  **Page Table Cache:** A 3D texture managed by CPU using LRU/MRU scheme [S1].
4.  **Block Cache:** A 3D texture in GPU memory [S1].
5.  **CPU Block Cache:** A larger version of the block cache in CPU memory [S1].

**Constants:**
*   $r_{pd}(l)$: Total number of entries in the page directory for resolution level $l$ [S1].
*   $r_{pt}(l)$: Total number of entries in the virtualized page table of level $l$ [S1].
*   $r_v(l)$: Total number of voxels in the virtualized volume of level $l$ [S1].
*   $b_{pt}$: Number of entries per page table block [S1].
*   $b_{vox}$: Number of voxels per voxel block [S1].

**Steps:**
1.  Retrieve the base offset for level $l$ from `pageDirBase` [S1].
2.  Compute the look-up index in `texPageDir` using normalized coordinates $p$ and constants $r_{pd}(l)$, $r_{pt}(l)$, $r_v(l)$, $b_{pt}$, and $b_{vox}$ [S1].
3.  Sample `texPageDir` at the computed index to obtain the page table entry [S1].
4.  Use the page table entry to determine the physical address in the block cache [S1].
5.  If a cache miss occurs, the ray-caster reports it to the CPU [S1].
6.  The CPU checks if the required block is in the CPU block cache [S1].
7.  If resident, the block is downloaded to the GPU block cache [S1].
8.  Empty blocks are skipped during ray-casting and do not trigger cache misses or require mapping [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def translate_address(l, p, pageDirBase, texPageDir, r_pd, r_pt, r_v, b_pt, b_vox):
    """
    Translates virtual multi-resolution address (l, p) to physical block cache address.
    """
    # Step 1: Get base offset for level l
    base_offset = pageDirBase[l]
    
    # Step 2: Compute look-up index
    # Note: Exact formula for index computation from p is not given in sources [S1]
    # It depends on r_pd(l), r_pt(l), r_v(l), b_pt, b_vox
    index = compute_index(p, l, r_pd(l), r_pt(l), r_v(l), b_pt, b_vox)
    
    # Step 3: Sample page directory
    # texPageDir is a 3D texture, index maps to 3D coordinates
    page_table_entry = sample_3d_texture(texPageDir, base_offset + index)
    
    # Step 4: Determine physical address
    # page_table_entry contains the physical address in block cache
    physical_address = extract_physical_address(page_table_entry)
    
    return physical_address

def handle_cache_miss(ray_caster, cpu_block_cache, gpu_block_cache):
    """
    Handles cache misses reported by the ray-caster.
    """
    # Ray-caster reports miss for a specific block
    block_id = ray_caster.get_missed_block()
    
    # Check CPU block cache
    if block_id in cpu_block_cache:
        # Download to GPU
        gpu_block_cache[block_id] = cpu_block_cache[block_id]
    else:
        # Fetch from storage (not detailed in sources)
        pass
```

**Edge Cases and Pitfalls:**
*   **Anisotropic Down-sampling:** Prevents storing the multi-resolution page directory as a regular mipmap, necessitating the custom packing into `texPageDir` [S1].
*   **Empty Blocks:** Empty space skipping is performed, but no cache miss is reported for empty blocks as they never need mapping [S1].

## Subtopics

### Page Table and Cache Management
This variant focuses on the management of the multi-resolution page directory, page table cache, and block cache, each implemented as a 3D texture managed by the CPU using LRU/MRU schemes [S1]. Cache misses are initiated by the ray-caster, and the CPU checks the CPU block cache before downloading to GPU [S1].

## Operating characteristics
**Time Complexity:** Not given in the sources.
**Memory Complexity:** Not given in the sources.
**Performance:** The method is designed for petascale microscopy data streams, implying scalability to large datasets [S1].
**Strengths:** Handles anisotropic down-sampling efficiently by packing levels into a single texture [S1].
**Limitations:** Requires custom packing logic due to anisotropic down-sampling [S1].
**When to use:** When rendering multi-resolution volume data with anisotropic down-sampling and needing efficient cache management for large datasets [S1].
**When not to use:** When regular mipmap structures are sufficient for down-sampling [S1].

## Background
The method was introduced in the context of interactive volume exploration of petascale microscopy data streams [S1]. It addresses the challenges of rendering large-scale volume data with multi-resolution support and efficient memory management [S1].

## Sources
[S1] PDF Interactive Volume Exploration of Petascale Microscopy Data Streams ..., https://vccvisualization.org/publications/2012_hadwiger_petascale.pdf, fetched 2026-10-28
