# Backpropagation with Per-Splat Parallelization

Also called: Per-splat backward, Per-splat parallelization, Recursive Per-Splat Parallelization for Backward Propagation, Splat-parallel backprop, Warp-level gradient accumulation

## Overview
This method is a GPU acceleration technique for the backward propagation pass in 3D Gaussian Splatting (3DGS) that eliminates atomic contention and depth-sorting constraints by processing gradients at the splat level within warps, using recursive state circulation [S2, unofficial]. It sets itself apart from the original 3DGS pixel-parallel approach by mapping threads to 2D splats rather than pixels, thereby avoiding serialized atomic operations when multiple threads contend for the same Gaussian's gradient [S1]. Unlike sibling methods like [DARBF Backpropagation](10254-darbf-backpropagation.md) which may rely on different basis function derivatives, or [Stochastic Gradient Estimation for Differentiable Rasterization](11381-stochastic-gradient-estimation-for-differentiable-rasterizat.md) which approximates gradients, this method provides an exact, deterministic gradient computation optimized for warp-level parallelism [S1][S2, unofficial].

## How it works
The method replaces the original 3DGS backward pass, where threads map to pixels and iterate over depth-sorted splats, with a scheme where threads maintain per-splat state and exchange smaller per-pixel states (transmittance $T$ and accumulated color $RGB$) [S1]. This approach leverages the differentiability of the forward rasterization defined in [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md).

### Inputs and Outputs
**Inputs:**
*   Gradient of the loss with respect to pixel colors: $\frac{\partial L}{\partial C_j}$ for all pixels $j$ in a tile.
*   Forward pass checkpoint data: Per-pixel states (transmittance $T$ and accumulated color $C$) stored at specific intervals during the forward pass.
*   Gaussian parameters: Position, covariance, opacity, spherical harmonics coefficients, and color for all active splats.
*   Tile configuration: A tile of $256$ pixels, partitioned into groups matching warp size.

**Outputs:**
*   Gradients with respect to all Gaussian parameters: $\frac{\partial L}{\partial \theta}$ for all splats $s$ in the tile.

### Data Structures
1.  **Per-Pixel State:** A tuple $(T_j, C_j)$ representing the transmittance and accumulated color for pixel $j$. In the forward pass, these are stored for every $N$-th splat (where $N=32$) [S1].
2.  **Per-Splat Gradient Accumulator:** Global memory location for each splat $s$ storing $\frac{\partial L}{\partial \theta_s}$.
3.  **Warp Shared Memory/Registers:** Used for intra-warp state exchange. Each lane $l$ in a warp of size $W=32$ holds state for a specific pixel or splat.

### Parameters
*   **Warp Size ($W$):** 32 threads [S1][S2, unofficial].
*   **Bucket Size:** Splats are divided into buckets of size 32, each scheduled to a CUDA warp [S1].
*   **Checkpoint Interval ($N$):** 32 splats [S1].
*   **Tile Size:** 256 pixels, partitioned into 8 groups of 32 pixels [S2, unofficial].

### Algorithm Steps
The backward pass is executed per tile. The tile is processed by warps, where each warp handles a bucket of 32 splats.

1.  **Initialization:**
    *   For each pixel $j$ in the tile, retrieve the initial per-pixel state $(T_j^{(0)}, C_j^{(0)})$ from the forward pass checkpoints. If the pixel has no splats, $T_j^{(0)} = 1.0, C_j^{(0)} = 0.0$ [S1].
    *   Each thread in the warp (lane $l$) is assigned to a specific pixel $j_l$ within the warp's group. Lane $l$ fetches the initial state for pixel $j_l$ [S2, unofficial].

2.  **State Circulation and Gradient Accumulation (Recursive Per-Splat):**
    *   The warp processes its bucket of 32 splats sequentially. Let the splats in the bucket be $s_0, s_1, \dots, s_{31}$.
    *   For each splat $s_k$ in the bucket (where $k$ ranges from 0 to 31):
        a.  **State Exchange:** The warp circulates the per-pixel states among its 32 lanes. Lane $l$ receives the current per-pixel state $(T_{j_l}, C_{j_l})$ for its assigned pixel $j_l$ [S1][S2, unofficial].
        b.  **Splat Evaluation:** The splat $s_k$ is evaluated at pixel $j_l$. Compute the splat's opacity $\alpha_{s_k, j_l}$ and color $c_{s_k, j_l}$ at pixel $j_l$ using the Gaussian parameters [S1].
        c.  **Blending Logic:** Apply the alpha compositing equation to update the pixel state for the next splat in the depth order. However, since this method processes splats in buckets and relies on forward checkpoints, the gradient calculation uses the stored forward states. Specifically, thread $l$ computes the gradient contribution for splat $s_k$ at pixel $j_l$.
        d.  **Gradient Computation:** The gradient for splat $s_k$ with respect to pixel $j_l$ is computed. Let $\frac{\partial L}{\partial C_{j_l}}$ be the incoming loss gradient. The contribution to the splat's gradient is derived from the chain rule through the alpha compositing equation. The thread accumulates this gradient into the global gradient accumulator for splat $s_k$ [S1].
        e.  **State Update for Next Splat:** To prepare for the next splat $s_{k+1}$ in the bucket, the per-pixel state $(T_{j_l}, C_{j_l})$ is updated to reflect the inclusion of splat $s_k$. This is done by applying the default alpha blending logic:
            $$ T_{new} = T_{old} \cdot (1 - \alpha_{s_k, j_l}) $$
            $$ C_{new} = C_{old} + T_{old} \cdot \alpha_{s_k, j_l} \cdot c_{s_k, j_l} $$
            The updated state is then passed to the next splat in the bucket via intra-warp shuffle [S1].

3.  **Completion:**
    *   Repeat the circulation and accumulation for all 8 groups of 32 pixels in the tile (since $256/32 = 8$) [S2, unofficial].
    *   Atomic updates to the global gradient accumulators are performed for each splat. Since each warp handles a unique bucket of splats, contention is minimized [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def backward_per_splat_parallelization(tile_pixels, splat_buckets, forward_checkpoints, loss_gradients):
    """
    tile_pixels: List of 256 pixel indices in the tile.
    splat_buckets: List of 8 buckets, each containing 32 splat indices.
    forward_checkpoints: Dictionary mapping (pixel_idx, splat_idx_interval) -> (T, C).
    loss_gradients: Array of shape (256,) containing dL/dC_j for each pixel.
    """
    W = 32  # Warp size
    N = 32  # Checkpoint interval
    
    # Initialize global gradient accumulators for all splats in the tile
    splat_gradients = {s: 0.0 for s in all_splats_in_tile}
    
    for group_idx in range(8):
        # Each warp handles one group of 32 pixels
        pixel_group = tile_pixels[group_idx * W : (group_idx + 1) * W]
        
        # Each lane l in the warp handles pixel pixel_group[l]
        for l in range(W):
            pixel_j = pixel_group[l]
            # Fetch initial state from forward pass checkpoints
            # Assuming checkpoint at splat 0 is always available
            T_j, C_j = forward_checkpoints[(pixel_j, 0)]
            
            # Process each bucket of 32 splats
            for bucket_idx in range(8):
                splat_bucket = splat_buckets[bucket_idx]
                
                # Circulate states within the warp for this bucket
                # Lane l holds state for pixel_j
                current_T = T_j
                current_C = C_j
                
                for k in range(W):
                    splat_s = splat_bucket[k]
                    
                    # Evaluate splat s at pixel j
                    alpha_s_j, color_s_j = evaluate_gaussian(splat_s, pixel_j)
                    
                    # Compute gradient contribution for splat s at pixel j
                    # dL/dtheta_s += dL/dC_j * dC_j/dtheta_s
                    # dC_j/dtheta_s involves alpha_s_j and color_s_j
                    grad_contrib = compute_gradient_contribution(
                        loss_gradients[pixel_j], 
                        current_T, 
                        alpha_s_j, 
                        color_s_j, 
                        splat_s
                    )
                    
                    # Accumulate gradient for splat s
                    splat_gradients[splat_s] += grad_contrib
                    
                    # Update state for next splat in the bucket
                    # T_new = T_old * (1 - alpha)
                    # C_new = C_old + T_old * alpha * color
                    current_T = current_T * (1 - alpha_s_j)
                    current_C = current_C + current_T * alpha_s_j * color_s_j
                    
                    # If not the last splat in bucket, shuffle state to next lane
                    # Note: In actual CUDA, this is done via __shfl_down_sync
                    if k < W - 1:
                        # Lane k+1 receives the updated state from lane k
                        # This is implicit in the loop structure if we assume
                        # each lane maintains its own pixel state across splats
                        pass 
                
                # After processing the bucket, the state (T_j, C_j) for pixel j
                # has been updated through all 32 splats in the bucket.
                # However, since we use checkpoints, we might need to reload
                # the state for the next bucket if the splats are not contiguous
                # in the depth sort. The method assumes checkpoints allow
                # restarting the state accumulation for each bucket.
                T_j, C_j = forward_checkpoints[(pixel_j, bucket_idx * N)]

    return splat_gradients
```
*Note: The pseudocode above simplifies the intra-warp shuffle mechanism. In practice, the state circulation is handled by CUDA warp shuffle instructions, where each lane holds the state for its assigned pixel and updates it as it processes each splat in the bucket, passing the updated state to the next splat's processing step within the same warp [S1][S2, unofficial].*

### Edge Cases and Pitfalls
*   **Atomic Contention:** The original 3DGS method suffers from heavy contention as many threads target the same memory locations for atomic writes [S2, unofficial]. This method mitigates this by parallelizing at the splat level, ensuring each warp handles a unique bucket of splats, thus reducing contention [S2, unofficial].
*   **Depth Sorting:** The original 3DGS requires strict depth order traversal, which can lead to redundant computations if the warp size does not align with the number of splats [S2, unofficial]. This method imposes no constraint on processing order, avoiding redundant arithmetic [S2, unofficial].
*   **Occlusion:** Iterating the tail of each tile’s depth-sorted list of splats often becomes redundant due to occlusion. The method tracks the last contributor across the tile to skip entire groups of splat-tile pairings in the backward pass, exploiting occlusion observed in the forward pass [S1].
*   **Precision:** Using tighter culling as proposed by Radl et al. [2024] minimizes redundant splats in both forward and backward passes, which can help maintain numerical stability by reducing the number of operations [S1].

## Subtopics

### [CUDA Backward Pass Optimization](5429-cuda-backward-pass-optimization.md)
This method applies low-level CUDA kernel optimizations to the 3D Gaussian Splatting backward pass to reduce memory access overhead and improve cache utilization. It specifically targets the reduction of redundant global memory transactions and improves warp-level parallelism without altering rendering results or optimization behavior. It differs from structural optimizations like Loss-Accumulation Guided (LAG) densification or pruning by reducing training time independently of the number of Gaussians, providing consistent speedups across all configurations.

## Operating characteristics
*   **Time Complexity:** The method reduces the time complexity of the backward pass by eliminating atomic contention and redundant computations. The original 3DGS backward pass has a time complexity proportional to the number of splats per pixel times the number of pixels, with a high constant factor due to atomic operations [S1]. This method reduces the constant factor by using warp-level parallelism and intra-warp shuffling [S1][S2, unofficial].
*   **Memory Complexity:** The method requires storing per-pixel states at checkpoint intervals during the forward pass. The memory overhead is proportional to the number of pixels times the number of checkpoints [S1]. This is a trade-off for the reduced memory bandwidth usage during the backward pass [S1].
*   **Measured Performance:** The method provides consistent speedups across all configurations, reducing training time independently of the number of Gaussians [S1]. It outperforms the original 3DGS backward pass and other concurrent works like DISTWAR [Durvasula et al. 2023] [S1].
*   **Strengths:** Eliminates atomic contention, avoids redundant computations, and leverages warp-level parallelism effectively [S1][S2, unofficial].
*   **Limitations:** Requires storing checkpoint data during the forward pass, which increases memory usage [S1]. The method assumes a warp size of 32, which is standard for CUDA but may need adjustment for other GPU architectures [S1][S2, unofficial].
*   **When to Use:** Use this method when training 3D Gaussian Splatting models on CUDA GPUs and performance is a bottleneck. It is particularly effective for scenes with a large number of splats and pixels [S1][S2, unofficial].
*   **When Not to Use:** Avoid this method if memory is extremely constrained, as the checkpoint storage increases memory usage [S1]. It is also not suitable for non-CUDA architectures without modification [S1][S2, unofficial].

## Background
The method was introduced in "Taming 3DGS: High-Quality Radiance Fields with Limited Resources" [S1] and further discussed in "SparseOIT: Improving Order-Independent Transparency 3DGS via Active Set Method" [S2, unofficial]. It addresses the performance bottleneck of the original 3DGS backward pass, which suffers from atomic contention and redundant computations due to its pixel-parallel design [S1][S2, unofficial]. The method builds on the principles of differentiable rasterization defined in [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md) and improves upon the pixel-parallel approach by introducing a splat-parallel strategy [S1][S2, unofficial].

## Sources
[S1] Taming 3DGS: High-Quality Radiance Fields with Limited Resources, https://doi.org/10.1145/3680528.3687694, fetched 2026-10-30
[S2] SparseOIT: Improving Order-Independent Transparency 3DGS via Active Set Method, https://doi.org/10.48550/arXiv.2605.13855, fetched 2026-10-29 (unofficial)
