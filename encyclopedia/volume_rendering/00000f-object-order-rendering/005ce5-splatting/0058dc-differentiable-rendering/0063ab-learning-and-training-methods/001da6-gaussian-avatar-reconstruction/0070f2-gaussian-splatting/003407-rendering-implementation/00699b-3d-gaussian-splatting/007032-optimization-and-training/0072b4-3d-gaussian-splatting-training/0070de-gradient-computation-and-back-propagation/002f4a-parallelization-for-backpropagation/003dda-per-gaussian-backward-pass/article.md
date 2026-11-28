# Per-Gaussian Backward Pass

Also called: Tile-based backward pass

## Overview
The Per-Gaussian Backward Pass is a backward pass optimization for 3D Gaussian Splatting that parallelizes gradient computation over Gaussians rather than pixels to reduce atomic contention [S1, unofficial]. It addresses the performance bottleneck of standard pixel-parallel backpropagation, where splats contributing to arbitrary numbers of pixels require frequent atomic operations for accumulation [S1, unofficial]. By computing gradients per Gaussian instead of per tile, it reduces false sharing and atomic contention during z-order sorting [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** The method takes the loss gradients with respect to rendered image pixels ($\frac{\partial L}{\partial I}$) and the forward pass state, including Gaussian parameters, tile assignments, and stored alpha blending states [S1, unofficial].
- **Output:** The method outputs the gradients with respect to Gaussian parameters, such as position, covariance, opacity, and spherical harmonics coefficients [S1, unofficial].

**Data Structures:**
- **Alpha Blending State Buffer:** A buffer stores the intermediate alpha blending state (accumulated opacity and color) at each non-empty pixel [S1, unofficial].
- This state is recorded after every 32nd splat in the respective tile list during the forward pass [S1, unofficial].

**Steps:**
1. **Forward Pass Recording:** During the forward rasterization pass, for each tile, store the alpha blending state at each non-empty pixel after processing every 32nd Gaussian (splat) in the tile's sorted list [S1, unofficial].
2. **Gradient Replay:** For each Gaussian, threads compute gradients locally based on the stored alpha states and the final pixel gradients, avoiding global memory synchronization and atomic operations [S1, unofficial].
3. **Parameter Update:** Accumulate the computed gradients for the Gaussians [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def per_gaussian_backward(tile_gaussians, alpha_states, pixel_gradients, tile_idx):
    """
    tile_gaussians: List of Gaussian parameters sorted by z-order for this tile.
    alpha_states: 2D array [num_pixels][num_steps] storing alpha blending state after every 32nd splat [S1, unofficial].
    pixel_gradients: Gradient of loss w.r.t. final pixel colors [num_pixels][3] [S1, unofficial].
    tile_idx: Index of the current tile.
    """
    num_gaussians = len(tile_gaussians)
    num_pixels = len(pixel_gradients)
    gradients = [None] * num_gaussians

    # Step 1: Forward pass recording is handled externally by storing states every 32 splats [S1, unofficial]

    # Step 2 & 3: Gradient Replay and Parameter Update
    for g_idx in range(num_gaussians):
        gaussian = tile_gaussians[g_idx]
        step_idx = g_idx // 32

        for p_idx in range(num_pixels):
            # Standard rasterization coverage check [G]
            if not gaussian.is_covered(p_idx):
                continue

            # Retrieve stored alpha blending state (accumulated opacity and color) at this checkpoint [S1, unofficial]
            alpha_state_before = alpha_states[p_idx][step_idx]

            # Compute gradient for this Gaussian using stored alpha state and final pixel gradient [G]
            # Exact derivative formula depends on the forward alpha blending equation [G]
            grad_g = gaussian.opacity * alpha_state_before.accumulated_opacity * pixel_gradients[p_idx]

            if gradients[g_idx] is None:
                gradients[g_idx] = grad_g
            else:
                gradients[g_idx] += grad_g

    return gradients
```

**Parameters:**
- **Bucket Size:** 32 Gaussians (one GPU warp), matching the checkpoint interval [S1, unofficial].
- **Tile Size:** Typically 16x16 pixels (256 pixels) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **VRAM Usage:** This approach increases VRAM usage due to the storage of alpha blending states [S1, unofficial].
- **Precision:** Storing intermediate alpha states may introduce precision issues if not handled carefully [G].
- **Empty Pixels:** Pixels not covered by any Gaussian in a bucket should be skipped to avoid unnecessary computation [G].

## Subtopics

### [Adaptive Scheduling for Backward Pass Implementations](10235-adaptive-scheduling-for-backward-pass-implementations.md)
Dynamically switches between rasterization implementations during the backward pass to minimize atomic contention and maximize thread utilization. It sets itself apart from static backward strategies by using a Thompson sampling scheduler to select the fastest implementation based on measured latency. Choose it when scene complexity or hardware configuration causes significant performance variance across different backward pass implementations.

### [Per-Gaussian Backward](10252-per-gaussian-backward.md)
Parallelizes gradient computation over Gaussians rather than pixels to reduce atomic contention. It sets itself apart from standard pixel-parallel backpropagation by operating on fixed-size buckets of 32 Gaussians (one GPU warp), allowing threads to collaboratively replay the alpha blending process using warp-level primitives. Choose it when warp stalls from global memory synchronization are the primary bottleneck.

## Operating characteristics
- **Time Complexity:** Reduces the number of required atomic operations by a factor equal to the number of pixels in each tile (256 for 16x16 tiles) [S1, unofficial].
- **Memory Complexity:** Increases VRAM usage [S1, unofficial].
- **Performance:** On Mip-NeRF360 outdoor scenes (RTX 4090), training time decreases from 16m56s to 14m14s (1.20x speedup over base), but VRAM increases from 6.33GiB to 7.69GiB (1.20x increase) and Gaussian count from 3.87M to 3.89M [S1, unofficial].
- On RTX 5090, training takes 163 seconds on average, a 5× improvement over the original implementation [S1, unofficial].
- **Strengths:** Significantly reduces warp stalls and false sharing during z-order sorting [S1, unofficial].
- **Limitations:** Increased VRAM usage [S1, unofficial].

## Background
The Per-Gaussian Backward Pass was introduced by Mallick et al. [56] (cited in [S1, unofficial]) to address the computational suboptimality of atomic operations in standard 3D Gaussian Splatting backpropagation [S1, unofficial].

## Sources
[S1] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
