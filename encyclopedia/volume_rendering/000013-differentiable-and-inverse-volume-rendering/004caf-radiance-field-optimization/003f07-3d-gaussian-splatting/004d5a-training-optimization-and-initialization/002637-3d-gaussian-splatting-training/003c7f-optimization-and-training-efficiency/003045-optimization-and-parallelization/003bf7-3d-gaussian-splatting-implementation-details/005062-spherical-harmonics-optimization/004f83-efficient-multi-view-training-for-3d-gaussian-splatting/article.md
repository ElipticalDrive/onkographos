# Efficient Multi-View Training for 3D Gaussian Splatting

Also called: Multi-View Mini-Batch Training for 3DGS, Thread-Efficient Partial Rendering for Multi-View 3DGS

## Overview
This method enables stable optimization of 3D Gaussian Splatting (3DGS) by processing multiple camera views per training iteration, addressing the high gradient variance inherent in the standard single-view approach. While standard 3DGS renders one image per iteration due to rasterization overhead, this method uses thread-efficient partial rendering to gather pixels from multiple views without idle thread waste, and introduces a 3D-space compatible loss to resolve adaptive density control conflicts. It sets itself apart from [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md) by focusing on batch geometry rather than radiance compression, and from [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md) by modifying the training loop structure rather than the gradient computation path.

## How it works
### Inputs and Outputs
**Inputs:**
- A set of $N$ training images with corresponding camera poses.
- A 3D Gaussian scene representation $\{G_j\}_{j=1}^M$ defined by 3D mean $\mu$, scale $S$, rotation $R$, opacity $o$, and spherical harmonic coefficients $\mathbf{c}$ (as defined in [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md)).
- A mini-batch size $K$ (number of views per iteration).

**Outputs:**
- Updated Gaussian parameters $\{G_j\}$ after optimization step.

### Data Structures
The rasterizer uses a tile-based structure. The image is split into tiles. A single thread block is allocated to a single tile. Threads within a block cooperate to preprocess Gaussians and render pixels. A thread is freed only when all peer threads in the same block have completed their work [S1, unofficial].

### Method Steps

1.  **Mini-batch Selection:** Randomly sample $K$ distinct views (images) from the training dataset to form a mini-batch. Unlike standard 3DGS which samples one view, this method aggregates rays from $K$ views [S1, unofficial].

2.  **Thread-Efficient Partial Rendering:**
    To avoid the idle thread overhead of naive multi-view rendering (where blocks are allocated per image, causing synchronization stalls), the method employs a partial rendering scheme.
    -   The rasterizer processes the selected $K$ views.
    -   Instead of allocating a full block per image, threads are managed to ensure no idle time occurs while waiting for other views to complete [S1, unofficial].
    -   For each pixel $p$ in the selected views, the rendered color $\hat{C}(p)$ is computed by alpha-blending sorted Gaussians $\{G_j\}_{j=1}^M$ projected to 2D:
        $$
        \hat{C}(p) = \sum_{i=1}^{M} c_i T_i o_i G_i(p)
        $$
        where $T_i = \prod_{k=1}^{i-1} (1 - o_k G_k(p))$ and $G_i(p) = \exp(-\frac{1}{2}(p - \mu'_i)^T (\Sigma'_i)^{-1} (p - \mu'_i))$. Here $\mu'$ and $\Sigma'$ are the projected 2D mean and covariance derived from 3D parameters [S1, unofficial].

3.  **Loss Computation (D-SSIM in 3D):**
    Standard 3DGS uses D-SSIM loss computed in 2D image space. This is incompatible with multi-view training because positional gradients from different views reside in different image planes, making vector arithmetic invalid for Adaptive Density Control (ADC).
    -   The method replaces 2D D-SSIM with a **3D D-SSIM** loss.
    -   This loss is computed directly in 3D space using the Gaussian primitives' parameters, ensuring that gradients for densification are consistent across views [S1, unofficial].

4.  **Gradient Estimation and Variance Reduction:**
    The variance of the mini-batch gradients $\nabla \bar{\phi}_k$ is defined as $V(x) := E[\|x - \mu\|_2^2]$ where $E[x] = \mu$.
    -   The method estimates variance using Monte Carlo sampling:
        $$
        V_{B_k \in S} (\nabla \bar{\phi}_k) = E[\|\nabla \bar{\phi}_k - P(w_t)\|_2^2]
        $$
    -   Experiments show that multi-view mini-batches yield smaller gradient variance than single-view, leading to more stable optimization [S1, unofficial].

5.  **Parameter Update:**
    Update weights $w$ (including Gaussian parameters) using the learning rate $\eta_t$:
    $$
    w_{t+1} = w_t - \eta_t \nabla \bar{\phi}_k (w_t)
    $$
    where $\nabla \bar{\phi}_k (w_t) = \frac{1}{|B_k|} \sum_{i \in B_k} \nabla \phi_i (w_t)$ [S1, unofficial].

6.  **Adaptive Density Control (ADC) Adjustment:**
    Standard ADC assumes all 2D positional gradients reside within a single plane. In multi-view training, gradients come from different planes.
    -   The method modifies ADC to operate on the 3D-space gradients derived from the 3D D-SSIM loss, preventing suboptimal densification caused by mixing vectors from different image spaces [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_step(gaussians, camera_poses, images, learning_rate, batch_size_k):
    """
    One training iteration using efficient multi-view rendering.
    """
    # 1. Sample K views
    selected_indices = random_sample(camera_poses, batch_size_k)
    selected_cameras = [camera_poses[i] for i in selected_indices]
    target_images = [images[i] for i in selected_indices]
    
    # 2. Thread-Efficient Partial Rendering
    # Render all K views in a single pass, managing thread blocks to avoid idle threads
    rendered_images = []
    for cam, target in zip(selected_cameras, target_images):
        # Project Gaussians to 2D for this camera
        projected_gaussians = project_gaussians(gaussians, cam)
        
        # Render using tile-based rasterizer with thread-efficient scheduling
        # This avoids allocating a full block per image which causes idle threads
        rendered = tile_rasterize_thread_efficient(projected_gaussians, cam.resolution)
        rendered_images.append(rendered)
        
    # 3. Compute 3D D-SSIM Loss
    # Compute loss in 3D space to ensure gradient consistency for ADC
    total_loss = 0.0
    for rendered, target in zip(rendered_images, target_images):
        # 3D D-SSIM computes similarity based on Gaussian parameters in 3D
        # rather than pixel coordinates in 2D
        loss = compute_3d_dssim_loss(gaussians, rendered, target)
        total_loss += loss
        
    avg_loss = total_loss / batch_size_k
    
    # 4. Backpropagation
    gradients = compute_gradients(avg_loss, gaussians)
    
    # 5. Update Parameters
    # w_{t+1} = w_t - eta * grad
    gaussians.update_parameters(gradients, learning_rate)
    
    # 6. Adaptive Density Control (ADC)
    # Use 3D gradients for densification to handle multi-view consistency
    gaussians.adaptive_density_control(gradients)
    
    return avg_loss
```

### Parameters
-   **Learning Rate ($\eta_t$):** Standard 3DGS learning rate schedule [S1, unofficial].
-   **Batch Size ($K$):** Number of views per iteration. Not given in the sources.
-   **Monte Carlo Samples ($K_{mc}$):** Number of mini-batches sampled to estimate gradient variance during analysis. Not given in the sources.

### Edge Cases and Pitfalls
-   **Idle Threads:** Naive multi-view rendering allocates a single block to a single tile per image. If one image renders faster, threads wait for others, causing significant overhead. The thread-efficient scheme mitigates this [S1, unofficial].
-   **ADC Incompatibility:** Standard ADC assumes gradients are in a single 2D plane. Multi-view training produces gradients in different spaces. Using standard ADC leads to suboptimal densification. The 3D D-SSIM loss resolves this [S1, unofficial].
-   **Gradient Variance:** Single-view training has higher gradient variance. Multi-view reduces this, improving stability [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** Higher per-iteration cost than single-view 3DGS due to rendering $K$ images, but thread-efficient rendering minimizes overhead. The reduction in gradient variance may allow for fewer iterations to converge.
-   **Memory Complexity:** Requires memory for $K$ rendered images and their gradients simultaneously.
-   **Performance:** Multi-view training shows smaller variance of mini-batch gradients than single-view, resulting in more stable optimization [S1, unofficial].
-   **Strengths:** More stable optimization; leverages multi-view consistency; reduces gradient noise.
-   **Limitations:** Requires modification of the rasterizer for thread efficiency; requires 3D-space loss computation; higher memory usage per iteration.
-   **When to use:** When training stability is critical and hardware supports parallel rendering of multiple views.
-   **When not to use:** When memory is constrained or when single-view convergence is sufficient.

## Background
-   **Origin:** Proposed by Minhyuk Choi, Injae Kim, and Hyunwoo J. Kim from Korea University and KAIST [S1, unofficial].
-   **History:** Published in June 2025 [S1, unofficial].
-   **Context:** Standard 3DGS uses single-view mini-batches because its rasterization pipeline makes gathering individual pixels from multiple images difficult. NeRF implicitly benefits from multi-view learning. This method adapts 3DGS to multi-view training [S1, unofficial].

## Sources
[S1] Efficient multi-view training for 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2506.12727, fetched 2026-10-26 (unofficial)
