# Per-Gaussian Backward Pass

Also called: Tile-based backward pass

## Overview
The Per-Gaussian Backward Pass is a parallelization strategy for the backward pass of 3D Gaussian Splatting that computes gradients by iterating over Gaussians rather than pixels [S1, unofficial]. It solves the high atomic contention and false sharing inherent in standard pixel-parallel backpropagation, where thousands of threads atomically update the same Gaussian parameters [S1, unofficial]. By reducing the number of required atomic operations by a factor equal to the number of pixels in a tile (typically 256 for 16×16 tiles), it significantly accelerates training time at the cost of increased VRAM usage [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: The forward pass state (sorted Gaussians per tile, alpha blending states), scene parameters (Gaussian positions, covariances, colors, opacities), and the loss gradient with respect to the rendered image.
- Output: Gradients for all Gaussian parameters (position, scale, rotation, color, opacity).

**Data Structures:**
- **Alpha Blending State Buffer:** During the forward pass, for each non-empty pixel in a tile, the intermediate alpha blending state (transmittance and accumulated color) is stored after every 32nd splat in the tile's Gaussian list [S1, unofficial]. This creates a checkpoint history required to replay the blending process during the backward pass.
- **Tile List:** A list of Gaussians sorted by depth (z-order) for each tile [S1, unofficial].

**Steps:**
1. **Forward Pass Checkpointing:** As Gaussians are rasterized into a tile, maintain the alpha blending state for each pixel. Store this state in a buffer after processing every 32nd Gaussian in the tile's sorted list [S1, unofficial]. This value of 32 corresponds to one GPU warp, allowing warp-level parallelism during the backward replay [S1, unofficial].
2. **Backward Pass Initialization:** For each tile, identify the Gaussians that contributed to the rendered pixels.
3. **Gradient Accumulation via Replay:** Instead of computing gradients pixel-by-pixel, iterate over the Gaussians in the tile. For each Gaussian, retrieve the stored alpha blending states from the checkpoints [S1, unofficial].
4. **Warp-Level Replay:** Threads within a warp collaboratively replay the alpha blending process using the stored states to compute the derivative of the loss with respect to the current Gaussian's parameters [S1, unofficial]. This avoids global memory latency stalls associated with atomic operations [S1, unofficial].
5. **Shared Memory Optimization:** To mitigate the increased VRAM usage from storing blending states, exploit shared memory to cache frequently accessed data and reduce global memory allocations [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def per_gaussian_backward(tile_gaussians, tile_pixels, loss_grad):
    """
    tile_gaussians: List of Gaussian objects sorted by depth for the tile.
    tile_pixels: List of pixel coordinates in the tile.
    loss_grad: Gradient of loss w.r.t. rendered image for these pixels.
    """
    # Step 1: Retrieve stored alpha blending states from forward pass
    # These are stored after every 32nd splat (warp size)
    blending_states = get_checkpointed_states(tile_gaussians)
    
    gradients = {}
    
    # Step 2: Iterate over Gaussians in the tile
    for i, gaussian in enumerate(tile_gaussians):
        grad = compute_gaussian_gradient(gaussian, blending_states, loss_grad)
        gradients[gaussian.id] = grad
        
    return gradients

def compute_gaussian_gradient(gaussian, blending_states, loss_grad):
    """
    Replays alpha blending for the specific Gaussian using checkpoints
    to compute its gradient without atomic contention.
    """
    # Find relevant checkpoints for this Gaussian's depth range
    relevant_states = get_relevant_checkpoints(gaussian, blending_states)
    
    # Initialize gradient accumulator
    grad_pos = zeros(3)
    grad_scale = zeros(3)
    grad_rot = zeros(4)
    grad_color = zeros(3)
    grad_opacity = 0.0
    
    # Replay blending process for affected pixels
    for pixel in gaussian.projected_pixels:
        # Get transmittance and accumulated color from checkpoints
        T_prev, C_acc = get_state_at_pixel(pixel, relevant_states)
        
        # Compute contribution of this Gaussian to the pixel
        alpha = gaussian.compute_alpha(pixel)
        color = gaussian.compute_color(pixel)
        
        # Derivative of pixel color w.r.t Gaussian parameters
        # dL/dG = dL/dI * dI/dG
        # dI/dG involves alpha, T_prev, and color
        dL_dI = loss_grad[pixel]
        
        # Compute partial derivatives
        # ... (standard differentiable rendering chain rules) ...
        
        # Accumulate gradients
        grad_pos += ...
        grad_scale += ...
        grad_rot += ...
        grad_color += ...
        grad_opacity += ...
        
    return {
        'pos': grad_pos,
        'scale': grad_scale,
        'rot': grad_rot,
        'color': grad_color,
        'opacity': grad_opacity
    }
```

**Parameters:**
- **Checkpoint Interval:** 32 Gaussians (one warp) [S1, unofficial].
- **Tile Size:** Typically 16×16 pixels (256 pixels) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **VRAM Usage:** Storing blending states increases VRAM consumption. This must be mitigated by using shared memory to reduce global memory allocations [S1, unofficial].
- **False Sharing:** Standard pixel-parallel backpropagation suffers from false sharing when threads in different tiles write to the same cache line. The per-Gaussian approach reduces this issue [S1, unofficial].
- **Z-Order Sorting:** Repeated z-ordering during densification can slow down training if not optimized [S1, unofficial].

## Subtopics

### Adaptive Scheduling for Backward Pass Implementations
A strategy to switch between two rasterization backward implementations to minimize atomic contention and maximize thread utilization, selected via a Thompson sampling scheduler [S2, unofficial]. Implementation 1 launches one thread block per tile and dynamically adjusts the Gaussian batch size $S$ based on the number of Gaussians $N$ binned to the tile, with $S$ chosen by rounding up $\sqrt{NP}$ to a multiple of subgroup size, capped to 128 [S2, unofficial]. Implementation 2 runs a forward pass parallelized over pixels to compute transmittance and its derivative with respect to Gaussian parameters for each Gaussian-pixel pair, storing them in shared memory, then runs a backward pass parallelized over Gaussians to fetch these pre-computed values [S2, unofficial]. Projected Gaussians are parameterized following [LDC* 25] to simplify opacity computation into matrix multiplication [S2, unofficial]. The Thompson sampling scheduler stores latency beliefs as distributions, randomly selects an implementation with higher probabilities for faster ones, and updates beliefs based on measured latency [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Reduces the number of atomic operations by a factor equal to the number of pixels in a tile (256 for 16×16 tiles) [S1, unofficial].
- **Memory Complexity:** Increases VRAM usage due to storing alpha blending states. On Mip-NeRF360 outdoor scenes (RTX 4090), VRAM increases from 6.33 GiB to 7.69 GiB (1.20x increase) [S1, unofficial].
- **Performance:** On Mip-NeRF360 outdoor scenes (RTX 4090), training time decreases from 16m56s to 14m14s (1.20x speedup) [S1, unofficial]. Newer GPUs exhibit greater speedups, with an RTX 5090 showing a 5× improvement over the original implementation [S1, unofficial].
- **Strengths:** Significantly reduces warp stalls caused by global memory latency and atomic contention [S1, unofficial].
- **Limitations:** Increased VRAM usage [S1, unofficial]. Performance may vary depending on scene characteristics and hardware, requiring adaptive scheduling in some implementations [S2, unofficial].

## Background
The Per-Gaussian Backward Pass was introduced by Mallick et al. [56] as a solution to the computational suboptimality of atomic operations in standard 3D Gaussian Splatting backpropagation [S1, unofficial]. It was later improved by Faster-GS to exploit shared memory and reduce VRAM costs [S1, unofficial]. Adaptive scheduling for backward pass implementations was introduced by VkSplat to dynamically select the most efficient backward strategy based on scene and hardware characteristics [S2, unofficial].

## Sources
[S1] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
[S2] VkSplat: High-Performance 3DGS Training in Vulkan Compute, https://doi.org/10.2312/egs.20261024, fetched 2026-10-29 (unofficial)
