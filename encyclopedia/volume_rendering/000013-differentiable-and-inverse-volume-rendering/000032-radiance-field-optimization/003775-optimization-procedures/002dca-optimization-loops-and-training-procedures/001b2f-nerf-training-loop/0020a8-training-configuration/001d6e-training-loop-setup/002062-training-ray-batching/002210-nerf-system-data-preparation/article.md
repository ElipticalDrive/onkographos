# NeRF System Data Preparation

Also called: Blender Dataset Loading, NeRF training data sampling, NeRF validation data sampling, Synthetic NeRF Dataset

## Overview
This method prepares training and validation data for Neural Radiance Fields (NeRF) by loading synthetic Blender dataset scenes, parsing JSON metadata for camera poses and images, and precomputing rays and RGB values [S1, S2, unofficial]. It differs from [LLFF Dataset Loading](2818-llff-dataset-loading.md) by assuming known, precise camera poses provided in JSON rather than estimating them, and by using fixed depth bounds (`near: 2.0`, `far: 6.0`) instead of NDC normalization [S1, S2, unofficial]. It differs from [NeRF Dataset Loading](3084-nerf-dataset-loading.md) by explicitly handling the specific preprocessing steps for the Blender synthetic dataset, including focal length computation from `camera_angle_x` and RGBA compositing [S1, S2, unofficial].

## How it works
**Inputs:**
- `transforms_{split}.json`: JSON file containing metadata for a specific split (train/val/test) [S1, unofficial].
- Image files: Corresponding to the paths in the JSON [S1, unofficial].
- Configuration parameters: `batch_image_sampling` (boolean), `train_num_rays` (integer), `background_color` (string: 'white' or 'random') [S2, unofficial].

**Outputs:**
- `rays`: Tensor of shape `(N, 6)` containing ray origins `(N, 3)` and normalized directions `(N, 3)` [S2, unofficial].
- `rgb`: Tensor of shape `(N, 3)` containing target RGB colors [S2, unofficial].
- `fg_mask`: Tensor of shape `(N, 1)` containing foreground masks (optional, depending on implementation) [S2, unofficial].
- `background_color`: Tensor of shape `(3,)` for compositing [S2, unofficial].

**Data Structures:**
- `rays`: Concatenated `[ray_o, ray_d]` where `ray_o` is origin and `ray_d` is unit direction [S2, unofficial].
- `c2w`: Camera-to-world transformation matrix, typically `(4, 4)` or `(3, 4)` [S1, unofficial].

**Steps:**

1. **Load Metadata and Compute Focal Length:**
   Parse `transforms_{split}.json`. Extract `camera_angle_x` (in radians). Compute focal length $f$:
   $$ f = \frac{0.5 \times W}{\tan(0.5 \times \text{camera\_angle\_x})} $$
   where $W$ is the image width [S1, unofficial].

2. **Select Images:**
   - If stage is 'train' and `batch_image_sampling` is true: Sample `train_num_rays` random indices from `self.dataset.all_images` [S2, unofficial].
   - If stage is 'train' and `batch_image_sampling` is false: Sample 1 random index [S2, unofficial].
   - If stage is 'val' or 'test': Use the first camera for the index [S2, unofficial].

3. **Load and Preprocess Images:**
   - Load the image for the selected index.
   - Convert to RGBA.
   - Resize to $(W, H)$ [S1, unofficial].
   - Composite onto a white background:
     $$ \text{rgb} = \text{rgba}[:, :, :3] \times \text{rgba}[:, :, 3:] + (1 - \text{rgba}[:, :, 3:]) $$
     [S1, unofficial].

4. **Generate Rays:**
   - Get camera-to-world matrix `c2w` for the selected index [S2, unofficial].
   - Generate ray origins `ray_o` and directions `ray_d` using `get_rays_np(H, W, focal, c2w)` [S1, unofficial].
     - `get_rays_np` computes pixel coordinates $(i, j)$, camera-space directions, and transforms them to world space using `c2w` [S1, unofficial].
   - Normalize ray directions:
     $$ \text{ray\_d} = \text{normalize}(\text{ray\_d}) $$
     [S2, unofficial].
   - Concatenate origins and directions:
     $$ \text{rays} = \text{concat}([\text{ray\_o}, \text{ray\_d}], \text{dim}=-1) $$
     [S2, unofficial].

5. **Sample Pixels (Training):**
   - If training, sample random pixel coordinates $(x, y)$ within image dimensions $w, h$ [S2, unofficial].
   - Extract RGB values and foreground masks for these pixels using the precomputed rays [S2, unofficial].
   - If validation/testing, use all directions and full image RGB/masks [S2, unofficial].

6. **Set Background Color:**
   - If `background_color` is 'white', set `self.model.background_color` to `torch.ones((3,), dtype=torch.float32)` [S2, unofficial].
   - If 'random', initialize randomly [S2, unofficial].

**Parameters:**
- `near`: 2.0 [S1, unofficial].
- `far`: 6.0 [S1, unofficial].
- `WHITE_BG`: Always True for Blender dataset [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
import json
from PIL import Image

def load_blender_data(json_path, split, config):
    """
    Loads and preprocesses Blender dataset data.
    
    Args:
        json_path: Path to transforms_{split}.json
        split: 'train', 'val', or 'test'
        config: Configuration object with batch_image_sampling, train_num_rays, background_color
        
    Returns:
        rays: Tensor (N, 6)
        rgb: Tensor (N, 3)
        fg_mask: Tensor (N, 1) or None
    """
    with open(json_path, 'r') as f:
        meta = json.load(f)
    
    # Compute focal length
    W = meta['w']
    H = meta['h']
    camera_angle_x = meta['camera_angle_x']
    focal = 0.5 * W / np.tan(0.5 * camera_angle_x)
    
    # Load images and poses
    images = []
    poses = []
    for frame in meta['frames']:
        img_path = frame['file_path']
        img = Image.open(img_path).convert('RGBA')
        img = img.resize((W, H), Image.Resampling.LANCZOS)
        images.append(np.array(img))
        poses.append(np.array(frame['transform_matrix'], dtype=np.float32))
    
    images = np.array(images)
    poses = np.array(poses)
    
    # Select indices
    if split == 'train':
        if config.batch_image_sampling:
            indices = np.random.choice(len(images), config.train_num_rays, replace=False)
        else:
            indices = np.random.choice(len(images), 1, replace=False)
    else:
        indices = np.array([0]) # Use first camera for val/test
    
    # Precompute rays for all pixels in selected images
    all_rays_o = []
    all_rays_d = []
    all_rgb = []
    all_fg_mask = []
    
    for idx in indices:
        c2w = poses[idx]
        img = images[idx]
        
        # Generate rays for the entire image
        ray_o, ray_d = get_rays_np(H, W, focal, c2w)
        
        # Normalize directions
        ray_d_norm = ray_d / np.linalg.norm(ray_d, axis=-1, keepdims=True)
        
        # Composite RGBA to RGB
        rgba = img.astype(np.float32) / 255.0
        rgb = rgba[:, :, :3] * rgba[:, :, 3:] + (1 - rgba[:, :, 3:])
        
        all_rays_o.append(ray_o)
        all_rays_d.append(ray_d_norm)
        all_rgb.append(rgb)
        all_fg_mask.append(rgba[:, :, 3:])
    
    # Stack all rays
    rays_o = np.concatenate(all_rays_o, axis=0).reshape(-1, 3)
    rays_d = np.concatenate(all_rays_d, axis=0).reshape(-1, 3)
    rgb_all = np.concatenate(all_rgb, axis=0).reshape(-1, 3)
    fg_mask_all = np.concatenate(all_fg_mask, axis=0).reshape(-1, 1)
    
    # Reshape to (N, 6)
    rays = np.concatenate([rays_o, rays_d], axis=-1)
    
    # Convert to tensors
    rays = torch.from_numpy(rays).float()
    rgb = torch.from_numpy(rgb_all).float()
    fg_mask = torch.from_numpy(fg_mask_all).float()
    
    return rays, rgb, fg_mask

def get_rays_np(H, W, focal, c2w, cx=None, cy=None):
    """NumPy version for dataset preprocessing."""
    if cx is None: cx = W / 2.0
    if cy is None: cy = H / 2.0
    
    i, j = np.meshgrid(np.arange(W, dtype=np.float32),
                       np.arange(H, dtype=np.float32),
                       indexing='xy')
    
    # Camera-space direction (OpenCV convention: z-forward, y-down, x-right)
    dirs = np.stack([
        (i - cx) / focal,
        -(j - cy) / focal,    # flip y (image y-down -> camera y-up)
        -np.ones_like(i),     # looking down -z in camera space (Blender/NeRF convention)
    ], axis=-1)  # (H, W, 3)
    
    # Rotate directions to world space
    ray_d = (dirs @ c2w[:3, :3].T)   # (H, W, 3)
    # All rays share the camera centre as origin
    ray_o = np.broadcast_to(c2w[:3, 3], (H, W, 3))                         # (H, W, 3)
    return ray_o, ray_d
```

## Subtopics

### [LLFF Dataset Loading](2818-llff-dataset-loading.md)
Loads real-world LLFF (Local Light Field Fusion) dataset scenes, parsing COLMAP poses from `poses_bounds.npy`, correcting axis conventions, and applying NDC (Normalized Device Coordinates) normalization. It differs from general NeRF dataset loading by enforcing forward-facing scene assumptions and specific axis corrections required for the LLFF capture rig.

### [NeRF Dataset Loading](3084-nerf-dataset-loading.md)
This method loads the NeRF-Synthetic (Blender) dataset by parsing JSON metadata for camera poses and images, handling half-resolution resizing, and returning normalized RGBA images and camera parameters. It differs from [LLFF Dataset Loading](2818-llff-dataset-loading.md) by assuming known, precise camera poses provided in JSON rather than estimating them from images, and by using a fixed scene bounding box.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths and Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
Implemented in open-source NeRF training pipelines for synthetic datasets [S1, S2, unofficial].

## Sources
[S1] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S2] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
