# Splat List (SVL) Generation with Multi-Level Downsampling

Also called: LOD Splatting, Sparse Voxel List

## Overview
A rendering technique that generates a sparse voxel list (SVL) of splat geometry across multiple levels of detail (LOD) by iteratively downsampling a voxel occupancy mask and extracting splat primitives for each level [S1, unofficial]. It differs from [Hierarchical Splatting](1168-hierarchical-splatting.md) by using a flat, contiguous buffer of splat indices rather than a tree structure for traversal, and from [Multi-Resolution Points](4784-multi-resolution-points.md) by explicitly managing LOD levels through mask downsampling rather than point clustering [S1, unofficial].

## How it works
**Inputs:** A `ChunkSet` containing a list of `ChunkMD` structures, each representing a spatial chunk of voxel data. Each chunk contains voxel data (`voxels`), an RLE representation (`rle`), and state flags.
**Outputs:** A contiguous `SVL` buffer (array of `uint16_t`) containing splat indices for all LOD levels, and updated `GeometrySVL` structures linking to this buffer.

**Data Structures:**
*   `Voxel`: Type alias for `uint8_t` [S1, unofficial].
*   `MAX_LOD_LEVEL`: Constant defining the number of splat LOD levels, set to 5 [S1, unofficial].
*   `struct GeometrySVL`:
    *   `lod`: `uint8_t`
    *   `vao`: `uint32_t`
    *   `vbo`: `uint32_t`
    *   `vbo_items`: `uint32_t`
*   `struct ChunkMD`:
    *   `svl`: `uint16_t*` (pointer to the SVL buffer for this chunk)
    *   `svl_items[MAX_LOD_LEVEL]`: `uint32_t` (count of items per LOD level)
    *   `svl_items_total`: `uint32_t` (total items across all LOD levels)
    *   `no_geometry`: `volatile uint8_t`
    *   `svl_dirty`: `volatile uint8_t`
    *   `mesh_dirty`: `volatile uint8_t`
    *   `remesh`: `volatile uint8_t`
    *   `voxels`: `Voxel*`
    *   `rle`: `Voxel*`
    *   `count`: `uint32_t`
    *   `mutex_svl`: mutex for SVL access
*   `struct ChunkSet`:
    *   `gsvl[MAX_LOD_LEVEL]`: `struct GeometrySVL*` (array of geometry structures for each LOD level)
    *   `chunks`: `struct ChunkMD*`
    *   `null_chunk`: `struct ChunkMD*`

**Algorithm:**
1.  **Initialization:** For each chunk `c` in `set->chunks`, check if it requires remeshing (`c->remesh` or `c->dirty`). If not, skip. If `c->mesh_vbo` exists, free it. [S1, unofficial].
2.  **Locking:** Lock the chunk for write access using `chunk_lock(set, c)`. [S1, unofficial].
3.  **Mask Generation:** Generate the initial voxel occupancy mask for the chunk using `chunk_make_mask`. This mask is stored in a work buffer. [S1, unofficial].
4.  **LOD Loop:** Iterate through LOD levels `i` from 0 to `MAX_LOD_LEVEL - 1`.
    *   **Splat Extraction:** Extract splat primitives from the current mask (at level `i`) into a geometry buffer. The number of items extracted is stored in `temp_lo_vbo_segments[i]`. [S1, unofficial].
    *   **Downsampling:** If `i` is not the final LOD level, downsample the current mask into a work buffer for level `i+1` using `chunk_mask_downsample`. [S1, unofficial].
    *   **Accumulation:** Accumulate the geometry item counts: `geom_items += temp_lo_vbo_segments[i]`. [S1, unofficial].
5.  **Buffer Allocation:**
    *   If `geom_items` is zero:
        *   Free any existing `c->svl` buffer.
        *   Set `c->svl_items` to zero for all levels.
        *   Set `c->svl_items_total` to 0.
        *   Set `c->no_geometry = 1`.
        *   Mark `c->svl_dirty = 1`.
    *   If `geom_items` is non-zero:
        *   Allocate a contiguous `SVL` buffer of size `geom_items * sizeof(uint16_t)`.
        *   Copy the accumulated geometry data from the mesher buffer into the `SVL` buffer.
        *   Update `c->svl` to point to this buffer.
        *   Update `c->svl_items[i]` with the count for each level.
        *   Set `c->svl_items_total = geom_items`.
        *   Mark `c->svl_dirty = 1`.
6.  **Geometry Structure Update:** Update the `GeometrySVL` structures in `set->gsvl[i]` with the `vbo` (pointer to `c->svl`), `vbo_items` (count for level `i`), and `lod` (level `i`). [S1, unofficial].
7.  **Unlocking:** Unlock the chunk using `chunk_unlock(set, c)`. [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_svl_list(set, c, mesher_buffers, omp_id):
    # Check if remeshing is needed
    if not (c.remesh or c.dirty):
        return

    # Lock chunk
    chunk_lock(set, c)

    # Clear buffers
    work_buf = mesher_buffers[omp_id].work
    mask_buf = mesher_buffers[omp_id].mask
    geom_buf = mesher_buffers[omp_id].geom
    memset(work_buf, 0, work_buf.size)
    memset(mask_buf, 0, mask_buf.size)
    memset(geom_buf, 0, geom_buf.size)

    geom_items = 0
    temp_lo_vbo_segments = [0] * MAX_LOD_LEVEL

    # Generate initial mask
    chunk_make_mask(c, mask_buf)

    # Iterate through LOD levels
    for i in range(MAX_LOD_LEVEL):
        # Extract splats from current mask
        # Returns number of items extracted for this LOD level
        count = extract_splats_from_mask(mask_buf, geom_buf, geom_items, i)
        temp_lo_vbo_segments[i] = count
        geom_items += count

        # Downsample mask for next level if not last level
        if i < MAX_LOD_LEVEL - 1:
            chunk_mask_downsample(mask_buf, work_buf)
            # Swap buffers or copy work_buf to mask_buf for next iteration
            # Implementation detail: assume work_buf becomes next mask_buf
            mask_buf = work_buf

    # Handle zero geometry case
    if geom_items == 0:
        if c.svl is not None:
            mem_free(c.svl)
        c.svl = None
        for j in range(MAX_LOD_LEVEL):
            c.svl_items[j] = 0
        c.svl_items_total = 0
        c.no_geometry = 1
        c.svl_dirty = 1
    else:
        # Allocate SVL buffer
        new_svl = mem_alloc(geom_items * sizeof(uint16_t))
        
        # Copy geometry data into SVL buffer
        # Implementation detail: geom_buf contains the indices/geometry data
        # The exact copy mechanism depends on how extract_splats_from_mask populates geom_buf
        # Assuming geom_buf is filled with uint16_t indices
        memcpy(new_svl, geom_buf, geom_items * sizeof(uint16_t))
        
        c.svl = new_svl
        for j in range(MAX_LOD_LEVEL):
            c.svl_items[j] = temp_lo_vbo_segments[j]
        c.svl_items_total = geom_items
        c.svl_dirty = 1

        # Update GeometrySVL structures
        for j in range(MAX_LOD_LEVEL):
            set.gsvl[j].vbo = new_svl
            set.gsvl[j].vbo_items = temp_lo_vbo_segments[j]
            set.gsvl[j].lod = j

    # Unlock chunk
    chunk_unlock(set, c)
```

**Parameters:**
*   `MAX_LOD_LEVEL`: 5 [S1, unofficial].
*   `Voxel`: `uint8_t` [S1, unofficial].
*   Coordinate/Iterator types: `uint16_t` for chunk/voxel locations, `uint32_t` for worldspace vectors and iterators [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Geometry:** If no splats are found at any LOD level, the SVL buffer is freed, and `no_geometry` is set to 1 [S1, unofficial].
*   **Buffer Overflow:** The mesher buffers must be large enough to hold the maximum possible geometry for all LOD levels combined. If buffers are exhausted, the thread skips the chunk [S1, unofficial].
*   **Concurrency:** Mutexes are used to protect chunk access. `svl_dirty` and `mesh_dirty` flags are used to coordinate between meshing and rendering threads [S1, unofficial].
*   **Null Chunk:** Null chunks are handled specially, copying the null chunk's voxel data [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels in the chunk and the number of LOD levels. Downsampling reduces the mask size exponentially, so higher LOD levels are faster to process.
*   **Memory Complexity:** Proportional to the number of splats generated. The SVL buffer is contiguous, which is cache-friendly for rendering.
*   **Strengths:** Simple data structure (flat array), easy to integrate with existing OpenGL/Vulkan pipelines, supports LOD for performance scaling.
*   **Limitations:** Requires full chunk remeshing if any voxel changes, which can be expensive for dynamic scenes. The flat SVL structure may lead to overdraw if not sorted or culled properly.
*   **When to use:** Static or slowly changing scenes where LOD is beneficial. Not ideal for highly dynamic scenes with frequent voxel changes.

## Background
This method is implemented in the VoxelPlat project by kosshi-net [S1, unofficial]. It is part of a larger voxel rendering system that uses chunk-based spatial partitioning and multi-resolution splatting for efficient rendering.

## Sources
[S1] kosshi-net/voxplat, https://github.com/kosshi-net/voxplat, fetched 2026-10-27 (unofficial)
