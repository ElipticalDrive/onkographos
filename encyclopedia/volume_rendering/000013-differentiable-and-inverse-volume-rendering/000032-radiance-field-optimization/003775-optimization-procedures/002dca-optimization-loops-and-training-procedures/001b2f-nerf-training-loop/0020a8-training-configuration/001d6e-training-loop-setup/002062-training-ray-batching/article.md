# Training Ray Batching

Also called: Random Ray Batching, Random Ray Batching for NeRF Training, Random Ray Sampling, batch ray selection, partial ray sampling

## Overview
Training Ray Batching is a data loading strategy for Neural Radiance Fields (NeRF) that selects a random subset of pixel coordinates (and their corresponding rays) from the full training image grid per optimization iteration, rather than rendering complete images [S1, S2, unofficial]. It solves the memory bottleneck of full-image rendering by processing only $N_{rand}$ rays at a time, enabling efficient gradient updates and better GPU memory utilization [S4, unofficial]. It differs from [Training Loop for Refractive Index Field](4178-training-loop-for-refractive-index-field.md) and [Training Loop](4290-training-loop.md) by explicitly managing the pre-computation, shuffling, and batch-slicing of ray tensors across multiple training images, whereas those siblings focus on the network forward/backward passes or specific physical models [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training images tensor `images` of shape $(N_{img}, H, W, 3)$; camera poses tensor `poses` of shape $(N_{img}, 3, 4)$; camera intrinsic matrix $K$ of shape $(3, 3)$; image dimensions $H, W$; batch size $N_{rand}$; boolean flag `use_batching` (default `True`).
*   **Outputs:** A batch tensor `batch` of shape $(3, N_{rand})$ containing ray origins, ray directions, and target RGB values, ready for the NeRF forward pass.

**Data Structures:**
The core data structure is a flat tensor `rays_rgb` of shape $(N_{total\_rays}, 3, 3)$, where $N_{total\_rays} = N_{train} \times H \times W$.
*   The first dimension indexes the specific ray (image index, row, column).
*   The second dimension indexes the component: index 0 is ray origin $(x, y, z)$, index 1 is ray direction $(d_x, d_y, d_z)$, and index 2 is target RGB color $(r, g, b)$.
*   The third dimension is the coordinate value (3 floats).
*   Data type is `float32`.

**Steps:**

1.  **Generate Rays:** For each training image $i \in [0, N_{img})$, compute the ray origin $o_i$ and direction $d_i$ for every pixel $(u, v)$ using the camera pose $P_i$ and intrinsics $K$. Stack these into a tensor `rays` of shape $(N_{img}, 2, H, W, 3)$, where the second dimension holds origins and directions [S2, unofficial].
2.  **Concatenate Targets:** Concatenate `rays` with the corresponding training image `images` along the component axis. If `rays` has shape $(N_{img}, 2, H, W, 3)$ and `images` is $(N_{img}, H, W, 3)$, the result `rays_rgb` has shape $(N_{img}, 3, H, W, 3)$ [S2, unofficial].
3.  **Transpose:** Transpose `rays_rgb` to move the spatial dimensions before the component dimensions. New shape: $(N_{img}, H, W, 3, 3)$ [S2, unofficial].
4.  **Filter Training Set:** Select only the images in the training set index list $i_{train}$. Stack them to form `rays_rgb` of shape $(N_{train}, H, W, 3, 3)$ [S2, unofficial].
5.  **Flatten:** Reshape `rays_rgb` to $(-1, 3, 3)$, resulting in shape $(N_{train} \times H \times W, 3, 3)$. This creates a flat list of all possible rays from all training images [S2, unofficial].
6.  **Cast and Shuffle:** Cast the tensor to `float32`. Randomly shuffle the order of the rays using a permutation algorithm (e.g., Fisher-Yates or library equivalent) to ensure uniform random sampling during training [S2, unofficial].
7.  **Batch Sampling (Per Iteration):**
    *   Maintain a global index `i_batch` initialized to 0.
    *   Slice the shuffled tensor: `batch_flat = rays_rgb[i_batch : i_batch + N_rand]`. Shape: $(N_{rand}, 3, 3)$.
    *   Transpose `batch_flat` to $(3, N_{rand}, 3)$ to separate origins, directions, and targets.
    *   Extract `batch_rays` (origins and directions) and `target_s` (RGB targets) [S2, unofficial].
    *   Increment `i_batch` by $N_{rand}$.
    *   If `i_batch >= rays_rgb.shape[0]`, shuffle `rays_rgb` again and reset `i_batch` to 0 [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def prepare_ray_batch(images, poses, K, i_train, H, W, N_rand):
    """
    Pre-computes and shuffles all rays for training.
    
    Args:
        images: np.ndarray of shape (N_img, H, W, 3)
        poses: np.ndarray of shape (N_img, 3, 4)
        K: np.ndarray of shape (3, 3)
        i_train: list of indices for training images
        H, W: int, image height and width
        N_rand: int, batch size (number of rays)
        
    Returns:
        rays_rgb: torch.Tensor of shape (N_train * H * W, 3, 3)
    """
    N_img = len(images)
    
    # Step 1: Generate rays for all images
    # get_rays_np(H, W, K, p) returns a tensor of shape (2, H, W, 3)
    # where dim 0 is [origins, directions]
    rays_list = []
    for p in poses[:N_img]:
        rays_list.append(get_rays_np(H, W, K, p))
    rays = np.stack(rays_list, axis=0)  # Shape: (N_img, 2, H, W, 3)
    
    # Step 2: Concatenate with target RGB
    # images[:, None] adds a dimension to make it (N_img, 1, H, W, 3)
    # Concatenate along axis 1 (component axis)
    rays_rgb = np.concatenate([rays, images[:, None]], axis=1)  # Shape: (N_img, 3, H, W, 3)
    
    # Step 3: Transpose to (N_img, H, W, 3, 3)
    rays_rgb = np.transpose(rays_rgb, [0, 2, 3, 1, 4])
    
    # Step 4: Filter for training set
    rays_rgb_train = np.stack([rays_rgb[i] for i in i_train], axis=0)  # Shape: (N_train, H, W, 3, 3)
    
    # Step 5: Reshape to flat list (-1, 3, 3)
    rays_rgb_flat = rays_rgb_train.reshape(-1, 3, 3)  # Shape: (N_train * H * W, 3, 3)
    
    # Step 6: Cast to float32 and Shuffle
    rays_rgb_flat = rays_rgb_flat.astype(np.float32)
    np.random.shuffle(rays_rgb_flat)
    
    # Convert to tensor for GPU
    rays_rgb = torch.Tensor(rays_rgb_flat)
    
    return rays_rgb

def sample_batch(rays_rgb, i_batch, N_rand):
    """
    Samples a batch of rays from the pre-shuffled tensor.
    
    Args:
        rays_rgb: torch.Tensor of shape (N_total, 3, 3)
        i_batch: int, current batch index
        N_rand: int, batch size
        
    Returns:
        batch_rays: torch.Tensor of shape (2, N_rand, 3)
        target_s: torch.Tensor of shape (N_rand, 3)
        new_i_batch: int, updated batch index
        needs_shuffle: bool, whether the tensor was reshuffled
    """
    N_total = rays_rgb.shape[0]
    
    # Slice batch
    batch_flat = rays_rgb[i_batch : i_batch + N_rand]  # Shape: (N_rand, 3, 3)
    
    # Transpose to (3, N_rand, 3)
    batch_transposed = torch.transpose(batch_flat, 0, 1)
    
    # Extract origins/directions (first 2) and targets (last 1)
    batch_rays = batch_transposed[:2]  # Shape: (2, N_rand, 3)
    target_s = batch_transposed[2]     # Shape: (N_rand, 3)
    
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

def get_rays_np(H, W, K, pose):
    """
    Helper function to generate rays for a single image.
    Returns shape (2, H, W, 3): [origins, directions]
    """
    # Standard NeRF ray generation logic
    # i, j are pixel coordinates
    i, j = np.meshgrid(np.arange(W, dtype=np.float32), np.arange(H, dtype=np.float32), indexing='xy')
    
    # Direction vectors in camera space
    dirs = np.stack([(i - K[0, 2]) / K[0, 0], 
                     (j - K[1, 2]) / K[1, 1], 
                     np.ones_like(i)], axis=-1)
    
    # Normalize directions
    dirs = dirs / np.linalg.norm(dirs, axis=-1, keepdims=True)
    
    # Transform to world space
    # pose is (3, 4) [R|t]
    R = pose[:, :3]
    t = pose[:, 3]
    
    # Rotate directions
    dirs_world = np.sum(np.expand_dims(dirs, -2) * R, axis=-1)
    
    # Origins are the camera center
    origins = np.dot(-R.T, t)
    origins_world = np.broadcast_to(origins, (H, W, 3))
    
    # Stack origins and directions
    rays = np.stack([origins_world, dirs_world], axis=0)  # (2, H, W, 3)
    return rays
```

**Parameters:**
*   `N_rand`: Number of rays per batch. Default value is $32 \times 32 \times 4 = 4096$ [S2, unofficial]. Controls memory footprint and gradient noise [S1, unofficial].
*   `use_batching`: Boolean flag. If `True` (default), rays are sampled from all training images globally. If `False`, rays are sampled from a single random image per iteration [S2, unofficial].
*   `K`: Camera intrinsic matrix, shape $(3, 3)$. Simplified form: $\begin{bmatrix} f_{camera} & 0 & W/2 \\ 0 & f_{camera} & H/2 \\ 0 & 0 & 1 \end{bmatrix}$ [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Ensure `float32` is used for ray coordinates to match the network's input precision and avoid overflow/underflow in accumulation [S2, unofficial].
*   **Boundary Conditions:** When `i_batch` reaches the end of the tensor, a full shuffle is required to maintain randomness for the next epoch. Failure to shuffle leads to biased sampling of the last portion of the dataset in subsequent epochs [S2, unofficial].
*   **Memory:** Pre-computing all rays requires memory proportional to $N_{train} \times H \times W \times 3 \times 4$ bytes. For high-resolution images, this can be significant. If memory is constrained, reduce `N_rand` or use lower resolution [S1, unofficial].

## Subtopics

### [NeRF System Data Preparation](2664-nerf-system-data-preparation.md)
This method prepares training and validation data for Neural Radiance Fields (NeRF) by loading synthetic Blender dataset scenes, parsing JSON metadata for camera poses and images, and precomputing rays and RGB values. It differs from [LLFF Dataset Loading](2818-llff-dataset-loading.md) by assuming known, precise camera poses provided in JSON rather than estimating them, and by using a fixed scene bounding box. It differs from [NeRF Dataset Loading](3084-nerf-dataset-loading.md) by explicitly handling the specific preprocessing steps for the Blender synthetic dataset, including focal length computation from `camera_angle_x` and RGBA compositing.

### [Batch Ray Sampling](2767-batch-ray-sampling.md)
Batch Ray Sampling is a data loading strategy for training radiance field models (such as NeRF) that generates batches of random rays from training images per optimization iteration, rather than rendering complete images. It solves the memory bottleneck of full-image rendering by processing only $N_{rand}$ rays at a time, enabling efficient gradient updates and better GPU memory utilization. It differs from sibling methods like [Pixel Batching](3134-pixel-batching.md) by focusing on ray generation and selection logic, whereas Pixel Batching encapsulates the full forward/backward rendering pipeline for a specific batch of pixels.

## Operating characteristics
*   **Time Complexity:** Pre-computation is $O(N_{img} \cdot H \cdot W)$. Per-iteration sampling is $O(N_{rand})$ for slicing and transposing, plus $O(N_{total\_rays})$ for shuffling at epoch boundaries.
*   **Memory Complexity:** $O(N_{train} \cdot H \cdot W)$ to store the pre-computed ray tensor. This is significantly less than storing full rendered images for all training steps, but still requires loading all rays into memory if `use_batching=True` [S2, unofficial].
*   **Measured Performance:** With $N_{rand}=4096$ and typical NeRF resolutions (e.g., 512x512), this method allows training on GPUs with 8-16GB VRAM, which would be insufficient for full-image rendering [S2, unofficial].
*   **Strengths:** Efficient GPU memory usage; enables training on large datasets; simple implementation.
*   **Limitations:** Pre-computation memory overhead; requires shuffling at epoch boundaries which can be costly for very large datasets; does not adaptively select "hard" rays (unlike importance sampling variants).
*   **When to use:** Standard NeRF training with limited GPU memory.
*   **When not to use:** When adaptive ray sampling is required for faster convergence, or when memory is extremely constrained (consider on-the-fly ray generation instead of pre-computation).

## Background
Training Ray Batching was introduced in the original NeRF paper (Mildenhall et al., 2020) as a fundamental technique to make volume rendering tractable on consumer hardware [S2, unofficial]. It addresses the high memory cost of rendering full images by processing rays in small batches. The method has become a standard component in NeRF-based training pipelines [S1, unofficial].

## Sources
[S1] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S2] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
