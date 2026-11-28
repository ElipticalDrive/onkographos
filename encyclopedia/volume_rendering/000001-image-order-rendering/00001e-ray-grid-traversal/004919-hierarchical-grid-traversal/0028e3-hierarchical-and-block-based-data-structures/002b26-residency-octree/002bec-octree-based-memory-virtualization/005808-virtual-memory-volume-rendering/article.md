# Virtual Memory Volume Rendering

Also called: Multi-Resolution Virtual Memory Scheme, Virtual Memory Architecture for Volume Rendering, Visualization-Driven Virtual Memory Volume Rendering

## Overview
Virtual Memory Volume Rendering is an out-of-core rendering architecture that virtualizes 3D volumes by subdividing them into small 3D blocks and mapping them to a physical block cache using a multi-resolution page table hierarchy [S1]. It solves the problem of interactive exploration of petascale microscopy data streams by decoupling the construction of the 3D multi-resolution representation from data acquisition and sample access time from hierarchy size [S1]. Unlike sibling methods such as [Octree-based out-of-core volume rendering](5171-octree-based-out-of-core-volume-rendering.md) which require explicit octree traversal and holding the entire path from leaf to root in GPU memory, this method avoids these drawbacks by using a virtual memory scheme with arbitrary packing and no pre-computed 3D hierarchy [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of 2D microscope image tiles (raw or processed into 2D mipmaps) and a camera/view state for ray-casting.
*   **Output:** A rendered image from single-pass ray-casting.

**Data Structures:**
1.  **Virtual Volume:** The logical 3D volume subdivided into small 3D blocks of size $B \times B \times B$ voxels. The standard block size is $32^3$ voxels [S1].
2.  **Multi-Resolution Page Directory:** A hierarchy of page tables corresponding to resolution levels $l$. This is the top-level structure in the virtual memory hierarchy [S1].
3.  **Page Table Cache:** A cache storing page table entries (PTEs). Each PTE maps a virtual block position to a physical position in the Block Cache or is flagged as unmapped [S1].
4.  **Block Cache:** A large 3D texture resident in GPU memory containing the working set of currently required 3D blocks. Blocks are packed arbitrarily into this cache [S1].
5.  **Cache Miss Hash Table:** A structure used to track requested voxel blocks to manage cache updates [S1].

**Numbered Steps:**
1.  **Ray-Casting in Virtual Space:** For each sample along a ray, the algorithm determines the multi-resolution level $l$ and the page table entry $p$ corresponding to the virtual voxel address [S1].
2.  **Address Translation:** The algorithm performs address translation using the Multi-Resolution Page Directory and the Page Table Cache to find the physical location in the Block Cache [S1].
3.  **Cache Miss Detection:** The system checks if the block is unmapped (not resident in the cache). Unmapped blocks produce cache misses [S1].
4.  **Backward Propagation:** Cache misses are propagated backwards to trigger on-demand out-of-core processing [S1].
5.  **On-Demand Construction:** The system fetches the required 2D sub-tiles from the input stream and stitches/resamples them into 3D blocks [S1].
6.  **Cache Update:** The constructed 3D blocks are stored in the Block Cache, and the Page Table Cache is updated to map the virtual address to the new physical location [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
BLOCK_SIZE = 32  # voxels per dimension

# Data Structures
# page_directory: Multi-resolution page directory hierarchy
# page_table_cache: Cache of page table entries (virtual -> physical block index)
# block_cache: 3D texture in GPU memory
# cache_miss_table: Hash table tracking requested blocks

def ray_casting_loop(ray):
    for sample in ray.samples:
        # 1. Determine virtual address components
        level_l = determine_resolution_level(sample.position)
        virtual_block_id = get_virtual_block_id(sample.position, level_l)
        
        # 2. Address Translation
        physical_block_index = translate_address(
            page_directory, 
            page_table_cache, 
            level_l, 
            virtual_block_id
        )
        
        # 3. Cache Miss Detection
        if physical_block_index is UNMAPPED:
            # 4. Propagate Cache Miss
            handle_cache_miss(level_l, virtual_block_id)
            
            # Retry translation after miss handling
            physical_block_index = translate_address(
                page_directory, 
                page_table_cache, 
                level_l, 
                virtual_block_id
            )
            
        # 5. Texture Fetch
        color = fetch_block_color(block_cache, physical_block_index, sample.offset)
        accumulate_color(color)

def handle_cache_miss(level_l, virtual_block_id):
    # 6. On-Demand Construction
    # Fetch 2D sub-tiles corresponding to the virtual block
    required_2d_tiles = fetch_2d_sub_tiles(virtual_block_id)
    
    # Stitch and resample into 3D block
    new_3d_block = stitch_and_resample(required_2d_tiles, BLOCK_SIZE)
    
    # Allocate space in Block Cache (arbitrary packing)
    physical_block_index = allocate_block_in_cache(block_cache, new_3d_block)
    
    # Update Page Table Cache
    update_page_table_cache(
        page_table_cache, 
        level_l, 
        virtual_block_id, 
        physical_block_index
    )
    
    # Record in cache miss table for consistency
    cache_miss_table.add(virtual_block_id)

def translate_address(page_directory, page_table_cache, level_l, virtual_block_id):
    # Check page table cache first
    if virtual_block_id in page_table_cache:
        return page_table_cache[virtual_block_id]
    
    # Traverse page directory hierarchy to find PTE
    # Not given in sources: specific traversal algorithm for multi-resolution page directory
    # [G] Assuming a standard hierarchical lookup where higher levels coarsen the address
    pte = lookup_page_directory(page_directory, level_l, virtual_block_id)
    
    if pte is VALID:
        page_table_cache[virtual_block_id] = pte.physical_index
        return pte.physical_index
    else:
        return UNMAPPED
```

**Parameters:**
*   **Block Size:** $32^3$ voxels [S1].

**Edge Cases and Pitfalls:**
*   **Missing Data:** Missing data are detected during ray-casting as cache misses. Unmapped blocks produce cache misses which must be handled by the backward propagation mechanism [S1].
*   **Precision:** Not given in the sources.
*   **Boundaries:** Not given in the sources.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** The system only stores the working set of currently required blocks in the Block Cache, avoiding the storage of the entire hierarchy in GPU memory [S1].
*   **Measured Performance:** The system is evaluated for scalability and performance on petascale microscopy data streams [S1].
*   **Scaling:** The architecture handles missing data naturally and does not pre-compute any 3D multi-resolution representation, allowing it to accept a constant stream of 2D image tiles [S1].
*   **Strengths:** Decouples construction of the 3D multi-resolution representation from data acquisition; decouples sample access time during ray-casting from the size of the multi-resolution hierarchy; avoids explicit octree traversal and holding the entire path from leaf to root in GPU memory; supports arbitrary packing of blocks in the cache [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** Interactive exploration of petascale microscopy data streams where data is acquired as a stream of 2D image tiles [S1].
*   **When not to use:** Not given in the sources.

## Background
Virtual Memory Volume Rendering was introduced in a 2012 paper by Hadwiger et al. [S1]. It is designed for real microscopy data from neuroscience, enabling interactive exploration of petascale volume data [S1]. The system is visualization-driven, restricting most computations to the visible volume data [S1]. It compares favorably to previous best-of-breed systems like Gigavoxels and CERA-TVR, which use explicit octree traversal [S1].

## Sources
[S1] Interactive Volume Exploration of Petascale Microscopy Data Streams, https://vccvisualization.org/publications/2012_hadwiger_petascale.pdf, fetched 2026-10-28
