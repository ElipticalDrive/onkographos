# Training Ray Batching

Also called: Random Ray Batching, Random Ray Batching for NeRF Training, Random Ray Sampling, batch ray selection, partial ray sampling

## Overview
Training Ray Batching is a data loading strategy for Neural Radiance Fields (NeRF) that selects a random subset of pixel coordinates (and their corresponding rays) from the full training image grid per iteration [S1, unofficial]. This reduces GPU memory usage during the backward pass of gradient descent by avoiding the computation of full-image gradients simultaneously [S1, unofficial]. It differs from image-based training, which renders complete images per step, by using smaller batches of random rays to improve memory utilization and convergence speed [S5, unofficial]. It builds on [Training Loop Setup](1522-training-loop-setup.md) by providing the structured ray tensors that are subsequently shuffled and sampled [S2, unofficial].

## How it works
**Inputs:**
- `images`: Array of training images with shape `(num_img, H, W, 3)`.
- `poses`: Array of camera poses with shape `(num_img, 3, 4)`.
- `K`: Camera intrinsic matrix with shape `(3, 3)`. Simplified form: `[[f, 0, W/2], [0, f, H/2], [0, 0, 1]]` where `f` is focal length [S2, unofficial].
- `i_train`: Indices of training images.
- `N_rand`: Number of random rays per batch. Default is 4096 (32x32x4) [S2, unofficial].
- `use_batching`: Boolean flag. If `True`, rays are sampled from all training images globally. If `False`, rays are sampled from a single random image per iteration [S2, unofficial].

**Outputs:**
- `batch_rays`: Tensor of shape `(2, N_rand, 3)` containing ray origins and directions.
- `target_s`: Tensor of shape `(N_rand, 3)` containing target RGB values.

**Data Structures:**
- `rays_rgb`: A flat tensor of shape `(N_total, 3, 3)` where `N_total = len(i_train) * H * W`. The first dimension indexes the ray, the second dimension indexes the component (0: origin, 1: direction, 2: target RGB), and the third dimension holds the 3D coordinate/value.

**Steps:**

1.  **Generate Rays:** For each training image `i` in `i_train`, compute ray origins and directions using `get_rays_np(H, W, K, poses[i])`. Stack these into a tensor `rays` of shape `(num_img, 2, H, W, 3)` [S2, unofficial].
2.  **Concatenate Targets:** Concatenate `rays` with the corresponding image pixels `images[i]` along the component axis to form `rays_rgb` of shape `(num_img, 3, H, W, 3)` [S2, unofficial].
3.  **Transpose:** Transpose `rays_rgb` to shape `(num_img, H, W, 3, 3)` to group spatial dimensions [S2, unofficial].
4.  **Filter Training Set:** Stack only the images in `i_train` to get shape `(len(i_train), H, W, 3, 3)` [S2, unofficial].
5.  **Flatten:** Reshape to `(-1, 3, 3)` resulting in shape `(N_total, 3, 3)` [S2, unofficial].
6.  **Cast and Shuffle:** Cast to `float32` and shuffle the entire array using `np.random.shuffle` [S2, unofficial].
7.  **Sample Batch:** In each iteration, slice `rays_rgb[i_batch : i_batch + N_rand]` [S2, unofficial].
8.  **Transpose Batch:** Transpose the batch to `(3, N_rand, 3)` and separate into `batch_rays` (first 2 components) and `target_s` (last component) [S2, unofficial].
9.  **Update Index:** Increment `i_batch` by `N_rand`. If `i_batch >= N_total`, shuffle `rays_rgb` again and reset `i_batch` to 0 [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def prepare_ray_batch(images, poses, K, i_train, N_rand=4096):
    """
    Prepare the ray batch tensor for random ray batching.
    
    Args:
        images: np.ndarray of shape (num_img, H, W, 3)
        poses: np.ndarray of shape (num_img, 3, 4)
        K: np.ndarray of shape (3, 3)
        i_train: list of indices for training images
        N_rand: int, number of rays per batch
        
    Returns:
        rays_rgb: torch.Tensor of shape (N_total, 3, 3)
    """
    num_img = len(poses)
    H, W = images.shape[1], images.shape[2]
    
    # Step 1: Generate rays for all training images
    # get_rays_np(H, W, K, p) returns (2, H, W, 3) -> (origins, directions)
    rays_list = []
    for p in poses[i_train]:
        rays_list.append(get_rays_np(H, W, K, p))
    rays = np.stack(rays_list, axis=0)  # (len(i_train), 2, H, W, 3)
    
    # Step 2: Concatenate with target RGB
    # images[i_train] has shape (len(i_train), H, W, 3)
    # We need to align dimensions for concatenation
    rays_rgb_list = []
    for i, idx in enumerate(i_train):
        # rays[i] is (2, H, W, 3), images[idx] is (H, W, 3)
        # Concatenate along axis 0 (component axis)
        rays_rgb_i = np.concatenate([rays[i], images[idx][None, ...]], axis=0)
        rays_rgb_list.append(rays_rgb_i)
        
    rays_rgb = np.stack(rays_rgb_list, axis=0)  # (len(i_train), 3, H, W, 3)
    
    # Step 3: Transpose to (len(i_train), H, W, 3, 3)
    rays_rgb = np.transpose(rays_rgb, [0, 2, 3, 1, 4])
    
    # Step 4: Flatten to (-1, 3, 3)
    rays_rgb = rays_rgb.reshape(-1, 3, 3)
    
    # Step 5: Cast to float32
    rays_rgb = rays_rgb.astype(np.float32)
    
    # Step 6: Shuffle
    np.random.shuffle(rays_rgb)
    
    # Convert to tensor
    rays_rgb = torch.Tensor(rays_rgb)
    
    return rays_rgb

def sample_ray_batch(rays_rgb, i_batch, N_rand):
    """
    Sample a batch of rays from the pre-shuffled rays_rgb tensor.
    
    Args:
        rays_rgb: torch.Tensor of shape (N_total, 3, 3)
        i_batch: int, current batch index
        N_rand: int, number of rays per batch
        
    Returns:
        batch_rays: torch.Tensor of shape (2, N_rand, 3)
        target_s: torch.Tensor of shape (N_rand, 3)
        new_i_batch: int, updated batch index
        needs_shuffle: bool, whether to shuffle again
    """
    N_total = rays_rgb.shape[0]
    
    # Step 7: Slice batch
    batch = rays_rgb[i_batch : i_batch + N_rand]  # (N_rand, 3, 3)
    
    # Step 8: Transpose to (3, N_rand, 3)
    batch = torch.transpose(batch, 0, 1)
    
    # Separate rays and targets
    batch_rays = batch[:2]  # (2, N_rand, 3)
    target_s = batch[2]     # (N_rand, 3)
    
    # Step 9: Update index
    new_i_batch = i_batch + N_rand
    needs_shuffle = new_i_batch >= N_total
    
    if needs_shuffle:
        # Shuffle again
        rand_idx = torch.randperm(N_total)
        rays_rgb = rays_rgb[rand_idx]
        new_i_batch = 0
        
    return batch_rays, target_s, new_i_batch, rays_rgb, needs_shuffle
```

**Parameters:**
- `N_rand`: Default 4096 [S2, unofficial].
- `image_size`: Determines the search space for random sampling [S1, unofficial].

**Edge Cases:**
- If `use_batching` is `False`, the method falls back to single-image sampling, selecting one random image and then `N_rand` pixels from it [S2, unofficial].
- Precision issues may arise if `N_rand` is too large for GPU memory, causing OOM errors [S1, unofficial].

## Subtopics

### [NeRF System Data Preparation](2664-nerf-system-data-preparation.md)
Prepares training data by loading synthetic scenes and precomputing ray origins/directions. Differs by handling coordinate conventions and background compositing. Builds on [Training Ray Batching](2252-training-ray-batching.md) by providing structured ray tensors.

### [Batch Ray Sampling](2767-batch-ray-sampling.md)
Generates batches by selecting random pixels and computing rays. Differs from image-based training by using smaller batches for better memory and convergence. Standard in modern NeRF methods.

## Operating characteristics
- **Time Complexity:** O(N) for shuffling, where N is the total number of rays. Sampling is O(1) per batch.
- **Memory Complexity:** O(N) to store the pre-computed `rays_rgb` tensor. This is significantly less than storing full image gradients.
- **Performance:** Reduces GPU memory usage, allowing for larger networks or higher resolutions [S1, unofficial]. Improves convergence speed by providing diverse gradient updates [S5, unofficial].
- **Strengths:** Memory efficient, simple to implement, works well with stochastic gradient descent.
- **Limitations:** Requires pre-computation of all rays, which may be prohibitive for very large datasets. Shuffling overhead increases with dataset size.
- **When to use:** When GPU memory is limited or when training on large datasets.
- **When not to use:** When training on very small datasets where full-image gradients are feasible and beneficial for stability.

## Background
Originated in the NeRF paper (Mildenhall et al., 2020) as a standard technique for training Neural Radiance Fields [S2, unofficial]. It addresses the memory bottleneck of rendering full images during training [S1, unofficial].

## Sources
[S1] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S2] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
