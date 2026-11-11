# Training Ray Batching

Also called: Random Ray Batching, Random Ray Batching for NeRF Training, Random Ray Sampling, batch ray selection, partial ray sampling

## Overview
Training Ray Batching is a data loading strategy for Neural Radiance Fields (NeRF) that selects a random subset of pixel coordinates (and their corresponding rays) from the full training image grid per optimization iteration, rather than rendering complete images [S1, S2, unofficial]. It reduces the GPU memory footprint during the backward pass of gradient descent by processing only $N_{rand}$ rays per step, which controls gradient noise and enables efficient training [S1, unofficial]. It differs from [Training Loop for Refractive Index Field](4178-training-loop-for-refractive-index-field.md) and [Training Loop](4290-training-loop.md) by explicitly managing the pre-computation, shuffling, and batch-slicing of ray tensors across multiple training images, whereas those siblings focus on the network forward/backward passes or specific physical models [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training images tensor `images` of shape $(N_{img}, H, W, 3)$; camera poses tensor `poses` of shape $(N_{img}, 3, 4)$; camera intrinsic matrix $K$ of shape $(3, 3)$; image dimensions $H, W$; batch size $N_{rand}$; boolean flag `use_batching` (default `True`) [S2, unofficial].
*   **Outputs:** A batch tensor `batch` of shape $(3, N_{rand}, 3)$ containing ray origins, ray directions, and target RGB values, ready for the NeRF forward pass [S2, unofficial].

**Data Structures:**
The core data structure is a flat tensor `rays_rgb` of shape $(N_{total\_rays}, 3, 3)$, where $N_{total\_rays} = N_{train} \times H \times W$ [S2, unofficial].
*   The first dimension indexes the specific ray (image index, row, column) [S2, unofficial].
*   The second dimension indexes the component: index 0 is ray origin $(x, y, z)$, index 1 is ray direction $(d_x, d_y, d_z)$, and index 2 is target RGB color $(r, g, b)$ [S2, unofficial].
*   The third dimension is the coordinate value (3 floats) [S2, unofficial].
*   Data type is `float32` [S2, unofficial].

**Steps:**
1.  **Generate Rays:** For each training image, call `get_rays_np(H, W, K, p)` for each pose $p$ in `poses[:,:3,:4]` and stack them into a tensor `rays` of shape $(N_{img}, ro+rd, H, W, 3)$, where $ro+rd=2$ represents ray origins and directions [S2, unofficial].
2.  **Concatenate Targets:** Concatenate `rays` with `images[:,None]` along axis 1 to create `rays_rgb` of shape $(N_{img}, ro+rd+rgb, H, W, 3)$ [S2, unofficial].
3.  **Transpose:** Transpose `rays_rgb` using `[0,2,3,1,4]` to shape $(N_{img}, H, W, ro+rd+rgb, 3)$ [S2, unofficial].
4.  **Filter Training Set:** Stack only the images in the training index list $i_{train}$ into `rays_rgb`, resulting in shape $(len(i_{train}), H, W, ro+rd+rgb, 3)$ [S2, unofficial].
5.  **Flatten:** Reshape `rays_rgb` to $(-1, 3, 3)$, resulting in shape $(N_{train} \times H \times W, 3, 3)$ [S2, unofficial].
6.  **Cast and Shuffle:** Cast `rays_rgb` to `np.float32`. Randomly shuffle the order of rays in `rays_rgb` using `np.random.shuffle(rays_rgb)` to ensure random sampling during training [S2, unofficial].
7.  **Batch Sampling (Per Iteration):** Maintain a global index `i_batch` initialized to 0. Slice `rays_rgb[i_batch:i_batch+N_rand]` to get the batch. Transpose the batch to shape $(2+1, N_{rand}, 3)$ where the first dimension represents ray origins, ray directions, and target values. Increment `i_batch` by $N_{rand}$. If `i_batch >= rays_rgb.shape[0]`, shuffle `rays_rgb` again using `torch.randperm` and reset `i_batch` to 0 [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def get_rays_np(H, W, K, pose):
    # pose is (3, 4) [R|t]
    # i, j are pixel coordinates
    i, j = np.meshgrid(np.arange(W, dtype=np.float32), np.arange(H, dtype=np.float32), indexing='xy')
    # Direction vectors in camera space
    dirs = np.stack([(i - K[0, 2]) / K[0, 0],
                     (j - K[1, 2]) / K[1, 1],
                     np.ones_like(i)], axis=-1)
    # Normalize directions
    dirs = dirs / np.linalg.norm(dirs, axis=-1, keepdims=True)
    # Transform to world space
    R = pose[:, :3]
    t = pose[:, 3]
    # Rotate directions
    dirs_world = np.sum(np.expand_dims(dirs, -2) * R, axis=-1)
    # Origins are the camera center
    origins = np.dot(-R.T, t)
    origins_world = np.broadcast_to(origins, (H, W, 3))
    # Stack origins and directions: shape (2, H, W, 3)
    rays = np.stack([origins_world, dirs_world], axis=0)
    return rays

def prepare_ray_batch(images, poses, K, i_train, H, W, N_rand):
    N_img = len(images)
    # Step 1: Generate rays for all images
    rays = np.stack([get_rays_np(H, W, K, p) for p in poses[:N_img]], axis=0)  # (N_img, 2, H, W, 3)
    # Step 2: Concatenate with target RGB
    rays_rgb = np.concatenate([rays, images[:, None]], axis=1)  # (N_img, 3, H, W, 3)
    # Step 3: Transpose to (N_img, H, W, 3, 3)
    rays_rgb = np.transpose(rays_rgb, [0, 2, 3, 1, 4])
    # Step 4: Filter for training set
    rays_rgb = np.stack([rays_rgb[i] for i in i_train], axis=0)  # (N_train, H, W, 3, 3)
    # Step 5: Reshape to flat list (-1, 3, 3)
    rays_rgb = rays_rgb.reshape(-1, 3, 3)  # (N_train * H * W, 3, 3)
    # Step 6: Cast to float32 and Shuffle
    rays_rgb = rays_rgb.astype(np.float32)
    np.random.shuffle(rays_rgb)
    # Convert to tensor for GPU
    rays_rgb = torch.Tensor(rays_rgb)
    return rays_rgb

def sample_batch(rays_rgb, i_batch, N_rand):
    N_total = rays_rgb.shape[0]
    # Slice batch
    batch_flat = rays_rgb[i_batch : i_batch + N_rand]  # (N_rand, 3, 3)
    # Transpose to (3, N_rand, 3)
    batch_transposed = torch.transpose(batch_flat, 0, 1)
    # Extract origins/directions (first 2) and targets (last 1)
    batch_rays = batch_transposed[:2]  # (2, N_rand, 3)
    target_s = batch_transposed[2]     # (N_rand, 3)
    new_i_batch = i_batch + N_rand
    needs_shuffle = False
    # Check if end of dataset reached
    if new_i_batch >= N_total:
        # Shuffle again
        rand_idx = torch.randperm(N_total)
        rays_rgb = rays_rgb[rand_idx]
        new_i_batch = 0
        needs_shuffle = True
    return batch_rays, target_s, new_i_batch, needs_shuffle
```

**Parameters:**
*   `N_rand`: Number of rays per batch. Default value is $32 \times 32 \times 4 = 4096$ [S2, unofficial]. Controls memory footprint and gradient noise [S1, unofficial].
*   `use_batching`: Boolean flag. If `True`, rays are sampled from all training images globally. If `False`, rays are sampled from a single random image per iteration [S2, unofficial].
*   `K`: Camera intrinsic calibration matrix, shape $(3, 3)$. Simplified form: $\begin{bmatrix} f_{camera} & 0 & W/2 \\ 0 & f_{camera} & H/2 \\ 0 & 0 & 1 \end{bmatrix}$ [S2, unofficial].
*   `image_size`: Resolution of the target image, determines the search space for random sampling [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** When `i_batch` reaches the end of the tensor, a full shuffle is required to maintain randomness for the next epoch [S2, unofficial].
*   **Precision:** The tensor is explicitly cast to `float32` during preparation to match standard training requirements [S2, unofficial].
*   **Memory:** Pre-computing all rays consumes memory proportional to the number of training pixels, which can be significant for high-resolution datasets [G].

## Subtopics

### [NeRF System Data Preparation](2664-nerf-system-data-preparation.md)
This method prepares training and validation data for Neural Radiance Fields (NeRF) by loading synthetic Blender dataset scenes, parsing JSON metadata for camera poses and images, and precomputing rays and RGB values. It differs from [LLFF Dataset Loading](2818-llff-dataset-loading.md) by assuming known, precise camera poses provided in JSON rather than estimating them, and by using fixed depth bounds (`near: 2.0`, `far: 6.0`) instead of NDC normalization. It differs from [NeRF Dataset Loading](3084-nerf-dataset-loading.md) by explicitly handling the specific preprocessing steps for the Blender synthetic dataset, including focal length computation from `camera_angle_x` and RGBA compositing.

### [Batch Ray Sampling](2767-batch-ray-sampling.md)
Batch Ray Sampling is a data loading strategy for training neural radiance fields (NeRF) that generates batches of random rays from training images per optimization iteration, rather than rendering complete images. It solves the memory bottleneck of full-image rendering by processing only a subset of pixels (e.g., 1024–8192 rays) per step, enabling efficient gradient updates and better memory utilization. It differs from image-based training by sampling randomly across the entire image pool, which allows for different training dynamics compared to frameworks like I-NGP that sample every pixel of $n$ images.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Reduces GPU memory usage during the backward pass of gradient descent [S1, unofficial]. Enables training on large datasets by processing only a subset of pixels per step [S2, unofficial].
*   **Limitations:** Requires pre-computation and shuffling of all rays, which consumes memory proportional to the training set size [S2, unofficial].
*   **When to use:** Standard NeRF training with limited GPU memory [S1, unofficial].
*   **When not to use:** When adaptive ray sampling is required for faster convergence, or when memory is extremely constrained (consider on-the-fly ray generation instead of pre-computation) [G].

## Background
Not given in the sources.

## Sources
[S1] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S2] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
