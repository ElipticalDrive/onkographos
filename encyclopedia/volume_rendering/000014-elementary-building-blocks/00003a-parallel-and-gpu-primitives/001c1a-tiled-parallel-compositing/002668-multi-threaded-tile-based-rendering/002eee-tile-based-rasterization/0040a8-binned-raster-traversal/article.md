# Binned Raster Traversal

Also called: Binned CUDA Rasterization

## Overview
Binned Raster Traversal is a GPU-accelerated rasterization method for Gaussian splatting that replaces the global depth-sorted scan of the baseline with a tile-local, bin-based linear scan [S1]. It solves the performance bottleneck of global sorting by partitioning the pair stream into coarse bins per tile, allowing each CUDA thread block to process a contiguous interval of pre-ordered data without per-bin branching [S1]. It differs from its parent, [Tile-Based Rasterization](5910-tile-based-rasterization.md), by explicitly managing the tile-primitive mapping on the CPU and staging primitive parameters in shared memory for cooperative compositing, rather than relying on a linear scan of a pre-binned pair stream [S1].

## How it works
**Inputs:**
*   Repaired pair stream $P$ (containing Gaussian IDs and sidecar data).
*   Bin offsets array `binOffsets[t, k]` for each tile $t$ and bin index $k \in [0, K]$.
*   Optional flat-ID sidecar.
*   Gaussian attributes (position, covariance, color, opacity).
*   Image and tile dimensions.

**Outputs:**
*   Rendered colors, alphas, and last contributing IDs for each pixel.

**Data Structures:**
*   `binOffsets`: A 2D array of size $T \times (K+1)$, where $T$ is the number of tiles and $K$ is the number of coarse bins per tile. `binOffsets[t, k]` stores the start index in the pair stream for the $k$-th bin of tile $t$. The interval for tile $t$ is $[binOffsets[t, 0], binOffsets[t, K])$.
*   Shared Memory: Used for candidate Gaussian IDs and optionally for staging candidate colors.

**Parameters:**
*   $K$: Number of coarse bins per tile. Default: 64 [S1].
*   Early-termination threshold: Transmittance $T_p$ below which a pixel is marked inactive.
*   Tile-alpha culling threshold: Minimum contribution threshold for tile-level culling.
*   Configuration flags: `color_staging` (enabled by default), `tile_alpha_culling` (enabled by default), `traversal_profiling` (disabled by default), `selective_exact_repair` (enabled by default) [S1].

**Steps:**
1.  Launch one CUDA block per tile in parallel.
2.  Determine tile pixel coordinates. Skip masked tiles.
3.  For all pixels $p$ in tile $t$:
    *   Initialize color accumulator $C_p \leftarrow 0$.
    *   Initialize transmittance accumulator $T_p \leftarrow 1$.
4.  Set scan interval $[a, b) \leftarrow [binOffsets[t, 0], binOffsets[t, K])$.
5.  Iterate through batches within the interval $[a, b)$:
    *   If all pixels in the tile have terminated, break the loop.
    *   Load candidate Gaussian IDs into shared memory.
    *   If tile-alpha culling is enabled, remove candidates below the tile-level contribution threshold.
    *   If color staging is enabled, stage candidate colors cooperatively in shared memory.
    *   For all Gaussians $g$ in the surviving batch order:
        *   For all active pixels $p$ in tile $t$:
            *   If traversal profiling is enabled, count candidate visits.
            *   Evaluate the same opacity/conic test as the baseline.
            *   Update $C_p$ and $T_p$ using baseline alpha compositing.
            *   If $T_p$ reaches the early-termination threshold, mark pixel $p$ as inactive.
6.  Write final color, alpha, and last contributing ID for each pixel.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def binned_raster_traversal(repaired_pair_stream, bin_offsets, gaussian_attrs, tile_dims, K=64):
    # Launch one CUDA block per tile
    for tile_t in parallel_tiles:
        # 1. Determine tile pixel coordinates; skip masked tiles
        if tile_t.is_masked:
            continue
        
        tile_pixels = tile_t.get_pixel_coordinates()
        
        # 2. Initialize accumulators for all pixels in the tile
        for p in tile_pixels:
            p.color_accumulator = 0.0
            p.transmittance_accumulator = 1.0
            p.is_active = True
            
        # 3. Set scan interval
        a = bin_offsets[tile_t.id, 0]
        b = bin_offsets[tile_t.id, K]
        
        # 4. Iterate through batches within the interval
        batch_start = a
        while batch_start < b:
            # Check if all pixels terminated
            if not any(p.is_active for p in tile_pixels):
                break
                
            # Load candidate Gaussian IDs into shared memory
            # Determine batch size based on stream layout or fixed chunk
            batch_end = min(batch_start + BATCH_SIZE, b)
            candidate_ids = load_candidates_from_stream(repaired_pair_stream, batch_start, batch_end)
            
            # Tile-alpha culling
            if ENABLE_TILE_ALPHA_CULLING:
                candidate_ids = cull_below_threshold(candidate_ids, TILE_ALPHA_THRESHOLD)
                
            # Color staging
            if ENABLE_COLOR_STAGING:
                stage_colors_cooperatively(candidate_ids, gaussian_attrs)
                
            # Process Gaussians
            for g_id in candidate_ids:
                g = gaussian_attrs[g_id]
                for p in tile_pixels:
                    if not p.is_active:
                        continue
                        
                    # Traversal profiling
                    if ENABLE_TRAVERSAL_PROFILING:
                        count_candidate_visit()
                        
                    # Evaluate opacity/conic test
                    if evaluate_opacity_conic_test(p, g):
                        # Update Cp and Tp using baseline alpha compositing
                        alpha = compute_alpha(p, g)
                        color = compute_color(p, g)
                        
                        p.color_accumulator = p.color_accumulator + color * alpha * p.transmittance_accumulator
                        p.transmittance_accumulator = p.transmittance_accumulator * (1.0 - alpha)
                        
                        # Early termination
                        if p.transmittance_accumulator < EARLY_TERMINATION_THRESHOLD:
                            p.is_active = False
                            
            batch_start = batch_end
            
        # 5. Write final results
        for p in tile_pixels:
            write_pixel_result(p, tile_t.id)
```

**Edge Cases and Pitfalls:**
*   **Bin Order:** Entries within a bin are not depth-sorted unless selective exact repair has restored the order. The method relies on the concatenation of bins being in front-to-back bin order, not necessarily within each bin [S1].
*   **Precision:** Standard floating-point precision is used for accumulators. Early termination thresholds must be chosen carefully to avoid visual artifacts from premature culling.
*   **Boundaries:** The scan interval $[a, b)$ is exclusive of $b$. Ensure `binOffsets` array is sized correctly to avoid out-of-bounds access.

## Subtopics

### [CUDA Tile-and-Bin Implementation](13681-cuda-tile-and-bin-implementation.md)
The CUDA Tile-and-Bin Implementation is a parallelization strategy for computing the forward and backward passes of 2D bitmap primitive rendering on a GPU. It partitions the image plane into tiles and assigns primitives to the tiles they overlap, using one CUDA thread block per tile to achieve complete pixel-level parallelism. It differs from its parent, [Binned Raster Traversal](10295-binned-raster-traversal.md), by explicitly managing the tile-primitive mapping on the CPU and staging primitive parameters in shared memory for cooperative compositing, rather than relying on a linear scan of a pre-binned pair stream.

## Operating characteristics
*   **Time Complexity:** The traversal is linear with respect to the number of candidates in the tile's bins. The overhead of bin construction is amortized over the rasterization pass.
*   **Memory Complexity:** Requires storage for the pair stream, bin offsets, and Gaussian attributes. Shared memory is used for candidate IDs and optionally colors.
*   **Measured Performance:** Evaluated on an RTX 4090 with 5M to 30M Gaussians. The method shows consistent performance scaling with workload [S1].
*   **Strengths:** Eliminates global sorting overhead; enables efficient tile-local processing; supports early termination.
*   **Limitations:** Performance depends on the quality of binning; poor binning can lead to unbalanced workloads.
*   **When to Use:** Suitable for high-resolution rendering with large numbers of Gaussians where global sorting is a bottleneck.
*   **When Not to Use:** For small scenes or low Gaussian counts where global sorting overhead is negligible.

## Background
Binned Raster Traversal was introduced in the TileGS paper, which proposes tile-local depth binning for Gaussian splatting rasterization [S1]. It addresses the scalability issues of previous methods by leveraging tile-local processing and binning strategies.

## Sources
[S1] TileGS: Tile-Local Depth Binning for Gaussian Splatting Rasterization, https://doi.org/10.2312/pg.20261013, fetched 2026-10-29
