# 3D Gaussian Splatting MVP

Also called: 3DGS MVP, pure PyTorch 3DGS

## Overview
3D Gaussian Splatting MVP is a self-contained, pure-PyTorch implementation of the core 3D Gaussian Splatting rendering algorithm, designed as a teaching reference rather than a production renderer [S1, unofficial]. It solves the problem of understanding the mathematical pipeline of projecting 3D Gaussians to 2D screen space and rasterizing them via alpha compositing, without relying on custom CUDA kernels or optimized tile-based rasterization [S1, unofficial]. It differs from production implementations by using explicit Python loops for rasterization and omitting spherical harmonics, adaptive density control, and hardware-accelerated sorting [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ 3D Gaussians defined by their parameters; a pinhole camera model defined by a view matrix $W$ and projection parameters; image dimensions $(H, W)$.
*   **Outputs:** A rendered RGB image of size $(H, W, 3)$ and an alpha mask of size $(H, W, 1)$.

**Data Structures:**
Each Gaussian $i$ is stored with the following fields [S1, unofficial]:
*   `mean`: 3D vector $(x, y, z)$ in world space.
*   `scale`: 3D vector $(s_x, s_y, s_z)$ stored in log-space.
*   `rotation`: Unit quaternion $(w, x, y, z)$.
*   `opacity`: Scalar $\alpha_i \in [0, 1]$.
*   `color`: 3D vector $(r, g, b)$.

**Formulas:**
1.  **Covariance Construction:** The 3D covariance matrix $\Sigma$ is constructed from scale $S$ and rotation $R$ to ensure it is symmetric positive semi-definite (PSD) [S1, unofficial]:
    $$ \Sigma = R S S^T R^T $$
    where $S = \text{diag}(e^{s_x}, e^{s_y}, e^{s_z})$ and $R$ is the rotation matrix derived from the unit quaternion.

2.  **2D Projection:** The 3D covariance $\Sigma$ is transformed into the camera frame and projected to 2D screen space using the Jacobian $J$ of the partial perspective projection [S1, unofficial]:
    $$ \Sigma_{2d} = J W \Sigma W^T J^T $$
    where $W$ is the view transformation matrix. A small isotropic dilation (blur) is added to $\Sigma_{2d}$ for numerical stability [S1, unofficial].

3.  **Rasterization:** Per-pixel front-to-back alpha compositing is performed using the volume-rendering equation with anisotropic 2D Gaussian weights [S1, unofficial].

**Steps:**
1.  **Camera Transform:** Transform Gaussian means from world space to camera space using the view matrix $W$ [S1, unofficial].
2.  **Covariance Transform:** Transform the 3D covariance matrix $\Sigma$ into the camera frame [S1, unofficial].
3.  **Projection:** Compute the 2D screen-space covariance $\Sigma_{2d}$ using the Jacobian $J$ and apply isotropic dilation [S1, unofficial].
4.  **Sorting:** Sort Gaussians by their depth (z-coordinate in camera space) for each pixel or globally. Note: `argsort` is used, which is non-differentiable across order changes, but gradients flow through the values given the fixed ordering [S1, unofficial].
5.  **Compositing:** Iterate over pixels and Gaussians to compute the final color and alpha using front-to-back blending [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def render_gaussians_mvp(gaussians, camera, image_h, image_w):
    """
    Pure PyTorch implementation of 3D Gaussian Splatting MVP.
    
    Args:
        gaussians: Dict with keys 'mean', 'scale', 'rotation', 'opacity', 'color'.
                   Shapes: mean (N,3), scale (N,3), rotation (N,4), opacity (N,1), color (N,3).
        camera: Dict with 'W' (view matrix, 4x4), 'J' (projection Jacobian, 2x3 or 3x3 depending on convention).
        image_h, image_w: Integer dimensions.
        
    Returns:
        rendered_image: Tensor (H, W, 3)
        rendered_alpha: Tensor (H, W, 1)
    """
    N = gaussians['mean'].shape[0]
    
    # 1. Transform means to camera space
    # Homogeneous coordinates for mean
    means_homo = torch.cat([gaussians['mean'], torch.ones(N, 1)], dim=1) # (N, 4)
    means_cam = (camera['W'] @ means_homo.t()).t()[:, :3] # (N, 3)
    
    # 2. Construct 3D Covariance Sigma
    # R from quaternion
    R = quaternion_to_rotation_matrix(gaussians['rotation']) # (N, 3, 3)
    # S from log-scale
    S = torch.diag_embed(torch.exp(gaussians['scale'])) # (N, 3, 3)
    # Sigma = R S S^T R^T
    Sigma_3d = R @ S @ S.transpose(1, 2) @ R.transpose(1, 2) # (N, 3, 3)
    
    # 3. Project Covariance to 2D
    # Sigma_2d = J W Sigma W^T J^T
    # Note: J is typically the Jacobian of the perspective projection.
    # For simplicity, assuming J is pre-computed or constant for the camera.
    # W is the view matrix (3x3 or 4x4 part relevant to rotation/translation).
    # Let's assume camera['W'] is 4x4. We need the 3x3 rotation part for coordinate transform.
    W_view = camera['W'][:3, :3]
    t_view = camera['W'][:3, 3]
    
    # Transform Sigma to camera frame
    Sigma_cam = W_view @ Sigma_3d @ W_view.transpose(1, 2)
    
    # Jacobian J for perspective projection
    # Standard EWA projection Jacobian involves depth.
    # J = d(x,y,z)_cam / d(x,y,z)_world ? No, J is d(screen) / d(camera).
    # Usually J = [ fx/z 0 -fx*x/z^2 ; 0 fy/z -fy*y/z^2 ]
    # For MVP, we might use a simplified constant J or compute per-gaussian.
    # The source says "J is the EWA / Zwicker-style projection Jacobian".
    # We will compute it per Gaussian based on its camera-space position.
    
    fx = camera.get('fx', 1.0)
    fy = camera.get('fy', 1.0)
    
    Sigma_2d_list = []
    for i in range(N):
        z = means_cam[i, 2]
        x = means_cam[i, 0]
        y = means_cam[i, 1]
        
        # Jacobian J_i (2x3)
        J_i = torch.tensor([
            [fx/z, 0, -fx*x/(z*z)],
            [0, fy/z, -fy*y/(z*z)]
        ], dtype=torch.float32)
        
        # Sigma_2d = J_i @ Sigma_cam[i] @ J_i^T
        s2d = J_i @ Sigma_cam[i] @ J_i.transpose(0, 1)
        
        # Add isotropic dilation for stability
        s2d[0, 0] += 0.3 # Example value for stability
        s2d[1, 1] += 0.3
        
        Sigma_2d_list.append(s2d)
        
    Sigma_2d = torch.stack(Sigma_2d_list) # (N, 2, 2)
    
    # 4. Depth Sort
    # Sort Gaussians by z-depth (camera space)
    depths = means_cam[:, 2] # (N,)
    sort_indices = torch.argsort(depths) # (N,)
    
    # Reorder all attributes
    sorted_means = means_cam[sort_indices]
    sorted_Sigma_2d = Sigma_2d[sort_indices]
    sorted_opacity = gaussians['opacity'][sort_indices]
    sorted_color = gaussians['color'][sort_indices]
    
    # 5. Rasterization Loop
    # Initialize output buffers
    final_color = torch.zeros((image_h, image_w, 3), dtype=torch.float32)
    final_alpha = torch.zeros((image_h, image_w, 1), dtype=torch.float32)
    
    # Precompute inverse covariance for Gaussian evaluation
    # Sigma_2d_inv = inv(Sigma_2d)
    Sigma_2d_inv = torch.inverse(Sigma_2d) # (N, 2, 2)
    
    # Create grid of pixel coordinates
    y_coords, x_coords = torch.meshgrid(torch.arange(image_h), torch.arange(image_w), indexing='ij')
    coords = torch.stack([x_coords.float(), y_coords.float()], dim=-1) # (H, W, 2)
    
    # For each Gaussian, update pixels it covers
    # MVP uses a Python loop over Gaussians for readability [S1, unofficial]
    for i in range(N):
        mu = sorted_means[i] # (3,) -> we only need x, y for screen
        mu_screen = mu[:2] # (2,)
        
        # Compute 2D Gaussian weight for all pixels
        # diff = coords - mu_screen # (H, W, 2)
        # diff = diff.unsqueeze(0) # (1, H, W, 2)
        # mu_screen_expanded = mu_screen.unsqueeze(0).unsqueeze(0) # (1, 1, 2)
        # diff = coords - mu_screen_expanded # (H, W, 2)
        
        # More efficient:
        diff = coords - mu_screen # Broadcasting works if shapes align? 
        # coords is (H, W, 2), mu_screen is (2,). 
        # We need to broadcast mu_screen to (1, 1, 2)
        mu_exp = mu_screen.view(1, 1, 2)
        diff = coords - mu_exp # (H, W, 2)
        
        # Mahalanobis distance: -0.5 * diff^T Sigma^-1 diff
        # diff is (H, W, 2), Sigma_inv is (2, 2)
        # temp = diff @ Sigma_inv[i] # (H, W, 2)
        # dist = torch.sum(temp * diff, dim=-1) # (H, W)
        
        # Vectorized over H, W
        # Sigma_inv[i] is (2, 2)
        # We can use einsum or matmul
        # dist = -0.5 * torch.sum((diff @ Sigma_2d_inv[i]) * diff, dim=-1)
        
        # Let's do it explicitly for clarity
        inv_cov = Sigma_2d_inv[i] # (2, 2)
        # (H, W, 2) @ (2, 2) -> (H, W, 2)
        temp = torch.einsum('hwk,kj->hwj', diff, inv_cov)
        dist = -0.5 * torch.sum(temp * diff, dim=-1) # (H, W)
        
        # Clamp negative distances to avoid exp(nan) or large values
        dist = torch.clamp(dist, max=0.0)
        
        # Weight = exp(dist)
        weight = torch.exp(dist) # (H, W)
        
        # Determine bounding box to avoid computing for all pixels?
        # MVP skips optimization, so we compute for all pixels.
        
        # Alpha contribution
        alpha_i = sorted_opacity[i] # Scalar
        color_i = sorted_color[i] # (3,)
        
        # Front-to-back compositing
        # C = sum(T * alpha_i * color_i)
        # T = product(1 - alpha_i)
        
        # We need to accumulate in order.
        # Since we loop over Gaussians in depth order, we can update the buffer.
        
        # Current alpha at pixel (h, w)
        # alpha_out = alpha_i * weight * (1 - current_alpha)
        # color_out += alpha_i * weight * color_i * (1 - current_alpha)
        # current_alpha += alpha_i * weight
        
        # However, standard alpha compositing is:
        # C_out = C_in + (1 - C_alpha_in) * alpha_i * weight * color_i
        # A_out = A_in + (1 - A_in) * alpha_i * weight
        
        # Let's apply this to the whole image
        current_alpha = final_alpha # (H, W, 1)
        current_color = final_color # (H, W, 3)
        
        # Broadcast alpha_i and color_i
        alpha_i_exp = alpha_i.view(1, 1, 1)
        color_i_exp = color_i.view(1, 1, 3)
        weight_exp = weight.view(image_h, image_w, 1)
        
        contribution_alpha = alpha_i_exp * weight_exp
        contribution_color = color_i_exp * weight_exp
        
        # Update
        final_alpha = current_alpha + (1 - current_alpha) * contribution_alpha
        final_color = current_color + (1 - current_alpha) * contribution_color
        
    return final_color, final_alpha

def quaternion_to_rotation_matrix(q):
    """Convert unit quaternion (w, x, y, z) to rotation matrix (3, 3)."""
    w, x, y, z = q[:, 0], q[:, 1], q[:, 2], q[:, 3]
    N = w*w + x*x + y*y + z*z
    w, x, y, z = w/N, x/N, y/N, z/N # Normalize
    
    xx, yy, zz = x*x, y*y, z*z
    xy, xz, yz = x*y, x*z, y*z
    wx, wy, wz = w*x, w*y, w*z
    
    # Rotation matrix
    R = torch.stack([
        1 - 2*(yy + zz), 2*(xy - wz), 2*(xz + wy),
        2*(xy + wz), 1 - 2*(xx + zz), 2*(yz - wx),
        2*(xz - wy), 2*(yz + wx), 1 - 2*(xx + yy)
    ], dim=-1).view(-1, 3, 3)
    
    return R
```

**Parameters:**
*   **Camera Model:** Pinhole camera with `look_at` function using OpenCV/COLMAP convention [S1, unofficial].
*   **Optimization:** Adam optimizer with L1 + MSE loss for fitting Gaussians to images [S1, unofficial].
*   **Iterations:** 600 iterations for the demo [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Depth Sort:** The use of `argsort` for depth sorting is non-differentiable across order changes. Gradients flow through the values given the ordering, which is accepted for the fit-from-scratch demo [S1, unofficial].
*   **Stability:** An isotropic dilation/blur is added to the 2D covariance matrix to prevent numerical instability during inversion [S1, unofficial].
*   **Performance:** The implementation uses a Python loop over Gaussians, making it slow for large N. It is intended for small scenes (e.g., 40 Gaussians) [S1, unofficial].

## Subtopics

### [3D Gaussian Splatting with Gaussian Unpooling](6133-3d-gaussian-splatting-with-gaussian-unpooling.md)
FSGS is a method for reconstructing 3D scenes from sparse-view inputs, addressing the challenge of insufficient initialization data for standard 3D Gaussian Splatting. It distinguishes itself from dense-view baselines by using Gaussian unpooling to expand a sparse set of Structure-from-Motion (SfM) points into a denser set of Gaussians, enabling faithful geometry reconstruction where standard methods fail due to lack of initial coverage.

### PRoGS
A post-training technique that approximates each Gaussian's contribution to the final scene to construct a prioritization order, enabling progressive rendering or streaming by dynamically selecting splats to display or transmit.

### Relightable 3DGS
Embeds physically-based material properties into each Gaussian for deferred shading, decomposing outgoing radiance into BRDF integration. It traces secondary rays from each Gaussian to the light to compute direct illumination from a point source, evaluating a shadow map. The specular BRDF is computed using the microfacet Cook-Torrance model: $f_s(\omega_0, \omega_i) = \frac{D(h; r) \cdot F(\omega_0, h) \cdot G(\omega_i, \omega_0, h; r)}{(n \cdot \omega_i) \cdot (n \cdot \omega_0)}$, where $h=(\omega_i+\omega_0)/2$ is the half vector, $D$ is the normal distribution function, $F$ is the Fresnel term, $G$ is the geometry term, $n$ is the normal, and $r$ is roughness. The PBR color is given by $c'(\omega_0) = \sum_{i=0}^{N_s} (f_d + f_s(\omega_0, \omega_i)) L_i(\omega_i) (\omega_i \cdot n) \Delta \omega_i$, where $f_d$ is the diffuse component, $L_i$ is incident radiance, and $\Delta \omega_i$ is the solid angle [S2, unofficial].

### 3D Gaussian Splatting with Vector Clustering and Entropy Encoding
SASCGS utilizes vector clustering to encode color and geometric attributes into two codebooks, using a sensitivity-aware KMeans method. For disk data storage, it utilizes the entropy encoding method DEFLATE, which combines the LZ77 algorithm and Huffman coding, to compress the data. On the MipNeRF360 dataset, SASCGS achieves SSIM 0.801, PSNR 26.98, LPIPS 0.238, and a size of 28.80 MB [S3].

### Vol3DGS
Enforces that alpha blending exactly satisfies the front-to-back absorption-emission model by deriving discrete compositing as an unbiased Monte Carlo estimator of the continuous integral. It introduces a continuous density field along the ray to resolve artifacts in scenes with semi-transparent objects. Volumetric alpha computation is defined as $\alpha_i = 1 - \exp(-\int G_i(x) dx)$, accumulating density along the ray in accordance with the volume rendering equation [S2, unofficial].

### HiFi4G
Successfully preserves spatial-temporal consistency in a more compact manner by combining 3DGS with the non-rigid tracking offered by its dual-graph mechanism for high-fidelity rendering [S3].

### 3D Gaussian Splatting with Mesh UV Parameterization
Enables real-time and high-quality animated humans by projecting the 3D Gaussian learning into a 2D texture space using mesh UV parameterization to capture the appearance [S3].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot H \cdot W)$ for the rasterization step, where $N$ is the number of Gaussians and $H, W$ are image dimensions. The MVP implementation uses explicit Python loops, resulting in very slow performance compared to GPU-optimized tile-based rasterizers.
*   **Memory Complexity:** $O(N + H \cdot W)$ for storing Gaussian attributes and intermediate buffers.
*   **Measured Performance:** End-to-end run takes ~30 seconds on CPU for a 64x64 image with 40 Gaussians [S1, unofficial].
*   **Strengths:** High readability, no custom CUDA kernels required, suitable for educational purposes and debugging the mathematical pipeline.
*   **Limitations:** Extremely slow for large scenes, lacks optimizations like tile-based rasterization, spherical harmonics for view-dependent color, adaptive density control, and SSIM losses [S1, unofficial].
*   **When to use:** For learning the core mathematics of 3D Gaussian Splatting, debugging, or small-scale demonstrations.
*   **When not to use:** For production rendering, large-scale scenes, or real-time applications.

## Background
3D Gaussian Splatting MVP is based on the core algorithm of 3D Gaussian Splatting (Kerbl et al., SIGGRAPH 2023) [S1, unofficial]. The MVP implementation was created as a self-contained Jupyter notebook to serve as a teaching reference, allowing users to "grind on the math" without the complexity of custom CUDA kernels [S1, unofficial]. It was developed as part of the `diffuse-experiment` repository [S1, unofficial].

## Sources
[S1] Add 3D Gaussian Splatting MVP notebook, https://github.com/fangyuan-ksgk/diffuse-experiment/pull/3, fetched 2026-10-26 (unofficial)
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S3] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
