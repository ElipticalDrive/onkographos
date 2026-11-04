# Span-based Column Encoding

Also called: VXL span serialization
## Overview
Span-based Column Encoding encodes a single $(x,y)$ column of a sparse voxel map into a sequence of `libvxl_span` structures containing air runs, top color runs, and bottom color runs [S1, unofficial]. It solves the problem of efficiently serializing sparse voxel data by reducing the storage of empty space and surface boundaries to compact run-length encoded structures [S1, unofficial].
## How it works
**Inputs:** A `libvxl_map` structure containing `width`, `height`, `depth`, and sparse block storage; coordinates $(x,y)$ defining the column to encode [S1, unofficial].
**Outputs:** A sequence of `libvxl_span` structures written to a buffer [S1, unofficial].
**Data Structures:**
* `libvxl_span`: Contains fields `color_start`, `color_end`, `air_start`, `length`, and `offset` [S1, unofficial].
* `libvxl_block`: Contains `position` (packed key) and `color` (uint32_t) [S1, unofficial].
* `position` key: Packed integer where $z$ is extracted via `key_getz()` and the rest via `key_discardz()` [S1, unofficial].
**Steps:**
1. Initialize `first_run = true`, get starting $z$ from the first block in the chunk [S1, unofficial].
2. Determine `top_start`: if `libvxl_geometry_get(map, x, y, z)` is true, `top_start = z`, else `top_start = key_getz(first_block->position)` [S1, unofficial].
3. Call `find_successive_surface` to find `top_end` and `last_surface_block` [S1, unofficial].
4. Determine `bottom_start`: if `top_end == map->depth` or `!libvxl_geometry_get(map, x, y, top_end)`, `bottom_start = top_end`. Else if `last_surface_block` points to a block at $z=0$, `bottom_start = key_getz(last_surface_block->position)` [S1, unofficial].
5. Write `desc->color_start = top_start`, `desc->color_end = top_end - 1`, `desc->air_start = first_run ? 0 : z`. Advance offset [S1, unofficial].
6. Write top colors: loop $k$ from `top_start` to `top_end`, write `(block->color & 0xFFFFFF) | 0x7F000000`. Advance offset [S1, unofficial].
7. If `bottom_end < map->depth`, set `desc->length = 1 + top_end - top_start + bottom_end - bottom_start`, write bottom colors, and set `z = bottom_end`. Else set `desc->length = 1 + top_end - top_start` and `z = bottom_start` [S1, unofficial].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_column(map, x, y):
    first_run = True
    chunk = map.chunks[y * map.width + x]
    if chunk.index == 0:
        return []
    first_block = next((b for b in chunk.blocks if b.position >= pos_key(x, y, 0)), None)
    z = key_getz(first_block.position)

    if libvxl_geometry_get(map, x, y, z):
        top_start = z
    else:
        top_start = key_getz(first_block.position)

    top_end = top_start
    last_surface_block = None
    while top_end < map.depth and libvxl_geometry_get(map, x, y, top_end):
        last_surface_block = get_block_at(map, x, y, top_end)
        top_end += 1

    if top_end == map.depth or not libvxl_geometry_get(map, x, y, top_end):
        bottom_start = top_end
    elif last_surface_block and key_getz(last_surface_block.position) == 0:
        bottom_start = key_getz(last_surface_block.position)
    else:
        bottom_start = top_end

    desc = libvxl_span()
    desc.color_start = top_start
    desc.color_end = top_end - 1
    desc.air_start = 0 if first_run else z
    desc.offset = 0
    spans = [desc]

    for k in range(top_start, top_end):
        block = get_block_at(map, x, y, k)
        if block:
            write_color((block.color & 0xFFFFFF) | 0x7F000000)

    bottom_end = bottom_start
    while bottom_end < map.depth and libvxl_geometry_get(map, x, y, bottom_end):
        bottom_end += 1

    if bottom_end < map.depth:
        desc.length = 1 + (top_end - top_start) + (bottom_end - bottom_start)
        for k in range(bottom_start, bottom_end):
            block = get_block_at(map, x, y, k)
            if block:
                write_color(block.color)
        z = bottom_end
    else:
        desc.length = 1 + (top_end - top_start)
        z = bottom_start

    return spans

# Helper definitions [G]
class libvxl_span:
    color_start = 0; color_end = 0; air_start = 0; length = 0; offset = 0
class libvxl_block:
    position = 0; color = 0
def pos_key(x, y, z): return (x << 24) | (y << 16) | z
def key_getz(pos): return pos & 0xFFFF
def libvxl_geometry_get(map, x, y, z): return map.geometry.get((x, y, z), False)
def get_block_at(map, x, y, z):
    chunk = map.chunks[y * map.width + x]
    return next((b for b in chunk.blocks if b.position == pos_key(x, y, z)), None)
def write_color(c): pass
```
**Parameters:**
* `map->depth`: Total height of the voxel map [S1, unofficial].
* `0x7F000000`: Alpha mask for surface blocks [S1, unofficial].
**Edge Cases:**
* **Empty Column:** If no blocks exist in the chunk, the function returns an empty list [S1, unofficial].
* **Boundary Conditions:** `top_end` can equal `map->depth` [S1, unofficial]. `bottom_start` logic depends on whether the surface touches the bottom ($z=0$) or ends at the top [S1, unofficial].
* **Precision:** Color packing uses bitwise AND with `0xFFFFFF` to mask RGB, preserving only the lower 24 bits [S1, unofficial].
## Subtopics

### [Voxel Map Streaming](4946-voxel-map-streaming.md)
Voxel Map Streaming is a sequential I/O method for dense voxel maps that outputs fixed-size binary chunks by iterating over $(x,y)$ columns. It solves the problem of efficient bulk transfer of voxel data by buffering encoded columns into a sliding window, allowing consumers to read data in predictable, fixed-size blocks without managing individual column boundaries. It differs from direct column access by abstracting the underlying [Span-based Column Encoding](4365-span-based-column-encoding.md) into a continuous byte stream with internal buffering and chunking logic.

## Operating characteristics
* **Time Complexity:** Linear with respect to the number of blocks in the column, as it iterates through surface blocks to determine runs [G].
* **Memory Complexity:** Proportional to the number of runs in the column [G].
* **Strengths:** Efficiently compresses sparse columns by ignoring large air gaps [G]. Explicitly handles surface colors with alpha masking [G].
* **Limitations:** Requires random access to blocks within a column to determine geometry and colors [G]. Not suitable for horizontal traversal without re-encoding [G].
* **When to Use:** When serializing voxel maps for storage or network transmission where column-wise access is natural [G].
* **When Not to Use:** When horizontal slicing or face-based mesh generation is required [G].
## Background
Span-based Column Encoding is part of the `libvxl` library, an implementation of the Voxel Space map format used in the game *Ace of Spades* [S1, unofficial]. It was developed by xtreme8000 to provide an efficient internal representation for sparse voxel data [S1, unofficial]. The method derives from the `.vxl` file format specifications, focusing on compact serialization of columnar data [S1, unofficial].
## Sources
[S1] xtreme8000/libvxl, https://github.com/xtreme8000/libvxl, fetched 2026-10-28 (unofficial)
