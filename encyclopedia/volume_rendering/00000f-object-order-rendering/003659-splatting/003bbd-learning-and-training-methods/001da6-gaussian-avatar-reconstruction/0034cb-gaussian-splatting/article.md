# Gaussian Splatting

Also called: 3D Gaussian Splatting (3DGS), Vanilla 3DGS rendering

## Overview
Gaussian Splatting (3DGS) is a novel-view synthesis method that represents a scene as an explicit collection of anisotropic 3D Gaussians, each carrying opacity and view-dependent color encoded by spherical harmonics [S5, unofficial]. It solves the trade-off between the high fidelity of implicit neural radiance fields (NeRFs) and the speed of point-based rendering by using a tile-based differentiable rasterizer to project these Gaussians onto the image plane, enabling real-time rendering (≥30 fps at 1080p) and fast training [S5, unofficial]. Unlike NeRFs, which require ray marching and neural network evaluations per pixel, 3DGS uses volume splatting to avoid costly queries in empty space, achieving high detail through adaptive density control that clones, splits, and prunes primitives based on image error [S1].

## How it works
**Inputs:** A set of posed input images $\{I_i\}$ and their corresponding camera poses (extrinsics $W$ and intrinsics $K$).
**Outputs:** A rendered image $I_f$ and gradients for optimizing Gaussian parameters.

**Data Structures:**
The scene is represented by a set of $M$ 3D Gaussians. Each Gaussian $g_i$ is defined by the following parameters [S1][S12, unofficial]:
*   **Center:** $\boldsymbol{\mu}_i \in \mathbb{R}^3$ (3D position).
*   **Covariance:** $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$ (shape and orientation). To ensure positive semi-definiteness during optimization, $\boldsymbol{\Sigma}_i$ is parameterized via factorization [S5, unofficial]:
    $$ \boldsymbol{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^\top \mathbf{R}_i^\top $$
    where $\mathbf{R}_i$ is a rotation matrix derived from a normalized quaternion $\mathbf{q}_i \in \mathbb{R}^4$ ($q_r^2 + q_i^2 + q_j^2 + q_k^2 = 1$), and $\mathbf{S}_i$ is a diagonal scaling matrix constructed from a scale vector $\mathbf{s}_i \in \mathbb{R}^3$ [S5, unofficial][S8]. The diagonal entries of $\mathbf{S}_i$ are optimized directly as $\mathbf{s}_i$ [S12, unofficial].
*   **Opacity:** $\sigma_i \in \mathbb{R}$ (learned peak opacity, often stored as a logit) [S4, unofficial][S12, unofficial].
*   **Color:** View-dependent color is encoded using spherical harmonics (SH) coefficients $\mathbf{c}_i$. The INRIA 3DGS format typically stores up to band 3, resulting in 48 coefficients per splat for RGB (16 per channel) [S4, unofficial]. The color for a viewing direction $\mathbf{d}$ is computed as $c_i = \text{SH}(\mathbf{c}_i, \mathbf{d})$ [S7, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Initialize the Gaussian centers $\boldsymbol{\mu}_i$ and colors $\mathbf{c}_i$ from a sparse point cloud obtained via Structure-from-Motion (SfM) [S1][S5, unofficial]. Random initialization is avoided as it introduces artifacts and "floaters" [S5, unofficial].
2.  **3D-to-2D Projection (Splatting):** For each Gaussian $i$, project it onto the 2D image plane [S8].
    *   Transform the center to camera coordinates: $\boldsymbol{\mu}_i^{cam} = \mathbf{W} \boldsymbol{\mu}_i$ [S12, unofficial].
    *   Approximate the perspective projection $\phi_{\mathbf{K}}$ using a first-order Taylor expansion (Jacobian $\mathbf{J} \in \mathbb{R}^{2 \times 3}$) around the center [S1][S12, unofficial]:
        $$ \tilde{\boldsymbol{\mu}}_i = \phi_{\mathbf{K}}(\mathbf{W} \boldsymbol{\mu}_i) $$
        $$ \tilde{\boldsymbol{\Sigma}}_i = \mathbf{J} \mathbf{W} \boldsymbol{\Sigma}_i \mathbf{W}^\top \mathbf{J}^\top $$
        where $\tilde{\boldsymbol{\mu}}_i$ and $\tilde{\boldsymbol{\Sigma}}_i$ are the 2D mean and covariance in screen space [S1][S12, unofficial].
    *   The 2D opacity and color are preserved: $\tilde{\sigma}_i = \sigma_i$ and $\tilde{c}_i = c_i$ [S1].
3.  **Tile-Based Sorting:** Partition the image into tiles (typically $16 \times 16$ pixels) [S8]. Assign Gaussians intersecting the view frustum to overlapping tiles. Sort the Gaussians within each tile by depth (z-coordinate of the center) using a GPU-optimized radix sort [S8][S15].
4.  **Alpha Compositing (Rasterization):** For each pixel, traverse the sorted list of Gaussians in front-to-back order and accumulate color using alpha blending [S8][S15]. The color $C_{\mathbf{x}}$ at pixel $\mathbf{x}$ is [S9][S12, unofficial]:
    $$ C_{\mathbf{x}} = \sum_{i=1}^{N} c_i \alpha_{(\mathbf{x}, i)} T_{(\mathbf{x}, i)} $$
    where $T_{(\mathbf{x}, i)} = \prod_{j=1}^{i-1} (1 - \alpha_{(\mathbf{x}, j)})$ is the transmittance, and $\alpha_{(\mathbf{x}, i)}$ is the effective opacity of the $i$-th Gaussian at $\mathbf{x}$ [S9].
    The effective opacity is computed as [S14, unofficial]:
    $$ \alpha_{(\mathbf{x}, i)} = \sigma_i \exp\left(-\frac{1}{2} (\mathbf{x} - \tilde{\boldsymbol{\mu}}_i)^\top \tilde{\boldsymbol{\Sigma}}_i^{-1} (\mathbf{x} - \tilde{\boldsymbol{\mu}}_i)\right) $$
    In implementation, the power is often calculated using the conic matrix $\mathbf{C} = \tilde{\boldsymbol{\Sigma}}_i^{-1}$:
    $$ \text{power} = -0.5 (C_{00} x^2 + C_{11} y^2) - C_{01} xy $$
    where $(x, y)$ are the local coordinates relative to the Gaussian center [S11, unofficial]. The alpha is clamped to a maximum of 0.99 to prevent full opacity blocking [S11, unofficial].
5.  **Optimization:** Compare the rendered image to the input view using a photometric loss $\mathcal{L}$, typically a combination of L1 and D-SSIM [S6, unofficial][S15]. Backpropagate gradients to update $\boldsymbol{\mu}_i, \mathbf{q}_i, \mathbf{s}_i, \sigma_i, \mathbf{c}_i$ using an optimizer like Adam [S12, unofficial].
6.  **Adaptive Density Control:** Every 100 iterations, adjust the number of Gaussians [S15]:
    *   **Clone:** Duplicate Gaussians in regions with high image error (underfitting).
    *   **Split:** Split large Gaussians that cover too much detail.
    *   **Prune:** Remove Gaussians with persistently low opacity [S1][S15].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def render_gaussians(gaussians, camera, image_shape):
    """
    Render a scene of 3D Gaussians to a 2D image.
    
    Args:
        gaussians: List of dicts with keys 'mu' (3,), 'q' (4,), 's' (3,), 
                   'opacity' (float), 'sh_coeffs' (48,).
        camera: Dict with 'W' (4x4 extrinsic), 'K' (3x3 intrinsic).
        image_shape: Tuple (height, width).
        
    Returns:
        image: (height, width, 3) RGB image.
    """
    height, width = image_shape
    tile_size = 16
    tiles_h = (height + tile_size - 1) // tile_size
    tiles_w = (width + tile_size - 1) // tile_size
    
    # Initialize output buffers
    # Using float32 for accumulation, final clamp to [0,1]
    acc_color = np.zeros((height, width, 3), dtype=np.float32)
    acc_alpha = np.zeros((height, width), dtype=np.float32)
    
    # Precompute camera matrices
    W = camera['W']
    K = camera['K']
    
    # Process each Gaussian
    for g in gaussians:
        mu_3d = g['mu']
        q = g['q']
        s = g['s']
        sigma_peak = g['opacity']
        sh_coeffs = g['sh_coeffs']
        
        # 1. Construct 3D Covariance Sigma
        R = quaternion_to_rotation_matrix(q)
        S = np.diag(s)
        Sigma_3d = R @ S @ S.T @ R.T
        
        # 2. Transform to Camera Space
        # Homogeneous coordinates
        mu_cam_h = W @ np.append(mu_3d, 1.0)
        mu_cam = mu_cam_h[:3] / mu_cam_h[3]
        
        # Jacobian of projection
        # P(x) = K * [x/z, y/z, 1]^T
        # J = d(P(x))/dx approximated at mu_cam
        J = compute_projection_jacobian(mu_cam, K)
        
        # 3. Project Covariance to 2D
        # Sigma_2d = J * W * Sigma_3d * W^T * J^T
        # Note: W in formula refers to the rotation part of extrinsic if mu is already transformed?
        # Standard derivation: Sigma_2d = J * R_cam * Sigma_3d * R_cam^T * J^T
        # Where R_cam is the rotation part of W. 
        # However, if we transform mu to camera space first, we must transform Sigma similarly.
        # Sigma_cam = R_cam @ Sigma_3d @ R_cam.T
        R_cam = W[:3, :3]
        Sigma_cam = R_cam @ Sigma_3d @ R_cam.T
        
        Sigma_2d = J @ Sigma_cam @ J.T
        
        # 4. Compute 2D Mean
        mu_2d_h = K @ (R_cam @ mu_3d + W[:3, 3]) # Or simply K @ mu_cam_h[:3] if mu_cam is correct
        # Using the linearized projection formula from sources:
        # mu_2d = phi_K(W * mu_3d)
        # phi_K is perspective projection.
        mu_2d = project_to_screen(mu_cam, K)
        
        # 5. Compute Conic Matrix (Inverse of Sigma_2d)
        # Add small epsilon for stability
        Sigma_2d += np.eye(2) * 1e-6
        Conic = np.linalg.inv(Sigma_2d)
        
        # 6. Compute Bounding Box
        # 3-sigma bounding box
        eigenvalues = np.linalg.eigvalsh(Sigma_2d)
        std_devs = np.sqrt(eigenvalues)
        bbox_size = 3.0 * std_devs
        
        # Convert to pixel coordinates
        # Approximate: scale by focal length? 
        # The sources mention normalizing to NDC. 
        # For simplicity in this pseudocode, we assume bbox_size is in pixels.
        # In practice, bbox_size needs to be scaled by the camera's focal length in pixels.
        # Let's assume a helper `sigma_to_pixel_scale` exists.
        bbox_px = sigma_to_pixel_scale(bbox_size, K)
        
        min_x = max(0, int(mu_2d[0] - bbox_px[0]))
        max_x = min(width, int(mu_2d[0] + bbox_px[0]))
        min_y = max(0, int(mu_2d[1] - bbox_px[1]))
        max_y = min(height, int(mu_2d[1] + bbox_px[1]))
        
        # 7. Rasterize within Bounding Box
        for y in range(min_y, max_y):
            for x in range(min_x, max_x):
                # Local coordinates relative to center
                dx = x - mu_2d[0]
                dy = y - mu_2d[1]
                
                # Power calculation
                power = -0.5 * (Conic[0,0]*dx*dx + Conic[1,1]*dy*dy + 2*Conic[0,1]*dx*dy)
                
                if power > 0.0:
                    continue # Outside effective area
                
                # Alpha
                alpha = sigma_peak * np.exp(power)
                alpha = min(alpha, 0.99) # Clamp
                
                if alpha < 1.0/255.0:
                    continue # Discard transparent fragments
                
                # Color from SH
                # View direction from camera to Gaussian center
                view_dir = mu_3d - camera['origin'] # Assuming camera origin known
                view_dir = view_dir / np.linalg.norm(view_dir)
                color = sh_evaluate(sh_coeffs, view_dir)
                
                # Alpha Blending
                # C_new = (1 - alpha) * C_old + alpha * color
                # But we need to track transmittance for correct order-independent blending?
                # Standard 3DGS sorts front-to-back and blends.
                # T_old = 1 - acc_alpha[y, x]
                # C_new = T_old * acc_color[y, x] + alpha * color
                # acc_alpha[y, x] += alpha * T_old
                
                T_old = 1.0 - acc_alpha[y, x]
                acc_color[y, x] = T_old * acc_color[y, x] + alpha * color
                acc_alpha[y, x] += alpha * T_old

    # Apply background color
    bg_color = np.array([1.0, 1.0, 1.0])
    final_image = acc_color + (1.0 - acc_alpha[:, :, np.newaxis]) * bg_color
    
    return np.clip(final_image, 0.0, 1.0)

def quaternion_to_rotation_matrix(q):
    qr, qi, qj, qk = q
    return np.array([
        [1 - 2*(qj**2 + qk**2), 2*(qi*qj - qr*qk), 2*(qi*qk + qr*qj)],
        [2*(qi*qj + qr*qk), 1 - 2*(qi**2 + qk**2), 2*(qj*qk - qr*qi)],
        [2*(qi*qk - qr*qj), 2*(qj*qk + qr*qi), 1 - 2*(qi**2 + qj**2)]
    ])

def compute_projection_jacobian(mu_cam, K):
    x, y, z = mu_cam
    if z <= 0:
        return np.zeros((2, 3))
    fx = K[0, 0]
    fy = K[1, 1]
    cx = K[0, 2]
    cy = K[1, 2]
    
    # J = d(u,v)/d(x,y,z)
    # u = fx * x/z + cx
    # v = fy * y/z + cy
    J = np.zeros((2, 3))
    J[0, 0] = fx / z
    J[0, 2] = -fx * x / (z**2)
    J[1, 1] = fy / z
    J[1, 2] = -fy * y / (z**2)
    return J

def project_to_screen(mu_cam, K):
    x, y, z = mu_cam
    if z <= 0:
        return np.array([0, 0])
    u = K[0, 0] * (x / z) + K[0, 2]
    v = K[1, 1] * (y / z) + K[1, 2]
    return np.array([u, v])

def sh_evaluate(coeffs, view_dir):
    # Simplified Band-0 evaluation
    # C0 = 0.28209479177387814
    C0 = 0.28209479177387814
    f_dc = coeffs[:3]
    color = np.clip(0.5 + C0 * f_dc, 0, 1)
    return color

def sigma_to_pixel_scale(sigma_sigma, K):
    # Approximate scaling from world units to pixels
    # Depends on depth and focal length
    # This is a placeholder for the exact conversion used in the rasterizer
    return sigma_sigma * K[0, 0] # Simplified
```

## Subtopics

### [Unnormalized 3D Gaussian Function](1026-unnormalized-3d-gaussian-function.md)
Parameterizes the Gaussian via its peak value (opacity) rather than its integral, making the amplitude the intensity at the center.

### [Row-constant hoisting for Gaussian Splatting Rasterization](1195-row-constant-hoisting-for-gaussian-splatting-rasterization.md)
Optimizes the inner pixel loop by hoisting row-constant terms outside the loop to minimize arithmetic operations.

### [2D Gaussian Projection with Covariance Stabilization](1198-2d-gaussian-projection-with-covariance-stabilization.md)
Uses a local affine transformation (Jacobian) to project 3D Gaussians to 2D ellipses, preserving the Gaussian form and enabling efficient rasterization.

### [Gaussian in the Wild (GS-W)](1386-gaussian-in-the-wild-gs-w.md)
Separates intrinsic material properties from dynamic environmental factors to handle unconstrained image collections without global latent embeddings.

### [3DGUT](1474-3dgut.md)
Uses an unscented transform to improve the affine approximation of the projection operator, reducing covariance approximation errors.

### [Transformer-based Reconstruction](1794-transformer-based-reconstruction.md)
Predicts 3DGS parameters directly from 2D images using a feed-forward network, bypassing iterative optimization.

### [3D Gaussian Ellipsoid Rendering](1869-3d-gaussian-ellipsoid-rendering.md)
Replaces rasterization with ray tracing to compute exact volume density distributions, solving occlusion ambiguity.

### [Stochastic Gaussian Splatting](6599-stochastic-gaussian-splatting.md)
Eliminates sorting bottlenecks by using stochastic sampling and atomic operations to render hundreds of millions of Gaussians.

### [Rendering Implementation](7155-rendering-implementation.md)
Defines the core differentiable kernel that projects 3D Gaussians to 2D ellipses and composites them via alpha blending.

### [Gsplat Transformation](7308-gsplat-transformation.md)
Provides semantic field access and partial update capabilities for the `Gsplat` data structure during optimization.

## Operating characteristics
**Time Complexity:** Rendering is $O(M)$ where $M$ is the number of Gaussians, but effectively $O(N \cdot K)$ per pixel where $N$ is the number of tiles and $K$ is the average number of Gaussians per tile. Sorting is $O(M \log M)$ per frame [S8].
**Memory Complexity:** $O(M)$ to store Gaussian parameters. Each Gaussian stores 3 (pos) + 3 (scale) + 4 (quat) + 1 (opacity) + 48 (SH) = 59 values [S15]. Typical scenes use 1–5 million Gaussians [S15].
**Performance:** Achieves real-time rendering (≥30 fps at 1080p) on consumer hardware [S5, unofficial]. On the Mip-NeRF360 benchmark, it reaches 27.21 dB PSNR and 0.214 LPIPS at 134 fps [S5, unofficial].
**Strengths:** Real-time rendering speed, high visual fidelity, explicit representation allows for easy editing and downstream tasks like surface reconstruction [S1].
**Limitations:** Large file sizes due to explicit storage [S15]. Temporal artifacts (flickering/popping) due to per-pixel depth sorting [S14, unofficial]. Struggles with highly reflective surfaces due to the emission-absorption model [S12, unofficial]. Moving objects can cause "flying artifacts" if not masked during training [S15].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S1][S5, unofficial]. It builds on EWA Volume Splatting [S2][S5, unofficial], which itself is based on Heckbert's thesis and traditional signal processing [S2]. The method was developed to address the computational cost of NeRFs, which require ray marching and neural network evaluations, by combining the volumetric image-formation model of radiance fields with the speed of splatting rasterization [S5, unofficial]. It has become a standard for real-time novel-view synthesis and 3D scene reconstruction.

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
[S3] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S4] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
[S5] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26 (unofficial)
[S6] Quantile Rendering: Efficiently Embedding High-dimensional Feature on 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2512.20927, fetched 2026-10-26 (unofficial)
[S7] WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU, https://doi.org/10.48550/arxiv.2602.03207, fetched 2026-10-26 (unofficial)
[S8] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S9] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S10] UltraGauss: Ultrafast Gaussian Reconstruction of 3D Ultrasound Volumes, https://doi.org/10.48550/arXiv.2505.05643, fetched 2026-10-27 (unofficial)
[S11] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S12] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S13] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
[S14] SparseOIT: Improving Order-Independent Transparency 3DGS via Active Set Method, https://doi.org/10.48550/arXiv.2605.13855, fetched 2026-10-29 (unofficial)
[S15] Gaussian Splatting - 高斯泼溅：实时3D 渲染, https://www.ultralytics.com/zh/glossary/gaussian-splatting, fetched 2026-10-30
