# Top Voxel Retrieval

## Overview
Top Voxel Retrieval finds the highest solid voxel at a given $(x,y)$ column within a chunked voxel map and returns its color and $z$-coordinate [S1, unofficial]. It differs from standard [Voxel Map Lookup](4226-voxel-map-lookup.md) by performing a vertical search for the maximum occupied $z$ rather than retrieving a specific $z$, and from [2D Voxel Shadow Map](5769-2d-voxel-shadow-map.md) by providing the raw data rather than a pre-computed texture [S1, unofficial].

## How it works
**Inputs:** A pointer to the `libvxl_map` structure, and integer coordinates $(x, y)$ defining the column.
**Outputs:** The color (integer) and $z$-coordinate of the topmost solid voxel. If no solid voxel exists, the behavior is undefined per source notes.

**Data Structures:**
The map is defined as [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```c
struct libvxl_map {
    size_t* chunk_offsets;
    size_t chunk_size;
    void* buffer;
    size_t buffer_offset;
    size_t pos;
};
```
Blocks within chunks are packed as [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```c
struct __attribute__((packed)) libvxl_kv6_block {
    int color;
    short z;
    unsigned char visfaces;
    unsigned char normal;
};
```
The `geometry` bitfield in a chunk copy indicates solid voxels.

**Steps:**
1.  Determine the chunk index containing $(x, y)$ using `chunk_offsets` and `chunk_size` [S1, unofficial].
2.  Load the chunk data into a `libvxl_chunk_copy` structure using `libvxl_copy_chunk` [S1, unofficial].
3.  Perform a lower-bound search for blocks where $z \ge 0$ using `libvxl_chunk_gequal_block` [S1, unofficial].
4.  Identify the block with the highest $z$ value among those marked solid in the `geometry` bitfield.
5.  Return the `color` and `z` fields of that block.

**Pitfalls:**
If no block exists satisfying the condition, assertions will trigger [S1, unofficial]. Precision issues may arise if $z$ values are not strictly monotonic in the sorted array, though the source implies a sorted structure.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_top_voxel(map: libvxl_map, x: int, y: int) -> tuple[int, int]:
    # 1. Find chunk index
    chunk_idx = find_chunk_index(map.chunk_offsets, x, y, map.chunk_size)
    
    # 2. Load chunk
    copy = libvxl_chunk_copy()
    libvxl_copy_chunk(map, copy, x, y)
    
    # 3. Search for top solid block
    # Note: libvxl_chunk_gequal_block performs lower-bound search for z >= 0
    # We must iterate or binary search the blocks_sorted array
    top_z = -1
    top_color = 0
    
    for i in range(copy.blocks_sorted_count):
        block = copy.blocks_sorted[i]
        # Check if solid using geometry bitfield
        if is_solid(copy.geometry, x, y, block.z):
            if block.z > top_z:
                top_z = block.z
                top_color = block.color
                
    # 4. Return result
    # If no solid block found, assertion triggers per source [S1, unofficial]
    return top_color, top_z
```

## Subtopics

### [2D Voxel Shadow Map](5769-2d-voxel-shadow-map.md)
A 2D Voxel Shadow Map is a rendering optimization that stores the highest occupied voxel Y-coordinate for each (X, Z) column in a 2D texture, enabling fast shadow determination without ray marching. It differs from standard depth buffers by storing world-space height values rather than view-space depth, and from ray-marched shadows by using a single texture lookup per fragment instead of iterative sampling. This method builds on [Top Voxel Retrieval](4236-top-voxel-retrieval.md) to provide occlusion data.

## Operating characteristics
Time complexity is dominated by chunk loading and block iteration, which is $O(N)$ where $N$ is the number of blocks in the column, or $O(\log N)$ if binary search is used on the sorted $z$ values. Memory complexity is $O(1)$ auxiliary space per query if the chunk copy is reused. Performance is high for sparse columns but degrades if many blocks exist. Strengths include direct access to world-space height. Limitations include the assertion failure on empty columns and the overhead of chunk copying. Use when shadow maps or height queries are needed; avoid when specific $z$ retrieval is required.

## Background
Originated in the `libvxl` library by xtreme8000 [S1, unofficial]. It is used in voxel-based game engines for efficient height queries.

## Sources
[S1] xtreme8000/libvxl, https://github.com/xtreme8000/libvxl, fetched 2026-10-28 (unofficial)
