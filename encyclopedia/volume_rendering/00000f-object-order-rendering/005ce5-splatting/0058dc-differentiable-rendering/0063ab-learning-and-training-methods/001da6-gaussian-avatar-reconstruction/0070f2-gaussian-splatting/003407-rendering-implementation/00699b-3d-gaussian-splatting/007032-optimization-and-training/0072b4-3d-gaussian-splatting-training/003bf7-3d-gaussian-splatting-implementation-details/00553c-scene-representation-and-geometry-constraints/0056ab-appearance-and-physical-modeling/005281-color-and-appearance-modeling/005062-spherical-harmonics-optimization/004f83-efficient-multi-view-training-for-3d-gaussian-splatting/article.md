# Efficient Multi-View Training for 3D Gaussian Splatting

Also called: Multi-View Mini-Batch Training for 3DGS, Thread-Efficient Partial Rendering for Multi-View 3DGS

## Overview
This method enables stable optimization of 3D Gaussian Splatting (3DGS) by training on multiple views per iteration, addressing the high variance of gradients inherent in the standard single-view approach [S1, unofficial]. It distinguishes itself from standard 3DGS by replacing the single-image mini-batch with a multi-image batch, while overcoming the rasterization overhead and adaptive density control incompatibilities that typically prevent such an implementation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with corresponding camera poses; 3D Gaussian primitives $\{G_i\}$ defined by mean $\mu_i$, covariance $\Sigma_i$, opacity $o_i$, and spherical harmonic coefficients for color $c_i$; learning rate $\eta_t$.
*   **Outputs:** Updated Gaussian parameters $w_{t+1}$; rendered images $\hat{C}(p)$.

**Data Structures:**
The rasterizer uses a tile-based architecture. The image is split into tiles. A single thread block is allocated to a single tile. Threads within a block cooperate to preprocess Gaussians and render pixels for that tile. A thread is freed only when all peer threads in the same block have completed their work [S1, unofficial].

**Method Steps:**

1.  **Mini-Batch Construction:**
    Instead of selecting a single image, select a subset of $K$ images (or all $N$ images) to form the mini-batch for the current iteration. This allows the gradient estimator to sample from a broader distribution of views [S1, unofficial].

2.  **Thread-Efficient Partial Rendering:**
    Standard naive multi-view rendering incurs significant overhead because the tile-based rasterizer allocates one block per tile per image. Rendering multiple images simultaneously causes idle threads when some blocks finish before others. To mitigate this, the method employs thread-efficient partial rendering [S1, unofficial]:
    *   The rendering workload is distributed such that threads are not blocked waiting for other images' tiles to complete if their own work is done, or the scheduling is adjusted to minimize idle time across the multi-view batch.
    *   Specifically, the rasterizer processes the multi-view batch in a way that avoids the synchronization stalls inherent in naive parallel rendering of multiple full images [S1, unofficial].

3.  **Gradient Estimation and Variance Reduction:**
    The method relies on the observation that multi-view mini-batches yield smaller gradient variance than single-view batches. The variance of the mini-batch gradient $\nabla \bar{\phi}_k$ is defined as:
    $$ V(\mathbf{x}) := E[\|\mathbf{x} - \mu\|_2^2] $$
    where $E[\mathbf{x}] = \mu$ [S1, unofficial].
    The variance for a mini-batch $B_k$ is estimated via Monte Carlo sampling:
    $$ V_{B_k \in S} (\nabla \bar{\phi}_k) = E[\|\nabla \bar{\phi}_k - P(w_t)\|_2^2] $$
    where $P(w_t)$ is the projection or expected gradient at weights $w_t$, and $K$ mini-batches are sampled to estimate this variance [S1, unofficial]. Lower variance leads to more stable optimization [S1, unofficial].

4.  **Weight Update:**
    The weights are updated using the standard SGD step:
    $$ w_{t+1} = w_t - \eta_t \nabla \bar{\phi}_k (w_t) $$
    where $\nabla \bar{\phi}_k (w_t) = \frac{1}{|B_k|} \sum_{i \in B_k} \nabla \phi_i (w_t)$ [S1, unofficial].

5.  **Adaptive Density Control (ADC) Adjustment:**
    Standard ADC assumes all 2D positional gradients reside within a single image plane. Multi-view training computes gradients across different image planes, making direct arithmetic between these vectors invalid and leading to suboptimal densification [S1, unofficial].
    *   The method requires a modified densification strategy that accounts for the multi-view nature of the gradients, ensuring that splitting and cloning operations are performed consistently across the 3D space rather than relying on single-plane gradient magnitudes [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_multi_view_3dgs(Gaussians, Images, Cameras, num_iterations):
    """
    Efficient Multi-View Training for 3D Gaussian Splatting.
    """
    for t in range(num_iterations):
        # 1. Select multiple images for the mini-batch
        # In standard 3DGS, this would be a single image.
        # Here, we select a subset or all images.
        batch_images = select_multi_view_batch(Images)
        batch_cameras = [Cameras[i] for i in batch_images]
        
        # 2. Render all images in the batch using thread-efficient partial rendering
        # This avoids the idle thread overhead of naive multi-view rendering.
        rendered_colors = []
        for img_idx, image, camera in zip(batch_images, batch_images, batch_cameras):
            # Tile-based rasterization with optimized thread scheduling
            # to minimize idle threads across the multi-view batch
            color = thread_efficient_rasterize(Gaussians, image, camera)
            rendered_colors.append(color)
            
        # 3. Compute loss and gradients
        # Assuming a loss function L between rendered and ground truth
        total_loss = 0
        gradients = []
        
        for color, gt_image in zip(rendered_colors, batch_images):
            loss = compute_loss(color, gt_image)
            total_loss += loss
            # Compute gradients with respect to Gaussian parameters
            grad = compute_gradients(loss, Gaussians)
            gradients.append(grad)
            
        # Average gradients over the multi-view batch
        # This reduces variance compared to single-view
        avg_gradient = sum(gradients) / len(gradients)
        
        # 4. Update Gaussian parameters
        # w_{t+1} = w_t - eta * grad_bar_k
        Gaussians.update(avg_gradient, learning_rate=get_lr(t))
        
        # 5. Adaptive Density Control (ADC)
        # Standard ADC is incompatible with multi-view gradients.
        # Use a modified ADC that handles gradients from multiple planes.
        Gaussians.adaptive_density_control_multi_view(gradients)
        
    return Gaussians

def thread_efficient_rasterize(Gaussians, image, camera):
    """
    Renders the image using a tile-based rasterizer optimized for
    multi-view batches to minimize idle threads.
    """
    # Split image into tiles
    tiles = split_image_into_tiles(image)
    
    # Allocate thread blocks to tiles
    # In naive multi-view, each image gets its own blocks, leading to
    # synchronization stalls. Here, we manage thread lifecycle efficiently.
    results = []
    
    for tile in tiles:
        # Preprocess Gaussians for this tile (culling, projection)
        visible_gaussians = preprocess(Gaussians, tile, camera)
        
        # Render pixels in the tile
        # Threads in the block cooperate; freed only when all peers finish
        tile_color = render_tile(visible_gaussians, tile, camera)
        results.append(tile_color)
        
    # Assemble image from tiles
    return assemble_image(results)

def adaptive_density_control_multi_view(gradients):
    """
    Modified ADC for multi-view training.
    Standard ADC assumes gradients are in a single plane.
    This function aggregates gradients from multiple views
    to make densification decisions in 3D space.
    """
    # Aggregate positional gradients from all views
    # Ensure arithmetic is performed in a consistent 3D space
    # rather than comparing 2D gradients from different image planes.
    aggregated_grads = aggregate_gradients_3d(gradients)
    
    # Split and clone Gaussians based on aggregated 3D gradients
    for gaussian in Gaussians:
        if gaussian.gradient_magnitude > threshold:
            split_or_clone(gaussian)
```

## Operating characteristics
*   **Time Complexity:** Theoretically higher per-iteration cost than single-view 3DGS due to rendering multiple images, but mitigated by thread-efficient partial rendering which reduces the overhead of idle threads [S1, unofficial].
*   **Memory Complexity:** Requires memory to store gradients and intermediate states for multiple views simultaneously [S1, unofficial].
*   **Performance:** Multi-view training results in smaller variance of mini-batch gradients, leading to more stable optimization compared to single-view training [S1, unofficial].
*   **Strengths:** More stable convergence; better utilization of training data per iteration.
*   **Limitations:** Requires careful handling of ADC to avoid suboptimal densification; naive implementation suffers from significant overhead [S1, unofficial].

## Background
3D Gaussian Splatting (3DGS) typically uses single-view mini-batch training because its rasterization pipeline makes gathering individual pixels from multiple images difficult, requiring rendering every pixel of all images in a single iteration which incurs considerable overhead [S1, unofficial]. NeRF, in contrast, uses multi-view mini-batches. The method was introduced to bridge this gap, showing that multi-view training leads to smaller gradient variance and more stable optimization [S1, unofficial].

## Sources
[S1] Efficient multi-view training for 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2506.12727, fetched 2026-10-26 (unofficial)
