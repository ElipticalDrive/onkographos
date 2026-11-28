# NeRF System Data Preparation

Also called: Blender Dataset Loading, NeRF training data sampling, NeRF validation data sampling, Synthetic NeRF Dataset

## Overview
NeRF System Data Preparation loads synthetic Blender dataset scenes, parsing JSON metadata for camera poses and images, and precomputes rays and RGB values for training and validation [S1, unofficial]. It differs from generic loaders by enforcing specific coordinate system conversions (OpenCV to NeRF's right-handed convention), half-resolution resizing for the synthetic dataset, and alpha compositing onto a white background to handle transparent foregrounds [S1, unofficial]. Unlike [LLFF Dataset Loading](2818-llff-dataset-loading.md), which handles sparse real-world poses and NDC normalization, this method assumes known, precise camera poses from synthetic renders [S1, unofficial].

## How it works
**Inputs:**
- `transforms_{split}.json`: Metadata file containing `camera_angle_x`, `frames` array (with `file_path`, `transform_matrix`), and optional `bg_color`.
- `split`: String, either `'train'`, `'val'`, or `'test'`.

**Outputs:**
- `rays`: Tensor of shape `(N, 6)` or `(H, W, 6)` containing concatenated ray origins and normalized directions.
- `rgb`: Tensor of shape `(N, 3)` or `(H, W, 3)` containing the target RGB colors.
- `masks`: Tensor of shape `(N, 1)` or `(H, W, 1)` containing foreground alpha masks (optional, used for validation).

**Data Structures:**
- `rays`: Float32 tensor. Columns 0-2 are ray origins $(o_x, o_y, o_z)$, columns 3-5 are normalized ray directions $(d_x, d_y, d_z)$.
- `c2w`: 4x4 camera-to-world transformation matrix.
- `focal`: Float scalar, computed from `camera_angle_x`.

**Parameters:**
- `WHITE_BG`: Boolean, always `True` for Blender dataset [S1, unofficial].
- `near`: Float, typically `2.0` [S1, unofficial].
- `far`: Float, typically `6.0` [S1, unofficial].
- `resize_factor`: Integer, typically `2` (images are resized to half resolution) [S1, unofficial].
- `train_num_rays`: Integer, number of random rays to sample per training step [S2, unofficial].
- `batch_image_sampling`: Boolean, if `True`, samples random indices from all training images; otherwise, samples 1 random index per step [S2, unofficial].

**Steps:**

1. **Parse Metadata:** Load `transforms_{split}.json`. Extract `camera_angle_x`. Compute focal length:
   $$ f = \frac{0.5 \times W}{\tan(0.5 \times \text{camera\_angle\_x})} $$
   where $W$ is the original image width [S1, unofficial].

2. **Load and Preprocess Images:** For each frame in the JSON:
   - Load the image specified by `file_path`.
   - Convert to RGBA.
   - Resize to $(W/2, H/2)$ [S1, unofficial].
   - Composite onto a white background:
     $$ \text{rgb} = \text{rgba}_{:,:3} \times \text{rgba}_{:,:,3} + (1 - \text{rgba}_{:,:,3}) $$
     Store the resulting RGB image and the alpha mask [S1, unofficial].

3. **Compute Ray Directions (Precomputation):**
   - Generate a grid of pixel coordinates $(i, j)$ for the resized image dimensions.
   - Compute camera-space directions using the focal length and principal point $(W/2, H/2)$:
     $$ d_x = \frac{i - W/2}{f}, \quad d_y = -\frac{j - H/2}{f}, \quad d_z = -1 $$
     (The negative sign on $d_y$ and $d_z$ accounts for the coordinate system flip from image space to NeRF's camera space) [S1, unofficial].
   - Store these directions in a tensor `self.dataset.directions` [S2, unofficial].

4. **Sample Rays (Per Iteration/Step):**
   - If `split` is `'train'`:
     - If `batch_image_sampling` is `True`, sample `train_num_rays` random indices from `self.dataset.all_images`. Else, sample 1 random index [S2, unofficial].
     - For each sampled index, retrieve the camera-to-world matrix `c2w` [S2, unofficial].
     - Sample random pixel coordinates $(x, y)$ within the image dimensions $w, h$ [S2, unofficial].
     - Extract ray origins from `c2w` and directions from `self.dataset.directions` based on the sampled coordinates [S2, unofficial].
     - Extract RGB values and masks for the sampled pixels [S2, unofficial].
   - If `split` is `'val'` or `'test'`:
     - Use the first camera for the index (or all cameras if generating full validation set) [S2, unofficial].
     - Extract all directions and full image RGB/masks [S2, unofficial].

5. **Normalize and Concatenate:**
   - Normalize ray directions to unit length:
     $$ \hat{d} = \frac{d}{\|d\|_2} $$
   - Concatenate origins and normalized directions:
     $$ \text{rays} = [\text{ray\_o}, \hat{d}] $$
     Shape is `(N, 6)` for sampled rays or `(H, W, 6)` for full images [S1, S2, unofficial].

6. **Set Background Color:**
   - If training, set `self.model.background_color` based on config. For Blender, this is `torch.ones((3,), dtype=torch.float32)` (white) [S1, S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
import json
from PIL import Image

def load_blender_data(data_dir, split, resize_factor=2):
    """
    Loads NeRF Blender dataset.
    Returns: rays, rgbs, masks (for validation), or ray_indices, rgbs, masks (for training)
    """
    with open(f"{data_dir}/transforms_{split}.json", 'r') as f:
        meta = json.load(f)
    
    W_orig = meta['w']
    H_orig = meta['h']
    focal = 0.5 * W_orig / np.tan(0.5 * meta['camera_angle_x'])
    
    W, H = W_orig // resize_factor, H_orig // resize_factor
    
    # Precompute directions for the resized image
    i, j = np.meshgrid(np.arange(W), np.arange(H), indexing='xy')
    dirs = np.stack([
        (i - W/2.0) / focal,
        -(j - H/2.0) / focal,
        -np.ones_like(i)
    ], axis=-1) # (H, W, 3)
    
    frames = meta['frames']
    all_images = []
    all_masks = []
    all_c2w = []
    
    for frame in frames:
        img_path = f"{data_dir}/{frame['file_path']}"
        img = Image.open(img_path).convert('RGBA')
        img = img.resize((W, H), Image.Resampling.BILINEAR)
        img = np.array(img).astype(np.float32) / 255.0
        
        # Composite on white background
        rgba = img
        rgb = rgba[..., :3] * rgba[..., 3:] + (1 - rgba[..., 3:])
        mask = rgba[..., 3:]
        
        all_images.append(rgb)
        all_masks.append(mask)
        
        # Convert 4x4 transform matrix to 3x4 (c2w)
        c2w = np.array(frame['transform_matrix'])[:3, :]
        all_c2w.append(c2w)
        
    all_images = np.stack(all_images, axis=0)
    all_masks = np.stack(all_masks, axis=0)
    all_c2w = np.stack(all_c2w, axis=0)
    
    return all_images, all_masks, all_c2w, dirs, focal

def sample_training_rays(all_images, all_masks, all_c2w, dirs, focal, train_num_rays=1024):
    """
    Samples random rays for a single training step.
    """
    num_images = all_images.shape[0]
    
    # Sample image indices
    if np.random.rand() < 0.5: # Example logic for batch_image_sampling
        img_indices = np.random.choice(num_images, size=train_num_rays, replace=False)
    else:
        img_indices = np.random.choice(num_images, size=1)
        
    rays_o_list = []
    rays_d_list = []
    rgbs_list = []
    masks_list = []
    
    for idx in img_indices:
        c2w = all_c2w[idx]
        rgb = all_images[idx]
        mask = all_masks[idx]
        
        # Sample random pixel coordinates
        h, w = rgb.shape[:2]
        x = np.random.randint(0, w)
        y = np.random.randint(0, h)
        
        # Get direction
        d = dirs[y, x]
        d = d / np.linalg.norm(d)
        
        # Get origin
        o = c2w[:, 3]
        
        rays_o_list.append(o)
        rays_d_list.append(d)
        rgbs_list.append(rgb[y, x])
        masks_list.append(mask[y, x])
        
    rays_o = np.stack(rays_o_list, axis=0)
    rays_d = np.stack(rays_d_list, axis=0)
    rgbs = np.stack(rgbs_list, axis=0)
    masks = np.stack(masks_list, axis=0)
    
    rays = np.concatenate([rays_o, rays_d], axis=-1)
    
    return rays, rgbs, masks

def sample_validation_rays(all_images, all_masks, all_c2w, dirs, focal):
    """
    Samples all rays for validation.
    """
    rays_list = []
    rgbs_list = []
    masks_list = []
    
    for idx in range(all_images.shape[0]):
        c2w = all_c2w[idx]
        rgb = all_images[idx]
        mask = all_masks[idx]
        h, w = rgb.shape[:2]
        
        # Generate all rays
        i, j = np.meshgrid(np.arange(w), np.arange(h), indexing='xy')
        dirs_full = np.stack([
            (i - w/2.0) / focal,
            -(j - h/2.0) / focal,
            -np.ones_like(i)
        ], axis=-1)
        
        # Normalize directions
        dirs_norm = dirs_full / np.linalg.norm(dirs_full, axis=-1, keepdims=True)
        
        # Origins
        o = c2w[:, 3]
        rays_o = np.broadcast_to(o, (h, w, 3))
        
        rays = np.concatenate([rays_o, dirs_norm], axis=-1)
        
        rays_list.append(rays)
        rgbs_list.append(rgb)
        masks_list.append(mask)
        
    rays = np.concatenate([r.reshape(-1, 6) for r in rays_list], axis=0)
    rgbs = np.concatenate([r.reshape(-1, 3) for r in rgbs_list], axis=0)
    masks = np.concatenate([m.reshape(-1, 1) for m in masks_list], axis=0)
    
    return rays, rgbs, masks
```

**Edge Cases and Pitfalls:**
- **Coordinate Systems:** The Blender dataset uses a right-handed coordinate system with Y-up and Z-forward (or similar, depending on the exporter), while NeRF assumes a specific camera space (Z-back, Y-down). The `get_rays` function must correctly flip the Y and Z axes of the direction vectors [S1, unofficial].
- **Alpha Compositing:** The synthetic dataset contains transparent backgrounds. Failing to composite onto a white background before training will result in the network learning to predict black or transparent values incorrectly, as the loss function expects RGB values [S1, unofficial].
- **Resolution:** The dataset is typically provided at high resolution (e.g., 800x800). Resizing to half resolution (400x400) is standard to reduce memory usage and training time [S1, unofficial].

## Subtopics

### NeRF-RPN Training Data Construction
Selects 90 scenes from ScanNet, SceneNN, and INRIA datasets. For each scene, video frames are uniformly divided into 100 bins, and the sharpest frame in each bin is selected based on the variance of the Laplacian. Provided depth maps and a depth-guided NeRF are used for training. Minimum bounding boxes are computed based on annotated meshes, and objects of certain categories (e.g., text cut off) are discarded [S3].

### [NeRF Dataset Loading](3084-nerf-dataset-loading.md)
Loads the NeRF-Synthetic (Blender) dataset by parsing JSON metadata for camera poses and images. It handles coordinate system conversions, half-resolution resizing, and alpha compositing to return normalized RGBA images and camera parameters. It differs from [LLFF Dataset Loading](2818-llff-dataset-loading.md) by assuming known, precise camera poses from synthetic renders rather than estimating them from sparse, unstructured real-world photos.

## Operating characteristics
- **Time Complexity:** Preprocessing is $O(N \cdot H \cdot W)$ where $N$ is the number of images, and $H, W$ are the image dimensions. Ray sampling is $O(R)$ where $R$ is the number of rays per batch.
- **Memory Complexity:** Storing all precomputed rays for validation is $O(N \cdot H \cdot W)$. For training, only the sampled rays are stored, $O(R)$. The images themselves are stored in memory, $O(N \cdot H \cdot W \cdot C)$.
- **Performance:** The method is efficient for synthetic datasets due to the small number of images and the ability to precompute rays. However, loading and resizing images can be a bottleneck if not cached.
- **Strengths:** Simple to implement, robust for synthetic data, handles transparency correctly.
- **Limitations:** Not suitable for real-world datasets without modification (e.g., LLFF, Tanks and Temples). Assumes perfect camera poses.

## Background
The NeRF System Data Preparation method originates from the original NeRF paper (Mildenhall et al., 2020) and its accompanying codebase, which introduced the standard pipeline for training on the Blender synthetic dataset [S1, unofficial]. It has become the de facto standard for evaluating NeRF variants on synthetic data.

## Sources
[S1] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S2] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S3] NeRF-RPN: A general framework for object detection in NeRFs, https://doi.org/10.1109/cvpr52729.2023.02253, fetched 2026-10-26
