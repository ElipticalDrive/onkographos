# DN-Splatter

Also called: DN-Splatter Regularization and Initialization, Depth and Normal Priors for Gaussian Splatting

## Overview
DN-Splatter is a regularization and initialization strategy for 3D Gaussian Splatting (3DGS) that incorporates depth and normal priors to improve photorealism and geometric accuracy in challenging indoor scenes, enabling direct mesh extraction [S2]. It distinguishes itself from standard 3DGS by using an edge-aware depth loss and normal alignment to resolve ambiguities in textureless regions, whereas its sibling [3D Gaussian Splatting with Normal Information for Mesh Extraction](12069-3d-gaussian-splatting-with-normal-information-for-mesh-extra.md) focuses primarily on normal extraction without the specific adaptive depth regularization and sensor-based initialization detailed here [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training images (RGB), optional sensor depth maps (e.g., from LiDAR or structured light), optional monocular depth/normal predictions, and camera poses [S2].
*   **Outputs:** Optimized 3D Gaussian primitives (means, covariances, opacities, colors) and a reconstructed mesh via Poisson surface reconstruction [S2].

**Data Structures:**
The method extends the standard 3DGS primitive representation [S2]. Each Gaussian $i$ is parameterized by:
*   Mean position $\mu_i \in \mathbb{R}^3$ [S2]
*   Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$, decomposed into scaling vector $s_i \in \mathbb{R}^3$ and rotation quaternion $q_i \in \mathbb{R}^4$ [S2]
*   Opacity $\alpha_i \in \mathbb{R}$ [S2]
*   Color $c_i \in \mathbb{R}^3$ (or spherical harmonics coefficients) [S2]

**Initialization:**
The Gaussian scene is initialized using either sparse COLMAP SfM points or dense sensor depth readings back-projected into 3D space [S2]. Sensor depth initialization is preferred as it outperforms COLMAP initialization in both mesh quality and novel-view synthesis [S2]. The number of initialized Gaussians is typically 1 million when using sensor depth [S2].

**Rendering Depth and Normals:**
Depth $\hat{D}$ is rendered per-pixel using a discrete volume rendering approximation [S2]:
$$ \hat{D} = \sum_{i \in N} d_i \alpha_i \prod_{j=1}^{i-1} (1-\alpha_j) $$
where $d_i$ is the $i$-th Gaussian's z-depth coordinate in view space, and $N$ is the set of Gaussians sorted by depth [S2]. Since 3DGS uses a single global sort rather than per-pixel sorting, this is an approximation [S2]. To handle pixels where accumulated transmittance $T_i \neq 1$, depth estimates are normalized by the final accumulated transmittance [S2]. Color and depths are rasterized simultaneously in a single CUDA kernel forward pass to improve speed [S2]. Normal directions are extracted from the Gaussians (derived from the covariance matrix) and rendered into normal maps for regularization [S2].

**Loss Functions:**
The total loss combines the standard RGB photometric loss with geometric regularization terms [S2].

1.  **Gradient-Aware Depth Loss ($L_{\hat{D}}$):**
    This loss regularizes the rendered depth $\hat{D}$ against ground-truth sensor depth $D_{\text{gt}}$ [S2]. It uses a logarithmic penalty for smoother reconstructions compared to linear or quadratic penalties [S2]. The loss is weighted by the image gradient to be adaptive:
    $$ L_{\hat{D}} = \sum_{p} w_p \| \log(\hat{D}_p) - \log(D_{\text{gt}, p}) \| $$
    where the weight $w_p$ is lower in regions with large RGB image gradients (edges) and higher in smooth, textureless regions [S2]. This mitigates uncertainties at object boundaries caused by raw depth sensor inaccuracies [S2].

2.  **Normal Loss ($L_{\hat{N}}$):**
    This loss aligns the rendered normal map $\hat{N}$ with estimated normal cues (from monocular networks or sensor data) [S2]. It improves mesh normal-completeness and F-scores [S2]. The specific formulation aligns Gaussians along scene geometry [S2].

3.  **Normal Smoothing Prior ($L_{\text{smooth}}$):**
    An optional prior applied to the rendered normal maps to ensure local smoothness, improving normal-completeness for Poisson meshing with minimal impact on other metrics [S2].

**Optimization:**
The Gaussian scene is optimized for 30,000 iterations [S2]. The optimization minimizes the combined loss, refining positions, opacities, and colors to satisfy both photometric and geometric constraints [S2].

**Mesh Extraction:**
After optimization, meshes are extracted directly from the Gaussian scene using Poisson surface reconstruction [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dn_splatter_optimization(
    images: List[np.ndarray], 
    depths: List[np.ndarray], 
    poses: List[np.ndarray],
    num_gaussians: int = 1_000_000
) -> List[Gaussian]:
    # 1. Initialization
    if sensor_depths_available:
        gaussians = initialize_from_sensor_depth(depths, poses, num_gaussians)
    else:
        gaussians = initialize_from_sfm(points, poses, num_gaussians)
        
    # 2. Optimization Loop
    for iteration in range(30_000):
        view_idx = sample_random_view()
        rgb_gt = images[view_idx]
        depth_gt = depths[view_idx]
        pose = poses[view_idx]
        
        # Forward Pass: Render Color, Depth, and Normal simultaneously
        # Single CUDA kernel for efficiency
        rgb_rendered, depth_rendered, normal_rendered, transmittance = render_gaussians(
            gaussians, pose
        )
        
        # Compute RGB Loss (Standard 3DGS photometric loss)
        loss_rgb = compute_rgb_loss(rgb_rendered, rgb_gt)
        
        # Compute Gradient-Aware Depth Loss
        img_grad = compute_gradient_magnitude(rgb_rendered)
        
        # Normalize depth by accumulated transmittance T to handle gaps
        depth_rendered_normalized = depth_rendered / (transmittance + 1e-8)
        
        # Logarithmic difference
        depth_diff = torch.log(depth_rendered_normalized) - torch.log(depth_gt)
        
        # Adaptive weighting: lower weight at high image gradients
        weight_map = 1.0 / (1.0 + img_grad)
        
        loss_depth = torch.sum(weight_map * torch.abs(depth_diff))
        
        # Compute Normal Loss (if normals are used)
        normal_gt = estimate_normals_monocular(rgb_rendered)
        loss_normal = compute_normal_alignment_loss(normal_rendered, normal_gt)
        
        # Compute Normal Smoothing Prior (optional)
        loss_smooth = compute_normal_smoothing_loss(normal_rendered)
        
        # Total Loss
        total_loss = loss_rgb + lambda_d * loss_depth + lambda_n * loss_normal + lambda_s * loss_smooth
        
        # Backpropagate and update Gaussians
        update_gaussians(gaussians, total_loss)
        
    return gaussians

def render_gaussians(gaussians: List[Gaussian], pose: np.ndarray):
    # Project 3D Gaussians to 2D screen space
    projected = project_gaussians(gaussians, pose)
    # Sort Gaussians globally by depth (not per-pixel)
    sorted_gaussians = sort_gaussians_by_depth(projected)
    
    # Rasterize
    height, width = projected[0].screen_bounds.shape
    color = np.zeros((height, width, 3))
    depth = np.zeros((height, width))
    normal = np.zeros((height, width, 3))
    transmittance = np.ones((height, width))
    
    for gaussian in sorted_gaussians:
        if gaussian.intersects_pixel():
            alpha = gaussian.opacity * gaussian.contribution_to_pixel
            color += gaussian.color * alpha * transmittance
            depth += gaussian.z_depth * alpha * transmittance
            normal += gaussian.normal * alpha * transmittance
            transmittance *= (1 - alpha)
            
    # Normalize depth by final transmittance to handle gaps
    depth = depth / (transmittance + 1e-8)
    
    return color, depth, normal, transmittance
```

## Operating characteristics
*   **Time Complexity:** Training takes approximately 17.9 minutes on an Nvidia 4090 GPU for 30,000 iterations on MuSHRoom scenes [S2]. The simultaneous rasterization of color and depth improves inference and training speed compared to separate steps [S2].
*   **Memory Complexity:** Uses 1 million Gaussians for sensor depth initialization [S2].
*   **Performance:** On the MuSHRoom left test set, DN-Splatter achieves Abs Rel 0.0228, Sq Rel 0.0089, RMSE 0.1280, $\delta<1.25$ 0.9854, PSNR 24.58, and SSIM 0.8558 [S2]. Mesh evaluation on ScanNet++ shows a Chamfer Distance (C-L1) of 0.0395 and Normal Consistency (NC) of 0.8316 [S2].
*   **Strengths:** Significantly improves reconstruction in ambiguous, textureless regions; enables direct mesh extraction with high normal completeness; sensor depth initialization outperforms COLMAP SfM [S2].
*   **Limitations:** Relies on the availability of sensor depth or accurate monocular estimates; sensor depths may have low resolution and inaccuracies at edges [S2]. The global sort approximation for depth rendering is not exact [S2].
*   **When to use:** High-fidelity 3D reconstruction of indoor scenes where both photorealism and geometric accuracy (for meshing) are required, especially in textureless areas [G].
*   **When not to use:** Scenes without textureless regions where standard 3DGS suffices, or when sensor depth data is unavailable and monocular estimates are unreliable [G].

## Background
DN-Splatter was introduced by Matias Turkulainen, Xuqian Ren, Iaroslav Melekhov, Otto Seiskari, Esa Rahtu, and Juho Kannala in the paper "DN-Splatter: Depth and Normal Priors for Gaussian Splatting and Meshing" (arXiv:2403.17822, WACV 2025) [S2]. It addresses the challenge that state-of-the-art methods often fail to achieve good results in both photorealism and geometry reconstruction for everyday indoor scenes [S2]. The method builds on 3D Gaussian Splatting [19] and leverages readily accessible geometric cues (sensor depth, monocular normals) to regularize optimization [S2].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] DN-Splatter: Depth and Normal Priors for Gaussian Splatting and Meshing, https://doi.org/10.1109/wacv61041.2025.00241, fetched 2026-10-26
