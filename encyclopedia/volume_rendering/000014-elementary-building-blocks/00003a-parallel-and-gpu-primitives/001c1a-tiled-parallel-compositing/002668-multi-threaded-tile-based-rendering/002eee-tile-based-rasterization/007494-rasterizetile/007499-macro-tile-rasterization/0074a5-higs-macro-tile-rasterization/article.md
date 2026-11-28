# HiGS Macro-tile Rasterization

Also called: HiGS rasterizer, macro-tile rasterization, HiGS tile rasterization

## Overview
HiGS Macro-tile Rasterization is a GPU-accelerated rasterization technique for 3D Gaussian Splatting that replaces standard per-tile bitmask generation and prefix-sum dispatch with inline visibility determination during gaussian loading [S1, unofficial]. It decouples work distribution from screen area by grouping render tiles into macro-tiles (8x4 or 8x8), bounding per-unit work to achieve resolution-stable frame times [S1, unofficial]. Unlike baseline rasterizers where partitioning and sorting dominate frame time at high resolutions, HiGS reduces the partitioning bottleneck by ensuring the macro-tile pair count grows significantly slower than screen resolution [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- A batch of 3D Gaussians defined by position, covariance (Cholesky factor $C$), opacity, and spherical harmonic coefficients.
- Camera projection parameters.
- Macro-tile layout configuration (8x4 or 8x8 render tiles per macro-tile).
- Global memory tile buffer (preallocated).

**Outputs:**
- Partial RGBT (Red, Green, Blue, Transmittance) results stored in a global-memory sparse tile buffer.
- An active-tile bitmask indicating which render tiles within the macro-tile received gaussian contributions.

### Data Structures
1.  **Global-Memory Tile Buffer:** A preallocated dense array indexed by `(macro-tile batch, render tile within macro-tile)`.
    -   **Size:** 16 KB per batch slot at default 8x8 render-tile size.
    -   **Calculation:** 32 tiles per macro-tile $\times$ 64 pixels per tile $\times$ 4 RGBT channels $\times$ 2 bytes (half precision).
    -   **Total Footprint:** $16B$ KB for $B$ macro-tile gaussian batches [S1, unofficial].
2.  **Active Mask:** A per-batch 32-bit integer stored out-of-band. Flags which render tiles received gaussians [S1, unofficial].
3.  **Shared Memory Ring Buffer:** A 2B-entry ring buffer storing gaussian indices, where $B$ is the batch size [S1, unofficial].
4.  **Register Storage:**
    -   32x32 bit matrix for mask transpose (one word per warp lane) [S1, unofficial].
    -   Gaussian attributes packed as `half2` to halve shared memory occupancy and register pressure [S1, unofficial].

### Parameters
-   **Macro-tile Layout:** 8x4 or 8x8 render tiles.
    -   For 8x4 layout, pixel coordinates $(x, y)$ relative to the macro-tile center satisfy $|x| \le 4$ and $|y| \le 2$ [S1, unofficial].
-   **Prescaling Factor:** Cholesky factors are prescaled to absorb log2 base conversion and coordinate scaling:
    $$ \tilde{C}'_{ij} = C_{ij} \times \frac{\text{tile\_size}}{2} \times \frac{\text{tile\_size}}{2} \times \text{tile\_size} $$
    This allows the rasterizer to evaluate opacity using a single half-precision `exp2` instruction without runtime base conversion [S1, unofficial].
-   **Batch Size ($B$):** Number of gaussians in a macro-tile batch.

### Algorithm Steps
1.  **Mask Transpose:**
    -   Each thread in a warp holds a 32x32 bit matrix in registers, where bit $j$ indicates if gaussian $j$ overlaps tile $i$.
    -   Perform a 5-stage warp-shuffle butterfly operation to transpose the matrix in registers.
    -   After transpose, lane $i$ holds a word where bit $j$ indicates gaussian $j$ overlaps tile $i$.
    -   This avoids atomics and shared-memory bank conflicts [S1, unofficial].

2.  **Build Tile Work Queue:**
    -   Warp 0 builds a work queue against a single shared-memory counter.
    -   Contention is kept warp-local [S1, unofficial].

3.  **Execute Blending Loop:**
    -   Drain gaussian indices from the shared memory ring buffer in batches of $B$.
    -   Hoist position delta $dx$ and base Cholesky term $C_{base\_0} = C_0 \cdot dx$ outside the pixel-pair loop.
    -   For each gaussian:
        -   Load 4 shared-memory values (using `half2` packing).
        -   Execute inner loop (~11 `half2` operations per pixel-pair).
        -   Evaluate opacity:
            $$ \text{opacity} \times \text{exp2}(-\tilde{C}) $$
            where $\tilde{C}$ is the prescaled Cholesky factor [S1, unofficial].
    -   Each gaussian requires exactly 4 shared-memory loads regardless of the number of pixel-pairs owned by the thread [S1, unofficial].

4.  **Check Saturation:**
    -   After draining each batch, perform a warp-unanimous ballot vote to test if all pixel-pairs have saturated transmittance.
    -   If saturated, the warp immediately claims the next tile [S1, unofficial].

5.  **Write Partial Results:**
    -   Write partial RGBT results to the global-memory tile buffer.
    -   Use cache-bypassing streaming stores to prevent evicting reused gaussian data from cache [S1, unofficial].

6.  **Post-Rasterization Compositing:**
    -   A separate kernel assigns one warp per render tile.
    -   Walk parent macro-tile batches in depth order.
    -   Read partial RGBT from the sparse tile buffer only if the per-batch active-tile bitmask has the tile's bit set.
    -   Accumulate using front-to-back alpha blending with transmittance early-out [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# HiGS Macro-tile Rasterization Kernel
# Assumptions: CUDA-like execution model, warp-level primitives available.

def higs_macro_tile_rasterization(gaussians, camera, macro_tile_config):
    """
    Main rasterization kernel for a single macro-tile batch.
    """
    # 1. Mask Transpose
    # Each thread holds a 32x32 bit matrix in registers.
    # bit_matrix[i][j] = 1 if gaussian j overlaps tile i
    bit_matrix = compute_overlap_mask(gaussians, camera, macro_tile_config)
    
    # Transpose 32x32 bit matrix using 5-stage warp-shuffle butterfly
    # After transpose: lane i holds word where bit j indicates gaussian j overlaps tile i
    transposed_matrix = warp_shuffle_butterfly(bit_matrix)
    
    # 2. Build Tile Work Queue
    # Warp 0 builds queue against shared-memory counter
    if thread_id == 0:
        work_queue = build_work_queue(transposed_matrix, shared_counter)
    
    # 3. Execute Blending Loop
    # Shared memory ring buffer for gaussian indices
    ring_buffer = load_gaussian_indices_from_global_to_shared()
    
    while not ring_buffer_empty():
        # Drain batch of B gaussians
        batch = ring_buffer.read_batch(B)
        
        for gaussian in batch:
            # Hoist invariant calculations
            dx = gaussian.position - macro_tile_center
            C_base_0 = gaussian.C[0] * dx
            
            # Load gaussian attributes (half2 packed)
            # Requires 4 shared-memory loads per gaussian
            attrs = load_gaussian_attributes(gaussian.id)
            
            # Inner blending loop (~11 half2 ops per pixel-pair)
            for pixel_pair in active_pixel_pairs:
                # Evaluate opacity using prescaled Cholesky factor
                # opacity * exp2(-C_tilde)
                C_tilde = compute_prescaled_chol(gaussian.C, dx, macro_tile_config)
                alpha = gaussian.opacity * exp2(-C_tilde)
                
                # Accumulate RGBT
                accumulate_pixel(pixel_pair, alpha, gaussian.color, gaussian.sh)
        
        # 4. Check Saturation
        # Warp-unanimous ballot vote
        all_saturated = ballot_all(saturated_transmittance)
        
        if all_saturated:
            # Claim next tile immediately
            claim_next_tile()
            break
            
    # 5. Write Partial Results
    # Use cache-bypassing streaming stores
    write_partial_results_to_global_buffer(partial_rgbt, cache_bypass=True)
    
    # 6. Update Active Mask
    # Set bits for tiles that received gaussians
    active_mask = compute_active_mask(transposed_matrix)
    write_active_mask(active_mask)

def post_rasterization_compositing(active_masks, tile_buffer, macro_tile_batches):
    """
    Separate kernel for final compositing.
    """
    # Assign one warp per render tile
    tile_id = get_tile_id()
    
    # Walk batches in depth order
    for batch in macro_tile_batches:
        # Check if this tile was active in this batch
        if active_masks[batch] & (1 << tile_id):
            # Read partial RGBT from sparse tile buffer
            partial_rgbt = read_tile_buffer(batch, tile_id)
            
            # Front-to-back alpha blending
            composite(partial_rgbt)
            
            # Transmittance early-out
            if transmittance_saturated():
                break
```

### Edge Cases and Pitfalls
-   **Shared Memory Pressure:** `half2` packing halves shared memory occupancy for gaussian attributes and doubles the blending loop read rate, while also halving register pressure [S1, unofficial].
-   **Precision:** The prescaling and tile-relative coordinates keep operands below the fp16 range limit, avoiding overflow during Mahalanobis evaluation [S1, unofficial].
-   **Cache Eviction:** Writing partial results without cache-bypassing would evict reused gaussian data from cache, degrading performance [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** The macro-tile pair count grows only 1.3–1.8x from 1080p to 4K, compared to 2.5–3.4x for render-tile schemes [S1, unofficial].
-   **Memory Complexity:** The global-memory tile buffer footprint is $16B$ KB for $B$ macro-tile gaussian batches [S1, unofficial].
-   **Performance:**
    -   At 1080p 8x8 tiles, HiGS is 4.8x faster than gsplat (0.52 ms vs 2.49 ms mean) [S1, unofficial].
    -   At 4K 8x8 tiles, HiGS is 8.4x faster (0.82 ms vs 6.92 ms mean) [S1, unofficial].
    -   Sort stage accounts for 30–41% of intersection+sort time and under 0.08 ms even at 4K [S1, unofficial].
-   **Strengths:** Resolution-stable frame times; reduced partitioning bottleneck; efficient use of fp16 and shared memory.
-   **Limitations:** Requires preallocation of global-memory tile buffer; complex warp-shuffle implementation for mask transpose.
-   **When to Use:** High-resolution rendering (4K+) where baseline rasterizers suffer from partitioning bottlenecks.
-   **When Not to Use:** Low-resolution rendering where the overhead of macro-tile management may not be justified; scenes with extremely sparse gaussian distributions where tile saturation is rare.

## Background
HiGS Macro-tile Rasterization was introduced in the HiGS paper (2026) as part of a hierarchical rendering architecture for real-time 3D Gaussian Splatting [S1, unofficial]. It addresses the limitations of baseline rasterizers like gsplat, where partitioning and sorting dominate frame time at high resolutions [S1, unofficial]. The method builds on the concept of macro-tiles to decouple work distribution from screen area, enabling resolution-stable performance [S1, unofficial].

## Sources
[S1] HiGS: A Hierarchical Rendering Architecture for Real-Time 3D Gaussian Splatting, https://doi.org/10.48550/arXiv.2606.00352, fetched 2026-10-29 (unofficial)
