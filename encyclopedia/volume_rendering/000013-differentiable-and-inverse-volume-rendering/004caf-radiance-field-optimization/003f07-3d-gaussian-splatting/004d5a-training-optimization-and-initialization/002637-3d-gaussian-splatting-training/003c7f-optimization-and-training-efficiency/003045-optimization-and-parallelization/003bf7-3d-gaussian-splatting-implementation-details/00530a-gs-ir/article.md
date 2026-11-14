# GS-IR
Also called: 3D Gaussian Splatting for Inverse Rendering

## Overview
GS-IR is an inverse rendering approach based on 3D Gaussian Splatting (3DGS) that leverages forward mapping volume rendering to achieve photorealistic novel view synthesis and relighting results, estimating scene geometry, surface material, and environment illumination from multi-view images captured under unknown lighting conditions [S1]. Unlike NeRF-based inverse rendering methods that use implicit neural representations and ray tracing (backward mapping) which suffer from low expressive power and high computational complexity, GS-IR uses 3D Gaussians and forward mapping splatting for real-time rendering efficiency [S1].

## How it works
The GS-IR framework consists of three stages: 1) Initial stage: optimize 3D Gaussians for geometry reconstruction using differentiable splatting and supervise normals using depth gradients; 2) Second stage: precompute occlusion based on learned geometry (depth and normal) and store it in a spherical harmonics-based architecture to model indirect illumination; 3) Final stage: combine differentiable splatting with the physical-based rendering (PBR) pipeline to optimize illumination and material-aware 3D Gaussians [S1].

### Inputs and Outputs
**Inputs:** Multi-view images captured under unknown lighting conditions [S1].
**Outputs:** Scene geometry (depth, normals), surface material (Albedo, Metallic, Roughness), and environment illumination [S1].

### Data Structures
The method extends the standard 3D Gaussian representation. Each Gaussian $i$ stores:
*   Position $\mu_i \in \mathbb{R}^3$ [S1].
*   Covariance $\Sigma_i$ (derived from rotation $R_i$ and scale $S_i$) [S1].
*   Opacity $\alpha_i$ [S1].
*   Spherical Harmonics coefficients for color/view-dependent appearance [S1].
*   Normal vector $n_i$ [S1].
*   Material properties: Albedo, Metallic, Roughness [S1].
*   Occlusion cache values stored in a spherical harmonics-based architecture [S1].

### Formulas
The 3D Gaussian function is defined as [S1]:
$$
g(x|\mu, \Sigma) = e^{-\frac{1}{2} (x-\mu)^T \Sigma^{-1} (x-\mu)}
$$
where $\mu \in \mathbb{R}^3$ is the mean and $\Sigma$ is the covariance matrix [S1].

To address normal estimation, GS-IR derives pseudo normals from the rendered depth map $\hat{D}$:
$$
\hat{n}_D = \nabla_{uv} \hat{D}
$$
where $\nabla_{uv}$ denotes the gradient in the image plane [S1]. These pseudo normals supervise the optimization of normals within the 3D Gaussians [S1].

### Steps
1.  **Initial Stage (Geometry and Normal Optimization):**
    *   Optimize 3D Gaussians for geometry reconstruction using differentiable splatting [S1].
    *   Render a depth map $\hat{D}$ [S1].
    *   Compute pseudo normals $\hat{n}_D = \nabla_{uv} \hat{D}$ from the depth gradient [S1].
    *   Use $\hat{n}_D$ to supervise and optimize the normal attributes $n_i$ stored in the 3D Gaussians [S1].

2.  **Second Stage (Occlusion Baking):**
    *   Precompute occlusion based on the learned geometry (depth and normal) [S1].
    *   Store the occlusion data in a spherical harmonics-based architecture to model indirect illumination [S1]. This is inspired by the "Indirect Lighting Cache" used in real-time rendering [S1].

3.  **Final Stage (Material and Illumination Optimization):**
    *   Combine differentiable splatting with a physical-based rendering (PBR) pipeline [S1].
    *   Optimize illumination and material-aware 3D Gaussians (Albedo, Metallic, Roughness) [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# GS-IR Optimization Loop
# Inputs: Multi-view images, Camera poses
# Outputs: Optimized 3D Gaussians with geometry, material, and illumination

def gs_ir_training(multi_view_images, camera_poses):
    # Initialize 3D Gaussians with position, covariance, opacity, color SH, normal, material
    gaussians = initialize_3d_gaussians()
    
    # Stage 1: Geometry and Normal Optimization
    for step in range(stage1_steps):
        # Render depth and color for each view
        rendered_depth, rendered_color = render_gaussians(gaussians, camera_poses)
        
        # Compute pseudo normals from depth gradient
        pseudo_normals = compute_gradient(rendered_depth)
        
        # Update Gaussians using differentiable splatting loss and normal supervision
        loss = compute_rendering_loss(rendered_color, multi_view_images) + \
               compute_normal_loss(pseudo_normals, gaussians.normals)
        update_gaussians(gaussians, loss)
        
    # Stage 2: Occlusion Baking
    # Precompute occlusion based on learned geometry
    occlusion_cache = bake_occlusion(gaussians.depth, gaussians.normals)
    # Store in spherical harmonics architecture
    indirect_illumination_model = fit_spherical_harmonics(occlusion_cache)
    
    # Stage 3: Material and Illumination Optimization
    for step in range(stage3_steps):
        # Render using PBR pipeline with indirect illumination
        rendered_pbr_color = render_pbr(gaussians, camera_poses, indirect_illumination_model)
        
        # Update Gaussians to optimize material (Albedo, Metallic, Roughness) and illumination
        loss = compute_pbr_loss(rendered_pbr_color, multi_view_images)
        update_gaussians_material(gaussians, loss)
        
    return gaussians
```

### Edge Cases and Pitfalls
*   **Normal Estimation:** 3DGS does not support producing plausible normals natively, and its adaptive control of Gaussian density may lead to loose geometry, making it difficult to estimate accurate scene normals [S1]. GS-IR addresses this with depth-derivation-based regularization [S1].
*   **Occlusion Calculation:** Forward mapping (e.g., rasterization and splatting) cannot trace occlusion like backward mapping (e.g., ray tracing), making it difficult to calculate occlusion for indirect lighting [S1]. GS-IR addresses this with a baking-based method [S1].

## Operating characteristics
*   **Efficiency:** GS-IR improves rendering efficiency compared to NeRF-based methods by using differentiable forward mapping volume rendering instead of ray tracing [S1].
*   **Expressiveness:** The flexible and expressive 3DGS representation allows for fast and compact geometry reconstruction [S1].
*   **Scene Scope:** Unlike TensoIR which leverages ray tracing of NeRF to directly model occlusion and indirect illumination but is limited to object-level, GS-IR handles occlusion via baking and is available for both objects and unbounded scenes [S1].

## Background
GS-IR is the first work to introduce the 3DGS technique for inverse rendering, simultaneously estimating scene geometry, materials, and illumination from multi-view images [S1]. It is motivated by the remarkable performance in quality and speed of 3DGS [S1]. The occlusion baking strategy is inspired by the "Indirect Lighting Cache" used in real-time rendering [S1].

## Sources
[S1] GS-IR: 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.1109/cvpr52733.2024.02045, fetched 2026-10-26
