# Multi-level, multi-resolution page table hierarchy

Also called: Multi-Level, Multi-Resolution Page Tables, Orthogonal page table hierarchy, Separate page tables per resolution, Single-Level

## Overview
This method decouples the spatial subdivision granularity of the residency octree from the resolution levels of the bricked volume hierarchy to enable scalable address translation for extreme-scale data. Unlike single-level page tables or tree structures where traversal depth and data resolution are coupled, it employs a separate page table hierarchy for each resolution level, allowing independent optimization of rendering quality and memory scalability.

## How it works
**Inputs and Outputs:**
*   **Input:** Ray sample position $(x, y, z)$, current ray direction, desired resolution level $L$ (determined by projected screen size), and the multi-resolution page directory structure resident in GPU memory.
*   **Output:** Physical address of the voxel brick containing the sample, allowing access to the voxel data cache texture.

**Data Structures:**
The structure consists of two orthogonal hierarchies:
1.  **Resolution Hierarchy:** A set of discrete resolution levels $L \in \{0, 1, \dots, N_{res}\}$.
2.  **Page Table Hierarchy:** For each resolution level $L$, a tree of page tables.
    *   **Multi-Resolution Page Directory:** The top-level page table for each resolution level. This is the only structure required to be resident in GPU memory. It is an array of pointers, one entry per resolution level.
    *   **Page Tables:** Virtualized structures below the directory. They map virtual page indices to physical brick indices.
    *   **Cache Textures:**
        *   **Voxel Cache Texture:** Stores active voxel bricks. All bricks, regardless of resolution level, must have the same physical resolution (number of voxels) to share this single texture.
        *   **Page Table Cache Texture:** Stores active page table bricks (the virtualized page tables).

**Address Layout:**
A virtual address is decomposed into fields. Let $B_v$ be the brick size in voxels for data, and $B_p$ be the brick size in entries for page tables.
1.  **LOD Selection:** Determine resolution level $L$ based on sample distance or projected size.
2.  **Directory Lookup:** Access the $L$-th entry in the Multi-Resolution Page Directory to get the root of the page table hierarchy for level $L$.
3.  **Page Table Traversal:** Traverse the page table hierarchy for level $L$.
    *   Let the virtual address be $A$.
    *   For each level $k$ in the page table hierarchy (from top to bottom):
        *   Extract the page index $P_k$ from $A$ using bit shifts/masks determined by $B_p$.
        *   Look up $P_k$ in the current page table brick.
        *   If the entry is valid, proceed to the next level.
        *   If invalid, a page fault occurs (handled by CPU/GPU driver to load the page table brick into the Page Table Cache Texture).
4.  **Voxel Lookup:** The final page table entry provides the physical brick index. Combine this with the offset within the brick (derived from the lower bits of $A$) to index into the Voxel Cache Texture.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
# B_v: Brick size in voxels (e.g., 16^3)
# B_p: Brick size in page table entries (e.g., 2^10)
# NUM_RES_LEVELS: Number of resolution levels
# PAGE_TABLE_DEPTH: Number of page table levels per resolution (e.g., 2 or 3)

# Resident Data
# multi_res_page_directory: Array[Pointer] of size NUM_RES_LEVELS
# voxel_cache_texture: Texture2D/3D storing voxel bricks
# page_table_cache_texture: Texture2D/3D storing page table bricks

# Cached State (per ray)
# cached_res_level: int
# cached_page_table_ptrs: Array[Pointer] of size PAGE_TABLE_DEPTH

def translate_address(ray_sample_pos, ray_dir):
    # 1. Determine Resolution Level
    # Example: based on projected voxel size or distance
    lod = compute_lod(ray_sample_pos, ray_dir)
    
    # 2. Look-up Caching: Exploit spatial coherence
    # If the resolution level hasn't changed and we have cached pointers,
    # we can skip directory lookup and potentially higher page table levels.
    if lod == cached_res_level and cached_page_table_ptrs is not None:
        current_ptr = cached_page_table_ptrs[0]
        start_level = 1 # Skip directory and first level if cached
    else:
        # Access the Multi-Resolution Page Directory (Resident)
        current_ptr = multi_res_page_directory[lod]
        cached_res_level = lod
        start_level = 0
        cached_page_table_ptrs = [None] * PAGE_TABLE_DEPTH

    # 3. Traverse Page Table Hierarchy
    virtual_address = compute_virtual_address(ray_sample_pos)
    
    for level in range(start_level, PAGE_TABLE_DEPTH):
        # Extract page index for this level
        # Assuming uniform brick size B_p for page tables
        page_index = (virtual_address >> (level * log2(B_p))) & (B_p - 1)
        
        # Load page table brick if not in cache
        # [G] Implementation detail: Check a hash map or texture lookup for presence
        page_table_brick = load_page_table_brick(current_ptr, page_index)
        
        # Get next level pointer or physical brick index
        entry = page_table_brick[page_index]
        
        if entry.is_valid:
            if level < PAGE_TABLE_DEPTH - 1:
                current_ptr = entry.pointer_to_next_level
                cached_page_table_ptrs[level] = current_ptr
            else:
                # Last level: entry contains physical brick index
                physical_brick_index = entry.physical_brick_index
                break
        else:
            # Page fault: Load the required page table brick
            # [G] This involves CPU-GPU synchronization or async copy
            load_faulted_page_table(virtual_address, level)
            # Retry logic would be needed here in a real implementation
            raise PageFaultException

    # 4. Access Voxel Data
    # Extract offset within the brick
    brick_offset = virtual_address & (B_v**3 - 1)
    
    # Access the Voxel Cache Texture
    # [G] The physical_brick_index maps to a region in the cache texture
    voxel_data = voxel_cache_texture[physical_brick_index][brick_offset]
    
    return voxel_data
```

**Parameters:**
*   **Brick Size (Voxel):** All active voxel bricks must have the same physical resolution (e.g., $16 \times 16 \times 16$ voxels) to share the single cache texture.
*   **Brick Size (Page Table):** Can be chosen independently of voxel brick size.
*   **Page Table Levels:** Two levels support volumes up to several hundred terabytes; three levels are sufficient for exa-scale data [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Virtual address bit-width must be sufficient to address the entire volume space across all resolution levels.
*   **Cache Coherence:** The page table cache texture must be kept coherent with the resident directory.
*   **Degenerate Input:** If a ray sample falls outside the volume bounds, address translation should fail gracefully or return a transparent value.

## Subtopics

### Single-Level, Multi-Resolution Page Tables
The simpler variant where each resolution level has only one page table level. It is easier to implement but scales poorly to very large data compared to the multi-level variant. Use when data size is moderate and implementation simplicity is prioritized. See [Single-Level, Multi-Resolution Page Tables](5252-single-level-multi-resolution-page-tables.md).

### Hash Tables
An alternative address translation structure using spatial hashing. Not discussed in detail here but mentioned as an alternative for GPU-based translation. See [Hash Tables](5253-hash-tables.md).

## Operating characteristics
*   **Time Complexity:** Address translation is $O(D)$ where $D$ is the page table depth (typically 2-3). Due to spatial coherence, the number of actual texture lookups for page tables is very low, as cached pointers are reused across samples along a ray.
*   **Memory Complexity:** The resident memory is proportional to the number of resolution levels (size of the multi-resolution page directory). Virtualized page tables are loaded on demand.
*   **Performance:** Scales well to extreme-scale data (hundreds of terabytes to exabytes). The performance difference between single-level and multi-level page tables is negligible due to caching [S1].
*   **Strengths:** Decouples resolution quality from traversal scalability. Supports very large datasets. Efficient caching due to coherence.
*   **Limitations:** Requires two cache textures (voxel and page table). More complex implementation than single-level page tables.
*   **When to Use:** Rendering extreme-scale volume data (e.g., electron microscopy) where single-level page tables or octrees are insufficient for scalability.
*   **When Not to Use:** Small to moderate datasets where simpler structures suffice.

## Background
The multi-level, multi-resolution page table hierarchy was described by Hadwiger et al. [HBJP12] and used for rendering extreme-scale electron microscopy data [BHL* 11, HBJP12]. It addresses the limitations of single-level page tables and tree structures by completely decoupling the address translation hierarchy from the multi-resolution rendering hierarchy. This allows independent optimization of data down-sampling (for quality) and page table size reduction (for scalability).

## Sources
[S1] State-of-the-Art in GPU-Based Large-Scale Volume Visualization, https://vcg.seas.harvard.edu/publications/20150513-stateof-the-art-in-gpu-based-large-scale-volume-visualization/paper, fetched 2026-10-24
