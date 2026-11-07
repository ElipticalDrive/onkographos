# Voxel Map Lookup


## Overview
Retrieves the color of a voxel at $(x,y,z)$ coordinates within a chunked voxel map [S1, unofficial]. It solves the problem of efficient sparse voxel access by combining a global geometry bitfield for quick existence checks with a binary search over sorted blocks within a specific chunk [S1, unofficial]. It differs from its parent [Chunked Voxel Map Serialization with Visfaces](4201-chunked-voxel-map-serialization-with-visfaces.md) by providing random-access retrieval rather than sequential serialization, and from [Top Voxel Retrieval](4236-top-voxel-retrieval.md) by returning the exact voxel color rather than the highest surface voxel in a column [S1, unofficial].

## How it works
**Inputs:**
- `map`: A pointer to a `libvxl_map` structure containing `width`, `height`, `depth`, a `geometry` bitfield, and an array of `libvxl_chunk` structures [S1, unofficial].
- `x, y, z`: Integer coordinates of the target voxel [S1, unofficial].

**Outputs:**
- `uint32_t`: The color of the voxel if found and solid; otherwise, `DEFAULT_COLOR(x,y,z)` or `0` [S1, unofficial].

**Data Structures:**
- `libvxl_map`: `width`, `height`, `depth` (dimensions), `chunks` (array of `libvxl_chunk`), `geometry` (array of `size_t` bitfield), `streamed` [S1, unofficial].
- `libvxl_chunk`: `blocks` (pointer to `libvxl_block` array), `length`, `index` (number of valid blocks) [S1, unofficial].
- `libvxl_block`: `position` (`uint32_t` key), `color` (`uint32_t`) [S1, unofficial].

**Key Encoding:**
Positions are encoded into a 32-bit integer using the macro `pos_key(x,y,z)`:
$$ \text{key} = (y \ll 20) \mid (x \ll 8) \mid z $$
This allocates 12 bits for $x$ (bits 8–19), 12 bits for $y$ (bits 20–31), and 8 bits for $z$ (bits 0–7) [S1, unofficial].

**Steps:**
1. **Bounds Check:** If `map` is null, or $x, y, z$ are outside $[0, \text{width}-1] \times [0, \text{height}-1] \times [0, \text{depth}-1]$, return `0` [S1, unofficial].
2. **Geometry Check:** Call `libvxl_geometry_get(map, x, y, z)`. This checks if the corresponding bit in `map->geometry` is set. If the bit is not set, return `0` [S1, unofficial].
3. **Chunk Selection:** Calculate the chunk index using `chunk_fposition(x, y, z)`. Retrieve the `libvxl_chunk` structure at that index [S1, unofficial].
4. **Binary Search:** Perform a binary search on `chunk->blocks` to find a block where `block.position == pos_key(x,y,z)` [S1, unofficial].
5. **Result:** If found, return `block.color`. If not found (block is solid in geometry but missing from chunk data), return `DEFAULT_COLOR(x,y,z)` [S1, unofficial].

**Parameters:**
- `DEFAULT_COLOR(x,y,z)`: Defined as `0x674028` [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Missing Block:** If the geometry bitfield indicates a voxel is solid but the block is not present in the chunk's sorted array, the method returns `DEFAULT_COLOR` instead of `0`. This can cause visual artifacts if the background is expected to be transparent or black [S1, unofficial].
- **Precision:** The key encoding limits $x$ and $y$ to 12 bits (max 4095) and $z$ to 8 bits (max 255). Coordinates outside the map bounds are rejected by the initial bounds check [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def libvxl_map_get(map, x, y, z):
    # 1. Bounds check
    if map is None or x < 0 or x >= map.width or \
       y < 0 or y >= map.height or z < 0 or z >= map.depth:
        return 0

    # 2. Geometry check
    if not libvxl_geometry_get(map, x, y, z):
        return 0

    # 3. Chunk selection
    chunk = chunk_fposition(map, x, y, z)
    if chunk is None:
        return 0

    # 4. Binary search
    key = (y << 20) | (x << 8) | z
    low = 0
    high = chunk.index - 1
    found_color = 0x674028  # DEFAULT_COLOR

    while low <= high:
        mid = (low + high) // 2
        block = chunk.blocks[mid]
        if block.position == key:
            found_color = block.color
            break
        elif block.position < key:
            low = mid + 1
        else:
            high = mid - 1

    # 5. Result
    return found_color

def libvxl_geometry_get(map, x, y, z):
    # Linear index calculation matching libvxl's layout
    offset = z + (x + y * map.width) * map.depth
    byte_index = offset // 8
    bit_index = offset % 8
    return (map.geometry[byte_index] & (1 << bit_index)) != 0

def chunk_fposition(map, x, y, z):
    # Standard chunk indexing based on world dimensions and chunk size [G]
    # Assumes fixed chunk dimensions or a provided chunk_size parameter in practice
    # Returns the chunk pointer at the calculated index
    chunk_idx = (z // CHUNK_DEPTH) * map.chunks_per_row + (y // CHUNK_HEIGHT) * map.chunks_per_col + (x // CHUNK_WIDTH)
    if chunk_idx < len(map.chunks):
        return map.chunks[chunk_idx]
    return None
```

## Subtopics

### Chunk Copy
Copies a map's geometry bitfield and a specific chunk's block array into an isolated `libvxl_chunk_copy` structure [S1, unofficial]. It allows for read-only or isolated access to a subset of the world [G]. It calculates the geometry size as $sg = \lceil \frac{\text{width} \times \text{height} \times \text{depth}}{8 \times \text{sizeof(size_t)}} \rceil \times \text{sizeof(size_t)}$ bytes [S1, unofficial]. It allocates `copy->blocks_sorted` of size `c->index * sizeof(struct libvxl_block)` [S1, unofficial]. The helper `libvxl_copy_chunk_is_solid` checks the copied geometry bitfield directly [S1, unofficial]. If the copy is null or coordinates are out of bounds, it returns `true` (solid) [S1, unofficial].

### Voxel Setting — libvxl_map_set
Sets a voxel to solid with a given color and automatically removes non-surface neighbors to prevent floating geometry [S1, unofficial]. It sets the geometry bit to 1, calls `libvxl_map_set_internal`, and then iterates over the 6 neighbors [S1, unofficial]. If a neighbor is not on the surface (`!libvxl_map_onsurface`), it calls `libvxl_map_setair_internal` to remove it [S1, unofficial]. A pitfall is that neighbors already on the surface are preserved even if they become isolated, maintaining structural integrity [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(\log N)$ for the binary search within a chunk, where $N$ is the number of blocks [G]. Bounds and geometry checks are $O(1)$ [G].
- **Memory Complexity:** $O(1)$ auxiliary space for lookup [G]. The geometry bitfield scales with world volume ($W \times H \times D / 8$ bytes) [G].
- **Strengths:** Fast random access to sparse voxel data. The geometry bitfield provides a quick rejection for air blocks [G].
- **Limitations:** The fixed key encoding limits world size ($x,y \le 4095, z \le 255$). The `DEFAULT_COLOR` fallback can mask missing data bugs [S1, unofficial].

## Background
Originated in the `libvxl` library by xtreme8000, a C library for handling voxel data in the VXL format [S1, unofficial]. It is used in voxel-based game engines and editors for efficient map manipulation [G]. The method relies on the specific binary serialization format where chunks contain sorted block arrays [S1, unofficial].

## Sources
[S1] xtreme8000/libvxl, https://github.com/xtreme8000/libvxl, fetched 2026-10-28 (unofficial)
