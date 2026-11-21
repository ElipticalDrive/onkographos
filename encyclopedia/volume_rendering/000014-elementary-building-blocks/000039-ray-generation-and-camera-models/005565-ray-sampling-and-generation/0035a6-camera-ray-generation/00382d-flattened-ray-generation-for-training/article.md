# Flattened Ray Generation for Training

Also called: Ray Generation for Training Views, Training Ray Sampling

## Overview
Flattened Ray Generation for Training is a data preparation method that converts a batch of training images into a single, flat tensor of rays and corresponding pixel colors, optimized for stochastic gradient descent in neural volume rendering. Unlike [Camera Ray Generation](7456-camera-ray-generation.md), which generates rays on-the-fly per batch or view, this method pre-aggregates rays from all training poses into a unified structure, enabling efficient random sampling of individual rays across the entire dataset during training. It differs from [Precomputed Ray Buffering](#precomputed-ray-buffering) by typically handling the flattening and indexing logic within the training loop or a dedicated sampler rather than storing the entire dataset in CPU RAM, though both aim to decouple ray generation from the forward pass.

## How it works
The method processes a set of training images and camera poses to produce flattened tensors of ray origins, directions, view directions, and target RGB values. It supports both regular grid-based scenes and irregular shapes, and can downsample rays to reduce memory usage.

### Inputs and Outputs
**Inputs:**
- `rgb_tr`: Tensor of training images, shape `[B, V, H, W, 3]` or list of images, where `B` is batch size, `V` is views per batch, `H` is height, `W` is width.
- `train_poses`: List or tensor of camera-to-world transformation matrices (`c2w`), shape `[B*V, 4, 4]`.
- `K`: Intrinsic camera matrix, shape `[3, 3]`.
- `world_size`: Size of the voxel grid, used for sampling step calculation.
- `xyz_min`, `xyz_max`: Bounding box of the scene.
- `near`, `far`: Near and far plane distances for ray clipping.
- `stepsize`: Base step size for ray sampling.
- `downrate`: Downsampling factor for rays (optional).
- `irregular_shape`: Boolean flag indicating if images have varying dimensions.
- `DEVICE`: Target device (e.g., `cuda`).

**Outputs:**
- `rays_o_tr`: Flattened ray origins, shape `[N, 3]`.
- `rays_d_tr`: Flattened ray directions, shape `[N, 3]`.
- `viewdirs_tr`: Flattened view directions, shape `[N, 3]`.
- `rgb_tr`: Flattened target RGB colors, shape `[N, 3]`.
- `imsz`: List of image sizes (number of pixels per image).

### Data Structures
- `rays_o_tr`, `rays_d_tr`, `viewdirs_tr`, `rgb_tr`: PyTorch tensors on the target device.
- `imsz`: List of integers, storing `H*W` for each image in the training set.
- `top`: Integer, current offset in the flattened tensors.

### Steps
1. **Initialize Output Tensors**:
   Calculate total number of pixels $N$ across all training images. Initialize `rays_o_tr`, `rays_d_tr`, `viewdirs_tr`, and `rgb_tr` as zero tensors of shape `[N, 3]` on the target device.

2. **Iterate Over Training Poses**:
   For each training pose index `i` (from `0` to `B*V - 1`):
   a. **Extract Pose and Image**: Get `c2w` from `train_poses[i]` and corresponding image `img` from `rgb_tr`.
   b. **Generate Rays**: Call `get_rays_of_a_view(H, W, K, c2w, ndc=False, inverse_y=False, flip_x=False, flip_y=False)` to obtain `rays_o`, `rays_d`, and `viewdirs` for this view. These are tensors of shape `[H, W, 3]`.
   c. **Flatten and Copy**: Flatten `rays_o`, `rays_d`, `viewdirs`, and `img` from `[H, W, 3]` to `[H*W, 3]`. Copy them into the global output tensors starting at index `top`:
      ```python
      n = H * W
      rgb_tr[top:top+n].copy_(img.flatten(0,1))
      rays_o_tr[top:top+n].copy_(rays_o.flatten(0,1).to(DEVICE))
      rays_d_tr[top:top+n].copy_(rays_d.flatten(0,1).to(DEVICE))
      viewdirs_tr[top:top+n].copy_(viewdirs.flatten(0,1).to(DEVICE))
      ```
   d. **Update Indices**: Append `n` to `imsz` and update `top += n`.

3. **Handle Irregular Shapes or Downsampling** (Optional):
   If `irregular_shape` is True, split the rays by image size. If `downrate` is specified, downsample the rays by selecting every `downrate`-th pixel, then flatten and split by 10000 for batch processing.

4. **Ray Sampling for Training** (Runtime):
   During training, sample random ray indices from the flattened tensors:
   a. Calculate `num_possible_rays = sum(imsz)`.
   b. Generate `num_rays` random indices: `ray_indices = torch.randint(num_possible_rays, (batch_size, num_rays), device=device)`.
   c. Retrieve the corresponding rays and colors using these indices.

### Formulas
- **Ray Generation** (via `get_rays_of_a_view`):
  For a pixel at $(u, v)$, the ray origin $o$ and direction $d$ are computed from the camera matrix $K$ and pose $c2w$.
  $$
  \begin{aligned}
  \text{pixel\_center} &= (u + 0.5, v + 0.5) \\
  \text{cam\_coord} &= K^{-1} \cdot \text{pixel\_center} \\
  o &= c2w_{0:3, 3} \\
  d &= c2w_{0:3, 0:3} \cdot \text{cam\_coord} \\
  d &= \frac{d}{\|d\|}
  \end{aligned}
  $$
  [S1, S2, S3, unofficial]

- **Ray Sampling Step** (for volume rendering):
  $$
  \begin{aligned}
  N_{\text{samples}} &= \lfloor \| \text{world\_size} + 1 \| / \text{stepsize} \rfloor + 1 \\
  \text{vec} &= \text{where}(rays\_d == 0, 1e-6, rays\_d) \\
  \text{rate\_a} &= (\text{xyz\_max} - rays\_o) / \text{vec} \\
  \text{rate\_b} &= (\text{xyz\_min} - rays\_o) / \text{vec} \\
  t_{\text{min}} &= \text{clamp}(\min(\text{rate\_a}, \text{rate\_b}), \text{min}=near, \text{max}=far) \\
  t_{\text{max}} &= \text{clamp}(\max(\text{rate\_a}, \text{rate\_b}), \text{min}=near, \text{max}=far) \\
  \text{step} &= \text{stepsize} \cdot \text{voxel\_size} \cdot \text{rng} \\
  \text{interpx} &= t_{\text{min}} + \text{step} / \|rays\_d\| \\
  \text{rays\_pts} &= rays\_o + rays\_d \cdot \text{interpx}
  \end{aligned}
  $$
  [S2, unofficial]

### Parameters
- `near`: Near plane distance. Default overwritten to `1e9` in some implementations for unbounded scenes. [S2, unofficial]
- `far`: Far plane distance. [S2, unofficial]
- `stepsize`: Step size for sampling. [S2, unofficial]
- `downrate`: Downsampling rate for rays. [S2, unofficial]
- `irregular_shape`: Boolean flag for irregular ray shapes. [S2, unofficial]

### Edge Cases and Pitfalls
- **Precision**: When `rays_d` is zero, a small epsilon (`1e-6`) is used to avoid division by zero in step calculation. [S2, unofficial]
- **Boundaries**: Ray clipping to `near` and `far` planes is essential to prevent sampling outside the scene bounds. [S2, unofficial]
- **Memory**: Flattening all rays into a single tensor can consume significant GPU memory for high-resolution datasets. [S4, unofficial]
- **Irregular Shapes**: If images have different sizes, `imsz` must be tracked to correctly index into the flattened tensors. [S2, unofficial]

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def get_training_rays_flatten(rgb_tr, train_poses, K, world_size, xyz_min, xyz_max, 
                              near, far, stepsize, downrate=1, irregular_shape=False, 
                              DEVICE='cuda'):
    """
    Generate flattened ray tensors for training.
    
    Args:
        rgb_tr: Tensor of shape [B, V, H, W, 3] or list of images.
        train_poses: Tensor of shape [B*V, 4, 4] or list of c2w matrices.
        K: Intrinsic matrix [3, 3].
        world_size: Size of the voxel grid.
        xyz_min, xyz_max: Scene bounding box.
        near, far: Near and far plane distances.
        stepsize: Base step size.
        downrate: Downsampling factor.
        irregular_shape: Flag for irregular image sizes.
        DEVICE: Target device.
        
    Returns:
        rays_o_tr, rays_d_tr, viewdirs_tr, rgb_tr_flat, imsz
    """
    # Calculate total number of pixels
    if isinstance(rgb_tr, list):
        imsz = [im.shape[0] * im.shape[1] for im in rgb_tr]
        N = sum(imsz)
    else:
        B, V, H, W, _ = rgb_tr.shape
        imsz = [H * W] * (B * V)
        N = B * V * H * W
        
    # Initialize output tensors
    rays_o_tr = torch.zeros(N, 3, device=DEVICE)
    rays_d_tr = torch.zeros(N, 3, device=DEVICE)
    viewdirs_tr = torch.zeros(N, 3, device=DEVICE)
    rgb_tr_flat = torch.zeros(N, 3, device=DEVICE)
    
    top = 0
    for i, (c2w, img) in enumerate(zip(train_poses, rgb_tr.flatten(0,1))):
        # Assuming img is already flattened to [H*W, 3] if irregular, else reshape
        if irregular_shape:
            H, W = int(img.shape[0]**0.5), int(img.shape[1]**0.5) # Approximate, better to store H,W
            # In practice, H and W should be known or stored
        else:
            H, W = img.shape[0], img.shape[1]
            
        # Generate rays for this view
        # get_rays_of_a_view is a helper that computes rays_o, rays_d, viewdirs
        rays_o, rays_d, viewdirs = get_rays_of_a_view(H, W, K, c2w, ndc=False, inverse_y=False, flip_x=False, flip_y=False)
        
        # Flatten and copy
        n = H * W
        rgb_tr_flat[top:top+n].copy_(img.flatten(0,1))
        rays_o_tr[top:top+n].copy_(rays_o.flatten(0,1).to(DEVICE))
        rays_d_tr[top:top+n].copy_(rays_d.flatten(0,1).to(DEVICE))
        viewdirs_tr[top:top+n].copy_(viewdirs.flatten(0,1).to(DEVICE))
        
        top += n
        
    return rays_o_tr, rays_d_tr, viewdirs_tr, rgb_tr_flat, imsz

def get_rays_of_a_view(H, W, K, c2w, ndc=False, inverse_y=False, flip_x=False, flip_y=False):
    """
    Helper function to generate rays for a single view.
    """
    # Create pixel coordinates
    i, j = torch.meshgrid(torch.arange(W, device=c2w.device), torch.arange(H, device=c2w.device), indexing='ij')
    dirs = torch.stack([(i - K[0,2])/K[0,0], (j - K[1,2])/K[1,1], torch.ones_like(i)], dim=-1)
    
    # Transform to world space
    rays_d = (dirs @ c2w[:3,:3].T).to(c2w.device)
    rays_o = c2w[:3,3].expand_as(rays_d).to(c2w.device)
    
    # Normalize directions
    rays_d = F.normalize(rays_d, dim=-1)
    
    # View directions (same as rays_d for perspective)
    viewdirs = rays_d
    
    return rays_o, rays_d, viewdirs
```

## Subtopics

### Precomputed Ray Buffering
Precomputed Ray Buffering stores all rays and RGB values in CPU RAM before training. It iterates over training images, computes rays for each, and concatenates them into large tensors (`self.all_rays` of shape `(B, 8)` and `self.all_rgbs` of shape `(B, 3)`). This approach simplifies the training loop by providing pre-computed data but can exhaust CPU RAM for high-resolution datasets (e.g., 800x800). Validation and test splits often bypass this buffer to save memory. [S4, unofficial]

## Operating characteristics
- **Time Complexity**: $O(N)$ for initial ray generation, where $N$ is the total number of pixels. Sampling is $O(1)$ per ray.
- **Memory Complexity**: $O(N)$ for storing flattened rays and colors. This can be significant for large datasets.
- **Performance**: Efficient for training as it avoids per-batch ray generation overhead. However, memory usage can be a bottleneck.
- **Strengths**: Simplifies data loading, enables random ray sampling across the entire dataset.
- **Limitations**: High memory consumption, especially for high-resolution images. Not suitable for very large datasets without downsampling or chunking.
- **When to Use**: When training on a fixed dataset with manageable memory requirements.
- **When Not to Use**: When memory is constrained or when dealing with extremely high-resolution images.

## Background
Flattened Ray Generation for Training is a common technique in neural radiance fields (NeRF) and related volume rendering methods. It was popularized by early NeRF implementations and is widely used in frameworks like [kwea123/nerf_pl](https://github.com/kwea123/nerf_pl) and [Jumpat/SegmentAnythingin3D](https://github.com/Jumpat/SegmentAnythingin3D). The method addresses the need for efficient data access during stochastic gradient descent, where random ray sampling is crucial for learning.

## Sources
[S1] sjtuytc/UnboundedNeRFPytorch, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
[S2] Jumpat/SegmentAnythingin3D, https://github.com/Jumpat/SegmentAnythingin3D, fetched 2026-10-26 (unofficial)
[S3] cvg/NoPoSplat, https://github.com/cvg/NoPoSplat, fetched 2026-10-27 (unofficial)
[S4] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
