# Splat List (SVL) Generation with Multi-Level Downsampling

Also called: LOD Splatting, Sparse Voxel List

## Overview
This method generates a Sparse Voxel List (SVL) containing splat geometry across multiple levels of detail (LOD) by iteratively downsampling a voxel occupancy mask and extracting simplified splat primitives for each level [S1, unofficial]. It differs from standard hierarchical splatting by explicitly constructing a contiguous buffer of geometry items for all LOD levels simultaneously during the meshing phase, rather than traversing a tree structure at render time to select primitives [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A `ChunkSet` containing a list of `ChunkMD` structures, each representing a spatial chunk of voxel data. Each chunk contains an occupancy mask (either raw `Voxel` array or RLE compressed).
*   **Output:** A contiguous SVL buffer (`uint16_t*`) stored in `ChunkMD.svl`, containing geometry item counts and references for each LOD level. The buffer size is determined by the total number of geometry items (`geom_items`) generated across all LODs.

**Data Structures**
The implementation relies on specific type aliases and structures defined in the source [S1, unofficial]:
*   `Voxel`: Type alias for `uint8_t`.
*   `MAX_LOD_LEVEL`: Constant integer set to 5.
*   `struct GeometrySVL`:
    *   `lod`: `uint8_t`
    *   `vao`: `uint32_t`
    *   `vbo`: `uint32_t`
    *   `vbo_items`: `uint32_t`
*   `struct ChunkMD` (relevant fields):
    *   `svl`: `uint16_t*` (pointer to the generated SVL buffer)
    *   `svl_items[MAX_LOD_LEVEL]`: `uint32_t` (count of items per LOD level)
    *   `svl_items_total`: `uint32_t` (total item count)
    *   `no_geometry`: `volatile uint8_t` (flag if no geometry exists)
    *   `svl_dirty`: `volatile uint8_t` (flag indicating SVL needs regeneration)
    *   `voxels`: `Voxel*` (raw voxel data)
    *   `count`: `uint32_t` (number of voxels in the chunk)
*   `struct ChunkSet` (relevant fields):
    *   `null_chunk`: Pointer to a null chunk structure used for empty space handling.

**Algorithm Steps**
1.  **Initialization:** For a given chunk `c`, initialize counters `geom_items = 0` and `temp_lo_vbo_segments[MAX_LOD_LEVEL] = {0}`. Initialize work buffers for the current LOD level.
2.  **Level Loop:** Iterate through LOD levels $i$ from 0 to `MAX_LOD_LEVEL - 1`.
    *   **Mask Generation:** If $i=0$, generate the initial voxel occupancy mask for the chunk using `chunk_make_mask`. If $i>0$, downsample the mask from level $i-1$ into a work buffer using `chunk_mask_downsample`.
    *   **Splat Extraction:** Extract splat primitives from the current level's mask. Accumulate the number of geometry items generated for this level into `temp_lo_vbo_segments[i]`.
    *   **Accumulation:** Add `temp_lo_vbo_segments[i]` to the running total `geom_items`.
    *   **Downsampling:** If $i < MAX\_LOD\_LEVEL - 1$, downsample the current mask into a work buffer for level $i+1$ using `chunk_mask_downsample`.
3.  **Buffer Allocation:** Allocate a contiguous SVL buffer of size `geom_items * sizeof(uint16_t)`.
4.  **Data Copy:** Copy the accumulated geometry data from the mesher buffer into the allocated SVL buffer.
5.  **Zero Geometry Handling:** If `geom_items` is zero:
    *   Free any existing SVL buffer.
    *   Set `c->svl_items` to zero.
    *   Set `c->svl_items_total` to 0.
    *   Set `c->no_geometry = 1`.
    *   Mark `c->svl_dirty = 1`.
6.  **State Update:** Update `c->svl` to point to the new buffer and set `c->svl_dirty = 0` (assuming successful generation).

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_LOD_LEVEL = 5
SIZEOF_UINT16 = 2

def generate_svl(chunk: ChunkMD, chunk_set: ChunkSet):
    # Initialize counters
    geom_items = 0
    temp_lo_vbo_segments = [0] * MAX_LOD_LEVEL
    
    # Work buffers for mask and geometry
    # In the source, these are pre-allocated per thread in mesher[omp_id]
    # Here we assume local buffers for clarity
    current_mask = allocate_zeroed_buffer(chunk.count)
    next_mask = allocate_zeroed_buffer(chunk.count)
    geom_buffer = allocate_zeroed_buffer(chunk.count * 4) # Assuming 4 bytes per item for simplicity
    
    # Level 0: Generate initial mask
    chunk_make_mask(chunk, current_mask)
    
    for level in range(MAX_LOD_LEVEL):
        # Extract splats from current_mask into geom_buffer
        # Returns count of items generated for this level
        items = extract_splats_from_mask(current_mask, geom_buffer, level)
        temp_lo_vbo_segments[level] = items
        geom_items += items
        
        # Downsample for next level if not the last level
        if level < MAX_LOD_LEVEL - 1:
            chunk_mask_downsample(current_mask, next_mask)
            # Swap masks for next iteration
            current_mask, next_mask = next_mask, current_mask
            
    # Handle zero geometry case
    if geom_items == 0:
        if chunk.svl is not None:
            free(chunk.svl)
        chunk.svl_items = [0] * MAX_LOD_LEVEL
        chunk.svl_items_total = 0
        chunk.no_geometry = 1
        chunk.svl_dirty = 1
        return

    # Allocate SVL buffer
    svl_buffer_size = geom_items * SIZEOF_UINT16
    chunk.svl = allocate_buffer(svl_buffer_size)
    
    # Copy accumulated geometry data
    # Note: The source mentions copying from mesher buffer to SVL buffer.
    # The exact layout of the SVL buffer (uint16_t) implies it stores indices or compacted data.
    copy_data_to_svl(geom_buffer, chunk.svl, geom_items)
    
    # Update chunk state
    chunk.svl_items = temp_lo_vbo_segments
    chunk.svl_items_total = geom_items
    chunk.svl_dirty = 0
```

**Parameters**
*   `MAX_LOD_LEVEL`: 5 [S1, unofficial].
*   `Voxel`: `uint8_t` [S1, unofficial].
*   Coordinate/Iterator types: `uint16_t` for chunk/voxel locations, `uint32_t` for worldspace vectors and iterators [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Empty Chunks:** If a chunk has no geometry (`geom_items == 0`), the SVL buffer is freed, and flags are set to indicate no geometry [S1, unofficial].
*   **Null Chunk Handling:** If a chunk's voxels point to the null chunk's voxels, it is treated as empty, and the RLE is set to NULL [S1, unofficial].
*   **Concurrency:** The meshing process uses mutexes (`mutex_write`) to protect chunk access. The mesher uses multiple threads with separate buffers to avoid contention [S1, unofficial].
*   **Dirty Flags:** The SVL is only regenerated if `c->remesh` is true, which is derived from `c->dirty` [S1, unofficial]. If `c->svl_dirty` or `c->mesh_dirty` is set, the chunk is skipped to avoid processing incomplete geometry [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The generation time depends on the number of voxels and the downsampling operations. The source notes a meshing time measurement but does not provide a closed-form complexity [S1, unofficial].
*   **Memory Complexity:** The SVL buffer size is proportional to the total number of geometry items across all LOD levels. Each chunk stores its own SVL buffer [S1, unofficial].
*   **Strengths:** Pre-computing the SVL allows for efficient rendering by providing a contiguous buffer of geometry items. Multi-level downsampling enables LOD rendering without runtime tree traversal [S1, unofficial].
*   **Limitations:** The method requires significant memory for storing SVL buffers for all chunks. The fixed `MAX_LOD_LEVEL` of 5 may not be optimal for all scenes [S1, unofficial].
*   **When to Use:** Suitable for dynamic voxel scenes where geometry changes frequently, and pre-computed LODs are beneficial for rendering performance [S1, unofficial].

## Background
This method is part of the VoxelPlat project, an unofficial implementation of voxel-based rendering techniques [S1, unofficial]. It builds upon the principles of hierarchical and multi-resolution splatting, specifically focusing on the generation of Sparse Voxel Lists (SVL) for efficient LOD rendering [S1, unofficial].

## Sources
[S1] kosshi-net/voxplat, https://github.com/kosshi-net/voxplat, fetched 2026-10-27 (unofficial)
