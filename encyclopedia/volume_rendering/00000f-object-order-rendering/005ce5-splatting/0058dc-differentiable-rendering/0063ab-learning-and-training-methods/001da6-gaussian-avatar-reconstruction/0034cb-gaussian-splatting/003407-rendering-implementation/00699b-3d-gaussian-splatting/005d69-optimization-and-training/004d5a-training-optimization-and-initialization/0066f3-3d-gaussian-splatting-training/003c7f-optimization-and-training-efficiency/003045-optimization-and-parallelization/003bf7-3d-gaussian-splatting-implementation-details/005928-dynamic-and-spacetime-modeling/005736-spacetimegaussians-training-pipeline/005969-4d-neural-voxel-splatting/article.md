# 4D Neural Voxel Splatting

Also called: 4D-NVS

## Overview
4D Neural Voxel Splatting (4D-NVS) is a dynamic scene rendering method that combines voxel-based representations with neural Gaussian splatting to decouple spatial structure from temporal dynamics [S1]. It generates neural Gaussians on-demand from persistent voxel anchors and applies unified deformations via a HexPlane encoder, reducing memory complexity to $O(fV + F)$ compared to the $O(N \cdot T)$ scaling of replicating Gaussians across timestamps [S1]. Unlike Scaffold-GS which generates static Gaussians, 4D-NVS produces time-varying Gaussians through learned temporal features, enabling real-time rendering with significantly lower memory consumption [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Sparse point cloud from Structure-from-Motion (SfM) [S1]; camera poses and timestamps [S1].
*   **Outputs:** Rendered images for arbitrary timestamps and viewpoints [S1].

**Data Structures:**
1.  **Neural Voxels:** A 4D voxel grid. Each voxel $v$ has a center position $x_v \in \mathbb{R}^3$, a spatial extent (scale factor) $l_v \in \mathbb{R}^3$, and a learnable feature vector $f_v \in \mathbb{R}^{f_v}$ [S1]. The feature dimension $f_v$ is 32 [S1].
2.  **Gaussian Offsets:** For each voxel, $k$ learnable offset vectors $O_i \in \mathbb{R}^3$ for $i=0,\dots,k-1$ [S1].
3.  **HexPlane Grids:** A decomposition of the 4D space into 2D planes (XY, XZ, YZ, XT, YT, ZT) with specific resolutions [S1].
4.  **Refinement Stack:** Stores metadata for problematic viewpoints, including failure type (quality vs. gradient), severity score, and temporal consistency flags [S1].

**Numbered Steps:**

1.  **Initialization:**
    *   Perform COLMAP initialization from the first frame (for Neu3D) or 200 random frames (for HyperNeRF) to yield 5,000–20,000 initial points [S1].
    *   Distribute voxel centers to ensure comprehensive spatial coverage using the sparse point cloud $P \in \mathbb{R}^{M \times 3}$ and voxel size $\epsilon$: $V = \{ \lfloor p/\epsilon \rfloor \cdot \epsilon \mid p \in P \}$ [S1].
    *   Train the scene using Scaffold-GS for 3,000 iterations with mixed timestamp samples to establish a foundational geometric representation without temporal deformation [S1].

2.  **Visibility Culling:**
    *   At each timestamp $t$, identify voxels within the camera frustum [S1].
    *   This reduces computational overhead from $O(N)$ to $O(V_{visible})$ where $V_{visible} \ll N$ [S1].

3.  **Neural Gaussian Generation:**
    *   For each visible voxel $v$, generate $k$ neural Gaussians [S1].
    *   Compute positions: $\mu_i = x_v + O_i \cdot l_v$ for $i=0,\dots,k-1$ [S1].
    *   Decode attributes using dedicated MLPs taking anchor features $f_v$, viewing distance $\delta_{vc}$, and directional vector $\hat{d}_{vc}$ as input [S1]:
        $$ \{\alpha_0,\dots,\alpha_{k-1}\} = F_\alpha(f_v, \delta_{vc}, \hat{d}_{vc}) $$
        $$ \{c_0,\dots,c_{k-1}\} = F_c(f_v, \delta_{vc}, \hat{d}_{vc}) $$
        $$ \{s_0,\dots,s_{k-1}\} = F_s(f_v, \delta_{vc}, \hat{d}_{vc}) $$
        $$ \{r_0,\dots,r_{k-1}\} = F_r(f_v, \delta_{vc}, \hat{d}_{vc}) $$
        where $F_\alpha, F_c, F_s, F_r$ are MLPs for opacity, color, scale, and rotation respectively [S1].

4.  **Deformation (Selective Strategy):**
    *   Integrate HexPlane features $f_h$ into integrated features $f_d$ using a shallow network $\phi_d$: $f_d = \phi_d(f_h)$ [S1].
    *   Predict offsets using single-layer MLPs $\phi_x, \phi_r, \phi_s$:
        $$ \Delta\mu = \phi_x(f_d), \quad \Delta r = \phi_r(f_d), \quad \Delta s = \phi_s(f_d) $$
        where $\phi_x$ outputs 3D position offsets, $\phi_r$ outputs quaternion rotations, and $\phi_s$ outputs 3D scale factors [S1].
    *   Apply deformation only to geometric attributes:
        $$ (\mu', r', s') = (\mu+\Delta\mu, r+\Delta r, s+\Delta s) $$
    *   Keep color $c$ and opacity $\alpha$ unchanged to prevent training instability [S1].

5.  **Density Control:**
    *   Grow anchors in high-gradient regions and prune those producing consistently low-opacity Gaussians [S1].
    *   Use densification threshold $\tau_g$ and opacity threshold $\tau_\alpha$ [S1].

6.  **View Refinement (Stage 3):**
    *   Identify crude viewpoints using PSNR-based detection: $PSNR_i < (1 + \gamma) \cdot EMA_{PSNR}$, where $\gamma$ decays from 0.05 to 0.02 and $EMA_{PSNR} = 0.4 \cdot PSNR_{current} + 0.6 \cdot EMA_{PSNR}$ [S1].
    *   Identify crude viewpoints using gradient-based detection: $\|\nabla L_{view}\| > (1 + \gamma) \cdot EMA_{grad}$ [S1].
    *   Add identified viewpoints to the refinement stack with metadata [S1].
    *   Perform focused training on crude viewpoints with sampling probability weighted by severity scores [S1].
    *   Use adjusted thresholds ($\tau_g=0.0001, \tau_\alpha=0.03$) for enhanced Gaussian generation in challenging regions [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
voxels = initialize_from_sfm(sparse_point_cloud)
train_scaffold_gs(voxels, iterations=3000)

# Main Training Loop
for iteration in range(14000):
    # 1. Visibility Culling
    visible_voxels = cull_frustum(voxels, camera_pose)
    
    # 2. Generate Gaussians
    gaussians = []
    for v in visible_voxels:
        # Position
        for i in range(k):
            mu_i = v.center + v.offsets[i] * v.scale
            # Attributes via MLPs
            alpha_i, color_i, scale_i, rot_i = decode_attributes(v.feature, delta_vc, d_vc)
            gaussians.append(Gaussian(mu_i, alpha_i, color_i, scale_i, rot_i))
    
    # 3. Deformation
    hex_features = get_hex_plane_features(voxels, timestamp)
    integrated_features = shallow_mlp(hex_features)
    for g in gaussians:
        delta_mu = position_mlp(integrated_features)
        delta_rot = rotation_mlp(integrated_features)
        delta_scale = scale_mlp(integrated_features)
        # Selective deformation: only geometric
        g.mu = g.mu + delta_mu
        g.rot = g.rot + delta_rot
        g.scale = g.scale + delta_scale
        # Color and opacity remain unchanged
    
    # 4. Render and Loss
    rendered_image = rasterize(gaussians, camera_pose)
    loss = compute_loss(rendered_image, ground_truth)
    loss += lambda_vol * volume_regularization_loss(gaussians)
    loss += lambda_tv * total_variation_loss(hex_features)
    update_parameters(loss)
    
    # 5. Density Control
    densify_gaussians(gaussians, threshold=0.0002)
    prune_gaussians(gaussians, opacity_threshold=0.05)

# View Refinement Loop
for iteration in range(14000):
    # Identify crude views
    crude_views = []
    for view in camera_list:
        psnr = compute_psnr(render(view))
        ema_psnr = 0.4 * psnr + 0.6 * ema_psnr
        if psnr < (1 + gamma) * ema_psnr:
            crude_views.append(view)
        
        grad_mag = compute_gradient_magnitude(view)
        if grad_mag > (1 + gamma) * ema_grad:
            crude_views.append(view)
    
    # Focused training on crude views
    for view in sample(crude_views, weights=severity_scores):
        visible_voxels = cull_frustum(voxels, view.pose)
        gaussians = generate_gaussians(visible_voxels)
        gaussians = deform_gaussians(gaussians)
        rendered_image = rasterize(gaussians, view.pose)
        loss = compute_loss(rendered_image, view.gt)
        update_parameters(loss)
        
        # Adjusted thresholds for refinement
        densify_gaussians(gaussians, threshold=0.0001)
        prune_gaussians(gaussians, opacity_threshold=0.03)
```

**Parameters:**
*   $k$ (Gaussians per voxel): 10 [S1].
*   $f_v$ (Feature dimension): 32 [S1].
*   HexPlane resolution: [64, 64, 64, 150] for DyNeRF; [64, 64, 64, 25] for HyperNeRF [S1].
*   HexPlane scale multipliers: [1, 2] [S1].
*   Learning rates: Gaussian offsets 0.01, color MLP 0.008, opacity MLP 0.002, rotation/scaling MLPs 0.004, HexPlane grids 0.0016, deformation MLPs 0.00016 [S1].
*   Learning rate decay: All rates decay to 1/10 of initial values by iteration 14,000 [S1].
*   $\lambda_{vol}$ (Volume regularization): 0.015 [S1].
*   $\lambda_{tv}$ (Total variation): 0.0002 [S1].
*   $\tau_g$ (Densification threshold): 0.0002 (coarse), 0.0001 (refinement) [S1].
*   $\tau_\alpha$ (Opacity threshold): 0.05 (coarse), 0.03 (refinement) [S1].
*   EMA momentum: 0.4 [S1].
*   $\gamma$ (PSNR threshold): Starts at 0.05, decays to 0.02 [S1].

**Edge Cases and Pitfalls:**
*   **Training Instability:** Deforming all Gaussian properties (including color and opacity) leads to instability and error accumulation because appearance properties are highly sensitive to deformation errors [S1]. The method fixes color and opacity to maintain stable gradients [S1].
*   **Large Motions:** Absence of initial points and inaccuracies in camera poses make optimization challenging [S1].
*   **Popping Artifacts:** View-dependent sorting solves the "static" part of popping artifacts but does not eliminate them entirely [S1].
*   **Blurry Effects:** Without volume regularization ($L_{vol}$), large Gaussians float in front of the camera causing blurry effects [S1].

## Operating characteristics
*   **Memory Complexity:** $O(fV + F)$, where $V$ is the number of voxels, $f$ is the feature dimension per voxel, and $F$ represents the deformation network parameters [S1]. This is a significant reduction from the traditional $O(N \cdot T)$ scaling [S1].
*   **Memory Usage:** 3,050 MiB on HyperNeRF [S1]. This is an 86% reduction vs TiNeuVox-B (21,558 MiB) and 32% vs 4D-GS (4,500 MiB) [S1].
*   **Training Time:** 17 minutes with refinement (13 minutes without) on HyperNeRF [S1]. This is a 94% speedup vs HyperNeRF (32 hours), 32% vs 4D-GS (25 minutes), and 89% vs GaGS (120 minutes) [S1].
*   **Rendering Performance:** 43 FPS on Neu3D dataset at 1,352x1,014 resolution [S1]. This is an order of magnitude faster than Im4D (5 FPS) and MSTH (2 FPS) [S1].
*   **Quality:** 28.5 PSNR and 0.872 MS-SSIM on HyperNeRF at 960x540 resolution [S1]. 33.12 PSNR and 0.021 D-SSIM on Neu3D at 1,352x1,014 resolution [S1].
*   **Strengths:** Real-time rendering, low memory footprint, fast training, stable convergence due to selective deformation [S1].
*   **Limitations:** Popping artifacts are not fully eliminated; large motions with poor initial points remain challenging [S1].

## Background
4D Neural Voxel Splatting was introduced in the paper "4D Neural Voxel Splatting: Dynamic Scene Rendering with Voxelized Gaussian Splatting" [S1]. It builds upon established techniques such as Scaffold-GS for voxel-based generation and HexPlane decomposition for temporal modeling [S1]. The method addresses the substantial memory overhead of extending 3D Gaussian Splatting to dynamic scenes by replicating Gaussians across frames [S1].

## Sources
[S1] 4D Neural Voxel Splatting: Dynamic Scene Rendering with Voxelized Guassian Splatting, https://doi.org/10.1109/icip61757.2026.11630026, fetched 2026-10-26
