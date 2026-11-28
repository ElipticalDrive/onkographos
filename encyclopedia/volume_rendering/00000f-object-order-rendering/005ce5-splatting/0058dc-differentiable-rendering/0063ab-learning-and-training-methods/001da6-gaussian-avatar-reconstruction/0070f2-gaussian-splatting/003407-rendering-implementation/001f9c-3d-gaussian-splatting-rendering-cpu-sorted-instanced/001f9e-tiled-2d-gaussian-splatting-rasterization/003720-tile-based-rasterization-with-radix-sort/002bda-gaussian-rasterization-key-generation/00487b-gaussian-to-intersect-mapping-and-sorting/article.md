# Gaussian-to-Intersect Mapping and Sorting

Also called: Gaussian rasterization preprocessing, Gaussian to Intersects Mapping, Tile-Based Gaussian Visibility Culling and Sorting, tile-key sorting

## Overview
Gaussian-to-Intersect Mapping and Sorting is a CUDA-based preprocessing pipeline that culls invisible Gaussians, maps 2D Gaussian projections to tile-based intersection IDs, sorts them for depth ordering, and computes tile bins to enable efficient tile-based rendering [S1, unofficial][S2, unofficial]. It solves the problem of efficiently organizing variable-length lists of Gaussian-tile overlaps for rasterization without resorting to a global composite-key sort, which is required by methods like Kerbl et al.'s original Gaussian Splatting [S1, unofficial]. It sets itself apart from its siblings by generating contiguous tile ranges in a depth-sorted Gaussian list, allowing downstream tile-based rasterizers to process Gaussians in spatially coherent batches [S1, unofficial].

## How it works
The method consists of a sequence of CUDA kernels and tensor operations that transform 3D Gaussian parameters into a sorted, tile-indexed format suitable for rasterization.

### Inputs and Outputs
**Inputs:**
*   `num_points`: Number of Gaussians [S2, unofficial].
*   `xys`: Tensor of shape `(batch, 2)` containing x,y locations of 2D Gaussian projections [S2, unofficial].
*   `depths`: Tensor of shape `(batch, 1)` containing z depth of Gaussians [S2, unofficial].
*   `radii`: Tensor of shape `(batch,)` containing the 2D radii of Gaussians [S2, unofficial].
*   `cov2d`: Tensor of shape `(batch, 3)` containing the upper triangular 2D covariance values [S2, unofficial].
*   `tile_bounds`: Tuple of `(tile_width, tile_height, num_tiles)` or `(tiles.x, tiles.y, 1)` [S2, unofficial].
*   `block_size`: Integer block size for CUDA kernel [S2, unofficial].

**Outputs:**
*   `isect_ids`: Tensor of shape `(cum_tiles_hit, 1)` containing unique IDs for each Gaussian in the form `(tile | depth id)` [S2, unofficial].
*   `gaussian_ids`: Tensor of shape `(cum_tiles_hit, 1)` that maps `isect_ids` back to `cum_tiles_hit` [S2, unofficial].
*   `tile_bins`: Tensor of shape `(num_intersects, 2)` where each row represents the range of Gaussian IDs hit per tile [S2, unofficial].
*   `conic`: Conic parameters derived from `cov2d` [S2, unofficial].
*   `radii`: Computed radii derived from `cov2d` [S2, unofficial].

### Data Structures
*   **GeometryState**: Contains allocated buffers for rasterization geometry [S1, unofficial].
    *   `depths`: float, size P
    *   `clamped`: float, size P*3
    *   `internal_radii`: uint32_t, size P
    *   `means2D`: float, size P
    *   `cov3D`: float, size P*6
    *   `conic_opacity`: float, size P
    *   `rgb`: float, size P*3
    *   `tiles_touched`: uint32_t, size P
*   **gaussian_keys_unsorted**: uint64_t array of size L. Each element packs tile ID in the upper 32 bits and depth in the lower 32 bits [S1, unofficial].
*   **gaussian_values_unsorted**: uint32_t array of size L. Stores the original Gaussian index corresponding to each key [S1, unofficial].
*   **ranges**: uint2 array of size equal to the number of tiles. Each element stores the start (x) and end (y) indices in the sorted key/value arrays for that tile [S1, unofficial].
*   **cum_tiles_hit**: Cumulative sum of `num_tiles_hit` (number of intersected tiles per Gaussian) along dimension 0 with dtype int32 [S2, unofficial]. The total number of intersects is the last element of this tensor.

### Steps

1.  **Frustum Culling**: Launch the `checkFrustum` kernel with P threads (P = total number of Gaussians). Each thread `idx` tests Gaussian `idx` against the view frustum using `in_frustum`, writing the boolean result to `present[idx]` [S1, unofficial].

2.  **Covariance Bounds**: Call `compute_cov2d_bounds` with `cov2d` (contiguous, shape batch x 3 upper triangular 2D covariance values). Asserts input shape ends in 3 and number of points > 0. Returns `conic` parameters and `radii` [S2, unofficial].

3.  **Key/Value Generation**: Launch the `duplicateWithKeys` kernel with P threads. For each Gaussian with `radii[idx] > 0`:
    *   Compute its bounding tile rectangle `(rect_min, rect_max)` using `getRect` [S1, unofficial].
    *   For each tile `(x, y)` in the rectangle:
        *   Compute key: `key = (y * grid.x + x) << 32 | depth` [S1, unofficial].
        *   Store `key` in `gaussian_keys_unsorted` and Gaussian index `idx` in `gaussian_values_unsorted` at offset `off` [S1, unofficial].
    *   Invisible Gaussians (`radii <= 0`) must be skipped to avoid emitting invalid or zero-sized tile overlaps [S1, unofficial].

4.  **Cumulative Sum**: Calculate `cum_tiles_hit` by taking the cumulative sum of `num_tiles_hit` (number of intersected tiles per Gaussian) along dimension 0 with dtype int32 [S2, unofficial].

5.  **Mapping to Intersects**: Call the CUDA kernel `map_gaussian_to_intersects` with inputs `num_points`, `num_intersects`, `xys` (contiguous), `depths` (contiguous), `radii` (contiguous), `cum_tiles_hit` (contiguous), `tile_bounds`, and `block_size`. This returns `isect_ids` (unique IDs in form `tile | depth id`) and `gaussian_ids` (mapping back to `cum_tiles_hit`) [S2, unofficial].

6.  **Sorting**:
    *   Sort `gaussian_keys_unsorted` and `gaussian_values_unsorted` together. Sorting by the composite key yields Gaussian IDs first sorted by tile ID, then by depth [S1, unofficial].
    *   Sort `isect_ids` using `torch.sort` to get `isect_ids_sorted` and `sorted_indices` [S2, unofficial].
    *   Reorder `gaussian_ids` using `torch.gather` with dimension 0 and `sorted_indices` to produce `gaussian_ids_sorted` [S2, unofficial].

7.  **Tile Range Identification**: Launch the `identifyTileRanges` kernel with L threads (L = total key/value pairs). For each index `idx`:
    *   Read `currtile = key >> 32` [S1, unofficial].
    *   If `idx == 0`, set `ranges[currtile].x = 0` [S1, unofficial].
    *   If `currtile != prevtile`, set `ranges[prevtile].y = idx` and `ranges[currtile].x = idx` [S1, unofficial].
    *   If `idx == L - 1`, set `ranges[currtile].y = L` [S1, unofficial].
    *   Tile range boundaries must correctly handle the first and last elements of the sorted list to ensure all tiles have valid start/end indices [S1, unofficial].

8.  **Tile Bin Computation**: Call `get_tile_bin_edges` with `num_intersects`, `isect_ids_sorted` (contiguous), and `tile_bounds`. This returns `tile_bins`, a tensor where `tile_bins[tile_idx]` returns the range (lower, upper) of Gaussian IDs that hit `tile_idx`. Expects intersection IDs to be sorted by increasing tile ID [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def gaussian_to_intersect_mapping_and_sorting(
    num_points: int,
    xys: torch.Tensor,       # Shape: (num_points, 2)
    depths: torch.Tensor,    # Shape: (num_points, 1)
    radii: torch.Tensor,     # Shape: (num_points,)
    cov2d: torch.Tensor,     # Shape: (num_points, 3)
    tile_bounds: tuple,      # (tile_width, tile_height, num_tiles)
    block_size: int
) -> tuple:
    """
    Preprocessing pipeline for 2D Gaussian rasterization.
    Returns: isect_ids_sorted, gaussian_ids_sorted, tile_bins, conic, radii_computed
    """
    
    # Step 1: Frustum Culling (Simulated as a boolean mask)
    # In practice, this is a CUDA kernel checkFrustum
    # present = check_frustum_kernel(...)
    present = torch.ones(num_points, dtype=torch.bool) # Placeholder for actual frustum test
    
    # Step 2: Covariance Bounds
    # Assert input shape ends in 3 and number of points > 0
    assert cov2d.shape[-1] == 3
    assert num_points > 0
    conic, radii_computed = compute_cov2d_bounds(cov2d)
    
    # Filter by radii > 0 and present
    valid_mask = (radii > 0) & present
    valid_indices = torch.nonzero(valid_mask, as_tuple=True)[0]
    num_valid = len(valid_indices)
    
    if num_valid == 0:
        return (
            torch.empty(0, dtype=torch.int64),
            torch.empty(0, dtype=torch.int32),
            torch.empty(0, dtype=torch.int64),
            conic,
            radii_computed
        )

    # Extract valid data
    xys_valid = xys[valid_indices]
    depths_valid = depths[valid_indices]
    radii_valid = radii[valid_indices]
    
    # Step 3: Key/Value Generation
    # Compute bounding tiles for each valid Gaussian
    # getRect returns (rect_min_x, rect_min_y, rect_max_x, rect_max_y)
    # This is a simplified simulation; actual implementation uses CUDA kernel duplicateWithKeys
    tile_width, tile_height, num_tiles = tile_bounds
    grid_x = tile_width # Assuming grid dimensions match tile bounds for simplicity in this abstract
    
    keys = []
    values = []
    
    for i in range(num_valid):
        idx = valid_indices[i].item()
        x, y = xys_valid[i]
        depth = depths_valid[i].item()
        r = radii_valid[i].item()
        
        # Simulate getRect
        rect_min_x = max(0, int(x - r))
        rect_min_y = max(0, int(y - r))
        rect_max_x = min(grid_x - 1, int(x + r))
        rect_max_y = min(tile_height - 1, int(y + r))
        
        for ty in range(rect_min_y, rect_max_y + 1):
            for tx in range(rect_min_x, rect_max_x + 1):
                tile_id = ty * grid_x + tx
                # Key: tile_id in upper 32 bits, depth in lower 32 bits
                # Depth needs to be cast to uint32 for packing
                key = (tile_id << 32) | (depth & 0xFFFFFFFF)
                keys.append(key)
                values.append(idx)
                
    if not keys:
        return (
            torch.empty(0, dtype=torch.int64),
            torch.empty(0, dtype=torch.int32),
            torch.empty(0, dtype=torch.int64),
            conic,
            radii_computed
        )

    gaussian_keys_unsorted = torch.tensor(keys, dtype=torch.uint64)
    gaussian_values_unsorted = torch.tensor(values, dtype=torch.uint32)
    L = len(keys)
    
    # Step 4: Cumulative Sum
    # num_tiles_hit is the number of tiles each Gaussian hit
    # We need to reconstruct num_tiles_hit from the generated keys
    # Count occurrences of each original Gaussian index in gaussian_values_unsorted
    unique, counts = torch.unique(gaussian_values_unsorted, return_counts=True)
    num_tiles_hit = torch.zeros(num_points, dtype=torch.int32)
    num_tiles_hit[unique] = counts
    
    cum_tiles_hit = torch.cumsum(num_tiles_hit, dim=0)
    total_intersects = cum_tiles_hit[-1].item() if total_intersects > 0 else 0
    
    # Step 5: Mapping to Intersects
    # This step generates isect_ids and gaussian_ids from the sorted/unsorted data
    # In the source, map_gaussian_to_intersects is called with cum_tiles_hit
    # It returns isect_ids (tile | depth id) and gaussian_ids (mapping back to cum_tiles_hit)
    # Since we have keys and values, we can derive these.
    # isect_ids is essentially the tile part of the key, or a unique ID.
    # gaussian_ids maps back to cum_tiles_hit.
    
    # Simulate map_gaussian_to_intersects
    # isect_ids: Tensor of shape (cum_tiles_hit, 1)
    # gaussian_ids: Tensor of shape (cum_tiles_hit, 1)
    
    # Extract tile IDs from keys
    isect_ids_raw = (gaussian_keys_unsorted >> 32).int()
    
    # gaussian_ids maps back to cum_tiles_hit. 
    # This usually means the index in the cum_tiles_hit array or the cumulative count.
    # Based on source, gaussian_ids maps isect_ids back to cum_tiles_hit.
    # We will use the values (original indices) to look up cum_tiles_hit.
    
    # Create a tensor for cum_tiles_hit lookups
    cum_tiles_hit_tensor = cum_tiles_hit.int()
    
    # Map original indices to their cumulative hit count
    # gaussian_ids_unsorted = cum_tiles_hit[gaussian_values_unsorted]
    gaussian_ids_unsorted = cum_tiles_hit_tensor[gaussian_values_unsorted]
    
    # Step 6: Sorting
    # Sort gaussian_keys_unsorted and gaussian_values_unsorted together
    # Sorting by composite key yields Gaussian IDs first sorted by tile ID, then by depth
    sorted_keys, sorted_indices = torch.sort(gaussian_keys_unsorted)
    sorted_values = gaussian_values_unsorted[sorted_indices]
    
    # Sort isect_ids and gaussian_ids using the same indices
    isect_ids_sorted = isect_ids_raw[sorted_indices].unsqueeze(1)
    gaussian_ids_sorted = gaussian_ids_unsorted[sorted_indices].unsqueeze(1)
    
    # Step 7: Tile Range Identification
    # Identify ranges for each tile
    num_tiles_actual = tile_bounds[2] # num_tiles from tile_bounds
    ranges = torch.zeros((num_tiles_actual, 2), dtype=torch.int64)
    
    if L > 0:
        # Get current tile for each entry
        current_tiles = sorted_keys >> 32
        
        # Initialize first range
        first_tile = current_tiles[0].item()
        ranges[first_tile, 0] = 0
        
        for idx in range(1, L):
            currtile = current_tiles[idx].item()
            prevtile = current_tiles[idx-1].item()
            
            if currtile != prevtile:
                ranges[prevtile, 1] = idx
                ranges[currtile, 0] = idx
        
        # Handle last element
        last_tile = current_tiles[L-1].item()
        ranges[last_tile, 1] = L
        
    # Step 8: Tile Bin Computation
    # get_tile_bin_edges returns tile_bins
    # tile_bins[tile_idx] returns the range (lower, upper) of Gaussian IDs that hit tile_idx
    # This is essentially the ranges tensor, but possibly formatted differently
    tile_bins = ranges
    
    return (
        isect_ids_sorted,
        gaussian_ids_sorted,
        tile_bins,
        conic,
        radii_computed
    )

def compute_cov2d_bounds(cov2d: torch.Tensor) -> tuple:
    """
    Computes conic parameters and radii from 2D covariance matrix.
    cov2d: Shape (batch, 3) upper triangular values [a, b, c]
    Returns: conic, radii
    """
    # Reconstruct 2x2 covariance matrix from upper triangular values
    # cov2d = [[a, b], [b, c]]
    a = cov2d[:, 0]
    b = cov2d[:, 1]
    c = cov2d[:, 2]
    
    # Determinant
    det = a * c - b * b
    
    # Conic parameters (inverse covariance)
    # conic = [[c, -b], [-b, a]] / det
    conic_a = c / det
    conic_b = -b / det
    conic_c = a / det
    
    conic = torch.stack([conic_a, conic_b, conic_c], dim=-1)
    
    # Radii (approximate based on eigenvalues or max extent)
    # Simplified: radius based on max of diagonal elements
    radii = torch.sqrt(torch.max(a, c))
    
    return conic, radii
```

### Parameters
*   `P`: Total number of Gaussians to process. Used as grid size for `checkFrustum` and `duplicateWithKeys` [S1, unofficial].
*   `L`: Total number of generated key/value pairs after `duplicateWithKeys`. Used as grid size for `identifyTileRanges` [S1, unofficial].
*   `num_points`: Number of Gaussians [S2, unofficial].
*   `num_intersects`: Total number of tile intersections [S2, unofficial].
*   `tile_bounds`: Tuple of `(tile_width, tile_height, num_tiles)` or `(tiles.x, tiles.y, 1)` [S2, unofficial].
*   `block_size`: Integer block size for CUDA kernel [S2, unofficial].

### Edge Cases and Pitfalls
*   **Invisible Gaussians**: Invisible Gaussians (`radii <= 0`) must be skipped during key/value generation to avoid emitting invalid or zero-sized tile overlaps [S1, unofficial].
*   **Tile Range Boundaries**: Tile range boundaries must correctly handle the first and last elements of the sorted list to ensure all tiles have valid start/end indices [S1, unofficial].
*   **Differentiability**: This function is not differentiable to any input. Gradients must be handled separately or these steps excluded from the differentiable graph [S2, unofficial].
*   **Input Shape**: Input `cov2d` must have a shape ending in 3 (upper triangular values). An assertion error is raised if this condition is not met [S2, unofficial].

## Subtopics

### [Gaussian Partitioning (Count-Then-Fill Binning)](23039-gaussian-partitioning-count-then-fill-binning.md)
Gaussian Partitioning is a two-pass CUDA binning algorithm that assigns 3D Gaussians to macro-tiles by constructing per-macro-tile, depth-keyed lists during the binning pass itself. It solves the parallel write-conflict problem of building variable-length lists without resorting to a global composite-key sort, which is required by methods like Kerbl et al.'s original Gaussian Splatting. By using batch-local atomics and hierarchical prefix scans, it eliminates the need for global sorting, leaving only in-segment ordering for downstream processing.

## Operating characteristics
*   **Time Complexity**: The method involves several passes: frustum culling (O(P)), key generation (O(L) where L is total tile intersections), sorting (O(L log L)), and range identification (O(L)). The sorting step is the dominant factor.
*   **Memory Complexity**: Requires storage for `gaussian_keys_unsorted` (uint64, size L), `gaussian_values_unsorted` (uint32, size L), `ranges` (uint2, size num_tiles), and intermediate tensors.
*   **Performance**: Performance depends on the number of Gaussians, tile size, and scene complexity. The use of CUDA kernels and efficient sorting allows for real-time rendering on modern GPUs.
*   **Strengths**: Efficiently handles variable-length lists of Gaussian-tile overlaps. Enables tile-based rendering by providing contiguous ranges of Gaussians for each tile.
*   **Limitations**: Not differentiable. Requires significant memory for large scenes with many Gaussians and tiles.
*   **When to Use**: When real-time rendering of 3D Gaussians is required, and the scene is complex enough that tile-based culling and sorting provide a performance benefit.
*   **When Not to Use**: When differentiability is required for the rendering step, or when the scene is small enough that global sorting is not a bottleneck.

## Background
The method is part of the Gaussian Splatting rendering pipeline, specifically the preprocessing stage that prepares Gaussian data for tile-based rasterization. It builds on [Gaussian Rasterization Key Generation](5133-gaussian-rasterization-key-generation.md). The approach is used in implementations of 3D Gaussian Splatting that aim for efficient, real-time rendering on GPUs.

## Sources
[S1] Lewis-Stuart-11/3DGS-to-PC, https://github.com/Lewis-Stuart-11/3DGS-to-PC, fetched 2026-10-27 (unofficial)
[S2] Gaussian-Splatting-Toolkit/Gaussian-Splatting-Toolkit, https://github.com/Gaussian-Splatting-Toolkit/Gaussian-Splatting-Toolkit, fetched 2026-10-27 (unofficial)
