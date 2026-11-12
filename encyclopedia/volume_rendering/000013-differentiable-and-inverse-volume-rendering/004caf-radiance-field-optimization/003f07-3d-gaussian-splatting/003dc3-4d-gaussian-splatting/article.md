# 4D Gaussian Splatting

Also called: 4D Gaussian Representation, 4D-GS, Dynamic Scene Gaussian Splatting

## Overview
4D Gaussian Splatting (4DGS) extends 3D Gaussian Splatting to model dynamic scenes by treating space and time as a unified 4D continuum, enabling real-time rendering of arbitrary time slices without per-frame optimization [S7, unofficial]. Unlike methods that treat time as an independent dimension or rely on MLP-based deformation fields, 4DGS uses native 4D Gaussian primitives with coherent spatiotemporal covariance, allowing for smooth interpolation of geometry and appearance between discrete keyframes [S1][S7, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Synchronized multi-view images or monocular video with known camera poses and timestamps $t \in [0, T]$ [S7, unofficial].
- **Outputs:** A set of 4D Gaussians $\{G_i\}_{i=1}^N$ and their associated parameters, enabling the rendering of an image $\mathcal{I}(u, v, t)$ at any spatial coordinate $(u, v)$ and time $t$ [S7, unofficial].

**Data Structures:**
Each 4D Gaussian $i$ is defined by the following parameters:
- **Mean:** $\boldsymbol{\mu}_i = (\mu_{x}, \mu_{y}, \mu_{z}, \mu_{t}) \in \mathbb{R}^4$ [S7, unofficial].
- **Covariance:** $\boldsymbol{\Sigma}_i \in \mathbb{R}^{4 \times 4}$, parameterized as $\boldsymbol{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^T \mathbf{R}_i^T$ [S7, unofficial].
  - **Scaling:** $\mathbf{S}_i = \text{diag}(s_{x}, s_{y}, s_{z}, s_{t})$, where $s_{\cdot} > 0$ [S7, unofficial].
  - **Rotation:** $\mathbf{R}_i \in \mathbb{R}^{4 \times 4}$ is constructed from two unit quaternions $\mathbf{q}_l = (a, b, c, d)$ and $\mathbf{q}_r = (p, q, r, s)$ representing left and right isoclinic rotations: $\mathbf{R}_i = L(\mathbf{q}_l)R(\mathbf{q}_r)$, where $L(\mathbf{q}_l)$ and $R(\mathbf{q}_r)$ are $4 \times 4$ matrices derived from the quaternion components [S7, unofficial].
- **Opacity:** $\alpha_i \in [0, 1]$ [S7, unofficial].
- **Color:** Represented by 4D Spherindrical Harmonics (4DSH) coefficients. The basis function is $Z_{nl}^m(t, \theta, \phi) = \cos\left(\frac{2\pi n}{T}t\right) Y_l^m(\theta, \phi)$, where $Y_l^m$ are 3D spherical harmonics, $n$ is the Fourier order, and $l, m$ are SH degree and order [S7, unofficial].

**Rendering Equation:**
The color at pixel $(u, v)$ and time $t$ is computed by sorting Gaussians by depth and accumulating opacity and color [S7, unofficial]:
$$ \mathcal{I}(u, v, t) = \sum_{i=1}^N p_i(t) p_i(u, v|t) \alpha_i c_i(d) \prod_{j=1}^{i-1} (1 - p_j(t) p_j(u, v|t) \alpha_j) $$
where:
- $p_i(t) = \mathcal{N}(t; \mu_{t,i}, \Sigma_{4,4,i})$ is the marginal probability of the Gaussian at time $t$ [S7, unofficial].
- $p_i(u, v|t)$ is the conditional 2D Gaussian probability density at $(u, v)$ given time $t$ [S7, unofficial].
- $c_i(d)$ is the view-dependent color evaluated using the 4DSH coefficients and viewing direction $d$ [S7, unofficial].

**Conditional 3D Gaussian Derivation:**
To render a specific time $t$, the 4D Gaussian is sliced into a 3D Gaussian [S3]. The conditional mean $\boldsymbol{\mu}_{3D|t}$ and covariance $\boldsymbol{\Sigma}_{3D|t}$ are derived from the properties of multivariate Gaussians [S7, unofficial]:
$$ \boldsymbol{\mu}_{3D|t} = \boldsymbol{\mu}_{1:3} + \boldsymbol{\Sigma}_{1:3,4} \boldsymbol{\Sigma}_{4,4}^{-1} (t - \mu_{t}) $$
$$ \boldsymbol{\Sigma}_{3D|t} = \boldsymbol{\Sigma}_{1:3,1:3} - \boldsymbol{\Sigma}_{1:3,4} \boldsymbol{\Sigma}_{4,4}^{-1} \boldsymbol{\Sigma}_{4,1:3} $$
where $\boldsymbol{\mu}_{1:3}$ is the spatial mean, $\boldsymbol{\Sigma}_{1:3,1:3}$ is the spatial covariance, and $\boldsymbol{\Sigma}_{1:3,4}$ is the cross-covariance between spatial and temporal dimensions [S7, unofficial]. The 2D projection covariance is then computed from $\boldsymbol{\Sigma}_{3D|t}$ using the standard 3DGS projection pipeline [S7, unofficial].

**Training and Optimization:**
1. **Initialization:**
   - For Plenoptic Video datasets: Initialize Gaussians using a colored point cloud from COLMAP on the first frame; distribute timestamps uniformly across the scene duration [S7, unofficial].
   - For D-NeRF datasets: Initialize 100,000 random points in volume $[-1.2, 1.2]^3$; set initial mean time to the scene's duration [S7, unofficial].
   - **Initial Time Scaling:** Set initial temporal scale $s_t$ to half of the total scene duration to leverage Gaussian anisotropy and prevent excessive Gaussian proliferation [S7, unofficial].
2. **Forward Pass:**
   - For each training iteration, sample a batch of images with random timestamps [S7, unofficial].
   - Compute conditional 3D Gaussians for each sampled time $t$ [S7, unofficial].
   - Render images using the extended rasterizer that accounts for $p_i(t)$ [S7, unofficial].
3. **Loss and Backpropagation:**
   - Compute rendering loss (e.g., L1 + D-SSIM) between rendered and ground truth images [S7, unofficial].
   - Backpropagate gradients through the conditional Gaussian formulas to update $\boldsymbol{\mu}, \boldsymbol{\Sigma}, \alpha$, and 4DSH coefficients [S7, unofficial].
4. **Density Control (Densification):**
   - Use average gradients of $\boldsymbol{\mu}_{t}$ as a density control indicator [S7, unofficial].
   - In over-reconstruction regions, perform Gaussian splitting using joint spatial and temporal position sampling [S7, unofficial].
   - Prune Gaussians with low opacity or large spatial/temporal extent [S7, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def render_4dgs(gaussians, camera, time_t):
    """
    Render a single frame at time_t using 4D Gaussians.
    gaussians: dict with keys 'mu' (N,4), 'cov' (N,4,4), 'alpha' (N,), 'sh_coeffs' (N, K)
    camera: dict with keys 'K' (3,3), 'R_w2c' (3,3), 't_w2c' (3,)
    time_t: float, current timestamp
    """
    N = gaussians['mu'].shape[0]
    H, W = camera['image_height'], camera['image_width']
    
    # 1. Compute conditional 3D Gaussian parameters
    mu_spatial = gaussians['mu'][:, :3]
    mu_temporal = gaussians['mu'][:, 3]
    
    Sigma_spatial = gaussians['cov'][:, :3, :3]
    Sigma_cross = gaussians['cov'][:, :3, 3]
    Sigma_temporal = gaussians['cov'][:, 3, 3]
    
    delta_t = time_t - mu_temporal
    conditional_mu = mu_spatial + (Sigma_cross * delta_t[:, None]) / Sigma_temporal[:, None]
    
    correction = (Sigma_cross[:, :, None] * Sigma_cross[:, None, :]) / Sigma_temporal[:, None, None]
    conditional_cov = Sigma_spatial - correction
    
    # 2. Project to 2D (Standard 3DGS projection)
    # Transform mean to camera space
    cam_mu = (camera['R_w2c'] @ conditional_mu.T).T + camera['t_w2c']
    depth = cam_mu[:, 2]
    
    # Project mean
    proj_mu = (camera['K'] @ cam_mu.T).T
    proj_mu[:, 0] /= proj_mu[:, 2]
    proj_mu[:, 1] /= proj_mu[:, 2]
    
    # Project covariance (Jacobian approximation)
    # Standard 3DGS projection: J = K @ R_w2c, cov_2d = J @ cov_3d @ J.T
    J = camera['K'] @ camera['R_w2c']
    proj_cov = np.einsum('ij,njk,kl->nil', J, conditional_cov, J.T)
    
    # 3. Compute Marginal Probability p(t)
    # p(t) = N(t; mu_t, Sigma_4,4)
    marginal_prob = 1.0 / math.sqrt(2 * math.pi * Sigma_temporal) * np.exp(-delta_t**2 / (2 * Sigma_temporal))
    
    # 4. Rasterize
    sorted_indices = np.argsort(depth)
    
    color = np.zeros((H, W, 3))
    alpha_accum = np.zeros((H, W))
    
    for i in sorted_indices:
        p_2d = 1.0 / (2 * math.pi * np.sqrt(np.clip(np.linalg.det(proj_cov[i]), 1e-6, None)))
        diff = np.array([proj_mu[i, 0] - np.arange(W), proj_mu[i, 1] - np.arange(H)])
        # Simplified 2D Gaussian PDF evaluation per pixel
        p_2d_grid = np.zeros((H, W))
        for y in range(H):
            for x in range(W):
                d = np.array([x - proj_mu[i, 0], y - proj_mu[i, 1]])
                cov_inv = np.linalg.inv(proj_cov[i])
                p_2d_grid[y, x] = np.exp(-0.5 * d @ cov_inv @ d)
        p_2d_grid *= p_2d
        
        weight = marginal_prob[i] * p_2d_grid * gaussians['alpha'][i]
        
        # View direction and 4DSH evaluation
        view_dir = np.zeros((H, W, 3))
        for y in range(H):
            for x in range(W):
                view_dir[y, x] = np.array([x - proj_mu[i, 0], y - proj_mu[i, 1], 1.0])
                view_dir[y, x] /= np.linalg.norm(view_dir[y, x])
                
        color_i = evaluate_4dsh(gaussians['sh_coeffs'][i], view_dir, time_t)
        
        color += color_i * weight
        alpha_accum += weight
        
        if np.all(alpha_accum > 0.99):
            break
            
    return color

def evaluate_4dsh(sh_coeffs, view_dir, time_t):
    """Evaluate 4DSH coefficients for given view direction and time."""
    # Standard 3D SH evaluation combined with temporal Fourier basis
    # Z_{nl}^m(t, theta, phi) = cos(2*pi*n/T * t) * Y_l^m(theta, phi)
    # Simplified: return SH(view_dir) * temporal_basis(time_t)
    return np.sum(sh_coeffs * np.cos(2 * np.pi * np.arange(sh_coeffs.shape[1]) / 100.0 * time_t), axis=1)
```

## Subtopics

### [Spacetime Gaussians (STG)](5997-spacetime-gaussians-stg.md)
Spacetime Gaussians (STG) is a 4D Gaussian Splatting method that models dynamic scenes using native 4D primitives, enabling real-time rendering of any arbitrary time slice without per-frame optimization. It distinguishes itself from MLP-based deformation approaches like DGS by explicitly parameterizing temporal opacity envelopes and polynomial trajectories for each Gaussian, allowing for smooth interpolation of geometry and appearance between discrete keyframes.

### Gaussian-splat timelapses
A time-lapse rendering architecture where each timepoint's splats are fitted independently and stored contiguously, with time acting as a hard ordering barrier to prevent LOD rung blending across timepoints [S8, unofficial]. It fits Gaussian splats per timepoint (e.g., 256K per frame) and streams timepoints sequentially, fetching chunks holding the new timepoint as the timeline advances [S8, unofficial]. Time-dependent elements like lineage tracks are rendered by clipping polylines and fading splats in/out based on their extent along the hidden temporal dimension [S8, unofficial].

### Urban Scene Adaptation for 4DGS
Training and rendering modifications for sparse, wide-baseline outdoor scenes to prevent overfitting and model infinite-distance backgrounds [S7, unofficial]. It integrates sparse depth supervision from LiDAR point clouds using an inverse L1 loss, adopts a cube map as the background model for the sky, and penalizes inverse depth values in sky-masked regions [S7, unofficial]. The temporal coefficient of 4DSH is deactivated (set to zero) during training to reduce overfitting to sparse observations [S7, unofficial]. Unlike indoor methods, it provides a unified representation of dynamic and static regions without explicit object masks [S7, unofficial].

### No-4DRot Variant
A variant that disables 4D rotation updates, resulting in less temporal variance and impaired motion fitting [S7, unofficial]. Compared to the Full variant, No-4DRot engages fewer Gaussians after filtering, reconstructs object rims poorly, and produces more flicker and blur due to reduced temporal variance and impaired inter-frame information exchange [S7, unofficial].

### Initial Time Scaling
Adjusts the temporal variance of initialized 4D Gaussians to match scene duration, preventing excessive Gaussian proliferation over long videos [S7, unofficial]. The initial time scaling is set to half of the total scene duration to leverage Gaussian anisotropy and avoid intractable Gaussian counts as video length increases [S7, unofficial].

### 4D-Rotor Gaussian Splatting
A method using 4D rotor-based primitives defined via Clifford algebra [S5, unofficial].

### Real-time 4DGS
A method that achieves interactive frame rates by building a specialized renderer that fuses static background Gaussians with a small, fast-updating set of dynamic foreground Gaussians [S9, unofficial].

### RetimeGS
A method that achieves continuous-time 4D reconstruction by modeling the trajectories with a neural ODE solver, allowing frame rates to be arbitrarily adjusted at test time [S9, unofficial].

## Operating characteristics
- **Time Complexity:** Rendering is $O(N)$ per pixel, similar to 3DGS, but requires computing conditional Gaussians for each time step [S7, unofficial]. Training involves backpropagation through the conditional Gaussian formulas, which adds constant overhead per iteration [S7, unofficial].
- **Memory Complexity:** Stores 4D parameters per Gaussian (mean, covariance, 4DSH coefficients) [S7, unofficial]. The number of Gaussians is comparable to 3DGS for static scenes but may increase for highly dynamic content if not properly regularized [S7, unofficial].
- **Measured Performance:** On the D-NeRF synthetic dataset (800x800), 4DGS achieves PSNR 31.79, SSIM 0.960, and LPIPS 0.051, with training time reduced from 18m09s to 6m22s (2.8x speedup) and VRAM from 1.9GiB to 1.2GiB compared to the reference implementation [S4, unofficial]. On the Plenoptic Video dataset, it achieves PSNR 32.01, DSSIM 0.014, LPIPS 0.055, and 114 FPS [S7, unofficial].
- **Strengths:** Real-time rendering of arbitrary time slices; coherent spatiotemporal modeling; efficient training with end-to-end optimization; no per-frame optimization required [S7, unofficial].
- **Limitations:** Struggles with highly complex, non-rigid deformations if the Gaussian count is insufficient; temporal flickering can occur in scenes with dramatic changes if batch sampling is not used [S7, unofficial].
- **When to use:** Dynamic scene novel view synthesis with synchronized multi-view cameras; applications requiring real-time exploration of time-varying content [S7, unofficial].
- **When not to use:** Scenes with extremely sparse temporal observations where interpolation is unreliable; scenes with infinite-distance backgrounds without specific adaptations (e.g., Urban Scene Adaptation) [S7, unofficial].

## Background
4D Gaussian Splatting was introduced by Yang et al. in CVPR'24 [S7, unofficial]. It extends the 3D Gaussian Splatting framework by incorporating a temporal dimension into the Gaussian primitives, allowing for the representation of dynamic scenes [S7, unofficial]. The method builds on the mathematical properties of multivariate Gaussians to derive conditional 3D Gaussians from 4D Gaussians at any given time [S7, unofficial]. It has been widely adopted for dynamic scene reconstruction and novel view synthesis, with various adaptations for specific domains such as urban scenes and microscopy [S7, unofficial][S8, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S3] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S4] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
[S5] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S6] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S7] “Real-time Photorealistic Dynamic Scene Representation and Rendering with 4D Gaussian Splatting”, https://radiancefields.com/papers/real-time-photorealistic-dynamic-scene-representation-and-rendering-with-4d-gaussian-splatting, fetched 2026-10-30 (unofficial)
[S8] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S9] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
