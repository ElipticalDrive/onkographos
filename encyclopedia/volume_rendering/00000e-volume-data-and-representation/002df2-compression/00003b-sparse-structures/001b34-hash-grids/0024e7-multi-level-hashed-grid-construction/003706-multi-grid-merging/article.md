# Multi-Grid Merging

Also called: Merging Multiple Grids, Min Compositing

## Overview
Multi-Grid Merging combines two sparse hash grids into a single structure by taking the union of their active voxels. It differs from [Replace Compositing](7905-multi-grid-merging.md) by resolving overlaps via a specific aggregation function (typically minimum, maximum, or sum) rather than overwriting, preserving information from both inputs [S1, unofficial]. This method is essential for building multi-level representations or combining data from different sources without losing overlapping details.

## How it works
**Inputs:**
*   Grid `a`: Source sparse hash grid.
*   Grid `b`: Source sparse hash grid.
*   `mode`: The compositing strategy (e.g., min, max, sum).

**Outputs:**
*   `data`: A new sparse hash grid containing the merged result.

**Data Structures:**
*   `data`: A hash map or sparse array where keys are voxel coordinates (e.g., integer triplets or hashed indices) and values are the voxel data (e.g., density, opacity, or color).

**Steps:**
1.  **Initialize:** Create an empty grid `data`.
2.  **Copy Grid A:** Iterate through all active voxels in grid `a`. For each voxel at coordinate `c` with value `v`, insert `data[c] = v`.
3.  **Merge Grid B:** Iterate through all active voxels in grid `b`. For each voxel at coordinate `c` with value `v`:
    *   If `c` exists in `data`:
        *   Let `existing = data[c]`.
        *   Update `data[c]` based on the `mode`:
            *   **Min Compositing:** `data[c] = min(existing, v)` [S1, unofficial].
            *   **Max Compositing:** `data[c] = max(existing, v)` [S1, unofficial].
            *   **Sum Compositing:** `data[c] = existing + v` [S1, unofficial].
    *   Else (`c` does not exist in `data`):
        *   Set `data[c] = v`.
4.  **Upsampling (Optional/Contextual):** If the merged grid is intended for propagation or rendering in an isotropic residuum context, upsample the merged grid to double the resolution of the individual propagation grids to prevent undersampling due to arbitrary grid orientations [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def merge_grids(grid_a, grid_b, mode="min"):
    """
    Merges two sparse hash grids.
    
    Args:
        grid_a: Dict-like object mapping coordinates to values.
        grid_b: Dict-like object mapping coordinates to values.
        mode: String, one of "min", "max", "sum".
        
    Returns:
        data: Dict-like object representing the merged grid.
    """
    data = {}
    
    # Step 2: Copy all active voxels from grid a
    for c, v in grid_a.items():
        data[c] = v
        
    # Step 3: Iterate active voxels from grid b
    for c, v in grid_b.items():
        if c in data:
            existing = data[c]
            if mode == "min":
                data[c] = min(existing, v)
            elif mode == "max":
                data[c] = max(existing, v)
            elif mode == "sum":
                data[c] = existing + v
            else:
                # Default or error handling for unsupported modes
                data[c] = min(existing, v) 
        else:
            data[c] = v
            
    return data
```

**Parameters:**
*   `mode`: Determines the overlap resolution. "min" is the default in some implementations [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point operations (especially sum) may accumulate errors if many grids are merged sequentially.
*   **Boundaries:** Ensure coordinate systems for `grid_a` and `grid_b` are aligned before merging.
*   **Degenerate Input:** Empty grids should be handled gracefully (returning the other grid).

## Subtopics

### Grid Deformation with Entropy Coding
Compresses data differences between registered adjacent data slices using entropy coding [S3, unofficial]. This variant focuses on storage efficiency rather than just structural merging, often applied after merging or as part of a compression pipeline for large-scale volume data [S3, unofficial].

### Max Compositing
Returns a new grid whose active voxels are the union of two grids, taking the maximum value at overlapping coordinates [S1, unofficial]. It differs from [Replace Compositing](7905-multi-grid-merging.md) by preserving the highest contribution in the overlap, making it suitable for scenarios where the strongest signal (e.g., highest density or opacity) is desired.

### Sum Compositing
Returns a new grid whose active voxels are the union of two grids, summing overlapping values [S1, unofficial]. This is suitable for additive blending of volumetric data, such as combining density fields from multiple sources.

### Multi-Level Grid Coarsening
A specialized merging strategy for generating multi-resolution grids [S4]. After 10,000 epochs, sampling points are randomly selected within each grid to calculate opacity. The vertex of the grid with the highest opacity among eight grids is selected as the vertex of the merged grid. This generates grids of 64×64×64, 32×32×32, 16×16×16, and 8×8×8 sequentially. Each level is tailored for specific observation heights to minimize rendering cost. For the same block, different levels use the same network, but gradients in rough levels only update mesh vertex coordinates to prevent coarse-scale meshes from providing undue supervision to shared networks [S4].

## Operating characteristics
*   **Time Complexity:** $O(N_a + N_b)$, where $N_a$ and $N_b$ are the number of active voxels in grid `a` and `b` respectively, assuming $O(1)$ hash map operations.
*   **Memory Complexity:** $O(N_a + N_b)$ for the resulting grid, as it stores the union of active voxels.
*   **Strengths:** Simple to implement; preserves all active data from both inputs; flexible compositing modes.
*   **Limitations:** Does not inherently reduce memory usage compared to the sum of inputs; requires careful handling of coordinate alignment.
*   **When to use:** When combining data from multiple sources or levels where overlap information must be preserved via aggregation.
*   **When not to use:** When strict memory reduction is required without preserving all details (consider pruning or thresholding).

## Background
Multi-grid merging is a fundamental operation in sparse volume rendering and compression, particularly in systems using hash grids or similar sparse structures. It enables the construction of multi-level representations and the integration of data from various sources. The specific "min compositing" variant is noted in implementations like Lyr.jl [S1, unofficial]. The concept of merging grids for multi-resolution rendering is also seen in neural radiance field applications, such as UE4-NeRF, where opacity-based selection is used for coarsening [S4].

## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] Principal-Ordinates Propagation for Real-Time Rendering ..., https://cg.ivd.kit.edu/publications/2015/principal-ordinates/ElekCAG2014.pdf, fetched 2026-10-29
[S3] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
[S4] UE4-NeRF: Neural Radiance Field for Real-Time Rendering of Large-Scale Scene, https://doi.org/10.48550/arXiv.2310.13263, fetched 2026-10-29
