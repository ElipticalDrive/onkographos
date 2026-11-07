# 3D Gaussian Splatting MVP

Also called: 3DGS MVP, pure PyTorch 3DGS

## Overview
3D Gaussian Splatting MVP is a self-contained, pure-PyTorch implementation of the core 3D Gaussian Splatting algorithm [S1, unofficial]. It renders a list of 3D Gaussians to a 2D buffer by projecting them to screen space, computing a 2D covariance matrix, and rasterizing the resulting ellipses [S1, unofficial]. Unlike production renderers, it intentionally skips tile-based CUDA rasterization, spherical harmonics, and adaptive density control, serving instead as a teaching reference for the underlying mathematics [S1, unofficial].

## How it works
**Inputs and Outputs:**
* **Inputs:** A set of $N$ 3D Gaussians defined by parameters; a pinhole camera model (view and projection matrices); image dimensions $(H, W)$ [S1, unofficial].
* **Outputs:** A rendered RGB image of size $(H, W, 3)$ and an alpha mask of size $(H, W, 1)$ [G].

**Data Structures:**
Each Gaussian $i$ is parameterized by:
* `mean`: 3D position vector $\mathbf{\mu}_i \in \mathbb{R}^3$.
* `scale`: 3D scale vector $\mathbf{s}_i \in \mathbb{R}^3$ (stored in log-space).
* `rotation`: Unit quaternion $\mathbf{q}_i \in \mathbb{R}^4$.
* `color`: RGB color vector $\mathbf{c}_i \in \mathbb{R}^3$.
* `opacity`: Scalar $\alpha_i \in [0, 1]$ [S1, unofficial].

The 3D covariance matrix $\Sigma_i$ is constructed from scale and rotation to ensure it is symmetric positive semi-definite (PSD) [S1, unofficial]:
$$ \Sigma_i = R_i S_i S_i^T R_i^T $$
where $R_i$ is the rotation matrix derived from $\mathbf{q}_i$, and $S_i = \text{diag}(\exp(\mathbf{s}_i))$ is the scale matrix [S1, unofficial].

**Algorithm Steps:**
1. **World-to-Camera Transform:** Transform the mean $\mathbf{\mu}_i$ and the 3D covariance $\Sigma_i$ from world space to camera space using the view transformation matrix $W$ [S1, unofficial].
    $$ \mathbf{\mu}'_i = W \mathbf{\mu}_i $$
    $$ \Sigma'_i = W \Sigma_i W^T $$
2. **2D Projection:** Project the 3D covariance into the 2D screen space using the EWA / Zwicker-style projection Jacobian $J$ [S1, unofficial]:
    $$ \Sigma_{2d, i} = J W \Sigma_i W^T J^T $$
    A small isotropic dilation (blur) is added to $\Sigma_{2d, i}$ for numerical stability [S1, unofficial].
3. **Depth Sort:** Sort the Gaussians by their depth (z-coordinate in camera space) using `argsort` [S1, unofficial]. Note that `argsort` is non-differentiable across order changes; gradients flow through the values given the fixed ordering [S1, unofficial].
4. **Rasterization:** Iterate over each Gaussian in sorted order using a Python loop rather than vectorizing over all $(H, W, N)$ at once for readability [S1, unofficial].
5. **Alpha Compositing:** Perform per-pixel front-to-back alpha compositing using the volume-rendering equation with anisotropic 2D Gaussian weights [S1, unofficial]:
    $$ C_{u,v} = \sum_{i \in \text{sorted}} c_i \cdot \alpha_i \cdot w_{i, u, v} \cdot \prod_{j < i} (1 - \alpha_j \cdot w_{j, u, v}) $$
    $$ \alpha_{u,v} = \sum_{i \in \text{sorted}} \alpha_i \cdot w_{i, u, v} \cdot \prod_{j < i} (1 - \alpha_j \cdot w_{j, u, v}) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def render_gaussians_mvp(gaussians, camera, H, W):
    """
    gaussians: dict with keys 'mean', 'scale', 'rotation', 'color', 'opacity'
    camera: object with 'view_matrix', 'fx', 'fy', 'cx', 'cy'
    H, W: image height and width
    """
    N = gaussians['mean'].shape[0]
    means = gaussians['mean']
    scales = gaussians['scale']
    quats = gaussians['rotation']
    colors = gaussians['color']
    opacities = gaussians['opacity']
    
    # 1. World to Camera Transform
    W_view = camera.view_matrix  # (4, 4)
    
    # Transform means
    means_homogeneous = torch.cat([means, torch.ones(N, 1)], dim=1)  # (N, 4)
    means_cam = (W_view @ means_homogeneous.T).T[:, :3]  # (N, 3)
    
    # Transform Covariance: Sigma_3d = R S S^T R^T
    S = torch.exp(scales)  # (N, 3)
    R = quat_to_rot_matrix(quats)  # (N, 3, 3)
    Sigma_3d = R @ (S.unsqueeze(2) * S.unsqueeze(1)) @ R.transpose(1, 2)  # (N, 3, 3)
    
    Sigma_cam = W_view[:3, :3] @ Sigma_3d @ W_view[:3, :3].transpose(1, 2)  # (N, 3, 3)
    
    # 2. Project to 2D
    fx, fy = camera.fx, camera.fy
    cx, cy = camera.cx, camera.cy
    
    X, Y, Z = means_cam[:, 0], means_cam[:, 1], means_cam[:, 2]
    J = torch.zeros(N, 2, 3)
    J[:, 0, 0] = fx / Z
    J[:, 0, 2] = -fx * X / (Z**2)
    J[:, 1, 1] = fy / Z
    J[:, 1, 2] = -fy * Y / (Z**2)
    
    # Sigma_2d = J Sigma_cam J^T
    Sigma_2d = torch.bmm(torch.bmm(J, Sigma_cam), J.transpose(1, 2))  # (N, 2, 2)
    
    # Add isotropic dilation for stability [G]
    Sigma_2d[:, 0, 0] += 0.3
    Sigma_2d[:, 1, 1] += 0.3
    
    # 3. Depth Sort
    depths = means_cam[:, 2]
    sort_indices = torch.argsort(depths, dim=0)
    
    means_cam_sorted = means_cam[sort_indices]
    Sigma_2d_sorted = Sigma_2d[sort_indices]
    colors_sorted = colors[sort_indices]
    opacities_sorted = opacities[sort_indices]
    
    # 4. Rasterization
    out_color = torch.zeros(H, W, 3)
    out_alpha = torch.zeros(H, W, 1)
    
    for i in range(N):
        # Project mean to 2D screen space
        x_proj = fx * means_cam_sorted[i, 0] / means_cam_sorted[i, 2] + cx
        y_proj = fy * means_cam_sorted[i, 1] / means_cam_sorted[i, 2] + cy
        mu_2d = torch.tensor([x_proj, y_proj])
        
        cov_2d = Sigma_2d_sorted[i]
        color = colors_sorted[i]
        alpha = opacities_sorted[i]
        
        # Compute bounding box (3-sigma radius) [G]
        eigenvalues = torch.linalg.eigvalsh(cov_2d)
        radius = torch.ceil(torch.sqrt(torch.max(eigenvalues) * 3.0))
        
        x_min = max(0, int(x_proj - radius))
        x_max = min(W, int(x_proj + radius))
        y_min = max(0, int(y_proj - radius))
        y_max = min(H, int(y_proj + radius))
        
        # Create pixel grid for this bounding box
        ys, xs = torch.meshgrid(torch.arange(y_min, y_max), torch.arange(x_min, x_max), indexing='ij')
        pixel_coords = torch.stack([xs.float(), ys.float()], dim=-1)  # (H', W', 2)
        
        # Compute weights
        diff = pixel_coords - mu_2d  # (H', W', 2)
        inv_cov = torch.inverse(cov_2d)  # (2, 2)
        temp = torch.einsum('...k,kl->...l', diff, inv_cov)  # (H', W', 2)
        dist = torch.sum(temp * diff, dim=-1)  # (H', W')
        weights = torch.exp(-0.5 * dist)  # (H', W')
        
        # Alpha compositing (front-to-back)
        current_alpha = out_alpha[y_min:y_max, x_min:x_max]  # (H', W', 1)
        current_color = out_color[y_min:y_max, x_min:x_max]  # (H', W', 3)
        
        alpha_expanded = (alpha * weights).unsqueeze(-1)  # (H', W', 1)
        
        new_alpha = current_alpha + alpha_expanded * (1 - current_alpha)
        new_color = current_color * (1 - alpha_expanded) + color.unsqueeze(0).unsqueeze(0) * alpha_expanded
        
        out_alpha[y_min:y_max, x_min:x_max] = new_alpha
        out_color[y_min:y_max, x_min:x_max] = new_color
        
    return out_color, out_alpha

def quat_to_rot_matrix(quats):
    # Converts [w, x, y, z] quaternions to rotation matrices [G]
    w, x, y, z = quats[:, 0], quats[:, 1], quats[:, 2], quats[:, 3]
    R = torch.zeros(quats.shape[0], 3, 3)
    R[:, 0, 0] = 1 - 2*(y**2 + z**2)
    R[:, 0, 1] = 2*(x*y - w*z)
    R[:, 0, 2] = 2*(x*z + w*y)
    R[:, 1, 0] = 2*(x*y + w*z)
    R[:, 1, 1] = 1 - 2*(x**2 + z**2)
    R[:, 1, 2] = 2*(y*z - w*x)
    R[:, 2, 0] = 2*(x*z - w*y)
    R[:, 2, 1] = 2*(y*z + w*x)
    R[:, 2, 2] = 1 - 2*(x**2 + y**2)
    return R
```

**Parameters:**
* **Camera Model:** Pinhole camera with `look_at` function using OpenCV/COLMAP convention [S1, unofficial].
* **Isotropic Dilation:** Added to $\Sigma_{2d}$ for stability [S1, unofficial]. Value not specified in sources, but a small constant (e.g., 0.3) is used [G].

**Edge Cases and Pitfalls:**
* **Depth Sort:** Uses `argsort` which is non-differentiable across order changes [S1, unofficial]. Gradients flow through values given the ordering [S1, unofficial]. This is accepted for the fit-from-scratch demo [S1, unofficial].
* **Performance:** The rasterizer uses a Python loop over Gaussians rather than vectorizing over all $(H, W, N)$ at once for readability [S1, unofficial].

## Operating characteristics
* **Time Complexity:** Not given in the sources [S1, unofficial].
* **Memory Complexity:** Not given in the sources [S1, unofficial].
* **Measured Performance:** End-to-end run takes ~30 seconds on CPU for a 64×64 image [S1, unofficial].
* **Strengths:** Pure PyTorch implementation, no custom CUDA kernels, easy to read and modify, suitable for teaching and debugging [S1, unofficial].
* **Limitations:** Slow due to Python loops, lacks optimizations like tile-based rasterization, spherical harmonics, and adaptive density control [S1, unofficial].
* **When to Use:** For educational purposes, prototyping, or when CUDA is unavailable [G].
* **When Not to Use:** For production rendering, large scenes, or real-time applications [G].

## Background
3D Gaussian Splatting MVP is based on the core algorithm described in "3D Gaussian Splatting for Real-Time Radiance Field Rendering" by Kerbl et al. (SIGGRAPH 2023) [S1, unofficial]. The MVP implementation is a self-contained Jupyter notebook intended as a teaching reference to allow users to implement the math without the complexity of custom CUDA kernels [S1, unofficial]. It is distributed as a standalone tutorial material within a larger experiment repository [S1, unofficial].

## Sources
[S1] Add 3D Gaussian Splatting MVP notebook, https://github.com/fangyuan-ksgk/diffuse-experiment/pull/3, fetched 2026-10-26 (unofficial)
