# Batch Ray Sampling

Also called: Batch Ray Generation, Random ray sampling, Ray batching, Ray-based training, batch ray selection

## Overview
Batch Ray Sampling is a data loading strategy for training radiance field models (such as NeRF) that generates batches of random rays from training images per optimization iteration, rather than rendering complete images [S4, unofficial]. It solves the memory bottleneck of full-image rendering by processing only $N_{rand}$ rays at a time, enabling efficient gradient updates and better GPU memory utilization [S4, unofficial]. It differs from sibling methods like [Pixel Batching](3134-pixel-batching.md) by focusing on ray generation and selection logic, whereas Pixel Batching encapsulates the full forward/backward rendering pipeline for a specific batch of pixels [S2, unofficial].

## How it works
The method generates ray origins, directions, and target colors for a batch of pixels. It supports two primary modes: precomputed batching (shuffled global buffer) and on-the-fly sampling (random image selection per step) [S1, unofficial][S5, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   Training images (RGB) and camera poses ($c2w$).
    *   Camera intrinsics (focal length, image dimensions $H, W$).
    *   Parameters: `batch_size` (or `N_rand`), `precompute_rays` (bool), `precrop_iters` (int), `precrop_frac` (float), `ndc` (bool, for forward-facing scenes) [S1, unofficial][S5, unofficial].
*   **Outputs**:
    *   `rays`: Tensor of shape $[N, 3]$ or $[N, 8+]$ containing ray origins, directions, near/far planes, and optionally view directions [S1, unofficial].
    *   `rgb_targets`: Tensor of shape $[N, 3]$ containing the ground truth RGB values for the sampled pixels [S5, unofficial].

### Data Structures
*   **Ray Tensor**: Concatenation of ray origin $(3)$, ray direction $(3)$, near distance $(1)$, and far distance $(1)$. If `use_viewdirs` is enabled, viewing direction $(3)$ is appended [S1, unofficial].
*   **Precomputed Buffer**: If `precompute_rays` is true, a buffer of shape $[N_{total}, 3, 3]$ (or similar flattened structure) stores all rays from all training images, concatenated with target colors, transposed, and shuffled [S1, unofficial].

### Steps

1.  **Determine Sampling Mode**: Check `precompute_rays` or `use_batching` flag [S1, unofficial][S5, unofficial].
2.  **Precomputed Mode (Global Shuffle)**:
    *   If not yet initialized, generate rays for all pixels in all training images.
    *   Concatenate rays with target RGB colors.
    *   Reshape and shuffle the entire dataset [S1, unofficial].
    *   Slice the next batch: `rays_batch = rays_buffer[i:i+N_rand]` [S2, unofficial].
    *   Update index `i` for next iteration [S2, unofficial].
3.  **On-the-Fly Mode (Random Image)**:
    *   Randomly select one training image index $k$ [S1, unofficial][S2, unofficial][S5, unofficial].
    *   Generate ray origins and directions for all pixels in image $k$ using camera pose $c2w_k$ and intrinsics [S1, unofficial].
    *   Apply NDC transformation if `ndc` is true [S1, unofficial].
    *   Select $N_{rand}$ random pixel coordinates $(u, v)$ [S1, unofficial][S2, unofficial].
4.  **Center Cropping (Optional)**:
    *   If current iteration $i < precrop_iters$, restrict pixel selection to a center crop of size $2 \cdot dH \times 2 \cdot dW$, where $dH = H \cdot precrop\_frac$ and $dW = W \cdot precrop\_frac$ [S1, unofficial].
5.  **Subpixel Sampling (Advanced/Differentiable)**:
    *   For differentiable rendering (e.g., Mitsuba), extend the wavefront to $batch\_size \times spp$ [S3, unofficial].
    *   Gather sensor and pixel indices using repeat indices [S3, unofficial].
    *   Sample subpixel offset: $offset = sampler.next\_2d()$ [S3, unofficial].
    *   Compute floating-point position: $pos\_f = pos + offset$ [S3, unofficial].
    *   Rescale to $[0, 1]^2$: $pos\_unit = \frac{pos\_f}{film\_size}$ [S3, unofficial].
6.  **Return**: Return the batched rays and corresponding target RGB values [S5, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_batch_rays(
    training_images, 
    training_poses, 
    intrinsics, 
    batch_size=1024, 
    precompute_rays=False,
    precrop_iters=0,
    precrop_frac=0.5,
    ndc=False,
    current_iter=0
):
    """
    Samples a batch of rays for training.
    """
    H, W = training_images[0].shape[:2]
    
    if precompute_rays:
        # Assumes global buffer is pre-shuffled
        # Fetch next batch from global index
        rays_batch, targets_batch = global_ray_buffer.get_next_batch(batch_size)
        return rays_batch, targets_batch

    # On-the-fly mode
    # 1. Pick random image
    img_idx = random.randint(0, len(training_images) - 1)
    img = training_images[img_idx]
    pose = training_poses[img_idx]
    
    # 2. Generate all rays for this image
    # rays_o: [H, W, 3], rays_d: [H, W, 3]
    rays_o, rays_d = generate_rays_from_pose(pose, intrinsics, H, W, ndc)
    
    # 3. Determine pixel coordinates
    if current_iter < precrop_iters:
        # Center crop
        dH = int(H * precrop_frac)
        dW = int(W * precrop_frac)
        y_min, y_max = (H - dH) // 2, (H + dH) // 2
        x_min, x_max = (W - dW) // 2, (W + dW) // 2
        # Sample from cropped region
        ys = random.randint(y_min, y_max - 1)
        xs = random.randint(x_min, x_max - 1)
        # Note: In practice, meshgrid is created for the crop region
        coords = generate_coords_in_crop(H, W, y_min, y_max, x_min, x_max, batch_size)
    else:
        # Full image
        coords = generate_random_coords(H, W, batch_size)
        
    # 4. Extract ray data and targets
    # Flatten coords to [batch_size, 2]
    ys, xs = coords[:, 0], coords[:, 1]
    
    rays_o_batch = rays_o[ys, xs] # [batch_size, 3]
    rays_d_batch = rays_d[ys, xs] # [batch_size, 3]
    targets = img[ys, xs]         # [batch_size, 3]
    
    # 5. Construct ray tensor (origin, direction, near, far)
    # near/far are typically constants or per-ray based on scene bounds
    near = 0.2
    far = 6.0
    rays = torch.cat([rays_o_batch, rays_d_batch, 
                      torch.full((batch_size, 1), near), 
                      torch.full((batch_size, 1), far)], dim=-1)
                      
    return rays, targets

def generate_random_coords(H, W, N):
    ys = torch.randint(0, H, (N,))
    xs = torch.randint(0, W, (N,))
    return torch.stack([ys, xs], dim=-1)
```

### Parameters
*   `N_rand` / `batch_size`: Number of rays per batch. Typical values: 1024–2048 for small/low-res scenes, 4096–8192 for medium scenes (default), 16384+ for large/high-res scenes [S1, unofficial][S4, unofficial].
*   `precrop_iters`: Number of iterations to use center cropping. Default: 0 [S1, unofficial].
*   `precrop_frac`: Fraction of image width/height for the center crop. Default: 0.5 [S1, unofficial].
*   `precompute_rays`: Boolean flag to use precomputed ray buffers. Default: False [S5, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: When using NDC coordinates for forward-facing scenes, ensure ray origins and directions are correctly transformed to avoid artifacts [S1, unofficial].
*   **Boundaries**: Ensure pixel coordinate sampling stays within $[0, H-1] \times [0, W-1]$ [S1, unofficial].
*   **Memory**: Precomputing all rays for large datasets (e.g., 100+ images at 1024x1024) can exceed CPU/GPU memory limits; on-the-fly sampling is preferred in such cases [S5, unofficial].

## Subtopics

### [Pixel Batching](3134-pixel-batching.md)
Pixel Batching is a ray-centric batched rendering operation that computes gradients of rendered images with respect to scene parameters using automatic differentiation (AD) in backward mode, supporting multiple sensors and pixels in a single batch. It serves as a Python-side wrapper that samples `batch_size` random (sensor, pixel) pairs across all sensors, renders those rays to a 1xN box-filter film, and supports a decorrelated PRB (Path Reuse/Reparameterization) backward pass without requiring kernel changes. Unlike the sensor-centric `mi.render` or sibling methods like [Batch Ray Sampling](2767-batch-ray-sampling.md) which prepare rays, this method encapsulates the full forward and backward rendering pipeline for a specific batch of pixels, enabling efficient training loops for inverse volume rendering.

### [Training Ray Shuffling](5446-training-ray-shuffling.md)
Training Ray Shuffling initializes the ray data for a training epoch by constructing rays from training cameras and ground truth, shuffling their order, and transferring the shuffled indices to the GPU. It serves as the setup phase for the training loop, ensuring that the batch of rays processed by the renderer is randomized to prevent training bias from spatial coherence in the dataset. It builds on [Batch Ray Sampling](2767-batch-ray-sampling.md) by handling the specific permutation and GPU transfer logic required for efficient training.

### Semantic Ray Sampling
Semantic Ray Sampling samples rays based on semantic segmentation masks, allowing for extra sampling of specific classes and masking away instances [S6, unofficial]. If `mask_away_instance` is true, it creates a kept mask where instance sum is 0: $kept\_mask = (instance\_seg.to(torch.float32).sum(axis=3) == 0).view(*prefix, H \times W)$ [S6, unofficial]. If `extra_sampling` is true, it iterates over classes in `self.class_ratio`, calculating extra rays for each class: $ray\_num\_extra = int(self.class\_ratio[class\_] \times N_{rays})$ [S6, unofficial]. It extracts coordinates for the class mask and records ray index ranges, subtracting extra rays from the remaining count [S6, unofficial]. A pitfall is that if the sum of sample ratios exceeds 1, `remain_rays` becomes negative, triggering an error [S6, unofficial].

### Batched Pixel Sampling
Batched Pixel Sampling samples a batch of pixels and sensors for rendering, generating three separate samplers for pixel selection, primal subpixel sampling, and adjoint subpixel sampling [S3, unofficial]. It determines the number of sensors $n_{sensors}$, creates samplers with sizes `batch_size`, `batch_size * spp`, and `batch_size * spp_grad`, and seeds them using `sub_seed = mi.sample_tea_32(seed, 17 * i + 5)[0]` [S3, unofficial]. It selects sensors via `sensor_idx = mi.UInt32(n_sensors * batch_samplers[0].next_1d())` and samples pixels via `pixels = mi.Point2u(mi.Point2f(film_size) * batch_samplers[0].next_2d())` [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N_{rand})$ per iteration for ray generation and sampling [S4, unofficial]. Precomputed mode has $O(1)$ sampling cost but $O(N_{total})$ initialization cost [S1, unofficial].
*   **Memory Complexity**: $O(N_{rand})$ for on-the-fly mode; $O(N_{total})$ for precomputed mode [S5, unofficial].
*   **Performance**: Significantly reduces GPU memory usage compared to full-image rendering, allowing for larger batch sizes or higher resolutions [S4, unofficial]. Convergence speed is generally faster due to more frequent gradient updates per epoch [S4, unofficial].
*   **Strengths**: Memory efficient, standard in modern NeRF methods, flexible (supports center cropping, NDC) [S4, unofficial].
*   **Limitations**: On-the-fly mode may have higher CPU overhead for ray generation; precomputed mode requires significant RAM for large datasets [S5, unofficial].
*   **When to use**: Standard for training NeRF and radiance field models [S4, unofficial]. Use precomputed mode for small datasets or when CPU ray generation is a bottleneck; use on-the-fly for large datasets [S5, unofficial].

## Background
Batch Ray Sampling originated with the NeRF (Neural Radiance Fields) paper by Mildenhall et al. (2020) [S1, unofficial]. It was introduced as a memory-efficient alternative to full-image rendering, enabling the training of neural networks on consumer GPUs. The method has since become the standard approach in differentiable volumetric rendering frameworks and radiance field reconstruction [S4, unofficial]. Variants like Semantic Ray Sampling have been developed to incorporate additional supervision signals from segmentation masks [S6, unofficial].

## Sources
[S1] yenchenlin/nerf-pytorch, https://github.com/yenchenlin/nerf-pytorch, fetched 2026-10-26 (unofficial)
[S2] yashbhalgat/HashNeRF-pytorch, https://github.com/yashbhalgat/HashNeRF-pytorch, fetched 2026-10-26 (unofficial)
[S3] https://github.com/rgl-epfl/unbiased-inverse-volume-rendering, https://github.com/rgl-epfl/unbiased-inverse-volume-rendering, fetched 2026-10-26 (unofficial)
[S4] JackZhouSz/tlir, https://github.com/JackZhouSz/tlir, fetched 2026-10-27 (unofficial)
[S5] InanXR/FLASH, https://github.com/InanXR/FLASH, fetched 2026-10-27 (unofficial)
[S6] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
