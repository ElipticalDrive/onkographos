# Two-Stage Inverse Rendering Optimization

Also called: Hybrid Differentiable Rendering for Inverse Rendering, Two-Stage Optimization Strategy

## Overview
Two-Stage Inverse Rendering Optimization decouples the recovery of scene geometry from material reflectance and illumination by alternating between an approximate differentiable renderer and a physically-based ray tracer [S1, S2]. It addresses the ill-posed nature of inverse rendering by first establishing a robust geometric proxy using silhouette constraints, then refining physically accurate albedo and environment maps while holding geometry fixed [S1, S2]. This approach differs from single-stage radiance field methods (e.g., [NeRF](44-radiance-field-optimization.md), [PhySG](44-radiance-field-optimization.md)) by explicitly modeling triangle meshes or Gaussian splats with PBR materials, enabling direct deployment in graphics engines and superior specular handling [S1].

## How it works
The method operates in two distinct phases [S1, S2]. The specific variables involved are: $\mu$ (position), $\Sigma$ (covariance), $c$ (color), $\alpha$ (opacity), $n$ (normal), $\rho$ (albedo), $r$ (roughness), and $L_{env}$ (environment-map radiance) [S2].

### Inputs and Outputs
*   **Inputs:** A set of multi-view images $\{I_k\}$, camera parameters, and an initial geometry estimate (either a coarse mesh from SfM/MVS [S1] or a pre-trained 3D Gaussian Splatting model [S2]).
*   **Outputs:** Optimized vertex positions $\theta_p$ (or Gaussian attributes), diffuse albedo texture $\theta_d$, specular albedo texture $\theta_s$ (or BRDF parameters $\rho, r$), and HDR environment map $L_{env}$ [S1, S2].

### Data Structures
1.  **Geometry Representation:**
    *   *Mesh-based [S1]:* Vertex position matrix $V \in \mathbb{R}^{n \times 3}$, Laplacian matrix $L \in \mathbb{R}^{n \times n}$, face normals $n_i$.
    *   *Gaussian-based [S2]:* 3D Gaussians defined by mean $\mu$, covariance $\Sigma$, opacity $\alpha$, color $c$, and normal $n$.
2.  **Material Representation:**
    *   Diffuse albedo $\theta_d$ (or $\rho_d$) and specular albedo $\theta_s$ (or $\rho_s$) stored as 2D texture maps [S1].
    *   Roughness denoted as $r$ in Gaussian variants [S2] or $\alpha$ in the GGX microfacet distribution for mesh variants [S1].
3.  **Lighting:**
    *   Environment map $L_{env}$, typically discretized on a grid (e.g., $512 \times 128$ [S1] or $16 \times 32$ [S2]).

### Numbered Steps

#### Stage 1: Geometry Optimization
The goal is to refine the initial geometry to accurately match the scene silhouette and surface structure [S1, S2].

1.  **Initialize:** Load initial geometry (mesh vertices $\theta_p$ or Gaussian parameters $\{\mu, \Sigma, c, \alpha, n\}$) and set environment map channels to $(0.5, 0.5, 0.5)$ [S1, S2].
2.  **Render Silhouette/Opacity:**
    *   *Mesh [S1]:* Use a soft rasterizer to compute silhouette image $I_{sil}(\theta_p)$ for each input view.
    *   *Gaussians [S2]:* Render opacity $\hat{\alpha}$ and color using standard Gaussian splatting.
3.  **Compute Losses:**
    *   *Silhouette Loss [S1]:*
        $$ L_{sil} = 1 - \frac{\tilde{I}_{sil} \otimes I_{sil}}{\sqrt{\sum \tilde{I}_{sil}^2 \sum I_{sil}^2}} $$
        where $\tilde{I}_{sil}$ is the ground truth silhouette, $\otimes$ is element-wise product, and normalization ensures scale invariance [S1].
    *   *Laplacian Loss [S1]:*
        $$ L_{lap} = \|LV\|_2 $$
        where $V$ is the vertex coordinate matrix and $L$ is the mesh Laplacian matrix [S1].
    *   *Normal Consistency Loss [S1]:*
        $$ L_{normal} = \sum_{i,j} [1 - (n_i \cdot n_j)] $$
        summed over all triangle pairs $(i,j)$ sharing a common edge [S1].
    *   *Edge Length Loss [S1]:*
        $$ L_{edge} = \sum_i e_i $$
        where $e_i$ is the length of the $i$-th edge [S1].
    *   *Opacity-Mask Loss [S2]:*
        $$ L_{mask} = \|\hat{\alpha} - M\|_1 $$
        where $M$ is the binary foreground mask [S2].
    *   *Color/Normal Loss [S2]:* $L_c$ (color reconstruction) and $L_n$ (normal consistency) from standard Gaussian splatting [S2].
4.  **Optimize Geometry:**
    *   *Mesh [S1]:* Minimize $L_{geo} = \lambda_{sil} L_{sil} + \lambda_{lap} L_{lap} + \lambda_{normal} L_{normal} + \lambda_{edge} L_{edge}$ by updating $\theta_p$.
    *   *Gaussians [S2]:* Minimize $L_{init} = L_c + L_n + \lambda_{mask} L_{mask}$ by updating $\{\mu, \Sigma, c, \alpha, n\}$.
5.  **Iterate:** Repeat for $N_1$ iterations (e.g., 30,000 for Gaussians [S2], or 200 iterations per view for meshes [S1]).

#### Stage 2: Reflectance and Lighting Optimization
The goal is to recover physically accurate materials and lighting [S1, S2].

1.  **Freeze Geometry:** Fix all geometric attributes ($\theta_p$ for mesh variants [S1], or $\{\mu, \Sigma, c, \alpha, n\}$ for Gaussian variants [S2]).
2.  **Initialize Materials:** Initialize diffuse $\theta_d$ (or $\rho_d$) and specular $\theta_s$ (or $\rho_s$) textures, roughness $r$, and environment map $L_{env}$ [S1, S2].
3.  **Physically-Based Rendering:**
    *   Trace rays using a physically-based differentiable renderer (e.g., Mitsuba 2 [S1] or OptiX [S2]).
    *   *Reflectance Model [S1]:*
        $$ f_r(\omega_o, \omega_i; x) = \theta_d(x) + \theta_s(\omega_o, \omega_i; x) $$
        $$ \theta_d(x) = \frac{\rho_d(x)}{\pi} $$
        $$ \theta_s(\omega_o, \omega_i; x) = \rho_s(x) \frac{D(h, \alpha) G(\omega_o, \omega_i, n(x))}{(n(x) \cdot \omega_i)(n(x) \cdot \omega_o) \pi} $$
        where $D$ is the GGX microfacet distribution [Walter et al., 2007], $G$ is the shadowing-masking function, and Fresnel effects are ignored [S1].
    *   *Rendering Equation [S1]:*
        $$ L_o(\omega_o; x) = \int_{\Omega} L_i(\omega_i) f_r(\omega_o, \omega_i; x) (\omega_i \cdot n) d\omega_i $$
4.  **Compute Losses:**
    *   *Photometric Loss [S1, S2]:*
        $$ L_{rgb} = \|\tilde{I}_{rgb} - I_{rgb}(\theta_r)\|_1 $$
        where $\tilde{I}_{rgb}$ is the ground truth image and $I_{rgb}(\theta_r)$ is the rendered image [S1].
    *   *Albedo Regularization [S1]:*
        $$ R(\theta_r) = \sum_p \left( \theta_s[p] - \frac{\sum_q \theta_s[q] k_{p,q}}{\sum_q k_{p,q}} \right)^2 + \sum_p \frac{(\theta_d[p] - \theta_d[q])^2}{2\sigma_2^2} $$
        where $k_{p,q} = \exp\left(-\frac{\|p-q\|_2^2}{2\sigma_1^2}\right)$ is a Gaussian kernel between pixels/vertices $p$ and $q$ [S1].
    *   *Total Loss [S1]:*
        $$ L_{ref} = \lambda_{rgb} L_{rgb} + \lambda_{reg} R(\theta_r) $$
    *   *Total Loss [S2]:*
        $$ L_{1pbr} = \|\tilde{I}_{rgb} - I_{rgb}\|_1 $$
5.  **Optimize Materials:**
    *   Update $\theta_d, \theta_s$ (or $\rho, r$) and $L_{env}$ to minimize $L_{ref}$ or $L_{1pbr}$ [S1, S2].
6.  **Iterate:** Repeat for $N_2$ iterations (e.g., 400 for both [S1, S2]).

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_silhouette_loss(I_sil, I_gt_sil):
    return 1.0 - np.sum(I_sil * I_gt_sil) / np.sqrt(np.sum(I_gt_sil**2) * np.sum(I_sil**2))

def compute_laplacian_loss(V, L):
    return np.linalg.norm(L @ V, ord='fro')

def compute_normal_consistency_loss(V, faces):
    normals = np.cross(V[faces[:,1]] - V[faces[:,0]], V[faces[:,2]] - V[faces[:,0]])
    normals = normals / np.linalg.norm(normals, axis=1, keepdims=True)
    loss = 0.0
    for i, j in shared_edges(faces):
        loss += 1.0 - np.dot(normals[i], normals[j])
    return loss

def compute_edge_length_loss(V, edges):
    return np.sum(np.linalg.norm(V[edges[:,1]] - V[edges[:,0]], axis=1))

def compute_opacity_mask_loss(alpha_hat, M):
    return np.sum(np.abs(alpha_hat - M))

def compute_l1_loss(I_pred, I_gt):
    return np.sum(np.abs(I_pred - I_gt))

def compute_albedo_regularization(theta_s, theta_d, sigma1=1.0, sigma2=1.0):
    # Simplified kernel computation for demonstration
    dist_sq = np.sum((theta_s[:, None, :] - theta_s[None, :, :])**2, axis=2)
    k = np.exp(-dist_sq / (2 * sigma1**2))
    reg_s = np.sum((theta_s - (k @ theta_s) / (k.sum(axis=1, keepdims=True) + 1e-8))**2)
    reg_d = np.sum((theta_d - theta_d)**2) / (2 * sigma2**2) # Placeholder for actual neighbor diff
    return reg_s + reg_d

def two_stage_inverse_rendering(images, cameras, initial_geometry, geometry_type):
    """
    Implements the two-stage optimization for inverse rendering.
    
    Args:
        images: List of input RGB images.
        cameras: List of camera parameters (extrinsics, intrinsics).
        initial_geometry: Initial mesh vertices or Gaussian parameters.
        geometry_type: 'mesh' or 'gaussian'
        
    Returns:
        optimized_geometry, albedo_textures, environment_map
    """
    if geometry_type == 'mesh':
        theta_p = initial_geometry.vertices.copy()
        theta_d = initial_geometry.diffuse_albedo.copy() if hasattr(initial_geometry, 'diffuse_albedo') else np.zeros_like(initial_geometry.vertices[:, :1])
        theta_s = initial_geometry.specular_albedo.copy() if hasattr(initial_geometry, 'specular_albedo') else np.zeros_like(initial_geometry.vertices[:, :1])
        L_env = np.full(3, 0.5) # White light initialization [S1]
        
        for _ in range(200 * len(images)): # [S1]
            I_sil = soft_rasterize(theta_p, cameras) # Approximate differentiable renderer [S1]
            L_sil = compute_silhouette_loss(I_sil, ground_truth_silhouettes)
            L_lap = compute_laplacian_loss(theta_p, mesh_laplacian)
            L_normal = compute_normal_consistency_loss(theta_p, mesh_faces)
            L_edge = compute_edge_length_loss(theta_p, mesh_edges)
            L_geo = lambda_sil * L_sil + lambda_lap * L_lap + lambda_normal * L_normal + lambda_edge * L_edge
            theta_p = optimize_step(theta_p, L_geo)
            
        # Stage 2
        frozen_theta_p = theta_p
        for _ in range(400): # [S1, S2]
            I_rgb = pbr_render(frozen_theta_p, theta_d, theta_s, L_env, cameras) # Physically-based renderer [S1]
            L_rgb = compute_l1_loss(I_rgb, ground_truth_images)
            R_theta = compute_albedo_regularization(theta_s, theta_d)
            L_ref = lambda_rgb * L_rgb + lambda_reg * R_theta
            theta_d, theta_s, L_env = optimize_step([theta_d, theta_s, L_env], L_ref)
            
    elif geometry_type == 'gaussian':
        mu, Sigma, c, alpha, n = initial_geometry
        theta_d = initial_geometry.diffuse_albedo.copy()
        theta_s = initial_geometry.specular_albedo.copy()
        L_env = np.full(3, 0.5)
        
        for _ in range(30000): # [S2]
            alpha_hat, color_hat = gaussian_splat_render(mu, Sigma, c, alpha, n, cameras)
            L_c = compute_l1_loss(color_hat, ground_truth_images)
            L_n = compute_normal_loss(n, ground_truth_normals)
            L_mask = compute_opacity_mask_loss(alpha_hat, foreground_masks)
            L_init = L_c + L_n + lambda_mask * L_mask
            mu, Sigma, c, alpha, n = optimize_step([mu, Sigma, c, alpha, n], L_init)
            
        # Stage 2
        frozen_mu, frozen_Sigma, frozen_c, frozen_alpha, frozen_n = mu, Sigma, c, alpha, n
        rho = theta_d # [S2]
        r = theta_s   # [S2]
        for _ in range(400): # [S2]
            I_rgb = ray_trace_render(frozen_mu, frozen_Sigma, frozen_c, frozen_alpha, frozen_n, rho, r, L_env, cameras)
            L_1pbr = compute_l1_loss(I_rgb, ground_truth_images)
            rho, r, L_env = optimize_step([rho, r, L_env], L_1pbr)
            
    return theta_p if geometry_type == 'mesh' else (mu, Sigma, c, alpha, n), theta_d, theta_s, L_env
```

### Parameters and Edge Cases
*   **Iterations:** Stage 1 typically requires significantly more iterations (e.g., 30,000 [S2]) than Stage 2 (e.g., 400 [S1, S2]).
*   **Ray Tracing Depth:** Limited to 3 bounces to manage memory and compute [S1].
*   **Sampling:** 4 samples per pixel (spp) and image downsampling by factor 4 are used during optimization to reduce cost [S1].
*   **Environment Map Size:** $512 \times 128$ [S1] or $16 \times 32$ [S2].
*   **Pitfall:** Extending Stage 2 training beyond the recommended iterations degrades relighting quality because the albedo begins to absorb illumination information [S2].
*   **Fresnel Effect:** Often ignored in the reflectance model to simplify the optimization landscape [S1].

## Operating characteristics
*   **Time Complexity:** Stage 1 is fast for meshes (~0.5 hours [S1]) or moderate for Gaussians (~15 minutes [S2]). Stage 2 is slower due to ray tracing (1.5–2.5 hours [S1] or ~10 minutes [S2] on high-end GPUs).
*   **Memory Complexity:** Depends on the geometry representation. Mesh-based methods have lower memory overhead than volumetric NeRFs [S1]. Gaussian splatting requires storing covariance and normal attributes per Gaussian [S2].
*   **Strengths:** Produces explicit, editable geometry (meshes or splats) compatible with standard graphics pipelines. Better handling of specular highlights compared to SDF-based methods like PhySG [S1].
*   **Limitations:** Requires a good initial geometry estimate. Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The method originates from the need to combine the robustness of approximate differentiable rendering for geometry with the physical accuracy of ray tracing for materials [S1]. It was introduced in [S1] (IJCAI 2023) as a hybrid approach using triangle meshes and Cook-Torrance PBR, differing from NeRF/IDR/PhySG which use SDFs and Spherical Gaussians. A variant using 3D Gaussian Splatting as the geometry proxy was later proposed in [S2] (2026), leveraging the speed of splatting for Stage 1 and ray tracing for occlusion-aware relighting in Stage 2.

## Sources
[S1] Efficient Multi-View Inverse Rendering Using a Hybrid Differentiable Rendering Method, https://doi.org/10.24963/ijcai.2023/205, fetched 2026-10-26
[S2] VisibilitySplat: Ray-Traced Occlusion for Relighting with 3D Gaussian Splatting, https://doi.org/10.24132/csrn.2026-e17, fetched 2026-10-26
