# LLFF Dataset Loading

Also called: LLFF Dataset, Real Scene NeRF Dataset, Local Light Field Fusion Dataset, Local Light Field Fusion Dataset Loading

## Overview
Loads real-world LLFF (Local Light Field Fusion) dataset scenes, parsing COLMAP poses from `poses_bounds.npy`, correcting axis conventions, and applying NDC (Normalized Device Coordinates) normalization [S1, S2, unofficial]. It differs from general NeRF dataset loading by enforcing forward-facing scene assumptions and specific axis corrections required for the LLFF capture rig [S1, S2, unofficial].

## How it works
**Inputs:**
- Path to `poses_bounds.npy` (NumPy array of shape `[N, 17]`).
- Path to image directory (e.g., `images/`).
- `downscale` factor (integer, typically 1, 2, 4, or 8).
- `holdout` interval for validation split (integer, typically 8).

**Outputs:**
- `c2w`: Camera-to-world matrices, shape `[N, 4, 4]`.
- `images`: Downscaled images, shape `[N, H', W', 3]`.
- `near`, `far`: Scalar bounds for ray sampling.
- `rays_o`, `rays_d`: Ray origins and directions in NDC space, shape `[N, H', W', 3]`.

**Data Structures:**
- `poses_bounds.npy`: A single NumPy array where each row corresponds to a camera. The last 17 values are `[H, W, focal, ... 12 values of pose data ..., near, far]`. The pose data occupies columns 3 to 14 (12 values), representing the first 3 rows of the 4x4 transformation matrix flattened row-major, excluding the last row which is `[0,0,0,1]`.

**Steps:**
1. Load `poses_bounds.npy` as a NumPy array of shape `[N, 17]` [S1, S2, unofficial].
2. Reshape the pose portion: `poses = poses_arr[:, :-2].reshape(N, 3, 5)` [S1, S2, unofficial]. The last 2 columns are `near` and `far` bounds.
3. Extract intrinsic parameters from the first camera: `H, W, focal = poses[0, :, 4]` [S1, unofficial].
4. Correct the axis convention to match the expected coordinate system (COLMAP uses a right-handed system with Z-forward, but NeRF often expects Y-up, Z-backward or similar variations). The correction is: `poses_np = np.concatenate([poses[:, :, 1:2], -poses[:, :, 0:1], poses[:, :, 2:]], axis=2)` [S1, unofficial]. This swaps X and Y axes and negates the new X axis.
5. Append the homogeneous row `[0, 0, 0, 1]` to each 3x4 pose matrix to form 4x4 `c2w` matrices [S1, unofficial].
6. Set near and far bounds: `self.near = np.min(bounds) * 0.9` and `self.far = np.max(bounds) * 1.0`, where `bounds` is `poses_arr[:, -2:]` [S1, unofficial].
7. Load images from the `images/` directory, sorted by filename [S2, unofficial].
8. If `downscale > 1`, downscale images. If a pre-downscaled directory exists, load from there; otherwise, generate and save [S2, unofficial].
9. Split data into train and validation sets using the holdout interval: `val_ids = list(range(0, len(c2ws), holdout))` [S1, unofficial].
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
- `WHITE_BG`: Always `False` for LLFF dataset [S1, unofficial].
- `holdout`: Typically 8 [S1, unofficial].
- `downscale`: Typically 1, 2, 4, or 8 [S2, unofficial].

**Edge Cases:**
- If `poses_bounds.npy` is missing or malformed, loading fails.
- If `downscale` directory does not exist, images must be resized; ensure sufficient memory for large images.
- NDC conversion assumes forward-facing scenes; non-forward-facing scenes may produce invalid NDC coordinates [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def load_llff_dataset(data_dir, downscale=1, holdout=8):
    # Step 1: Load poses_bounds.npy
    poses_bounds = np.load(f"{data_dir}/poses_bounds.npy")
    N = poses_bounds.shape[0]
    
    # Step 2: Reshape poses
    poses_arr = poses_bounds.reshape(N, -1, 17)[:, :, :-2]
    poses = poses_arr[:, :, :3].reshape(N, 3, 5)
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

## Operating characteristics
- **Time Complexity:** Linear in the number of images $N$ and pixels $H \times W$ for ray generation.
- **Memory Complexity:** $O(N \cdot H \cdot W)$ for storing rays.
- **Strengths:** Handles real-world COLMAP data with axis corrections; NDC normalization simplifies sampling for forward-facing scenes.
- **Limitations:** Only suitable for forward-facing scenes; NDC can cause artifacts if scenes are not strictly forward-facing.

## Background
The LLFF dataset was introduced in "Local Light Field Fusion" (CVPR 2016) for novel view synthesis. It is widely used in NeRF research for evaluating forward-facing scene reconstruction [S1, S2, unofficial].

## Sources
[S1] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S2] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
