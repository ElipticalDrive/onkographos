# Tile Binning for Splatting

Also called: Tile Binning, Tsplat tile binning

## Overview
Tile Binning for Splatting groups depth-sorted splats into per-tile buckets to improve cache locality during the compositing phase of Gaussian Splatting rendering [S1, unofficial]. It solves the problem of scattered memory access patterns by organizing splats into coarse depth bins (typically 16x16 pixel blocks) within each tile, serving as the foundational data structure for subsequent rasterization passes [S1, unofficial]. It is distinguished from its siblings by its reliance on a count-then-scatter two-pass approach using direct atomic operations for the count/scatter phase, which provides the default performance baseline on modern hardware [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A list of projected splats, each with a bounding box in screen space and a depth value. The splats are assumed to be pre-sorted by depth [S1, unofficial].
- **Output:** A flattened array of splat indices (`bins.splat_indices`) ordered by tile, and an offset array (`bins.offsets`) indicating the start index of each tile's splats in the flattened array.

**Data Structures:**
- `bins.offsets`: An array of size $num\_tiles + 1$. `bins.offsets[i]` stores the starting index in `bins.splat_indices` for tile $i$. `bins.offsets[num_tiles]` stores the total number of splats.
- `bins.cursor`: A temporary array of size $num\_tiles$, initialized as a copy of `bins.offsets[..num_tiles]`, used during the scatter pass to track the current write position for each tile.
- `bins.splat_indices`: A dynamic array (or pre-allocated buffer of size equal to the total number of splats) storing the indices of the splats, ordered by tile.

**Parameters:**
- `TILE_W`, `TILE_H`: Tile dimensions in pixels. Standard value is 16x16 [S1, unofficial][S2].
- `width`, `height`: Render resolution in pixels.

**Steps:**

1. **Calculate Tile Grid:**
   Compute the number of tiles in x and y directions:
   $$num\_tiles\_x = \lceil width / TILE\_W \rceil$$
   $$num\_tiles\_y = \lceil height / TILE\_H \rceil$$
   $$num\_tiles = num\_tiles\_x \cdot num\_tiles\_y$$

2. **Pass 1: Count Splat-Tile Intersections:**
   Initialize `bins.offsets` of size $num\_tiles + 1$ to zero.
   For each splat $p$ in the input list:
   - Determine the tile range $[tx0, tx1] \times [ty0, ty1]$ covered by the splat's bounding box.
   - For each tile $(tx, ty)$ in this range, calculate the linear tile index $t = ty \cdot num\_tiles\_x + tx$.
   - Increment a counter for tile $t$. In the atomic implementation, this is done by atomically incrementing `bins.offsets[t + 1]` (using $t+1$ to distinguish from the prefix sum offset logic, or by using a separate count buffer if offsets are used for prefix sums directly). *Note: The source [S1, unofficial] describes incrementing `bins.offsets[row + tx + 1]` where `row` likely refers to the tile row index, implying a specific indexing scheme. The standard interpretation is incrementing a count per tile.*

3. **Prefix Sum:**
   Perform an exclusive prefix sum on the counts to determine start offsets.
   For $i$ from 1 to $num\_tiles$:
   $$bins.offsets[i] += bins.offsets[i-1]$$
   After this step, `bins.offsets[i]` holds the start index for tile $i$ in the output array. The total number of splats is `bins.offsets[num_tiles]`.

4. **Initialize Cursors:**
   Copy the start offsets to the cursor array:
   $$bins.cursor[i] = bins.offsets[i] \quad \text{for } i \in [0, num\_tiles-1]$$

5. **Pass 2: Scatter Splat Indices:**
   For each splat index $idx$ and splat $p$:
   - Determine the tile range $[tx0, tx1] \times [ty0, ty1]$ covered by the splat's bounding box.
   - For each tile $(tx, ty)$ in this range, calculate the linear tile index $t = ty \cdot num\_tiles\_x + tx$.
   - Place the splat index $idx$ into the output array at the current cursor position for tile $t$:
     $$bins.splat\_indices[bins.cursor[t]] = idx$$
   - Increment the cursor for tile $t$:
     $$bins.cursor[t] += 1$$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tile_bin_splats(splats, width, height, TILE_W=16, TILE_H=16):
    num_tiles_x = (width + TILE_W - 1) // TILE_W
    num_tiles_y = (height + TILE_H - 1) // TILE_H
    num_tiles = num_tiles_x * num_tiles_y
    
    # Initialize offsets and cursors
    offsets = [0] * (num_tiles + 1)
    cursors = [0] * num_tiles
    
    # Pass 1: Count
    # Using a temporary count array to avoid atomic contention if not using atomics
    # However, the source [S1, unofficial] implies direct atomic increments on offsets[counts]
    # Here we simulate the count phase with a separate array for clarity, 
    # then prefix sum into offsets.
    counts = [0] * num_tiles
    
    for splat in splats:
        # Determine bounding box in pixels
        x_min, y_min, x_max, y_max = splat.bounding_box
        # Determine tile range
        tx0 = x_min // TILE_W
        ty0 = y_min // TILE_H
        tx1 = (x_max + TILE_W - 1) // TILE_W
        ty1 = (y_max + TILE_H - 1) // TILE_H
        
        # Clamp to tile grid
        tx0 = max(0, min(tx0, num_tiles_x - 1))
        ty0 = max(0, min(ty0, num_tiles_y - 1))
        tx1 = max(0, min(tx1, num_tiles_x - 1))
        ty1 = max(0, min(ty1, num_tiles_y - 1))
        
        for ty in range(ty0, ty1 + 1):
            for tx in range(tx0, tx1 + 1):
                t = ty * num_tiles_x + tx
                counts[t] += 1
                
    # Prefix Sum to get offsets
    offsets[0] = 0
    for i in range(1, num_tiles + 1):
        offsets[i] = offsets[i-1] + counts[i-1]
        
    # Initialize cursors
    for i in range(num_tiles):
        cursors[i] = offsets[i]
        
    # Pass 2: Scatter
    splat_indices = [0] * offsets[num_tiles]
    
    for idx, splat in enumerate(splats):
        x_min, y_min, x_max, y_max = splat.bounding_box
        tx0 = max(0, min((x_min // TILE_W), num_tiles_x - 1))
        ty0 = max(0, min((y_min // TILE_H), num_tiles_y - 1))
        tx1 = max(0, min(((x_max + TILE_W - 1) // TILE_W), num_tiles_x - 1))
        ty1 = max(0, min(((y_max + TILE_H - 1) // TILE_H), num_tiles_y - 1))
        
        for ty in range(ty0, ty1 + 1):
            for tx in range(tx0, tx1 + 1):
                t = ty * num_tiles_x + tx
                pos = cursors[t]
                splat_indices[pos] = idx
                cursors[t] += 1
                
    return splat_indices, offsets
```

**Edge Cases and Pitfalls:**
- **Precision:** Bounding box calculations must handle floating-point to integer conversion correctly to avoid off-by-one errors in tile assignment [S1, unofficial].
- **Boundaries:** Tiles outside the render resolution must be clamped or ignored to prevent out-of-bounds access [S1, unofficial].
- **Degenerate Input:** Splats with zero area or completely outside the screen should be handled gracefully (e.g., skipped or assigned to no tiles) [S1, unofficial].

## Subtopics

### [Tile-Local Bin Construction](10280-tile-local-bin-construction.md)
Tile-Local Bin Construction assigns Gaussian-tile intersection entries to coarse depth bins within each tile and constructs a flattened tile-bin-major pair stream using direct atomic operations for count and scatter. It solves the problem of organizing splats for cache-efficient compositing by grouping them into depth-ordered buckets, serving as the foundational data structure for subsequent rasterization passes. It is distinguished from its siblings by its reliance on direct atomics for the count/scatter phase, which provides the default performance baseline on modern hardware.

### Stable Tile ID Sorting
Stable Tile ID Sorting groups depth-sorted fragments into per-tile lists using a stable radix sort on tile IDs, preserving the depth order within each tile [S2]. It computes a 16-bit tile ID for each fragment and uses it as the key for a 16-bit radix sort [S2]. After sorting, it computes the first and last indices for each tile by launching a kernel with one thread per fragment, updating the previous tile's `lastIndex` and the current tile's `firstIndex` when tile IDs differ [S2].

### Splat Compaction
Splat Compaction is a GPU compute shader that builds a compact list of visible splat indices to enable subsequent passes to only process visible splats [S3, unofficial]. It uses input buffers `SplatClusterIndexBuffer`, `ClusterVisibilityBitmap`, `LODClusterSelectedBitmap`, and `SelectedClusterBuffer` (all `StructuredBuffer<uint>`) and outputs `CompactedSplatIndices` (`RWStructuredBuffer<uint>`) and `VisibleSplatCount` (`RWStructuredBuffer<uint>`) [S3, unofficial]. Parameters include `TotalSplatCount`, `OriginalSplatCount`, `UseLODRendering`, and `THREADGROUP_SIZE` (256) [S3, unofficial].

### Splat Reordering
Splat Reordering is a technique to reduce reconstruction buffer bandwidth by storing splats in a pre-rasterization cache and rasterizing all splats for a specific tile consecutively as a batch [S4]. It maintains a list of splats for each reconstruction buffer tile, selects one tile at a time, and rasterizes all splats inside that tile as a batch, processing tiles in scan-line order [S4]. It uses a modified priority queue ordering criterion for cyclic processing: $i \prec j$ is defined as: false if $i < I_c \leq j$; true if $j < I_c \leq i$; $i < j$ otherwise, where $I_c$ is the currently processed tile index [S4]. The tile size is typically 8x8 pixels [S4]. A pitfall is that using N linked lists on-chip fails for large output resolutions because it cannot support more than N concurrent lists [S4]. Conventional heap insertion/removal requires O(log n) operations, but pipelined heap insertion and removal achieve constant throughput (two cycles in FPGA implementation), independent of heap size [S4].

## Operating characteristics
- **Time Complexity:** The count-then-scatter approach is O(N) where N is the number of splats, assuming constant time tile lookup and atomic operations [S1, unofficial]. Stable radix sort is O(N log K) where K is the number of tiles [S2].
- **Memory Complexity:** O(N) for storing splat indices and O(T) for tile offsets/cursors, where T is the number of tiles [S1, unofficial].
- **Measured Performance:** The atomic-based binning provides a strong baseline on modern GPUs due to efficient parallel atomic operations [S1, unofficial].
- **Strengths:** Improves cache locality during compositing by grouping splats per tile [S1, unofficial]. Simple to implement and highly parallelizable [S1, unofficial].
- **Limitations:** Atomic contention can become a bottleneck for very high splat densities [S1, unofficial]. Requires pre-sorting by depth [S1, unofficial].
- **When to Use:** When cache efficiency is critical and the number of splats is large [S1, unofficial].
- **When Not to Use:** When the number of tiles is small relative to splats, or when hardware atomic performance is poor [S1, unofficial].

## Background
Tile Binning for Splatting originated from the need to optimize Gaussian Splatting rendering on GPUs. The count-then-scatter approach is widely used in modern Gaussian Splatting implementations, such as those described in [S1, unofficial] and [S2]. The concept of tile-based rendering and binning has roots in traditional rasterization hardware, as noted in [S4]. The specific application to Gaussian Splatting involves adapting these techniques to handle the unique properties of Gaussian primitives, such as their bounding boxes and depth sorting.

## Sources
[S1] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
[S2] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
[S3] YuanBaoSMadLab/OpenSplat4D, https://github.com/YuanBaoSMadLab/OpenSplat4D, fetched 2026-10-27 (unofficial)
[S4] A hardware architecture for surface splatting, https://doi.org/10.1145/1275808.1276490, fetched 2026-10-28
