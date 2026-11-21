# libvxl Map Storage and Access

Also called: Ace of Spades map, Chunk-based Voxel Map Storage, chunked voxel grid, vxl map format

## Overview
libvxl Map Storage and Access is a sparse voxel map storage system for 'Ace of Spades' (Voxel Space) maps, using a compressed internal format derived from .vxl files [S1, unofficial]. It supports floating block detection and top-layer block retrieval [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Map dimensions ($W, H, D$), optional initial data (binary stream of `libvxl_span` descriptors), or empty initialization [S1, unofficial].
- Output: A `libvxl_map` structure containing chunked block data and a geometry bitmask array [S1, unofficial].

**Data Structures:**
The core structure `libvxl_map` contains:
- `width`, `height`, `depth`: Dimensions of the map [S1, unofficial].
- `chunks`: Pointer to an array of `libvxl_chunk` structures [S1, unofficial].
- `geometry`: Pointer to an array of `size_t` values acting as a bitmask for solid/empty status [S1, unofficial].
- `streamed`: Counter for streaming operations [S1, unofficial].

Each `libvxl_chunk` contains:
- `blocks`: Pointer to an array of `libvxl_block` structures [S1, unofficial].
- `length`: Current allocated capacity of the `blocks` array [S1, unofficial].
- `index`: Number of valid blocks currently stored [S1, unofficial].

Each `libvxl_block` contains:
- `position`: A 32-bit integer encoding $(x, y, z)$ coordinates. Format: `0xYYYXXXZZ` where X and Y are 12 bits (0-4095) and Z is 8 bits (0-255) [S1, unofficial].
- `color`: A 32-bit integer representing the block color [S1, unofficial].

The `libvxl_span` structure (used for input parsing) is a packed struct with:
- `length`: Total length of the span data [S1, unofficial].
- `color_start`: Z-coordinate of the start of the colored surface [S1, unofficial].
- `color_end`: Z-coordinate of the end of the colored surface [S1, unofficial].
- `air_start`: Z-coordinate where air runs begin [S1, unofficial].

**Parameters:**
- `LIBVXL_CHUNK_SIZE`: 16. Internal chunk size for splitting the map [S1, unofficial].
- `LIBVXL_CHUNK_GROWTH`: 2. Factor by which chunk buffer grows when full [S1, unofficial].
- `LIBVXL_CHUNK_SHRINK`: 4. Factor by which chunk buffer shrinks [S1, unofficial].
- `DEFAULT_COLOR`: 0x674028. Default color for solid blocks with no specific color (e.g., underground) [S1, unofficial].

**Steps:**

1. **Coordinate to Chunk Mapping:** Calculate chunk count: $C = \lceil W / \text{LIBVXL\_CHUNK\_SIZE} \rceil$. Calculate chunk indices: $c_x = \lfloor x / \text{LIBVXL\_CHUNK\_SIZE} \rfloor$, $c_y = \lfloor y / \text{LIBVXL\_CHUNK\_SIZE} \rfloor$. Return pointer to `map->chunks + c_x + c_y * C` [S1, unofficial].
2. **Geometry Bit Access:** Calculate linear offset: $O = z + (x + y \cdot W) \cdot D$. Access bit at $O \pmod{8 \cdot \text{sizeof}(size\_t)}$ in word at `map->geometry + O / (8 * sizeof(size_t))`. Return 1 if bit is set, 0 otherwise [S1, unofficial].
3. **Geometry Bit Set:** Calculate offset $O$ as above. Get pointer `val = map->geometry + O / (8 * sizeof(size_t))`. Get bit position `bit = O % (8 * sizeof(size_t))`. Update `*val = (*val & ~((size_t)1 << bit)) | (state << bit)` [S1, unofficial].
4. **Block Insertion/Update:** Perform binary search for `position` in `chunk->blocks`. If found, update `color`. If not found, find insertion point `start`. If `chunk->index == chunk->length`, grow buffer: `chunk->length *= LIBVXL_CHUNK_GROWTH`, `realloc` blocks. `memmove` blocks from `start+1` to shift space. Insert new block at `start`. Increment `chunk->index` [S1, unofficial].
5. **Block Lookup:** Binary search for `position` in `chunk->blocks`. If found, return pointer to block. If not found, return pointer to `chunk->blocks + start` (insertion point) [S1, unofficial].
6. **Span Length Calculation:** If `s->length > 0`, return `s->length * 4`. Otherwise, return `(s->color_end + 2 - s->color_start) * 4` [S1, unofficial].
7. **Map Creation and Initialization:** Calculate chunk grid dimensions: $sx = \lceil W / \text{LIBVXL\_CHUNK\_SIZE} \rceil$, $sy = \lceil H / \text{LIBVXL\_CHUNK\_SIZE} \rceil$. Allocate `map->chunks` and for each chunk, allocate `blocks` with initial capacity $\text{LIBVXL\_CHUNK\_SIZE} \times \text{LIBVXL\_CHUNK\_SIZE} \times 2$. Allocate `geometry` array with size $\lceil (W \cdot H \cdot D) / (8 \cdot \text{sizeof}(size\_t)) \rceil \cdot \text{sizeof}(size\_t)$ bytes, initialized to 0xFF (solid) or 0x00 (empty) [S1, unofficial].
8. **Parsing Input Data:** Read `libvxl_span` descriptors. For each span at $(x,y)$, set geometry to 0 for $z \in [\text{air\_start}, \text{color\_start})$. Set color for $z \in [\text{color\_start}, \text{color\_end}]$. Handle bottom runs if `desc->length > 0` [S1, unofficial].
9. **Boundary Conditions:** Check `libvxl_geometry_get` at map edges. Insert default colors if a solid voxel is adjacent to an empty one at the boundary [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Memory and type helpers (simulating C behavior in Python)
WORD_BITS = 64  # sizeof(size_t) * 8 on 64-bit systems
def realloc(ptr, new_size):
    if len(ptr) < new_size:
        ptr.extend([None] * (new_size - len(ptr)))
    return ptr
def sizeof_block():
    return 8  # Simulated size of libvxl_block

def libvxl_chunk_fposition(map, x, y):
    chunk_cnt = (map.width + 16 - 1) // 16
    chunk_x = x // 16
    chunk_y = y // 16
    return map.chunks[chunk_x + chunk_y * chunk_cnt]

def libvxl_geometry_get(map, x, y, z):
    if x >= map.width or y >= map.height or z >= map.depth:
        return False
    offset = z + (x + y * map.width) * map.depth
    word_idx = offset // WORD_BITS
    bit_idx = offset % WORD_BITS
    return (map.geometry[word_idx] & (1 << bit_idx)) > 0

def libvxl_geometry_set(map, x, y, z, state):
    if x >= map.width or y >= map.height or z >= map.depth:
        return
    offset = z + (x + y * map.width) * map.depth
    word_idx = offset // WORD_BITS
    bit_idx = offset % WORD_BITS
    map.geometry[word_idx] = (map.geometry[word_idx] & ~(1 << bit_idx)) | (state << bit_idx)

def libvxl_chunk_insert(chunk, pos, color):
    if chunk.index == chunk.length:
        chunk.length *= 2
        chunk.blocks = realloc(chunk.blocks, chunk.length * sizeof_block())
        
    start = 0
    end = chunk.index
    while end > start:
        mid = (start + end) // 2
        if pos > chunk.blocks[mid].position:
            start = mid + 1
        elif pos < chunk.blocks[mid].position:
            end = mid
        else:
            chunk.blocks[mid].color = color
            return
            
    if start < chunk.index:
        # Shift blocks from start..index-1 to start+1..index
        chunk.blocks[start+1 : start+1 + (chunk.index - start)] = chunk.blocks[start : chunk.index]
        
    chunk.blocks[start].position = pos
    chunk.blocks[start].color = color
    chunk.index += 1

def libvxl_chunk_gequal_block(chunk, pos):
    start = 0
    end = chunk.index
    while end > start:
        mid = (start + end) // 2
        if pos > chunk.blocks[mid].position:
            start = mid + 1
        elif pos < chunk.blocks[mid].position:
            end = mid
        else:
            return chunk.blocks[mid]
    return chunk.blocks[start]
```

**Pitfalls:**
- Out of bounds access: Functions like `libvxl_map_issolid`, `libvxl_map_get`, and `libvxl_map_set` check bounds. `libvxl_map_issolid` returns non-solid (0) for out-of-bounds. `libvxl_map_gettop` leaves result unmodified for out-of-bounds. `libvxl_map_set` does nothing for out-of-bounds [S1, unofficial].
- Coordinate System: All functions use voxlap's coordinate system where X and Y are 12 bits (0-4095) and Z is 8 bits (0-255) in the key format `0xYYYXXXZZ` [S1, unofficial].
- Chunk growth triggers when `chunk->index == chunk->length`, multiplying length by `LIBVXL_CHUNK_GROWTH` and reallocating blocks [S1, unofficial].

## Subtopics

### [Chunked Voxel Map Serialization with Visfaces](4201-chunked-voxel-map-serialization-with-visfaces.md)
This method serializes a chunked voxel map into a compact file format by iterating through the spatial grid and computing a 6-bit visibility face mask (visfaces) for each block based on its immediate solid neighbors.

### [Span-based Column Encoding](4365-span-based-column-encoding.md)
Span-based Column Encoding encodes a single $(x,y)$ column of a sparse voxel map into a sequence of `libvxl_span` structures containing air runs, top color runs, and bottom color runs. It solves the problem of efficiently serializing sparse voxel data by reducing the storage of empty space and surface boundaries to compact run-length encoded structures.

## Operating characteristics
- **Time Complexity:** Block lookup and insertion are $O(\log N)$ where $N$ is the number of blocks in a chunk, due to binary search [S1, unofficial]. Geometry access is $O(1)$ [S1, unofficial].
- **Memory Complexity:** Storage is proportional to the number of blocks plus the geometry bitmask [S1, unofficial].
- **Performance:** Lower chunk sizes (e.g., 16) speed up access but increase memory fragmentation [S1, unofficial].
- **Strengths:** Fast direct access to blocks, support for floating block detection, and efficient serialization via column encoding [S1, unofficial].
- **Limitations:** Sparse data may still consume significant memory if not properly chunked. Boundary conditions require explicit handling [S1, unofficial].
- **When to Use:** Suitable for voxel-based games and simulations requiring frequent block updates and queries [G].
- **When Not to Use:** Not ideal for highly dense, fully populated voxel worlds where a simple 3D array might be more efficient [G].

## Background
libvxl is a library for reading and writing Ace of Spades map files easily and fast [S1, unofficial]. It uses a compressed internal format derived from .vxl files [S1, unofficial]. It supports enhanced features for special cases such as floating block detection and getting the top layer block, useful for map overviews [S1, unofficial]. All functions use voxlap's coordinate system where X and Y are 12 bits (0-4095) and Z is 8 bits (0-255) in the key format `0xYYYXXXZZ` [S1, unofficial].

## Sources
[S1] xtreme8000/libvxl, https://github.com/xtreme8000/libvxl, fetched 2026-10-28 (unofficial)
