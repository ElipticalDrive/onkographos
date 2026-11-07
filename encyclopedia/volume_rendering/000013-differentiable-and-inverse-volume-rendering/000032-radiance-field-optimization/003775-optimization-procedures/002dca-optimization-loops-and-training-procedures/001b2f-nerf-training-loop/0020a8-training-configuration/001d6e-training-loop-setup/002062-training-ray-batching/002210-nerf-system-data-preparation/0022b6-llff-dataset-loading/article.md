# LLFF Dataset Loading

Also called: LLFF Bounds & Scale Normalization, Real Scene NeRF Dataset, Local Light Field Fusion Dataset, Local Light Field Fusion Dataset Loading

## Overview
Loads real-world LLFF (Local Light Field Fusion) dataset scenes, parsing COLMAP poses from `poses_bounds.npy`, correcting axis conventions, and applying NDC (Normalized Device Coordinates) normalization [S2, unofficial]. It differs from general NeRF dataset loading by enforcing forward-facing scene assumptions and specific axis corrections required for the LLFF capture rig [S3, unofficial].

## How it works
**Inputs:**
- Path to `poses_bounds.npy` (NumPy array of shape `[N, 17]`) [S2, unofficial].
- Path to image directory (e.g., `images/`) [S3, unofficial].
- `downscale` factor (integer, Not given in the sources.)
- `holdout` interval for validation split (integer, Not given in the sources.)

**Outputs:**
- `c2w`: Camera-to-world matrices, shape `[N, 4, 4]` [S1, unofficial].
- `images`: Downscaled images, shape `[N, H', W', 3]` [S3, unofficial].
- `near`, `far`: Scalar bounds for ray sampling [S2, unofficial].
- `rays_o`, `rays_d`: Ray origins and directions in NDC space, shape `[N, H', W', 3]` [S1, unofficial].

**Data Structures:**
- `poses_bounds.npy`: A single NumPy array where each row corresponds to a camera. Each row contains 17 values: the first 15 form the 3x5 pose matrix, and the last 2 are `near` and `far` bounds [S1, unofficial].

**Steps:**
1. Load `poses_bounds.npy` as a NumPy array of shape `[N, 17]` [S2, unofficial].
2. Reshape the pose portion: `poses = poses_arr[:, :-2].reshape(N, 3, 5)` [S1, unofficial].
3. Extract intrinsic parameters from the first camera: `H, W, focal = poses[0, :, 4]` [S1, unofficial].
4. Correct the axis convention: `poses_np = np.concatenate([poses[:, :, 1:2], -poses[:, :, 0:1], poses[:, :, 2:]], axis=2)` [S1, unofficial].
5. Append the homogeneous row `[0, 0, 0, 1]` to each 3x4 pose matrix to form 4x4 `c2w` matrices [S1, unofficial].
6. Set near and far bounds: `self.near = np.min(bounds) * 0.9` and `self.far = np.max(bounds) * 1.0`, where `bounds` is `poses_arr[:, -2:]` [S2, unofficial].
7. Load images from the `images/` directory, sorted by filename [S3, unofficial].
8. If `downscale > 1`, downscale images. If a pre-downscaled directory exists, load from there; otherwise, generate and save [S2, unofficial].
9. Split data into train and validation sets using the holdout interval: `val_ids = list(range(0, len(c2ws), holdout))` [S2, unofficial].
10. For each image, generate rays. Compute pixel centers `(i, j)` and direction vectors in camera space:
    $$ \text{dirs} = \left[ \frac{i - c_x}{f}, -\frac{j - c_y}{f}, -1 \right] $$
    where $c_x = W/2$, $c_y = H/2$, and $f$ is focal length [S1, unofficial].
11. Transform ray directions to world space: $\text{ray\_d} = \text{dirs} \cdot \text{c2w}_{0:3, 0:3}$ [S1, unofficial].
12. Set ray origins to camera center: $\text{ray\_o} = \text{c2w}_{0:3, 3}$ [S1, unofficial].
13. Convert rays to NDC space using `ndc_rays(H, W, focal, near, ray_o, ray_d)` [S1, unofficial].

**NDC Conversion Formula:**
The `ndc_rays` function shifts ray origins to the near plane and projects them into NDC space [S1, unofficial]:
1. Compute shift $t$:
   $$ t = -\frac{\text{near} + \text{ray\_o}_z}{\text{ray\_d}_z} $$
2. Shift origin:
   $$ \text{o0} = \text{ray\_o} + t \cdot \text{ray\_d} $$
3. Project to NDC:
   $$ \text{o0x} = -\frac{2f}{W} \frac{\text{o0}_x}{\text{o0}_z}, \quad \text{o0y} = -\frac{2f}{H} \frac{\text{o0}_y}{\text{o0}_z}, \quad \text{o0z} = 1 + \frac{2 \cdot \text{near}}{\text{o0}_z} $$
   $$ \text{dx} = -\frac{2f}{W} \left( \frac{\text{ray\_d}_x}{\text{ray\_d}_z} - \frac{\text{o0}_x}{\text{o0}_z} \right), \quad \text{dy} = -\frac{2f}{H} \left( \frac{\text{ray\_d}_y}{\text{ray\_d}_z} - \frac{\text{o0}_y}{\text{o0}_z} \right), \quad \text{dz} = -\frac{2 \cdot \text{near}}{\text{o0}_z} $$
4. Return $\text{ray\_o\_ndc} = [\text{o0x}, \text{o0y}, \text{o0z}]$ and $\text{ray\_d\_ndc} = [\text{dx}, \text{dy}, \text{dz}]$ [S1, unofficial].

**Parameters:**
- `WHITE_BG`: Always `False` for LLFF dataset [S3, unofficial].
- `holdout`: Not given in the sources.
- `downscale`: Not given in the sources.

**Edge Cases:**
- If `downscale` directory does not exist, images must be resized; ensure sufficient memory for large images [S2, unofficial].
- NDC conversion assumes forward-facing scenes; non-forward-facing scenes may produce invalid NDC coordinates [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import os
import numpy as np
import torch
from PIL import Image

def load_llff_dataset(data_dir, downscale=1, holdout=8):
    # Step 1: Load poses_bounds.npy
    poses_bounds = np.load(f"{data_dir}/poses_bounds.npy")
    N = poses_bounds.shape[0]

    # Step 2: Reshape poses and extract bounds
    poses = poses_bounds[:, :-2].reshape(N, 3, 5)
    bounds = poses_bounds[:, -2:]

    # Step 3: Extract intrinsics
    H, W, focal = poses[0, :, 4]
    H, W = int(H), int(W)

    # Step 4: Correct axis convention
    poses_np = np.concatenate([
        poses[:, :, 1:2],
        -poses[:, :, 0:1],
        poses[:, :, 2:]
    ], axis=2)

    # Step 5: Append homogeneous row
    c2ws = np.zeros((N, 4, 4))
    c2ws[:, :3, :4] = poses_np
    c2ws[:, 3, 3] = 1.0

    # Step 6: Set near/far
    near = np.min(bounds) * 0.9
    far = np.max(bounds) * 1.0

    # Step 7: Load images
    image_dir = f"{data_dir}/images"
    image_paths = sorted([f"{image_dir}/{f}" for f in os.listdir(image_dir) if f.endswith('.png') or f.endswith('.jpg')])
    images = [np.array(Image.open(p)) for p in image_paths]

    # Step 8: Downscale
    if downscale > 1:
        new_H, new_W = H // downscale, W // downscale
        images = [np.array(Image.fromarray(img).resize((new_W, new_H), Image.LANCZOS)) for img in images]
        H, W = new_H, new_W

    # Step 9: Split train/val
    val_ids = list(range(0, N, holdout))
    train_ids = [i for i in range(N) if i not in val_ids]

    # Step 10-13: Generate rays and NDC
    i, j = np.meshgrid(np.arange(W, dtype=np.float32), np.arange(H, dtype=np.float32), indexing="xy")
    cx, cy = W / 2.0, H / 2.0
    dirs = np.stack([(i - cx) / focal, -(j - cy) / focal, -np.ones_like(i)], -1)

    rays_o_list = []
    rays_d_list = []

    for idx in train_ids:
        c2w = c2ws[idx]
        ray_d = np.einsum("hwc,kc->hwk", dirs, c2w[:3, :3])
        ray_o = np.broadcast_to(c2w[:3, 3], ray_d.shape)

        # NDC conversion
        ray_o_t = torch.from_numpy(ray_o)
        ray_d_t = torch.from_numpy(ray_d)
        ray_o_ndc, ray_d_ndc = ndc_rays(H, W, focal, near, ray_o_t, ray_d_t)

        rays_o_list.append(ray_o_ndc.numpy())
        rays_d_list.append(ray_d_ndc.numpy())

    rays_o = np.stack(rays_o_list, axis=0)
    rays_d = np.stack(rays_d_list, axis=0)

    return c2ws, images, near, far, rays_o, rays_d

def ndc_rays(H, W, focal, near, ray_o, ray_d):
    # Shift to near plane
    t = -(near + ray_o[..., 2]) / ray_d[..., 2]
    o0 = ray_o + t.unsqueeze(-1) * ray_d

    # NDC projection
    o0x = -(2 * focal / W) * o0[..., 0] / o0[..., 2]
    o0y = -(2 * focal / H) * o0[..., 1] / o0[..., 2]
    o0z = 1.0 + 2.0 * near / o0[..., 2]

    dx = -(2 * focal / W) * (ray_d[..., 0] / ray_d[..., 2] - o0[..., 0] / o0[..., 2])
    dy = -(2 * focal / H) * (ray_d[..., 1] / ray_d[..., 2] - o0[..., 1] / o0[..., 2])
    dz = -2.0 * near / o0[..., 2]

    ray_o_ndc = torch.stack([o0x, o0y, o0z], dim=-1)
    ray_d_ndc = torch.stack([dx, dy, dz], dim=-1)
    return ray_o_ndc, ray_d_ndc
```

## Subtopics

### LLFF NDC Contraction
A method to handle forward-facing scenes in the LLFF dataset by contracting the entire scene into NDC space for reconstruction and mesh extraction [S4]. It condenses the entire scene into NDC space for reconstruction and mesh extraction [S4]. For this dataset, it employs DiffMC with a resolution of 375 and uses 9× sample per-pixel SSAA [S4].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
[S2] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S3] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S4] NeuManifold: Neural Watertight Manifold Reconstruction with Efficient and High-Quality Rendering Support, https://doi.org/10.1109/wacv61041.2025.00081, fetched 2026-10-29
