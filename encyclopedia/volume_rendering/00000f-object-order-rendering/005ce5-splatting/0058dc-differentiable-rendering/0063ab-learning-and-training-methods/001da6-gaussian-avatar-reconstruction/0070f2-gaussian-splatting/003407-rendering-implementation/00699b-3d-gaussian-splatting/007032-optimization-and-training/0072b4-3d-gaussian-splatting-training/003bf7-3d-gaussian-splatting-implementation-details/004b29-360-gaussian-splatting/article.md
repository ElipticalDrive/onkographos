# 360 Gaussian Splatting

Also called: 360-GS

## Overview
360 Gaussian Splatting is a method for generating high-quality 3D spatial representations from 360-degree camera images by combining OpenSfM for sparse point cloud generation and Gaussian Splatting for dense Gaussian conversion [S1, unofficial]. It solves the problem that standard Gaussian Splatting assumes perspective cameras and cannot directly process equirectangular images from 360-degree cameras, a limitation that previously forced users to crop or split images [S1, unofficial]. It sets itself apart from standard implementations by modifying the differentiable rasterization module to support spherical rendering and using OpenSfM, which natively supports equirectangular inputs, unlike Colmap [S1, unofficial].

## How it works
**Inputs:** A set of 360-degree equirectangular images [S1, unofficial].
**Outputs:** A set of 3D Gaussians (positions, covariances, colors, opacities) optimized to render the input images [S1, unofficial].

**Data Structures:**
*   **Sparse Point Cloud:** Derived from OpenSfM's `reconstruction.json` [S1, unofficial]. Contains 3D positions [S1, unofficial]. The `error` field present in the source data is ignored during training and set to 0 [S1, unofficial].
*   **Camera Data:** Derived from OpenSfM's `reconstruction.json` [S1, unofficial]. Contains camera positions and intrinsic/extrinsic parameters [S1, unofficial].
*   **Gaussian Representation:** Standard 3D Gaussian Splatting representation (position $\mathbf{\mu}$, covariance $\mathbf{\Sigma}$, spherical harmonics coefficients, opacity $\alpha$, color $c$) [G].

**Steps:**
1.  **Structure from Motion (SfM):** Generate a sparse point cloud and camera poses from the 360-degree images using OpenSfM [S1, unofficial]. OpenSfM is chosen because it supports equirectangular images directly, whereas Colmap does not [S1, unofficial].
2.  **Data Loading:**
    *   Read camera positions and parameters from `reconstruction.json` using a function analogous to `read_opensfm` [S1, unofficial].
    *   Read sparse point cloud positions from `reconstruction.json` using a function analogous to `read_opensfm_points3D` [S1, unofficial]. Initialize the `error` attribute to 0 [S1, unofficial].
3.  **Rasterization Modification:** Modify the differentiable Gaussian rasterization module (specifically `diff_gaussian_rasterization`) to support equirectangular rendering [S1, unofficial]. This involves adding a `render_spherical` function to the Python interface and updating the CUDA kernels (`forward.cu`, `backward.cu`, `auxiliary.h`) [S1, unofficial].
4.  **Projection:** Project 3D points to equirectangular coordinates [S1, unofficial]. In `auxiliary.h`, the device function `point_to_equirect` transforms a point $\mathbf{p}_{orig}$ using the view matrix $\mathbf{V}$:
    $$ \mathbf{d} = \mathbf{V} \cdot \mathbf{p}_{orig} $$
    $$ \text{length} = \|\mathbf{d}\| $$
    $$ \text{longitude} = \operatorname{atan2}(d_x, d_z) $$
    $$ \text{latitude} = \operatorname{atan2}(d_y, \sqrt{d_x^2 + d_z^2}) $$
    $$ u = \frac{\text{longitude}}{\pi} $$
    $$ v = \frac{\text{latitude}}{\pi/2} $$
    The resulting normalized coordinates $(u, v)$ map to the equirectangular image plane [S1, unofficial].
5.  **Training:** Optimize the Gaussian parameters to minimize the difference between rendered equirectangular images and input images [S1, unofficial]. To manage VRAM, training images are loaded onto the GPU only during the forward/backward pass, not kept resident in memory [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
import json

def point_to_equirect(p_orig: torch.Tensor, viewmatrix: torch.Tensor) -> torch.Tensor:
    """Project 3D points to equirectangular coordinates [S1, unofficial]."""
    # Transform point to camera space [G]
    p_homogeneous = torch.cat([p_orig, torch.ones(p_orig.shape[0], 1, device=p_orig.device)], dim=1)
    direction_vector = torch.matmul(p_homogeneous, viewmatrix.t())[:, :3]
    # Compute length and normalize [G]
    length = torch.sqrt(torch.sum(direction_vector ** 2, dim=1))
    length = torch.clamp(length, min=1e-6)
    direction_norm = direction_vector / length.unsqueeze(1)
    dx, dy, dz = direction_norm[:, 0], direction_norm[:, 1], direction_norm[:, 2]
    # Compute longitude and latitude [S1, unofficial]
    longitude = torch.atan2(dx, dz)
    latitude = torch.atan2(dy, torch.sqrt(dx**2 + dz**2))
    # Normalize to [-1, 1] range [S1, unofficial]
    u = longitude / np.pi
    v = latitude / (np.pi / 2.0)
    return torch.stack([u, v], dim=1)

def read_opensfm_points3d(reconstruction_path: str) -> torch.Tensor:
    """Read sparse point cloud positions from OpenSfM reconstruction.json [S1, unofficial]."""
    with open(reconstruction_path, 'r') as f:
        data = json.load(f)
    points_3d = [point_data['position'] for point_data in data['points3D'].values()]
    # Ignore 'error' value, set to 0 as per source [S1, unofficial]
    return torch.tensor(points_3d, dtype=torch.float32)

def read_opensfm_cameras(reconstruction_path: str) -> list:
    """Read camera positions and parameters from OpenSfM reconstruction.json [S1, unofficial]."""
    with open(reconstruction_path, 'r') as f:
        data = json.load(f)
    return list(data['cameras'].values())

def render_spherical(gaussians, cameras, image_width, image_height):
    """Modified rasterization function supporting equirectangular rendering [S1, unofficial].
    The source specifies adding this to the Python interface and updating CUDA kernels [S1, unofficial].
    [Comment] The core rasterization loop (sorting, projecting, blending) follows standard 3D Gaussian Splatting,
    but uses point_to_equirect for coordinate projection instead of perspective projection [S1, unofficial].
    """
    # Implementation invokes the modified CUDA kernels (forward.cu, backward.cu) with equirectangular projection [S1, unofficial].
    pass
```

## Operating characteristics
*   **VRAM Usage:** Training images are loaded onto the GPU only during the training step, not kept resident [S1, unofficial]. This reduces VRAM usage to approximately 11GB, allowing training on a 16GB GPU (e.g., NVIDIA T4 on Paperspace) with 5,000 images [S1, unofficial].
*   **Performance:** Training is slower compared to keeping images in VRAM, but feasible on consumer/prosumer hardware [S1, unofficial].
*   **Strengths:** Direct support for 360-degree equirectangular images without cropping; lower VRAM requirements than standard 24GB+ recommendations [S1, unofficial].
*   **Limitations:** Relies on OpenSfM, which may have different feature matching characteristics (e.g., using SuperPoint, Disk, or ALIKED with LightGlue) compared to Colmap [S1, unofficial].

## Background
360 Gaussian Splatting was developed to address the incompatibility of the original Gaussian Splatting method with 360-degree cameras [S1, unofficial]. Standard Gaussian Splatting relies on Colmap for SfM, which does not support equirectangular images directly, requiring users to crop images [S1, unofficial]. The method was presented in an unofficial blog post by inuex35, detailing modifications to the Gaussian Splatting repository to integrate OpenSfM and spherical rasterization [S1, unofficial].

## Sources
[S1] 360度カメラ画像からの空間再構成 - 360 Gaussian Splatting, https://qiita.com/inuex35/items/af720689e253cd3b617b, fetched 2026-10-29 (unofficial)
