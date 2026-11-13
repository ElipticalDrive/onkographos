# Top Voxel Retrieval


## Overview
Top Voxel Retrieval finds the highest solid voxel at a given $(x, y)$ column within a chunked voxel map and returns its color and $z$-coordinate [S1, unofficial]. It builds on [Voxel Map Lookup](4226-voxel-map-lookup.md) by restricting the search to the vertical axis to identify the surface for lighting or shadowing calculations [G].

## How it works
**Inputs and Outputs:**
- Input: A pointer to the `libvxl_map` structure, and integer coordinates $(x, y)$ representing the column.
- Output: The color and $z$-coordinate of the highest solid voxel. If no solid voxel exists, the function triggers an assertion [S1, unofficial].

**Data Structures:**
The map is stored in a binary format. The core structure `libvxl_map` contains:
- `size_t* chunk_offsets`: An array mapping chunk indices to byte offsets in the buffer.
- `size_t chunk_size`: The dimensions of a single chunk.
- `void* buffer`: The raw memory buffer containing serialized chunk data.
- `size_t buffer_offset`: Current read position in the buffer.
- `size_t pos`: Current position in the chunk offsets array.

Each chunk contains a header `libvxl_kv6` and a series of blocks `libvxl_kv6_block`. The block structure is packed:
- `int color`: 32-bit integer representing the voxel color.
- `short z`: 16-bit signed integer representing the $z$-coordinate.
- `unsigned char visfaces`: 8-bit visibility flags.
- `unsigned char normal`: 8-bit normal vector encoding.

**Steps:**
1. **Locate Chunk:** Compute chunk indices `chunk_x = global_x // chunk_size` and `chunk_y = global_y // chunk_size`. Retrieve the byte offset from `map_struct.chunk_offsets[chunk_y * chunks_per_row + chunk_x]` [G].
2. **Parse Header:** Read the 32-byte `libvxl_kv6` header at the chunk offset to obtain `width`, `height`, `depth`, and `len` [S1, unofficial].
3. **Local Coordinates:** Compute `local_x = global_x % chunk_size` and `local_y = global_y % chunk_size` [G].
4. **Search Blocks:** Iterate `z` from `depth - 1` down to `0`. For each `z`, compute the block index in the packed 3D layout: `block_idx = (z * header.height + local_y) * header.width + local_x` [G]. Read the 8-byte `libvxl_kv6_block` at `buffer_offset + block_idx * 8` [S1, unofficial].
5. **Solid Check:** Determine if the block is solid by checking if `normal != 0` or `visfaces != 0` [G].
6. **Return Result:** Return `color` and `z` of the first solid block found. If the loop completes without finding a solid block, trigger an assertion [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_top_voxel(map_struct, global_x, global_y, chunks_per_row):
    chunk_size = map_struct.chunk_size
    chunk_x = global_x // chunk_size
    chunk_y = global_y // chunk_size
    chunk_offset = map_struct.chunk_offsets[chunk_y * chunks_per_row + chunk_x]
    
    # Header size is 32 bytes: magic(4) + width(4) + height(4) + depth(4) + pivot(12) + len(4)
    header_width = int.from_bytes(map_struct.buffer[chunk_offset+4:chunk_offset+8], 'little')
    header_height = int.from_bytes(map_struct.buffer[chunk_offset+8:chunk_offset+12], 'little')
    header_depth = int.from_bytes(map_struct.buffer[chunk_offset+12:chunk_offset+16], 'little')
    
    local_x = global_x % chunk_size
    local_y = global_y % chunk_size
    blocks_start = chunk_offset + 32
    
    for z in range(header_depth - 1, -1, -1):
        # [G] Assumes packed 3D layout: block index = (z * height + y) * width + x
        block_idx = (z * header_height + local_y) * header_width + local_x
        block_off = blocks_start + block_idx * 8
        
        color = int.from_bytes(map_struct.buffer[block_off:block_off+4], 'little')
        block_z = int.from_bytes(map_struct.buffer[block_off+4:block_off+6], 'little')
        visfaces = map_struct.buffer[block_off+6]
        normal = map_struct.buffer[block_off+7]
        
        if block_z == z and (normal != 0 or visfaces != 0):
            return color, block_z
            
    raise AssertionError("No solid voxel found at column")
```

**Parameters:**
- `chunks_per_row`: Number of chunks along the X-axis. Not given in the sources.
- `z` range: $0$ to `depth - 1`.

**Edge Cases:**
- **No Solid Voxel:** If the column is empty, the function triggers an assertion [S1, unofficial].
- **Precision:** Coordinates are integers. No floating-point issues.
- **Boundaries:** Ensure global $(x, y)$ is within the map bounds before lookup [G].

## Subtopics

### [2D Voxel Shadow Map](5769-2d-voxel-shadow-map.md)
A 2D Voxel Shadow Map stores the highest occupied voxel Y-coordinate per (X, Z) column to determine occlusion for lighting without ray marching. It builds on [Top Voxel Retrieval](4236-top-voxel-retrieval.md) to provide occlusion data [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ per column lookup when using the packed 3D layout indexing, or $O(N)$ in worst-case scenarios where $N$ is the number of blocks in the chunk if scanning linearly [G].
- **Memory Complexity:** $O(1)$ additional memory for the retrieval itself [G].
- **Performance:** Fast for surface queries. Slower if chunks are not cached [G].
- **Strengths:** Efficient for lighting calculations where only the surface matters [G].
- **Limitations:** Only returns the top voxel. Does not provide information about lower voxels [G].

## Background
Top Voxel Retrieval is part of the libvxl library ecosystem [S1, unofficial]. It is used in voxel-based games and simulations where efficient surface queries are required for rendering and physics [G].

## Sources
[S1] xtreme8000/libvxl, https://github.com/xtreme8000/libvxl, fetched 2026-10-28 (unofficial)
