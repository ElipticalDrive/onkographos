# Top Voxel Retrieval

## Overview
Top Voxel Retrieval finds the highest solid voxel at a given $(x, y)$ column within a chunked voxel map and returns its color and $z$-coordinate [S1, unofficial]. It builds on [Voxel Map Lookup](4226-voxel-map-lookup.md) by restricting the search to the vertical axis to identify the surface for lighting or shadowing calculations. Unlike standard lookup which retrieves arbitrary $(x,y,z)$ data, this method performs a directional search to find the first occupied block from the top down.

## How it works
**Inputs and Outputs:**
- Input: A pointer to the `libvxl_map` structure, and integer coordinates $(x, y)$ representing the column.
- Output: The color and $z$-coordinate of the highest solid voxel. If no solid voxel exists, the behavior is undefined or triggers an assertion [S1, unofficial].

**Data Structures:**
The map is stored in a binary format. The core structure `libvxl_map` contains:
- `size_t* chunk_offsets`: An array mapping chunk indices to byte offsets in the buffer.
- `size_t chunk_size`: The dimensions of a single chunk (width, height, depth).
- `void* buffer`: The raw memory buffer containing serialized chunk data.
- `size_t buffer_offset`: Current read position in the buffer.
- `size_t pos`: Current position in the chunk offsets array.

Each chunk contains a header `libvxl_kv6` and a series of blocks `libvxl_kv6_block`. The block structure is packed:
- `int color`: 32-bit integer representing the voxel color.
- `short z`: 16-bit signed integer representing the $z$-coordinate.
- `unsigned char visfaces`: 8-bit visibility flags.
- `unsigned char normal`: 8-bit normal vector encoding.

**Steps:**
1. **Locate Chunk:** Determine which chunk contains the global $(x, y)$ coordinate. Use the `chunk_offsets` array to find the byte offset in `buffer` where the target chunk begins.
2. **Parse Header:** Read the `libvxl_kv6` header to obtain `width`, `height`, and `depth` of the chunk.
3. **Local Coordinates:** Calculate local $(x', y', z')$ coordinates within the chunk. Since we are searching for the top voxel, we iterate $z$ from `depth - 1` down to $0$.
4. **Search Blocks:** Iterate through the blocks in the chunk. The blocks are not necessarily sorted by $z$ in the raw file, but the lookup function `libvxl_chunk_gequal_block` suggests a binary search or lower-bound search is used for efficiency [S1, unofficial]. However, for "Top" retrieval, one must find the block with the maximum $z$ such that the voxel at $(x, y, z)$ is solid.
5. **Solid Check:** For each candidate block at local $(x', y')$, check if it is solid. The function `libvxl_copy_chunk_is_solid` can be used if the chunk is loaded into memory, or check the `visfaces`/`normal` fields in the raw format [S1, unofficial].
6. **Return Result:** Return the `color` and `z` of the first solid block found starting from the highest $z$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_top_voxel(map_struct, global_x, global_y):
    # 1. Determine chunk index
    # Assuming chunk_size is uniform and known
    chunk_size = map_struct.chunk_size
    chunk_x = global_x // chunk_size.width
    chunk_y = global_y // chunk_size.height
    
    # 2. Get chunk offset
    chunk_offset = map_struct.chunk_offsets[chunk_y * (map_struct.width // chunk_size.width) + chunk_x]
    
    # 3. Read chunk header
    header = read_kv6(map_struct.buffer, chunk_offset)
    
    # 4. Calculate local coordinates
    local_x = global_x % chunk_size.width
    local_y = global_y % chunk_size.height
    
    # 5. Search for top solid voxel
    # Iterate from top z down to 0
    for z in range(header.depth - 1, -1, -1):
        # Find block at (local_x, local_y, z)
        # In raw format, blocks might be stored in a specific order.
        # If using libvxl_copy_chunk interface:
        # copy = load_chunk(map_struct, chunk_x, chunk_y)
        # if libvxl_copy_chunk_is_solid(copy, local_x, local_y, z):
        #     color = libvxl_copy_chunk_get_color(copy, local_x, local_y, z)
        #     libvxl_copy_chunk_destroy(copy)
        #     return color, z
        
        # Raw implementation assumption:
        # Blocks are stored sequentially. We need to find the block matching (local_x, local_y, z).
        # This requires parsing the block list.
        block = find_block_at(map_struct.buffer, chunk_offset + header_size, local_x, local_y, z)
        if block is not None and is_solid(block):
            return block.color, block.z
            
    # No solid voxel found
    raise AssertionError("No solid voxel found at column")

def is_solid(block):
    # Check if the block represents a solid voxel
    # Based on libvxl_kv6_block, visfaces or normal might indicate emptiness
    # If normal is 0 or visfaces indicates no faces, it might be empty.
    # Specific logic depends on the serialization format details not fully provided.
    # [G] Assuming a non-zero normal or specific visface flag indicates solidity.
    return block.normal != 0
```

**Parameters:**
- `chunk_size`: Typically $16 \times 16 \times 16$ or $32 \times 32 \times 32$ [S1, unofficial].
- `z` range: $0$ to `depth - 1`.

**Edge Cases:**
- **No Solid Voxel:** If the column is empty, the function should handle this. The source notes mention assertions will trigger if `libvxl_chunk_gequal_block` fails to find a block [S1, unofficial].
- **Precision:** Coordinates are integers. No floating-point issues.
- **Boundaries:** Ensure global $(x, y)$ is within the map bounds before lookup.

## Subtopics

### [2D Voxel Shadow Map](5769-2d-voxel-shadow-map.md)
A 2D Voxel Shadow Map stores the highest occupied voxel Y-coordinate per (X, Z) column to determine occlusion for lighting without ray marching. It builds on [Top Voxel Retrieval](4236-top-voxel-retrieval.md) to provide occlusion data [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ if the chunk is pre-loaded and indexed, or $O(\log N)$ if binary search is used within the chunk [S1, unofficial]. In raw format, it may be $O(N)$ where $N$ is the number of blocks in the chunk if no index is maintained.
- **Memory Complexity:** $O(1)$ additional memory if using pre-loaded chunk structures. Raw access requires minimal memory.
- **Performance:** Fast for surface queries. Slower if chunks are not cached.
- **Strengths:** Efficient for lighting calculations where only the surface matters.
- **Limitations:** Only returns the top voxel. Does not provide information about lower voxels.

## Background
Top Voxel Retrieval is part of the libvxl library ecosystem, a C library for voxel data handling [S1, unofficial]. It is used in voxel-based games and simulations where efficient surface queries are required for rendering and physics.

## Sources
[S1] xtreme8000/libvxl, https://github.com/xtreme8000/libvxl, fetched 2026-10-28 (unofficial)
