# Splat List (SVL) Generation with Multi-Level Downsampling

Also called: LOD Splatting, Sparse Voxel List

## Overview
Splat List (SVL) Generation with Multi-Level Downsampling is a rendering technique that generates a sparse voxel list (SVL) of splat geometry across multiple levels of detail (LOD) by iteratively downsampling a voxel occupancy mask and extracting splat primitives for each level [S1, unofficial]. It differs from standard hierarchical splatting by explicitly constructing a multi-resolution hierarchy through iterative mask downsampling rather than traversing a pre-existing spatial tree, allowing for efficient generation of LOD splats directly from voxel data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel occupancy mask for a chunk, represented as an array of `Voxel` (alias for `uint8_t`) [S1, unofficial].
*   **Output:** A contiguous SVL buffer containing geometry items for all LOD levels, and metadata in `struct ChunkMD` [S1, unofficial].

**Data Structures:**
The method relies on specific structures defined in the source implementation [S1, unofficial]:

1.  **`struct GeometrySVL`**: Represents a single splat LOD entry.
    *   `lod`: `uint8_t`
    *   `vao`: `uint32_t`
    *   `vbo`: `uint32_t`
    *   `vbo_items`: `uint32_t`
2.  **`struct ChunkMD`**: Represents a chunk's state and geometry data. Key fields relevant to SVL generation include:
    *   `svl`: `uint16_t*` (pointer to the SVL buffer)
    *   `svl_items[MAX_LOD_LEVEL]`: `uint32_t` (count of items per LOD level)
    *   `svl_items_total`: `uint32_t` (total item count)
    *   `no_geometry`: `volatile uint8_t`
    *   `svl_dirty`: `volatile uint8_t`
    *   `voxels`: `Voxel*` (pointer to voxel data)
    *   `count`: `uint32_t` (number of voxels)
3.  **`struct ChunkSet`**: Manages the set of chunks. Contains `gsvl[MAX_LOD_LEVEL]` (array of `struct GeometrySVL*`) and `chunks` (array of `struct ChunkMD*`) [S1, unofficial].

**Parameters:**
*   `MAX_LOD_LEVEL`: Constant defining the number of splat LOD levels, set to 5 [S1, unofficial].
*   `Voxel`: Type alias for `uint8_t` [S1, unofficial].
*   Coordinate/Iterator types: Chunk/voxel locations use `uint16_t`, worldspace vectors use `uint32_t`, iterators use `uint32_t` [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** For each chunk `c` in the `ChunkSet`, check if remeshing is required. If `c->remesh` is 0 and geometry is not dirty, skip. Otherwise, lock the chunk for writing [S1, unofficial].
2.  **Mask Generation:** Generate the initial voxel occupancy mask for the chunk using `chunk_make_mask` [S1, unofficial].
3.  **LOD Loop:** Iterate through LOD levels `i` from 0 to `MAX_LOD_LEVEL - 1`:
    a.  **Splat Extraction:** Extract splat primitives for the current level `i` from the current mask. Accumulate the geometry item counts into a running total `geom_items` [S1, unofficial].
    b.  **Downsampling:** After extracting splats for level `i`, downsample the current mask into a work buffer for level `i+1` using `chunk_mask_downsample`, unless `i` is the final LOD level (`i == MAX_LOD_LEVEL - 1`) [S1, unofficial].
4.  **Buffer Allocation:** Allocate a contiguous SVL buffer of size `geom_items * sizeof(uint16_t)` [S1, unofficial].
5.  **Data Copy:** Copy the accumulated geometry data from the mesher buffer into the allocated SVL buffer [S1, unofficial].
6.  **State Update:**
    *   If `geom_items` is zero:
        *   Free any existing SVL buffer.
        *   Zero out `c->svl_items`.
        *   Set `c->svl_items_total` to 0.
        *   Set `c->no_geometry` to 1.
        *   Mark `c->svl_dirty` to 1 [S1, unofficial].
    *   Otherwise, update `c->svl_items` and `c->svl_items_total` with the generated counts [S1, unofficial].
7.  **Cleanup:** Unlock the chunk and mark it as no longer dirty for meshing [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_LOD_LEVEL = 5
Voxel = uint8_t

# Data Structures (simplified for pseudocode)
class GeometrySVL:
    lod: uint8_t
    vao: uint32_t
    vbo: uint32_t
    vbo_items: uint32_t

class ChunkMD:
    voxels: Voxel*
    count: uint32_t
    svl: uint16_t*
    svl_items: array[uint32_t, MAX_LOD_LEVEL]
    svl_items_total: uint32_t
    no_geometry: uint8_t
    svl_dirty: uint8_t
    mesh_dirty: uint8_t
    remesh: uint8_t
    last_meshing: float

class ChunkSet:
    chunks: array[ChunkMD]
    null_chunk: ChunkMD

def generate_svl(set: ChunkSet, c: ChunkMD):
    # Check if remeshing is needed
    if c.remesh == 0 and not c.mesh_dirty:
        return

    # Lock chunk
    chunk_lock(set, c)

    # Ensure voxels are available
    if c.voxels is None:
        if c.voxels == set.null_chunk.voxels:
            c.voxels = mem_alloc(c.count * sizeof(Voxel))
            memcpy(c.voxels, set.null_chunk.voxels, c.count * sizeof(Voxel))
        else:
            # Handle RLE decompression if necessary (simplified)
            pass

    # Initialize buffers
    geom_buffer = mem_calloc(some_size)
    mask_buffer = mem_calloc(some_size)
    work_buffer = mem_calloc(some_size)

    # Step 1: Generate initial voxel occupancy mask
    chunk_make_mask(c, mask_buffer)

    geom_items = 0
    temp_lo_vbo_segments = [0] * MAX_LOD_LEVEL

    # Step 2: Iterate through LOD levels
    for i in range(MAX_LOD_LEVEL):
        # Step 3a: Extract splats for level i
        # This function populates geom_buffer and updates geom_items
        extract_splats_for_level(
            mask_buffer, 
            geom_buffer, 
            i, 
            &geom_items, 
            &temp_lo_vbo_segments[i]
        )

        # Step 3b: Downsample mask for next level
        if i < MAX_LOD_LEVEL - 1:
            chunk_mask_downsample(mask_buffer, work_buffer)
            # Swap buffers or copy work_buffer back to mask_buffer
            mask_buffer = work_buffer
            work_buffer = mem_calloc(some_size) # Reallocate or reuse

    # Step 4: Allocate SVL buffer
    if geom_items > 0:
        c.svl = mem_alloc(geom_items * sizeof(uint16_t))
        # Step 5: Copy geometry data
        memcpy(c.svl, geom_buffer, geom_items * sizeof(uint16_t))
        
        # Update metadata
        c.svl_items_total = geom_items
        # Copy per-level counts from temp_lo_vbo_segments or similar
        for i in range(MAX_LOD_LEVEL):
            c.svl_items[i] = temp_lo_vbo_segments[i]
        c.no_geometry = 0
    else:
        # Step 6: Handle empty geometry
        if c.svl is not None:
            mem_free(c.svl)
            c.svl = None
        for i in range(MAX_LOD_LEVEL):
            c.svl_items[i] = 0
        c.svl_items_total = 0
        c.no_geometry = 1
        c.svl_dirty = 1

    # Cleanup
    mem_free(geom_buffer)
    mem_free(mask_buffer)
    mem_free(work_buffer)

    # Mark as clean
    c.remesh = 0
    c.mesh_dirty = 0
    c.svl_dirty = 0 # Or keep dirty if upload pending? Source says set svl_dirty=1 if empty, implies clean if not?
    # Source says: "mark c->svl_dirty = 1" only if geom_items is zero.
    # Assuming if not zero, it's ready for upload, so maybe not dirty?
    # The source snippet: "If geom_items is zero, ... set c->svl_dirty = 1"
    # It does not explicitly say set to 0 if not zero, but logically it should be clean.
    # However, the source also has a check "if( c->svl_dirty || c->mesh_dirty ) continue;"
    # So if we just generated it, it might need to be marked dirty for upload?
    # The source snippet for the zero case sets svl_dirty=1.
    # For the non-zero case, it doesn't explicitly set svl_dirty.
    # Let's assume it's clean for meshing purposes but maybe needs upload.
    # The source code snippet shows:
    # if( c->svl_dirty || c->mesh_dirty ) continue;
    # ...
    # if( geom_items == 0 ) { ... c->svl_dirty = 1; }
    # It does NOT set c->svl_dirty = 0 in the non-zero case in the provided snippet.
    # However, the loop condition checks for svl_dirty.
    # If we don't set it to 0, the next frame will skip it.
    # This implies that either:
    # 1. The upload process clears svl_dirty.
    # 2. The snippet is incomplete.
    # Given the instruction to be complete, I will add a comment about this potential pitfall.
    # [G] Assuming the upload mechanism clears svl_dirty, or it should be cleared here.
    # To be safe, if the source doesn't specify, I will note it.
    # But the source says "mark c->svl_dirty = 1" ONLY for the zero case.
    # This suggests that for non-zero, it might NOT be dirty? Or the snippet is partial.
    # Let's stick to the source: only set to 1 if zero.
    # [G] In a real implementation, svl_dirty likely needs to be cleared after successful upload.
    # The source snippet doesn't show the upload code, only the generation.
    # I will leave svl_dirty state as per source: only explicitly set to 1 if empty.

    chunk_unlock(set, c)
```

**Edge Cases and Pitfalls:**
*   **Empty Geometry:** If `geom_items` is zero, the SVL buffer is freed, and `no_geometry` is set to 1. Failure to handle this can lead to invalid memory access or incorrect rendering state [S1, unofficial].
*   **Buffer Management:** The mesher uses a pool of buffers (`MESHER_THREAD_BUFFERS`). If all buffers are busy, the thread may skip processing. This requires careful synchronization [S1, unofficial].
*   **Precision:** The use of `uint16_t` for chunk/voxel locations limits the spatial resolution of the chunk coordinates. `uint32_t` is used for worldspace vectors, which may have implications for large scenes [S1, unofficial].
*   **Concurrency:** The method uses mutexes (`mutex_write`) to protect chunk access. Deadlocks or race conditions can occur if not handled correctly, especially with the `chunk_lock` and `chunk_unlock` calls [S1, unofficial].

## Subtopics

### Hierarchical Splatting
Uses a pre-built spatial hierarchy (e.g., octree) to select nodes for rendering based on screen-space footprint or error thresholds. Choose when the hierarchy is already built or when dynamic updates are frequent and localized. [Hierarchical Splatting](1168-hierarchical-splatting.md)

### Multi-Resolution Points
Focuses on point-based rendering with LOD, often using screen-space error metrics. Choose when point primitives are sufficient and high performance is needed for large point clouds. [Multi-Resolution Points](4784-multi-resolution-points.md)

### 3D Gaussian Splatting LOD methods
Extends 3D Gaussian Splatting with LOD techniques, typically involving covariance-based simplification. Choose when high-quality rendering of Gaussian primitives is required with LOD. [3D Gaussian Splatting LOD methods](5439-3d-gaussian-splatting-lod-methods.md)

### Octree-based Splatting
Uses an octree structure to manage voxel data and splat generation. Choose when the data has a natural octree structure or when efficient spatial queries are needed. [Octree-based Splatting](8067-octree-based-splatting.md)

## Operating characteristics
*   **Time Complexity:** The complexity is dominated by the mask generation and downsampling steps. Downsampling reduces the problem size geometrically, so the total work is roughly $O(N)$ where $N$ is the number of voxels in the finest level, as each level processes a fraction of the previous level's data.
*   **Memory Complexity:** The SVL buffer size is proportional to the number of splats generated, which is typically much less than the number of voxels. The mask buffers require $O(N)$ memory for the finest level, with less for subsequent levels.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales well with the number of LOD levels, as each level reduces the data size. The use of multiple threads (via OpenMP in the source) allows for parallel processing of chunks.
*   **Strengths:** Efficient generation of LOD splats directly from voxel data. Simple hierarchy construction through downsampling.
*   **Limitations:** Requires iterative downsampling, which may be less flexible than pre-built hierarchies. The fixed `MAX_LOD_LEVEL` of 5 may not be optimal for all scenes.
*   **When to use:** When rendering voxel-based data with LOD requirements and when a pre-built spatial hierarchy is not available or too costly to maintain.
*   **When not to use:** When the scene has a complex, non-voxel structure, or when dynamic updates require frequent rebuilding of the hierarchy.

## Background
The method is described in the context of the `voxplat` project by kosshi-net [S1, unofficial]. The specific implementation details, including data structures and algorithm steps, are derived from the source code of this project. The approach is a practical implementation of multi-resolution splatting for voxel data, leveraging iterative downsampling to generate LOD splats.

## Sources
[S1] kosshi-net/voxplat, https://github.com/kosshi-net/voxplat, fetched 2026-10-27 (unofficial)
