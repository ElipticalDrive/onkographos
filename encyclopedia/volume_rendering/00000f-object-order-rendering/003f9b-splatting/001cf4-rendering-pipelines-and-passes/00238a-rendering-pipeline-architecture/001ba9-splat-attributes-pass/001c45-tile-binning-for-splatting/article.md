# Tile Binning for Splatting

Also called: Tile Binning, Tsplat tile binning

## Overview
Tile Binning for Splatting groups depth-sorted splats into per-tile buckets (16x16 pixel blocks) using a count-then-scatter two-pass approach to improve cache locality during compositing [S1, unofficial]. It differs from stable radix-sort approaches by explicitly computing tile boundaries via prefix sums rather than relying on global sort stability to preserve depth order within tiles [S1, unofficial][S2].

## How it works
**Inputs:** A list of $N$ depth-sorted splats, each with a screen-space bounding box; image dimensions $W \times H$; tile dimensions $TILE\_W \times TILE\_H$ (typically 16x16) [S1, unofficial].
**Outputs:** A flat array `bins.splat_indices` containing splat indices grouped by tile, and metadata `bins.offsets` defining the start range for each tile in that array [S1, unofficial].

**Data Structures:**
*   `bins.offsets`: Array of size $num\_tiles + 1$ [S1, unofficial]. Stores the cumulative start index for each tile [S1, unofficial]. `bins.offsets[i]` is the start index for tile $i$ [S1, unofficial]. `bins.offsets[num_tiles]` is the total number of splats [S1, unofficial].
*   `bins.cursor`: Array of size $num\_tiles$ [S1, unofficial]. A mutable copy of `bins.offsets[..num_tiles]` used during scattering to track the next available slot in each tile's bucket [S1, unofficial].
*   `bins.splat_indices`: Array of size $N$ [S1, unofficial]. Stores the original index of each splat [S1, unofficial].

**Steps:**

1.  **Calculate Tile Grid:**
    Compute the number of tiles in x and y directions [S1, unofficial]:
    $$num\_tiles\_x = \lceil W / TILE\_W \rceil$$
    $$num\_tiles\_y = \lceil H / TILE\_H \rceil$$
    $$num\_tiles = num\_tiles\_x \cdot num\_tiles\_y$$

2.  **Pass 1: Count:**
    Initialize `bins.offsets` of size $num\_tiles + 1$ to zero [S1, unofficial].
    For each splat $s$ in the input list:
    a. Determine the tile range $[tx0, tx1] \times [ty0, ty1]$ covered by the splat's bounding box [S1, unofficial].
    b. For each tile $(tx, ty)$ in this range, calculate the linear tile ID:
       $$tile\_id = ty \cdot num\_tiles\_x + tx$$
    c. Increment the count for this tile:
       $$bins.offsets[tile\_id + 1] += 1$$

3.  **Prefix Sum:**
    Perform an exclusive prefix sum on `bins.offsets` to convert counts to start indices [S1, unofficial]:
    $$bins.offsets[i] = bins.offsets[i] + bins.offsets[i-1] \quad \text{for } i = 1 \dots num\_tiles$$
    After this step, `bins.offsets[i]` holds the starting index in `bins.splat_indices` for tile $i$ [S1, unofficial]. The total number of splats is `bins.offsets[num_tiles]` [S1, unofficial].

4.  **Initialize Cursors:**
    Copy the start indices to the cursor array for mutable access during scattering [S1, unofficial]:
    $$bins.cursor[i] = bins.offsets[i] \quad \text{for } i = 0 \dots num\_tiles - 1$$

5.  **Pass 2: Scatter:**
    For each splat index $idx$ and splat $s$ in the input list:
    a. Determine the tile range $[tx0, tx1] \times [ty0, ty1]$ covered by the splat's bounding box [S1, unofficial].
    b. For each tile $(tx, ty)$ in this range:
       i. Calculate $tile\_id = ty \cdot num\_tiles\_x + tx$.
       ii. Place the splat index into the bin:
           $$bins.splat_indices[bins.cursor[tile\_id]] = idx$$
       iii. Increment the cursor for this tile:
            $$bins.cursor[tile\_id] += 1$$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tile_bin_splats(splats, width, height, tile_w=16, tile_h=16):
    num_tiles_x = (width + tile_w - 1) // tile_w
    num_tiles_y = (height + tile_h - 1) // tile_h
    num_tiles = num_tiles_x * num_tiles_y
    
    # Initialize offsets array with size num_tiles + 1
    offsets = [0] * (num_tiles + 1)
    
    # Pass 1: Count
    for splat in splats:
        # Determine tile range from splat bounding box [G]
        tx0 = max(0, splat.min_x // tile_w)
        tx1 = min((width - 1) // tile_w, splat.max_x // tile_w)
        ty0 = max(0, splat.min_y // tile_h)
        ty1 = min((height - 1) // tile_h, splat.max_y // tile_h)
        
        for ty in range(ty0, ty1 + 1):
            for tx in range(tx0, tx1 + 1):
                tile_id = ty * num_tiles_x + tx
                offsets[tile_id + 1] += 1
                
    # Prefix Sum
    for i in range(1, num_tiles + 1):
        offsets[i] += offsets[i - 1]
        
    # Initialize cursors
    cursors = offsets[:num_tiles]
    
    # Pass 2: Scatter
    splat_indices = [0] * len(splats)
    
    for idx, splat in enumerate(splats):
        # Determine tile range from splat bounding box [G]
        tx0 = max(0, splat.min_x // tile_w)
        tx1 = min((width - 1) // tile_w, splat.max_x // tile_w)
        ty0 = max(0, splat.min_y // tile_h)
        ty1 = min((height - 1) // tile_h, splat.max_y // tile_h)
        
        for ty in range(ty0, ty1 + 1):
            for tx in range(tx0, tx1 + 1):
                tile_id = ty * num_tiles_x + tx
                cursor_pos = cursors[tile_id]
                splat_indices[cursor_pos] = idx
                cursors[tile_id] += 1
                
    return splat_indices, offsets
```

**Edge Cases and Pitfalls:**
*   **Precision:** Tile ID calculation must use integer arithmetic to avoid floating-point rounding errors at tile boundaries [S1, unofficial].
*   **Boundaries:** Splats partially outside the image bounds must be clamped to the valid tile range $[0, num\_tiles\_x-1] \times [0, num\_tiles\_y-1]$ [S1, unofficial].
*   **Degenerate Input:** If a splat has zero area or is completely outside the image, it should not increment any tile count [G].

## Subtopics

### Stable Tile ID Sorting
A variant that groups depth-sorted fragments into per-tile lists using a stable radix sort on tile IDs, preserving the depth order within each tile [S2]. It computes a 16-bit tile ID for each fragment and uses it as the key for a 16-bit radix sort [S2]. After sorting, it computes the first and last indices for each tile by launching a kernel with one thread per fragment; if the current fragment's tile ID differs from the previous, it updates the previous tile's `lastIndex` and the current tile's `firstIndex` [S2]. This approach is chosen when the underlying data structure already supports efficient stable sorting [S2].

## Operating characteristics
*   **Time Complexity:** $O(N + T)$, where $N$ is the number of splats and $T$ is the number of tiles [S1, unofficial]. The two-pass approach involves one pass over splats for counting and one for scattering, plus a linear prefix sum over tiles [S1, unofficial].
*   **Memory Complexity:** $O(T + N)$ for the `offsets`, `cursors`, and `splat_indices` arrays [S1, unofficial].
*   **Performance:** In the benchmarked pipeline, this method is part of a full rendering loop processing 200,000 splats, showing efficient integration with projection and compositing steps [S1, unofficial].
*   **Strengths:** Improves cache locality during compositing by grouping splats per tile [S1, unofficial]. Simple to implement on CPU and GPU without complex synchronization [S1, unofficial].
*   **Limitations:** Requires two passes over the splat data [S1, unofficial]. Does not inherently preserve depth order within tiles unless the input is pre-sorted and the scatter is stable [S1, unofficial][S2].
*   **When to use:** When cache locality is critical and the input splats are already depth-sorted [S1, unofficial].

## Background
Tile binning for splatting is implemented in open-source Gaussian splatting renderers to optimize the compositing phase by reducing cache misses during the rendering of large numbers of splats [S1, unofficial]. The method is distinct from the stable radix sort approach used in Splatshop, which leverages GPU radix sort algorithms to group fragments by tile ID while preserving depth order [S2].

## Sources
[S1] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
[S2] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
