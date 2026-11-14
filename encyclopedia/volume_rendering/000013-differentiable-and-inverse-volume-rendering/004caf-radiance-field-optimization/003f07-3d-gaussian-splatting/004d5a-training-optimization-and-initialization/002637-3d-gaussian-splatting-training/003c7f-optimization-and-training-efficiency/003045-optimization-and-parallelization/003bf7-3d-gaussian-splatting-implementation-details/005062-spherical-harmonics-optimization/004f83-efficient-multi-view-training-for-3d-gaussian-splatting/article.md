# Efficient Multi-View Training for 3D Gaussian Splatting

Also called: Multi-View Mini-Batch Training for 3DGS, Thread-Efficient Partial Rendering for Multi-View 3DGS

## Overview
This method enables stable optimization of 3D Gaussian Splatting (3DGS) by processing multiple training images per iteration, addressing the high gradient variance inherent in the standard single-view approach. Unlike standard 3DGS which renders one image per step due to rasterization overhead, this method uses thread-efficient partial rendering to gather pixels from multiple views without idle thread waste, while replacing the incompatible Adaptive Density Control (ADC) with a 3D-space compatible densification strategy.

## How it works
### Inputs and Outputs
- **Inputs**: A set of $N$ training images with corresponding camera poses; 3D Gaussian primitives defined by mean $\mu$, covariance $\Sigma$ (derived from scale $S$ and rotation $R$), opacity $o$, and spherical harmonic coefficients $\mathbf{c}$; learning rate $\eta_t$.
- **Outputs**: Updated Gaussian parameters $w_{t+1}$; rendered images $\hat{C}(p)$.

### Data Structures
- **Tile-based Rasterizer State**: The image is split into tiles. A single thread block is allocated to a single tile. Threads within a block cooperate to preprocess Gaussians and render pixels. A thread is freed only when all peer threads in the same block complete their work [S1, unofficial].
- **Mini-batch Buffer**: Stores indices of pixels sampled from multiple views.

### Method Steps
1. **Preprocessing**: Cull, project, and organize 3D Gaussian primitives for the current set of cameras [S1, unofficial].
2. **Thread-Efficient Partial Rendering**:
   - Instead of rendering full images, sample a mini-batch of pixels from $K$ different views.
   - Assign these pixels to tiles. To avoid idle threads (which occur in naive multi-view rendering when some tiles finish before others), the rasterizer must ensure balanced workload distribution across blocks. If a block finishes early, it should not block the synchronization barrier for the entire iteration if possible, or the workload must be balanced such that all blocks in the iteration complete simultaneously [S1, unofficial].
   - Render the sampled pixels using the standard alpha-blending equation:
     $$ \hat{C}(p) = \sum_{i=1}^{M} c_i T_i o_i G_i(p) $$
     where $T_i = \prod_{k=1}^{i-1} (1 - o_k G_k(p))$ and $G_i(p) = \exp(-\frac{1}{2}(p - \mu'_i)^T (\Sigma'_i)^{-1} (p - \mu'_i))$ [S1, unofficial].
3. **Loss Computation**:
   - Compute the loss $\phi_i(w)$ for each sampled pixel $i$ using a differentiable loss function (e.g., D-SSIM adapted for 3D space or L1/L2) [S1, unofficial].
4. **Gradient Aggregation**:
   - Compute the mini-batch gradient:
     $$ \nabla \bar{\phi}_k (w_t) = \frac{1}{|B_k|} \sum_{i \in B_k} \nabla \phi_i (w_t) $$
     where $B_k$ is the set of sampled pixels from multiple views [S1, unofficial].
5. **Parameter Update**:
   - Update weights:
     $$ w_{t+1} = w_t - \eta_t \nabla \bar{\phi}_k (w_t) $$
     [S1, unofficial].
6. **Adaptive Density Control (ADC) Adjustment**:
   - Standard ADC assumes gradients reside in a single 2D plane. For multi-view training, gradients come from different image planes. The method replaces standard 2D gradient accumulation with a 3D-space compatible metric (e.g., D-SSIM computed in 3D or accumulated 3D positional gradients) to decide densification (splitting/pruning) [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_step(gaussians, cameras, images, learning_rate, num_views):
    """
    One training iteration using efficient multi-view rendering.
    """
    # 1. Sample pixels from multiple views
    # Instead of rendering all pixels, sample a subset B_k from K views
    batch_pixels = []
    for view_idx in range(num_views):
        # Sample random pixel coordinates (u, v) and camera index
        pixels = sample_pixels(images[view_idx], count=NUM_SAMPLES_PER_VIEW)
        batch_pixels.extend(pixels)
    
    # 2. Thread-Efficient Rendering
    # The rasterizer must handle partial rendering.
    # Allocate tiles. Ensure workload balance to prevent idle threads.
    # Naive approach: Render all pixels from all views.
    # Efficient approach: Only render pixels in batch_pixels.
    # The rasterizer splits image into tiles. A single block is allocated to a single tile.
    # Threads are freed when all peers in the block complete.
    
    rendered_colors = {}
    for pixel in batch_pixels:
        # Project Gaussians to 2D for this pixel's camera
        projected_gaussians = project_gaussians(gaussians, pixel.camera)
        
        # Alpha blend
        color = 0.0
        transmittance = 1.0
        for g in sorted(projected_gaussians, key=lambda x: x.depth):
            # Evaluate Gaussian at pixel
            g_val = exp(-0.5 * (pixel.pos - g.mu_2d).T @ inv(g.Sigma_2d) @ (pixel.pos - g.mu_2d))
            color += g.color * transmittance * g.opacity * g_val
            transmittance *= (1.0 - g.opacity * g_val)
        
        rendered_colors[pixel.id] = color

    # 3. Compute Loss and Gradients
    total_loss = 0.0
    gradients = {} # Accumulate gradients for Gaussians
    
    for pixel in batch_pixels:
        gt_color = images[pixel.camera].pixel(pixel.u, pixel.v)
        pred_color = rendered_colors[pixel.id]
        
        # Loss function (e.g., L1 or D-SSIM)
        loss = compute_loss(pred_color, gt_color)
        total_loss += loss
        
        # Backpropagate to get gradients for this pixel
        grad = compute_gradient(loss, pixel, gaussians)
        
        # Accumulate gradients
        for g_id, g_grad in grad.items():
            if g_id not in gradients:
                gradients[g_id] = 0.0
            gradients[g_id] += g_grad

    # 4. Update Parameters
    for g_id, g_grad in gradients.items():
        # Average gradient over batch size
        avg_grad = g_grad / len(batch_pixels)
        gaussians[g_id].update(learning_rate, avg_grad)

    # 5. Adaptive Density Control (3D-compatible)
    # Use 3D-space metrics for densification to avoid mixing 2D planes
    densify_gaussians_3d(gaussians, gradients)
    
    return total_loss
```

### Parameters
- **Learning Rate ($\eta_t$)**: Standard Adam or SGD learning rate.
- **Number of Sampled Mini-batches ($K$)**: Used for variance estimation in analysis, but in training, the batch size is the total number of pixels sampled from multiple views [S1, unofficial].
- **Number of Views per Batch**: Typically small (e.g., 2-4) to balance memory and gradient stability [S1, unofficial].

### Edge Cases and Pitfalls
- **Idle Threads**: Naive multi-view rendering causes significant overhead because the tile-based rasterizer allocates a single block to a single tile. If one view's tiles finish before others, threads idle. The method requires thread-efficient scheduling to mitigate this [S1, unofficial].
- **ADC Incompatibility**: Standard ADC assumes all 2D positional gradients reside within a single plane. Multi-view training risks performing arithmetic between vectors in different spaces (different image planes), resulting in suboptimal densification. A 3D-space compatible densification mechanism is required [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Higher per-iteration cost than single-view due to rendering multiple views, but reduced by partial rendering (sampling pixels instead of full images).
- **Memory Complexity**: Increased memory usage for storing gradients and intermediate states for multiple views.
- **Performance**: Multi-view mini-batch has smaller variance of mini-batch gradients than single-view, leading to more stable optimization [S1, unofficial].
- **Strengths**: More stable convergence; better utilization of GPU parallelism if thread efficiency is maintained.
- **Limitations**: Requires custom rasterizer modifications for thread efficiency; ADC must be redesigned for 3D compatibility.

## Background
- **Origin**: Proposed by Minhyuk Choi, Injae Kim, and Hyunwoo J. Kim from Korea University and KAIST [S1, unofficial].
- **History**: Published in 2025 (arXiv:2506.12727) [S1, unofficial].
- **Context**: Standard 3DGS uses single-view mini-batches because its rasterization pipeline makes gathering individual pixels from multiple images difficult. NeRF implicitly benefits from multi-view learning. This method bridges that gap for 3DGS [S1, unofficial].

## Sources
[S1] Efficient multi-view training for 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2506.12727, fetched 2026-10-26 (unofficial)
