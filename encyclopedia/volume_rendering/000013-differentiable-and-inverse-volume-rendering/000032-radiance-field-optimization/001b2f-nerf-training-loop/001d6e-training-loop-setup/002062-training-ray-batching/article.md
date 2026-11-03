# Training Ray Batching

Also called: Random Ray Batching, Random Ray Batching for NeRF Training

## Overview
Training Ray Batching is a data loading strategy for NeRF optimization that pre-computes rays for all training images, concatenates them with target RGB colors, and samples random subsets per iteration [S1, unofficial]. It differs from Single Image Sampling by drawing rays globally across the entire training dataset rather than restricting them to one image per step, ensuring uniform exposure to all views [S1, unofficial].

## How it works
**Inputs:**
- `poses`: Array of camera poses, shape `(num_img, 3, 4)` [S1, unofficial].
- `images`: Array of training images, shape `(num_img, H, W, 3)` [S1, unofficial].
- `K`: Camera intrinsic matrix, shape `(3, 3)` [S1, unofficial].
- `i_train`: List of indices for training images [S1, unofficial].
- `N_rand`: Number of rays per batch [S1, unofficial].

**Data Structures:**
- `rays_rgb`: A flat tensor of shape `(N_total, 3, 3)` where `N_total = len(i_train) * H * W` [S1, unofficial]. The first dimension indexes the ray, the second dimension has size 3 representing `[ray_origin, ray_direction, target_rgb]`, and the third dimension is the vector size 3 [S1, unofficial].

**Steps:**
1. **Generate Rays:** For each pose `p` in `poses[:,:3,:4]`, call `get_rays_np(H, W, K, p)` to generate ray origins and directions [S1, unofficial]. Stack these into a tensor `rays` of shape `(num_img, ro+rd, H, W, 3)` where `ro+rd = 2` [S1, unofficial].
2. **Concatenate Targets:** Concatenate `rays` with `images[:,None]` along axis 1 [S1, unofficial]. `images` is expanded to `(num_img, 1, H, W, 3)` [S1, unofficial]. Result `rays_rgb` has shape `(num_img, ro+rd+rgb, H, W, 3)` which equals `(num_img, 3, H, W, 3)` [S1, unofficial].
3. **Transpose:** Transpose `rays_rgb` from `(num_img, 3, H, W, 3)` to `(num_img, H, W, 3, 3)` using `np.transpose(rays_rgb, [0, 2, 3, 1, 4])` [S1, unofficial].
4. **Filter Training Set:** Stack only the images in `i_train` into `rays_rgb` [S1, unofficial]. Result shape is `(len(i_train), H, W, 3, 3)` [S1, unofficial].
5. **Reshape:** Flatten the spatial and image dimensions by reshaping to `(-1, 3, 3)` [S1, unofficial]. The first dimension becomes `len(i_train) * H * W` [S1, unofficial].
6. **Cast:** Convert to `np.float32` [S1, unofficial].
7. **Shuffle:** Randomly shuffle the entire array using `np.random.shuffle(rays_rgb)` [S1, unofficial].
8. **Sample Batch:** In each iteration, slice `rays_rgb[i_batch : i_batch + N_rand]` [S1, unofficial]. Transpose the batch to shape `(3, N_rand, 3)` (equivalent to `2+1, N_rand, 3`) to separate origins, directions, and targets [S1, unofficial].
9. **Update Index:** Increment `i_batch` by `N_rand` [S1, unofficial]. If `i_batch >= rays_rgb.shape[0]`, shuffle `rays_rgb` again using `torch.randperm` and reset `i_batch` to 0 [S1, unofficial].

**Parameters:**
- `N_rand`: Default `32 * 32 * 4 = 4096` [S1, unofficial].
- `no_batching`: Boolean [S1, unofficial]. If `True`, disables this method in favor of single-image sampling [S1, unofficial].
- `K`: Simplified intrinsic `[[f, 0, W/2], [0, f, H/2], [0, 0, 1]]` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def generate_rays(H, W, K, pose):
    """Standard ray generation routine [G]. Returns (2, H, W, 3) for origins and directions."""
    # Standard implementation computes camera centers and per-pixel ray directions
    # from the pose matrix and intrinsic calibration K.
    origins = np.zeros((H, W, 3), dtype=np.float32)
    directions = np.zeros((H, W, 3), dtype=np.float32)
    # ... standard ray casting logic ...
    return np.stack([origins, directions], axis=0)

def prepare_ray_batch(poses, images, K, i_train, N_rand=4096):
    num_img = poses.shape[0]
    H, W = images.shape[1], images.shape[2]
    
    # Step 1: Generate rays for all images
    rays_list = [generate_rays(H, W, K, p) for p in poses[:, :3, :4]]
    rays = np.stack(rays_list, axis=0)  # (num_img, 2, H, W, 3)
    
    # Step 2: Concatenate with target RGB
    rays_rgb = np.concatenate([rays, images[:, None]], axis=1)  # (num_img, 3, H, W, 3)
    
    # Step 3: Transpose to (num_img, H, W, 3, 3)
    rays_rgb = np.transpose(rays_rgb, [0, 2, 3, 1, 4])
    
    # Step 4: Filter for training set
    rays_rgb = np.stack([rays_rgb[i] for i in i_train], axis=0)  # (len(i_train), H, W, 3, 3)
    
    # Step 5: Reshape to flat list (-1, 3, 3)
    rays_rgb = np.reshape(rays_rgb, [-1, 3, 3])
    
    # Step 6: Cast to float32
    rays_rgb = rays_rgb.astype(np.float32)
    
    # Step 7: Shuffle
    np.random.shuffle(rays_rgb)
    
    # Move to GPU
    device = torch.device('cuda')  # Standard device selection [G]
    rays_rgb = torch.Tensor(rays_rgb).to(device)
    
    i_batch = 0
    return rays_rgb, i_batch

def sample_batch(rays_rgb, i_batch, N_rand):
    # Step 8: Slice batch
    batch = rays_rgb[i_batch : i_batch + N_rand]  # (N_rand, 3, 3)
    
    # Transpose to (3, N_rand, 3)
    batch = torch.transpose(batch, 0, 1)
    
    # Separate rays and targets
    batch_rays = batch[:2]  # (2, N_rand, 3)
    target_s = batch[2]     # (N_rand, 3)
    
    # Step 9: Update index
    i_batch += N_rand
    if i_batch >= rays_rgb.shape[0]:
        # Shuffle again if end of dataset reached
        rand_idx = torch.randperm(rays_rgb.shape[0])
        rays_rgb = rays_rgb[rand_idx]
        i_batch = 0
        
    return batch_rays, target_s, i_batch
```

**Edge Cases:**
- If `N_rand` does not divide the total number of rays evenly, the last batch in an epoch will be smaller unless padding is implemented [S1, unofficial]. The source code simply slices until the end, then shuffles and resets [S1, unofficial].
- Precision issues may arise if `H` and `W` are large, leading to very large `rays_rgb` tensors [S1, unofficial]. Memory usage is `O(num_img * H * W)` [S1, unofficial].

## Subtopics

### Single Image Sampling
Samples `N_rand` rays from a single randomly chosen training image per iteration [S1, unofficial]. Chosen when memory is constrained or when specific view-dependent effects are studied in isolation [S1, unofficial].

### Chunked Rendering
Renders rays in chunks to manage GPU memory during the forward pass [S1, unofficial]. This is a rendering optimization, not a data loading strategy, but often used in conjunction with batching [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Pre-computation is `O(num_img * H * W)` [S1, unofficial]. Sampling is `O(N_rand)` per iteration [S1, unofficial]. Shuffling is `O(N_total)` per epoch [S1, unofficial].
- **Memory Complexity**: `O(num_img * H * W)` to store the pre-computed `rays_rgb` tensor [S1, unofficial].
- **Performance**: Default `N_rand=4096` provides a balance between gradient noise and computational efficiency [S1, unofficial].
- **Strengths**: Ensures all training views are seen uniformly per epoch [S1, unofficial]. Reduces overhead of ray generation during training [S1, unofficial].
- **Limitations**: High memory usage for high-resolution images [S1, unofficial]. Shuffling large tensors can be slow [S1, unofficial].

## Background
Originated in the original NeRF implementation by Mildenhall et al. [S1, unofficial]. The method was designed to efficiently train neural radiance fields by leveraging pre-computed ray data to accelerate the training loop [S1, unofficial].

## Sources
[S1] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
