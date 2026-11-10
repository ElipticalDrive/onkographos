# 3D Gaussian Splatting MVP

Also called: 3DGS MVP, pure PyTorch 3DGS

## Overview
3D Gaussian Splatting MVP is a self-contained, pure-PyTorch implementation of the core 3D Gaussian Splatting algorithm, designed as a teaching reference rather than a production renderer [S1, unofficial]. It renders a list of 3D Gaussians to a 2D buffer by projecting them to screen space, computing a 2D covariance matrix, and rasterizing the resulting ellipses, building on [3D Gaussian Splatting Rendering (Software)](2805-3d-gaussian-splatting-rendering-software.md). It distinguishes itself from production variants by intentionally omitting tile-based CUDA rasterization, spherical harmonics, adaptive density control, and SSIM losses, relying instead on a readable Python loop over Gaussians and standard PyTorch operations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ 3D Gaussians defined by their parameters, a pinhole camera model (look_at, OpenCV/COLMAP convention), and image dimensions $(H, W)$ [S1, unofficial].
*   **Outputs:** A rendered RGB image of size $(H, W, 3)$ and, during training, gradients for the Gaussian parameters [S1, unofficial].

**Data Structures:**
Each Gaussian $i$ is parameterized by the following fields [S1, unofficial]:
*   `mean`: 3D position vector $\mathbf{\mu}_i \in \mathbb{R}^3$.
*   `scale`: Log-space scale vector $\mathbf{s}_i \in \mathbb{R}^3$. The actual scale matrix $S$ is diagonal with entries $\exp(\mathbf{s}_i)$.
*   `rotation`: Unit quaternion $\mathbf{q}_i \in \mathbb{R}^4$ (normalized). Converted to a rotation matrix $R_i \in \mathbb{R}^{3 \times 3}$.
*   `color`: Per-Gaussian RGB color vector $\mathbf{c}_i \in \mathbb{R}^3$.
*   `opacity`: Scalar opacity $\alpha_i \in [0, 1]$.

**Formulas:**
1.  **Covariance Construction:** The 3D covariance matrix $\Sigma_i$ is constructed to be automatically symmetric positive semi-definite (PSD) [S1, unofficial]:
    $$ \Sigma_i = R_i S_i S_i^T R_i^T $$
    where $R_i$ is the rotation matrix derived from the unit quaternion and $S_i$ is the diagonal scale matrix.

2.  **Camera Space Transformation:** The mean $\mathbf{\mu}_i$ and covariance $\Sigma_i$ are transformed from world space to camera space using the view transformation matrix $W$ [S1, unofficial]:
    $$ \mathbf{\mu}_{i,cam} = W \mathbf{\mu}_i $$
    $$ \Sigma_{i,cam} = W \Sigma_i W^T $$

3.  **2D Screen-Space Projection:** The 3D covariance is projected to 2D screen space using the EWA / Zwicker-style projection Jacobian $J$ [S1, unofficial]:
    $$ \Sigma_{2d,i} = J W \Sigma_i W^T J^T $$
    A small isotropic dilation/blur is added to $\Sigma_{2d,i}$ for numerical stability [S1, unofficial].

4.  **Alpha Compositing:** Per-pixel front-to-back alpha compositing is performed using the volume-rendering equation with anisotropic 2D Gaussian weights [S1, unofficial]. For a pixel $p$, the color $C(p)$ is computed by iterating over Gaussians sorted by depth:
    $$ C(p) = \sum_{i \in \text{sorted}} T_i \alpha_i \mathbf{c}_i, \quad T_i = \prod_{j < i} (1 - \alpha_j) $$
    where $\alpha_i$ is the alpha value of the 2D Gaussian at pixel $p$, derived from the 2D covariance $\Sigma_{2d,i}$ and the distance from the pixel center to the Gaussian mean.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def render_gaussians_mvp(gaussians, camera, H, W):
    """
    Render 3D Gaussians using pure PyTorch.
    
    Args:
        gaussians: Dict with keys 'mean', 'scale', 'rotation', 'color', 'opacity'.
                   Shapes: mean (N,3), scale (N,3), rotation (N,4), color (N,3), opacity (N,).
        camera: Dict with 'W' (3x4 view matrix) and 'J' (2x3 projection Jacobian).
        H, W: Image height and width.
        
    Returns:
        rendered_image: (H, W, 3) tensor.
    """
    N = gaussians['mean'].shape[0]
    
    # 1. Transform means to camera space
    # Add homogeneous coordinate 1 to means
    means_h = torch.cat([gaussians['mean'], torch.ones(N, 1)], dim=1) # (N, 4)
    means_cam = (camera['W'] @ means_h.t()).t() # (N, 3)
    
    # 2. Transform covariances to camera space
    # Build rotation matrices from quaternions
    R = quaternion_to_rotation_matrix(gaussians['rotation']) # (N, 3, 3)
    
    # Build scale matrices from log-scale
    S = torch.diag_embed(torch.exp(gaussians['scale'])) # (N, 3, 3)
    
    # Sigma = R S S^T R^T
    Sigma_3d = R @ S @ S.transpose(1, 2) @ R.transpose(1, 2) # (N, 3, 3)
    
    # Transform 3D covariance to camera space: Sigma_cam = W Sigma W^T
    # Note: W is 3x4, Sigma is 3x3. We need to handle the homogeneous part correctly.
    # Usually, the covariance transformation uses the upper 3x3 of W if W is affine.
    # Assuming W is the 3x4 view matrix, we take the 3x3 rotation part for covariance transform.
    W_rot = camera['W'][:, :3] # (3, 3)
    Sigma_cam = W_rot @ Sigma_3d @ W_rot.transpose(1, 2) # (N, 3, 3)
    
    # 3. Project to 2D screen space
    # Sigma_2d = J W Sigma W^T J^T
    # J is 2x3, W is 3x4 (but we used W_rot which is 3x3)
    # So Sigma_2d = J @ W_rot @ Sigma_3d @ W_rot.T @ J.T
    Sigma_2d = camera['J'] @ Sigma_cam @ camera['J'].transpose(1, 2) # (N, 2, 2)
    
    # Add small isotropic dilation for stability
    Sigma_2d += torch.eye(2).unsqueeze(0) * 0.3 # (N, 2, 2)
    
    # 4. Depth Sort
    # Use argsort on z-coordinate (means_cam[:, 2])
    # Note: argsort is non-differentiable across order changes [S1, unofficial]
    depth_sort_indices = torch.argsort(means_cam[:, 2], dim=0) # (N,)
    
    # Gather sorted Gaussians
    sorted_means = means_cam[depth_sort_indices]
    sorted_Sigma_2d = Sigma_2d[depth_sort_indices]
    sorted_colors = gaussians['color'][depth_sort_indices]
    sorted_opacities = gaussians['opacity'][depth_sort_indices]
    
    # 5. Rasterize
    # Create grid of pixel centers
    y, x = torch.meshgrid(torch.arange(H), torch.arange(W), indexing='ij')
    pixel_centers = torch.stack([x.float(), y.float()], dim=-1) # (H, W, 2)
    
    # Initialize output
    rendered_image = torch.zeros(H, W, 3)
    transmittance = torch.ones(H, W)
    
    # Loop over Gaussians (for readability) [S1, unofficial]
    for i in range(N):
        mean_2d = sorted_means[i, :2] # (2,)
        cov_2d = sorted_Sigma_2d[i] # (2, 2)
        color = sorted_colors[i] # (3,)
        opacity = sorted_opacities[i] # ()
        
        # Compute 2D Gaussian weights at each pixel
        # dist = (p - mean)^T Sigma^{-1} (p - mean)
        diff = pixel_centers - mean_2d # (H, W, 2)
        # Inverse covariance
        inv_cov = torch.inverse(cov_2d) # (2, 2)
        # Mahalanobis distance
        dist = torch.sum(diff @ inv_cov * diff, dim=-1) # (H, W)
        
        # Gaussian weight
        weight = torch.exp(-0.5 * dist) # (H, W)
        
        # Alpha value
        alpha = weight * opacity # (H, W)
        
        # Front-to-back compositing
        # C_new = C_old * (1 - alpha) + color * alpha
        # T_new = T_old * (1 - alpha)
        
        # Vectorized update for all pixels
        rendered_image = rendered_image * (1 - alpha).unsqueeze(-1) + color.unsqueeze(0).unsqueeze(0) * alpha.unsqueeze(-1)
        transmittance = transmittance * (1 - alpha)
        
    return rendered_image

def quaternion_to_rotation_matrix(q):
    """Convert unit quaternions to rotation matrices."""
    # q: (N, 4) -> (w, x, y, z)
    w, x, y, z = q[:, 0], q[:, 1], q[:, 2], q[:, 3]
    N = q.shape[0]
    
    R = torch.zeros(N, 3, 3)
    
    R[:, 0, 0] = 1 - 2 * (y**2 + z**2)
    R[:, 0, 1] = 2 * (x*y - w*z)
    R[:, 0, 2] = 2 * (x*z + w*y)
    
    R[:, 1, 0] = 2 * (x*y + w*z)
    R[:, 1, 1] = 1 - 2 * (x**2 + z**2)
    R[:, 1, 2] = 2 * (y*z - w*x)
    
    R[:, 2, 0] = 2 * (x*z - w*y)
    R[:, 2, 1] = 2 * (y*z + w*x)
    R[:, 2, 2] = 1 - 2 * (x**2 + y**2)
    
    return R
```

**Parameters:**
*   **Camera Model:** Pinhole camera with `look_at` function using OpenCV/COLMAP convention [S1, unofficial].
*   **Isotropic Dilation:** A small constant (e.g., 0.3) is added to the diagonal of the 2D covariance matrix for stability [S1, unofficial].
*   **Optimization:** Adam optimizer is used for fitting Gaussians to training images [S1, unofficial].
*   **Loss:** Photometric L1 + MSE loss [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Depth Sort Non-Differentiability:** The depth sort uses `argsort`, which is non-differentiable across order changes. Gradients flow through values given the ordering, which is accepted for the fit-from-scratch demo [S1, unofficial].
*   **Performance:** The Python loop over Gaussians is slow. End-to-end run takes ~30 seconds on CPU for a 64x64 image [S1, unofficial].
*   **Missing Features:** This implementation intentionally skips tile-based CUDA rasterizer, spherical-harmonics view-dependent color, adaptive density control, real captures, and SSIM losses [S1, unofficial].

## Subtopics

### [3D Gaussian Splatting with Gaussian Unpooling](6133-3d-gaussian-splatting-with-gaussian-unpooling.md)
FSGS is a method for reconstructing 3D scenes from sparse-view inputs, addressing the challenge of insufficient initialization data for standard 3D Gaussian Splatting. It distinguishes itself from dense-view baselines by using Gaussian unpooling to expand a sparse set of Structure-from-Motion (SfM) points into a denser set of Gaussians, enabling faithful geometry reconstruction where standard methods fail due to lack of initial coverage. See [3D Gaussian Splatting with Gaussian Unpooling](6133-3d-gaussian-splatting-with-gaussian-unpooling.md).

### [Relightable 3DGS](7851-relightable-3dgs.md)
Relightable 3DGS extends standard 3D Gaussian Splatting by embedding physically-based material properties into each Gaussian to enable deferred shading. It solves the inability of standard rasterization to accurately represent specular reflections, glossy materials, and global illumination by decomposing outgoing radiance into a BRDF integration. Unlike standard 3DGS which relies on view-dependent spherical harmonics, this method supports relighting under novel illumination conditions while preserving real-time performance. See [Relightable 3DGS](7851-relightable-3dgs.md).

### [PRoGS](8773-progs.md)
PRoGS is a post-training technique for 3D Gaussian Splatting that approximates each Gaussian's contribution to the final scene to construct a prioritization order, enabling progressive rendering or streaming by dynamically selecting splats to display or transmit. Unlike existing web-viewers that require full scene retrieval before rendering, PRoGS loads a representative subset early, significantly reducing time-to-first-paint while maintaining perceptual quality. It is designed for on-device rendering applications constrained by memory and streaming applications requiring minimal bandwidth. See [PRoGS](8773-progs.md).

### [3D Gaussian Splatting Rendering Pipeline (RViz Plugin)](10297-3d-gaussian-splatting-rendering-pipeline-rviz-plugin.md)
This article describes the OpenGL-based rendering pipeline for visualizing 3D Gaussian Splatting data within the ROS RViz environment. It solves the problem of efficiently rendering millions of 2D-projected Gaussian splats as a single instanced draw call, handling depth sorting and view-dependent shading. It differs from pure software implementations by leveraging GPU-based sorting and texture buffer objects (TBOs) for data storage, and from other RViz plugins by supporting both exact back-to-front sorting and Order-Independent Transparency (WBOIT) approximations. See [3D Gaussian Splatting Rendering Pipeline (RViz Plugin)](10297-3d-gaussian-splatting-rendering-pipeline-rviz-plugin.md).

### 3D Gaussian Splatting with Vector Clustering and Entropy Encoding
A method that utilizes vector clustering to encode color and geometric attributes into two codebooks, using a sensitivity-aware KMeans method. For disk data storage, SASCGS utilizes the entropy encoding method DEFLATE, which combines the LZ77 algorithm and Huffman coding, to compress the data. On the MipNeRF360 dataset, SASCGS achieves SSIM 0.801, PSNR 26.98, LPIPS 0.238, and a size of 28.80 MB [S2].

### Vol3DGS
Enforces that alpha blending exactly satisfies the front-to-back absorption-emission model by deriving discrete compositing as an unbiased Monte Carlo estimator of the continuous integral. It introduces a continuous density field along the ray to resolve artifacts in scenes with semi-transparent objects. Volumetric alpha computation is defined as $\alpha_i = 1 - \exp(-\int G_i(x) dx)$, accumulating density along the ray in accordance with the volume rendering equation [S3, unofficial].

### HiFi4G
This successfully preserves spatial-temporal consistency in a more compact manner. It combines 3DGS with the non-rigid tracking offered by its dual-graph mechanism for high-fidelity rendering [S2].

### 3D Gaussian Splatting with Mesh UV Parameterization
This enables real-time and high-quality animated humans. It projects the 3D Gaussian learning into a 2D texture space using mesh UV parameterization to capture the appearance [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot H \cdot W)$ per iteration due to the Python loop over $N$ Gaussians for each pixel in the $H \times W$ image [S1, unofficial].
*   **Memory Complexity:** $O(N + H \cdot W)$ for storing Gaussian parameters and intermediate rasterization buffers [S1, unofficial].
*   **Measured Performance:** End-to-end run takes ~30 seconds on CPU for a 64x64 image [S1, unofficial].
*   **Strengths:** High readability, no custom CUDA kernels required, suitable for teaching and debugging the core algorithm [S1, unofficial].
*   **Limitations:** Very slow compared to GPU-accelerated implementations, lacks advanced features like spherical harmonics and adaptive densification [S1, unofficial].
*   **When to Use:** Educational purposes, debugging, or when custom CUDA kernels are not available.
*   **When Not to Use:** Production rendering, large-scale scenes, or real-time applications.

## Background
3D Gaussian Splatting MVP is based on the core algorithm of 3D Gaussian Splatting (Kerbl et al., SIGGRAPH 2023) [S1, unofficial]. It was implemented as a self-contained Jupyter notebook to serve as a teaching reference, allowing users to "grind on the math" without the complexity of custom CUDA kernels [S1, unofficial]. The implementation was contributed to the `diffuse-experiment` repository [S1, unofficial].

## Sources
[S1] Add 3D Gaussian Splatting MVP notebook, https://github.com/fangyuan-ksgk/diffuse-experiment/pull/3, fetched 2026-10-26 (unofficial)
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S3] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
