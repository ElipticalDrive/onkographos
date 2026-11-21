# Per-Gaussian Backward Pass

Also called: Tile-based backward pass

## Overview
The Per-Gaussian Backward Pass is a backward pass optimization for 3D Gaussian Splatting that parallelizes gradient computation over Gaussians rather than pixels to reduce atomic contention [S1, unofficial]. It addresses the performance bottleneck of standard pixel-parallel backpropagation, where splats contributing to arbitrary numbers of pixels require frequent atomic operations for accumulation [S1, unofficial]. By operating on fixed-size buckets of 32 Gaussians (one GPU warp), it significantly reduces warp stalls and false sharing associated with global memory synchronization [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Gradients of the loss with respect to the rendered image pixels ($\frac{\partial L}{\partial I}$), the forward pass state (Gaussian parameters, tile assignments, and stored alpha blending states).
- **Output:** Gradients with respect to Gaussian parameters (position, covariance, opacity, spherical harmonics coefficients).

**Data Structures:**
- **Alpha Blending State Buffer:** A buffer storing the intermediate alpha blending state (accumulated opacity and color) at each non-empty pixel. This state is recorded after every 32nd splat in the respective tile list during the forward pass.
- **Gaussian Buckets:** Gaussians are grouped into buckets of size 32, corresponding to one GPU warp.

**Steps:**
1. **Forward Pass Recording:** During the forward rasterization pass, for each tile, store the alpha blending state at each non-empty pixel after processing every 32nd Gaussian (splat) in the tile's sorted list [S1, unofficial].
2. **Bucket Assignment:** Group the Gaussians into buckets of 32. Each warp is assigned to process one bucket [S1, unofficial].
3. **Gradient Replay:** For each bucket, threads collaboratively replay the alpha blending process using warp-level primitives. Instead of writing gradients to global memory and accumulating via atomics, threads compute gradients locally based on the stored alpha states and the final pixel gradients [S1, unofficial].
4. **Parameter Update:** Accumulate the computed gradients for the Gaussians in the current bucket.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Per-Gaussian Backward Pass
# Assumes forward pass has stored alpha_states[tile_idx][pixel_idx][step]
# where step is a multiple of 32.

def per_gaussian_backward(tile_gaussians, alpha_states, pixel_gradients):
    """
    tile_gaussians: List of Gaussians assigned to this tile, sorted by z-order.
    alpha_states: 3D array [num_tiles][num_pixels_in_tile][num_steps]
                  storing alpha blending state after every 32nd splat.
    pixel_gradients: Gradient of loss w.r.t. final pixel colors [num_tiles][num_pixels_in_tile][3]
    """
    num_gaussians = len(tile_gaussians)
    # Initialize gradients for all Gaussians in this tile
    gradients = [ZeroGradient() for _ in tile_gaussians]
    
    # Process Gaussians in buckets of 32 (one warp)
    for bucket_start in range(0, num_gaussians, 32):
        bucket_end = min(bucket_start + 32, num_gaussians)
        bucket = tile_gaussians[bucket_start:bucket_end]
        
        # For each pixel in the tile
        for pixel_idx in range(num_pixels_in_tile):
            if pixel_idx is empty or not covered by any gaussian in bucket:
                continue
                
            # Get final pixel gradient
            grad_final = pixel_gradients[tile_idx][pixel_idx]
            
            # Retrieve stored alpha states for this pixel
            # We need states at steps: 32, 64, ..., up to the last multiple of 32 <= bucket_end
            # And potentially the final state if bucket_end is not a multiple of 32
            
            # Replay blending for the bucket
            # This involves iterating through the bucket's Gaussians and 
            # computing their contribution to the pixel gradient using the 
            # stored alpha states to avoid recomputing the forward pass entirely.
            
            # Simplified logic for one pixel in one bucket:
            # 1. Start with final gradient grad_final.
            # 2. Iterate backwards through the bucket's Gaussians.
            # 3. Use stored alpha states to determine the "background" color/alpha 
            #    seen by each Gaussian in the bucket.
            # 4. Compute local gradient for each Gaussian based on its opacity, 
            #    color, and the gradient of the background.
            
            # Note: Exact implementation details of the replay loop depend on 
            # the specific alpha blending formula used in the forward pass.
            # The key is that atomic operations are avoided by computing 
            # gradients locally within the warp.
            
            for i in range(len(bucket) - 1, -1, -1):
                gaussian = bucket[i]
                # Retrieve alpha state before this gaussian was processed
                # This requires mapping the global index to the stored step index
                step_idx = get_alpha_state_step_index(bucket_start + i)
                alpha_state_before = alpha_states[tile_idx][pixel_idx][step_idx]
                
                # Compute gradient for this gaussian
                # grad_g = grad_final * d(I)/d(gaussian)
                # d(I)/d(gaussian) depends on gaussian's opacity, color, and 
                # the accumulated alpha from previous gaussians (alpha_state_before)
                
                grad_g = compute_gaussian_gradient(gaussian, alpha_state_before, grad_final)
                gradients[bucket_start + i] += grad_g
                
                # Update grad_final for the next (previous in z-order) gaussian
                # This is effectively reversing the blending step
                grad_final = reverse_blend_gradient(grad_final, gaussian, alpha_state_before)

    return gradients
```

**Parameters:**
- **Bucket Size:** 32 Gaussians (one warp) [S1, unofficial].
- **Tile Size:** Typically 16x16 pixels (256 pixels) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **VRAM Usage:** This approach increases VRAM usage due to the storage of alpha blending states. This is mitigated by exploiting shared memory to reduce memory costs and overall VRAM allocations [S1, unofficial].
- **Precision:** Storing intermediate alpha states may introduce precision issues if not handled carefully, though the sources do not specify exact precision requirements [G].
- **Empty Pixels:** Pixels not covered by any Gaussian in a bucket should be skipped to avoid unnecessary computation [G].

## Subtopics

### [Adaptive Scheduling for Backward Pass Implementations](10235-adaptive-scheduling-for-backward-pass-implementations.md)
Optimizes the backward pass by dynamically selecting between parallelization strategies to minimize atomic contention.

### [Per-Gaussian Backward](10252-per-gaussian-backward.md)
The core method of parallelizing gradient computation over Gaussians in fixed-size buckets to reduce atomic contention.

## Operating characteristics
- **Time Complexity:** Reduces the number of required atomic operations by a factor equal to the number of pixels in each tile (256 for 16x16 tiles) [S1, unofficial].
- **Memory Complexity:** Increases VRAM usage. On Mip-NeRF360 outdoor scenes (RTX 4090), VRAM increases from 6.33GiB to 7.69GiB (1.20x increase) [S1, unofficial].
- **Performance:** On Mip-NeRF360 outdoor scenes (RTX 4090), training time decreases from 16m56s to 14m14s (1.20x speedup over base) [S1, unofficial]. On RTX 5090, training takes 163 seconds on average, a 5x improvement over the original implementation [S1, unofficial].
- **Strengths:** Significantly reduces warp stalls and false sharing during z-order sorting [S1, unofficial].
- **Limitations:** Increased VRAM usage [S1, unofficial].

## Background
The Per-Gaussian Backward Pass was introduced by Mallick et al. [56] (cited in [S1, unofficial]) to address the computational suboptimality of atomic operations in standard 3D Gaussian Splatting backpropagation [S1, unofficial]. It is part of the broader effort to optimize training efficiency in differentiable volume rendering.

## Sources
[S1] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
