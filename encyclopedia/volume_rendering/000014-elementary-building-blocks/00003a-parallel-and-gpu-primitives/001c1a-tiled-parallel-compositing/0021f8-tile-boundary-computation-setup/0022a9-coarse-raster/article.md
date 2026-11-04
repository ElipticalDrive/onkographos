# Coarse Raster

Also called: Coarse Rasterizer

## Overview
The Coarse Rasterizer performs triangle setup and emits a conservative list of all triangles touching each tile in a tiled volume rendering pipeline [S1]. It differs from typical chunkers by avoiding per-tile queues and expensive tile coverage tests during the setup phase, instead relying on a global radix-sort to organize triangle-tile relationships [S1]. This approach minimizes inter-thread communication and leverages the high compute-to-bandwidth ratio of modern GPUs by recomputing setup data per tile rather than caching it [S1].

## How it works
The method consists of three main phases: Triangle Setup, Tile Sorting, and Tile Splitting.

### Inputs and Outputs
*   **Input:** A list of triangles defined in world or voxel space.
*   **Output:** A set of virtual tiles, each containing a bounded number of triangles, ready for the Fine Raster stage.

### Data Structures
1.  **Triangle Descriptor:** A 32-bit integer (or similar word-sized type) used for sorting.
    *   Bits [0:1]: Dominant axis of the triangle (2 bits).
    *   Bits [2:31]: Tile ID (remaining bits).
2.  **Unsorted List:** An array of pairs `(triangle_index, descriptor)`.
3.  **Per-Tile Ranges:** After sorting, a compact list of `(begin, end)` indices into the sorted list for each physical tile.
4.  **Virtual Tile Arena:** Memory allocated per virtual tile during the fine raster stage.

### Parameters
*   $M_{tile}$: The maximum number of triangles allowed in a virtual tile [S1].

### Steps

1.  **Triangle Setup and Emission**
    For each triangle, a single CUDA thread processes the triangle [S1].
    a.  Compute the integer bounding box of the triangle.
    b.  Determine the dominant axis of the triangle.
    c.  Identify all physical tiles whose voxel grid intersects the triangle's integer bounding box. This is a conservative estimate; expensive precise coverage tests are skipped [S1].
    d.  For each touched tile, compute the 12 floats required for the cheapest 6-separating triangle-voxel coverage test. These floats are recomputed for each tile rather than computed once and reused, optimizing for GPU compute-to-bandwidth ratios [S1].
    e.  Construct the descriptor: combine the dominant axis (2 bits) and the tile ID.
    f.  Emit a pair `(triangle_index, descriptor)` to a global unsorted list [S1].

2.  **Tile Sorting**
    a.  Perform a global radix sort on the unsorted list of `(triangle_index, descriptor)` pairs based on the descriptor [S1].
    b.  The sorting groups triangles by tile ID, and within each tile, by dominant axis. This grouping increases SIMD efficiency in the subsequent fine raster stage [S1].
    c.  Generate a compact list of per-tile `(begin, end)` ranges that point to the start and end indices of each tile's triangles in the sorted array [S1].

3.  **Tile Splitting**
    a.  Iterate through the physical tiles.
    b.  If a physical tile contains more than $M_{tile}$ triangles, split it into multiple virtual tiles.
    c.  Each virtual tile contains at most $M_{tile}$ triangles.
    d.  Virtual tiles may overlap on the voxel grid. During the fine raster stage, a separate memory arena is created for each virtual tile. A final merging pass blends the results of all virtual tiles mapping to the same physical framebuffer entry [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Phase 1: Triangle Setup and Emission
# Global atomic counter for the unsorted output list
output_counter = 0
output_list = [] # Array of (triangle_idx, descriptor)

def process_triangle(triangle):
    global output_counter
    
    # 1. Compute integer bounding box
    bbox_min, bbox_max = compute_integer_bounding_box(triangle)
    
    # 2. Determine dominant axis
    dominant_axis = compute_dominant_axis(triangle)
    
    # 3. Find all tiles intersecting the bounding box
    touched_tiles = find_tiles_intersecting_bbox(bbox_min, bbox_max)
    
    # 4. Emit for each touched tile
    for tile_id in touched_tiles:
        # Recompute 12 floats for this specific tile
        setup_data = compute_setup_floats(triangle, tile_id)
        
        # Construct descriptor: 2 bits for axis, rest for tile ID
        descriptor = (tile_id << 2) | dominant_axis
        
        # Store in global list
        idx = atomic_add(output_counter, 1)
        output_list[idx] = (triangle.id, descriptor)

# Phase 2: Tile Sorting
def sort_tiles():
    # Sort the output_list by descriptor
    sorted_list = radix_sort(output_list, key=lambda x: x.descriptor)
    
    # Generate per-tile ranges
    tile_ranges = {}
    current_tile = -1
    start_idx = 0
    
    for i, (tri_idx, descriptor) in enumerate(sorted_list):
        tile_id = descriptor >> 2
        
        if tile_id != current_tile:
            if current_tile != -1:
                tile_ranges[current_tile] = (start_idx, i)
            current_tile = tile_id
            start_idx = i
            
    # Close the last tile
    if current_tile != -1:
        tile_ranges[current_tile] = (start_idx, len(sorted_list))
        
    return sorted_list, tile_ranges

# Phase 3: Tile Splitting
def split_tiles(tile_ranges, max_triangles=M_tile):
    virtual_tiles = []
    
    for tile_id, (start, end) in tile_ranges.items():
        num_triangles = end - start
        if num_triangles <= max_triangles:
            # No split needed
            virtual_tiles.append({
                'physical_tile_id': tile_id,
                'triangles': sorted_list[start:end],
                'is_virtual': False
            })
        else:
            # Split into virtual tiles
            num_splits = ceil(num_triangles / max_triangles)
            for i in range(num_splits):
                chunk_start = start + i * max_triangles
                chunk_end = min(start + (i + 1) * max_triangles, end)
                virtual_tiles.append({
                    'physical_tile_id': tile_id,
                    'triangles': sorted_list[chunk_start:chunk_end],
                    'is_virtual': True,
                    'virtual_index': i
                })
                
    return virtual_tiles
```

### Edge Cases and Pitfalls
*   **Low-resolution voxelizations:** May have too few occupied tiles to fill the GPU machine, or skewed triangle distributions. Tile splitting addresses this by creating more virtual tiles to improve load balancing [S1].
*   **Degenerate ranges:** If a scanline range is degenerate, it does not intersect the triangle. The coarse rasterizer avoids this by using conservative bounding box tests, potentially emitting triangles to tiles they do not strictly intersect, which is handled by the fine rasterizer's precise tests [S1].

## Operating characteristics
*   **Time Complexity:** Efficient due to low data amplification; triangles often touch one or a few tiles only [S1]. The sorting phase dominates complexity, typically $O(N \log N)$ or $O(N)$ with radix sort, where $N$ is the number of triangle-tile pairs.
*   **Memory Complexity:** Requires storage for the unsorted list, sorted list, and per-tile ranges. Virtual tile arenas are allocated during fine raster.
*   **Strengths:** Avoids inter-thread communication primitives (like per-tile queues) by using global radix-sort counting and scattering [S1]. The recomputation of setup data exploits high compute-to-bandwidth ratios [S1]. Sorting by dominant axis improves SIMD efficiency in the fine raster stage [S1].
*   **Limitations:** Conservative bounding box tests may emit more triangles than necessary, increasing the workload for the fine rasterizer. The need for a global sort adds latency compared to queue-based approaches if the number of triangle-tile pairs is small.
*   **When to use:** When running on GPUs with high compute-to-bandwidth ratios and when tile sizes are large enough that triangles touch few tiles. Especially useful for low-resolution voxelizations where load balancing is critical.

## Background
The Coarse Rasterizer is part of the VoxelPipe pipeline, introduced by researchers at NVIDIA in 2011 [S1]. It was designed to address the inefficiencies of previous tiled volume rendering approaches that relied on per-tile queues and inter-thread communication [S1]. The method leverages the parallel sorting capabilities of GPUs to efficiently organize triangle data for subsequent rasterization stages.

## Sources
[S1] VoxelPipe: A Programmable Pipeline for 3D Voxelization, https://research.nvidia.com/sites/default/files/pubs/2011-08_VoxelPipe-A-Programmable/voxel-pipe.pdf, fetched 2026-10-27
