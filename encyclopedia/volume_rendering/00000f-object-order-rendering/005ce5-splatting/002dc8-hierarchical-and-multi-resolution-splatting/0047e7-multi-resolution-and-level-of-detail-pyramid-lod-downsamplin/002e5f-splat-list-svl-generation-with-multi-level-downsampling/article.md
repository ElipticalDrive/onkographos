# Splat List (SVL) Generation with Multi-Level Downsampling

Also called: LOD Splatting, Sparse Voxel List

## Overview
This method generates a Sparse Voxel List (SVL) of splat geometry across multiple levels of detail (LOD) by iteratively downsampling a voxel occupancy mask and extracting representative splat primitives for each level [S1, unofficial].

## How it works
**Inputs:** A voxel chunk containing an occupancy mask (array of `Voxel` types, where `Voxel` is `uint8_t`) and metadata defining the chunk's spatial bounds and resolution [S1, unofficial].
**Outputs:** A contiguous SVL buffer (`uint16_t*`) containing geometry items for all LOD levels, and a `GeometrySVL` structure describing the buffer's layout [S1, unofficial].

**Data Structures:**
The core output structure is `struct GeometrySVL`:
- `lod`: `uint8_t`, the level of detail index [S1, unofficial].
- `vao`: `uint32_t`, vertex array object identifier [S1, unofficial].
- `vbo`: `uint32_t`, vertex buffer object identifier [S1, unofficial].
- `vbo_items`: `uint32_t`, number of items in the buffer [S1, unofficial].

The chunk metadata is stored in `struct ChunkMD`, which includes:
- `svl`: `uint16_t*`, pointer to the SVL buffer [S1, unofficial].
- `svl_items[MAX_LOD_LEVEL]`: `uint32_t`, count of items per LOD level [S1, unofficial].
- `svl_items_total`: `uint32_t`, total item count across all LODs [S1, unofficial].
- `svl_dirty`: `volatile uint8_t`, flag indicating the SVL needs regeneration [S1, unofficial].
- `no_geometry`: `volatile uint8_t`, flag set if the chunk contains no geometry [S1, unofficial].
- `mutex_write`: `pthread_mutex_t`, protects chunk access during meshing [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:** For each chunk `c`, check if `c->svl_dirty` is set or if `c->mesh_dirty` is set [S1, unofficial].
2. **Mask Generation:** Generate the initial voxel occupancy mask for the chunk using `chunk_make_mask` [S1, unofficial].
3. **Iterative Downsampling and Extraction:** Loop through LOD levels `i` from 0 to `MAX_LOD_LEVEL - 1`:
   a. **Splat Extraction:** Extract splat primitives from the current occupancy mask for level `i` [S1, unofficial].
   b. **Accumulation:** Add the number of generated geometry items for this level to a running total `geom_items` [S1, unofficial].
   c. **Downsampling:** If `i` is not the final LOD level, downsample the current mask into a work buffer for level `i+1` using `chunk_mask_downsample` [S1, unofficial].
4. **Buffer Allocation:** Allocate a contiguous SVL buffer of size `geom_items * sizeof(uint16_t)` [S1, unofficial].
5. **Data Copy:** Copy the accumulated geometry data from the mesher buffer into the allocated SVL buffer [S1, unofficial].
6. **Metadata Update:** Update `c->svl_items` with the counts per level and `c->svl_items_total` with the total count [S1, unofficial].
7. **Empty Chunk Handling:** If `geom_items` is zero, free any existing SVL buffer, zero out `c->svl_items`, set `c->svl_items_total` to 0, set `c->no_geometry` to 1, and mark `c->svl_dirty` as 1 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_svl(chunk, set, max_lod_level=5):
    # Check dirty flags; skip if SVL or mesh needs regeneration [S1, unofficial]
    if chunk.svl_dirty or chunk.mesh_dirty:
        return

    geom_items = 0
    svl_items_per_level = [0] * max_lod_level
    mask = chunk_make_mask(chunk)  # Generate initial voxel occupancy mask [S1, unofficial]
    work_buffer = [0] * len(mask)  # Work buffer for downsampled masks [S1, unofficial]

    for i in range(max_lod_level):
        # Extract splat primitives from current mask for level i [S1, unofficial]
        items = extract_splats_from_mask(mask, level=i)
        count = len(items)
        svl_items_per_level[i] = count
        geom_items += count

        # Downsample current mask into work buffer for level i+1 [S1, unofficial]
        if i < max_lod_level - 1:
            chunk_mask_downsample(mask, work_buffer)
            mask = work_buffer

    # Handle empty chunk case [S1, unofficial]
    if geom_items == 0:
        if chunk.svl is not None:
            mem_free(chunk.svl)
        chunk.svl = None
        chunk.svl_items = [0] * max_lod_level
        chunk.svl_items_total = 0
        chunk.no_geometry = 1
        chunk.svl_dirty = 1
        return

    # Allocate and copy SVL buffer [S1, unofficial]
    buffer_size = geom_items * 2  # sizeof(uint16_t)
    svl_buffer = mem_alloc(buffer_size)
    copy_geometry_to_svl(mesher_buffer, svl_buffer, geom_items)

    chunk.svl = svl_buffer
    chunk.svl_items = svl_items_per_level
    chunk.svl_items_total = geom_items
    chunk.svl_dirty = 0

# Steps not detailed in sources:
# chunk_make_mask(chunk): Generates initial voxel occupancy mask.
# extract_splats_from_mask(mask, level): Extracts representative splat primitives.
# chunk_mask_downsample(mask, work_buffer): Downsamples mask for next LOD.
# copy_geometry_to_svl(src, dst, count): Copies accumulated geometry data.
```

**Parameters:**
- `MAX_LOD_LEVEL`: Constant defining the number of splat LOD levels. Set to 5 [S1, unofficial].
- `Voxel`: Type alias for `uint8_t` [S1, unofficial].
- Coordinate/Iterator types: Chunk/voxel locations use `uint16_t`, worldspace vectors use `uint32_t`, iterators use `uint32_t` [S1, unofficial].

**Edge Cases:**
- **Empty Chunk:** If no geometry is generated (`geom_items == 0`), the SVL buffer is freed, and flags are reset to indicate no geometry [S1, unofficial].
- **Concurrency:** The method uses `mutex_write` to protect chunk access during meshing [S1, unofficial].
- **Precision:** Coordinate/Iterator types use `uint16_t` for chunk/voxel locations and `uint32_t` for worldspace vectors and iterators [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, unofficial].
- **Memory Complexity:** Not given in the sources [S1, unofficial].
- **Strengths:** Not given in the sources [S1, unofficial].
- **Limitations:** Not given in the sources [S1, unofficial].
- **When to Use:** Not given in the sources [S1, unofficial].
- **When Not to Use:** Not given in the sources [S1, unofficial].

## Background
This method is described in the kosshi-net/voxplat repository [S1, unofficial].

## Sources
[S1] kosshi-net/voxplat, https://github.com/kosshi-net/voxplat, fetched 2026-10-27 (unofficial)
