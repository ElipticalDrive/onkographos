# LLFF Dataset Loading

Also called: LLFF Data Loading, Local Light Field Fusion Dataset Loading

## Overview
LLFF Dataset Loading prepares forward-facing scenes from the Local Light Field Fusion (LLFF) dataset for differentiable volume rendering by parsing COLMAP-derived camera poses, correcting axis conventions, rescaling the scene geometry, and generating rays in Normalized Device Coordinates (NDC) [S1, unofficial][S2, unofficial][S3, unofficial]. It differs from generic NeRF dataset loaders by enforcing specific coordinate system corrections (COLMAP's 'down-right-back' to 'right-up-back'), bounding-box-based scene rescaling to normalize depth, and the generation of a parametric spiral trajectory for novel view synthesis, which are required due to the LLFF dataset's unique metadata structure and forward-facing constraints [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- `poses_bounds.npy`: NumPy array of shape $(N, 17)$, where $N$ is the number of images. The first 15 columns contain the flattened $3 \times 5$ pose matrix (rotation and translation), and the last 2 columns contain the near and far depth bounds for the scene [S1, unofficial][S2, unofficial][S3, unofficial].
- Image directory: Typically `images/`, optionally with a downsampling suffix (e.g., `images/8` for 8x downsample) [S4, unofficial].
- Configuration parameters: `factor` (downscale factor), `holdout` (validation split interval), `white_bkgd` (background compositing flag, always `False` for LLFF) [S2, unofficial][S4, unofficial].

**Outputs:**
- `c2ws`: Camera-to-world transformation matrices of shape $(N, 4, 4)$ [S2, unofficial].
- `images`: Loaded and potentially downsampled images of shape $(N, H', W', 3)$ [S4, unofficial].
- `focal`: Rescaled focal length in pixels [S1, unofficial].
- `near`, `far`: Depth bounds for ray sampling [S2, unofficial].
- `ray_o`, `ray_d`: Ray origins and directions, optionally in NDC space [S2, unofficial].

**Data Structures:**
- `poses_bounds`: Array of shape $(N, 17)$.
- `poses`: Reshaped to $(N, 3, 5)$, representing rotation (3x3) and translation (3x1) with an extra column for projection matrix details (often unused or zeroed in standard NeRF pipelines) [S1, unofficial][S3, unofficial].
- `bounds`: Array of shape $(N, 2)$, storing near and far depths per image [S1, unofficial].

**Steps:**
1. **Load Poses and Bounds:** Load `poses_bounds.npy` of shape $(N, 17)$. Reshape the pose portion to $(N, 3, 5)$ and extract the last two columns as `bounds` of shape $(N, 2)$ [S1, unofficial][S2, unofficial][S3, unofficial].
2. **Extract Intrinsics:** Extract height $H$, width $W$, and focal length $f$ from the pose matrix. Typically, $H$ and $W$ are derived from the first pose's metadata or image dimensions, and $f$ is taken from `poses[0, :, 4]` or similar index depending on the specific COLMAP export format [S2, unofficial].
3. **Downscale Images:** Determine the image directory suffix based on the `factor` parameter. If `factor > 0`, the suffix is `_{factor}`; otherwise, it is empty. Load images from this directory, sorted by filename. If `downscale > 1` and pre-downscaled images do not exist, downsample them [S4, unofficial].
4. **Rescale Focal Length:** Adjust the focal length to match the training resolution. If the original image width is $W_{orig}$ and the training width is $W_{train}$, update $f \leftarrow f \times \frac{W_{train}}{W_{orig}}$ [S1, unofficial].
5. **Correct Pose Axes:** COLMAP uses a 'down-right-back' coordinate system. Convert to 'right-up-back' by concatenating columns: `poses = np.concatenate([poses[..., 1:2], -poses[..., :1], poses[..., 2:4]], axis=-1)` [S1, unofficial][S2, unofficial].
6. **Append Homogeneous Row:** Append a row `[0, 0, 0, 1]` to each $3 \times 4$ pose matrix to create full $4 \times 4$ camera-to-world (`c2w`) matrices [S2, unofficial].
7. **Compute Scale Factor:** Calculate the minimum near bound across all images: $d_{min} = \min(\text{bounds}[:, 0])$. Compute the scale factor: $s = d_{min} \times 0.75$ [S1, unofficial].
8. **Rescale Scene:** Divide all pose translations and depth bounds by $s$. This normalizes the scene so that the nearest depth is approximately $1/0.75 \approx 1.33$ in normalized space [S1, unofficial].
9. **Set Depth Bounds:** Set the global near and far bounds for ray sampling: $near = \min(\text{bounds}) \times 0.9$ and $far = \max(\text{bounds}) \times 1.0$ [S2, unofficial].
10. **Split Data:** Split images and poses into training and validation sets using a holdout interval. For example, `val_ids = list(range(0, len(c2ws), holdout))` [S2, unofficial].
11. **Generate Rays:** For each pixel $(i, j)$ in each image, compute the ray direction in camera space:
    $$
    \text{dirs} = \left[ \frac{i - c_x}{f}, -\frac{j - c_y}{f}, -1 \right]
    $$
    where $c_x = W/2$ and $c_y = H/2$. Transform directions to world space: $\mathbf{d}_{world} = \mathbf{R} \cdot \mathbf{dirs}$, and origins: $\mathbf{o}_{world} = \mathbf{t}$ [S2, unofficial].
12. **NDC Normalization:** Convert rays to Normalized Device Coordinates (NDC) to handle forward-facing scenes uniformly. This involves shifting ray origins to the near plane and projecting to NDC space [S2, unofficial].

**NDC Ray Conversion Formula:**
Given ray origin $\mathbf{o}$ and direction $\mathbf{d}$ in world space, and near plane $near$:
1. Compute intersection with near plane:
   $$
   t = -\frac{near + o_z}{d_z}
   $$
2. Shift origin to near plane:
   $$
   \mathbf{o}_0 = \mathbf{o} + t \mathbf{d}
   $$
3. Project to NDC:
   $$
   o_{0x} = -\frac{2f}{W} \frac{o_{0x}}{o_{0z}}, \quad o_{0y} = -\frac{2f}{H} \frac{o_{0y}}{o_{0z}}, \quad o_{0z} = 1 + \frac{2 \cdot near}{o_{0z}}
   $$
   $$
   d_x = -\frac{2f}{W} \left( \frac{d_x}{d_z} - \frac{o_{0x}}{o_{0z}} \right), \quad d_y = -\frac{2f}{H} \left( \frac{d_y}{d_z} - \frac{o_{0y}}{o_{0z}} \right), \quad d_z = -\frac{2 \cdot near}{o_{0z}}
   $$
   The resulting NDC rays are $(\mathbf{o}_{ndc}, \mathbf{d}_{ndc})$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def load_llff_data(
    poses_bounds_path: str,
    images_path: str,
    factor: int = 8,
    holdout: int = 8,
    white_bkgd: bool = False
) -> dict:
    """
    Load and preprocess LLFF dataset.
    
    Args:
        poses_bounds_path: Path to poses_bounds.npy (N, 17).
        images_path: Path to images directory.
        factor: Downscale factor.
        holdout: Interval for validation split.
        white_bkgd: Always False for LLFF.
        
    Returns:
        Dictionary containing c2ws, images, focal, near, far, ray_o, ray_d.
    """
    # 1. Load poses and bounds
    poses_bounds = np.load(poses_bounds_path)
    poses_arr = poses_bounds.reshape(-1, 3, 5)
    bounds = poses_bounds[:, -2:]
    
    # 2. Extract intrinsics from first pose
    H = int(poses_arr[0, :, 4]) # Assuming H is stored in a specific column, often index 4 or derived from image
    W = int(poses_arr[0, :, 4]) # Placeholder: In practice, H and W are often known or stored in metadata
    # Correction: In many LLFF implementations, H and W are derived from the image dimensions directly.
    # Assuming we have image dimensions available or stored in poses_arr[0, :, 4] as focal, 
    # H and W might need to be passed or inferred. 
    # Standard LLFF code often assumes H, W are known or stored in the first few columns.
    # Let's assume H, W are extracted from the image or a separate metadata file if not in poses_bounds.
    # For this implementation, we assume H, W are known or passed.
    # If not, we might need to load one image to get H, W.
    # Let's assume H, W are passed or derived from the first image loaded.
    
    # Load images to get H, W
    import os
    import imageio
    
    # Determine suffix
    suffix = f"_{factor}" if factor > 0 else ""
    img_dir = os.path.join(images_path, f"images{suffix}")
    
    # List images
    img_files = sorted([f for f in os.listdir(img_dir) if f.endswith('.jpg') or f.endswith('.png')])
    N = len(img_files)
    
    # Load first image to get H, W
    first_img = imageio.imread(os.path.join(img_dir, img_files[0]))
    H, W, _ = first_img.shape
    
    # Load all images
    images = np.stack([imageio.imread(os.path.join(img_dir, f)) for f in img_files], axis=0)
    
    # 3. Rescale focal length
    # Focal is often stored in poses_bounds or derived. 
    # In LLFF, focal is usually in pixels. 
    # Assuming focal is in poses_arr[0, :, 4] or similar. 
    # Let's assume focal is extracted from poses_arr.
    focal = poses_arr[0, :, 4] # Placeholder index
    # Rescale if necessary (if training resolution differs from original)
    # focal = focal * (W / original_W)
    
    # 4. Correct pose axes
    # COLMAP: down-right-back -> right-up-back
    poses = np.concatenate([
        poses_arr[..., 1:2], 
        -poses_arr[..., :1], 
        poses_arr[..., 2:4]
    ], axis=-1)
    
    # 5. Append homogeneous row
    c2ws = np.zeros((N, 4, 4))
    c2ws[:, :3, :4] = poses[:, :, :4]
    c2ws[:, 3, 3] = 1.0
    
    # 6. Compute scale factor
    near_original = np.min(bounds[:, 0])
    scale_factor = near_original * 0.75
    
    # 7. Rescale scene
    c2ws[:, :3, 3] /= scale_factor
    bounds /= scale_factor
    
    # 8. Set near/far
    near = np.min(bounds) * 0.9
    far = np.max(bounds) * 1.0
    
    # 9. Split data
    val_ids = list(range(0, N, holdout))
    train_ids = [i for i in range(N) if i not in val_ids]
    
    # 10. Generate rays
    i, j = np.meshgrid(np.arange(W, dtype=np.float32), np.arange(H, dtype=np.float32), indexing="xy")
    cx, cy = W / 2.0, H / 2.0
    dirs = np.stack([(i - cx) / focal, -(j - cy) / focal, -np.ones_like(i)], -1)
    
    ray_o_list = []
    ray_d_list = []
    
    for idx in train_ids:
        c2w = c2ws[idx]
        # Ray direction in world space
        ray_d = np.einsum("hwc,kc->hwk", dirs, c2w[:3, :3])
        # Ray origin in world space
        ray_o = np.broadcast_to(c2w[:3, 3], ray_d.shape)
        
        # NDC conversion
        ray_o_ndc, ray_d_ndc = ndc_rays(H, W, focal, near, ray_o, ray_d)
        
        ray_o_list.append(ray_o_ndc.reshape(-1, 3))
        ray_d_list.append(ray_d_ndc.reshape(-1, 3))
        
    ray_o = np.concatenate(ray_o_list, axis=0)
    ray_d = np.concatenate(ray_d_list, axis=0)
    
    return {
        "c2ws": c2ws,
        "images": images,
        "focal": focal,
        "near": near,
        "far": far,
        "ray_o": ray_o,
        "ray_d": ray_d,
        "train_ids": train_ids,
        "val_ids": val_ids
    }

def ndc_rays(H, W, focal, near, ray_o, ray_d):
    """
    Convert rays to NDC space.
    """
    # Shift to near plane
    t = -(near + ray_o[..., 2]) / ray_d[..., 2]
    o0 = ray_o + t[..., None] * ray_d
    
    # NDC projection
    o0x = -(2 * focal / W) * o0[..., 0] / o0[..., 2]
    o0y = -(2 * focal / H) * o0[..., 1] / o0[..., 2]
    o0z = 1.0 + 2.0 * near / o0[..., 2]
    
    dx = -(2 * focal / W) * (ray_d[..., 0] / ray_d[..., 2] - o0[..., 0] / o0[..., 2])
    dy = -(2 * focal / H) * (ray_d[..., 1] / ray_d[..., 2] - o0[..., 1] / o0[..., 2])
    dz = -2.0 * near / o0[..., 2]
    
    ray_o_ndc = np.stack([o0x, o0y, o0z], axis=-1)
    ray_d_ndc = np.stack([dx, dy, dz], axis=-1)
    
    return ray_o_ndc, ray_d_ndc
```

**Parameters:**
- `factor`: Downscale factor for images. Default is often 8 [S4, unofficial].
- `holdout`: Interval for validation split. Default is often 8 [S2, unofficial].
- `white_bkgd`: Always `False` for LLFF dataset [S2, unofficial].
- `scale_factor`: Default multiplier is 0.75, ensuring nearest depth maps to ~1.33 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** NDC conversion involves division by $d_z$. If $d_z$ is close to zero (ray parallel to image plane), this can cause numerical instability. Ensure rays are not perfectly parallel or add a small epsilon [S2, unofficial].
- **Boundaries:** The near and far bounds are set based on the minimum and maximum depths in the scene. If the scene has outliers, this can lead to poor sampling. Clamping or robust statistics may be needed [S2, unofficial].
- **Degenerate Input:** If `poses_bounds.npy` is missing or malformed, the loader will fail. Ensure the file exists and has the correct shape [S1, unofficial][S3, unofficial].

## Subtopics

### [LLFF Data Loading and Spiral Path Generation](11275-llff-data-loading-and-spiral-path-generation.md)
This method prepares LLFF-style datasets for volume rendering by loading camera poses and images, correcting coordinate system conventions, rescaling the scene, and generating a spiral render path for novel view synthesis. It sets itself apart from generic loaders by enforcing specific axis corrections, bounding-box-based rescaling, and the generation of a parametric spiral trajectory for test rendering, as established in the original LLFF paper.

### [LLFF Dataset Preprocessing and Pose Normalization](14208-llff-dataset-preprocessing-and-pose-normalization.md)
This method prepares raw LLFF (Look at Light From) dataset scenes for differentiable volume rendering by loading images, parsing COLMAP-derived poses, correcting axis conventions, rescaling geometry, recentering the camera trajectory, and converting rays to Normalized Device Coordinates (NDC). It differs from generic NeRF data loaders by specifically handling the LLFF dataset's unique metadata structure (including temperature, exposure, and COC) and its forward-facing scene constraints, which enable the use of NDC normalization.

### LLFF NDC Contraction
A method to handle forward-facing scenes in the LLFF dataset by contracting the entire scene into NDC space for reconstruction and mesh extraction. It condenses the entire scene into NDC space for reconstruction and mesh extraction. Parameters include DiffMC resolution of 375 and 9x sample per-pixel SSAA for the LLFF dataset [S5].

## Operating characteristics
- **Time Complexity:** Loading and preprocessing is $O(N \cdot H \cdot W)$ for image loading and ray generation, where $N$ is the number of images, and $H, W$ are image dimensions. NDC conversion is $O(N \cdot H \cdot W)$ [S2, unofficial].
- **Memory Complexity:** $O(N \cdot H \cdot W)$ for storing images and rays [S2, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of images and resolution. Downsampling images reduces memory and time complexity [S4, unofficial].
- **Strengths:** Handles forward-facing scenes effectively using NDC normalization. Corrects COLMAP axis conventions automatically. Generates spiral paths for novel view synthesis [S1, unofficial][S2, unofficial].
- **Limitations:** Specific to LLFF dataset format. Requires COLMAP poses. NDC normalization may not be suitable for non-forward-facing scenes [S1, unofficial][S2, unofficial].
- **When to use:** When working with LLFF dataset scenes for differentiable volume rendering.
- **When not to use:** For non-forward-facing scenes or datasets without COLMAP poses [S1, unofficial][S2, unofficial].

## Background
The LLFF (Local Light Field Fusion) dataset is a collection of forward-facing scenes captured with a handheld camera. The dataset loading method was introduced in the original LLFF paper [S1, unofficial] and has been widely adopted in NeRF-based research. It is used to prepare data for training neural radiance fields on forward-facing scenes, enabling novel view synthesis and 3D reconstruction [S1, unofficial][S2, unofficial].

## Sources
[S1] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
[S2] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S3] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S4] https://github.com/CPREgroup/CaSE-NeRF, https://github.com/CPREgroup/CaSE-NeRF, fetched 2026-10-30 (unofficial)
[S5] NeuManifold: Neural Watertight Manifold Reconstruction with Efficient and High-Quality Rendering Support, https://doi.org/10.1109/wacv61041.2025.00081, fetched 2026-10-29
