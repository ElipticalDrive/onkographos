# Gaussian Splatting Rendering Pipeline

Also called: 3D Gaussian Splatting Rendering Pipeline, Gaussian Splatting Rasterization and Rendering Pipeline

## Overview
The Gaussian Splatting Rendering Pipeline orchestrates the differentiable rasterization of a scene represented as a set of 3D Gaussians into a 2D image for a single camera view [S1][S6, unofficial]. It solves the problem of efficiently rendering millions of adaptive, view-dependent primitives by projecting 3D covariances to 2D, evaluating spherical harmonics for color, and compositing the results via alpha blending [S3, S6, unofficial]. It sets itself apart from standard polygon rasterization by handling soft, overlapping volumetric primitives without geometry clipping or complex shader pipelines, and from NeRFs by offering explicit, differentiable scene representations that render at interactive frame rates [S1]. It builds on [GaussianRasterizer Initialization and Input Validation](2923-gaussianrasterizer-initialization-and-input-validation.md) and [GaussianRasterizationSettings](2924-gaussianrasterizationsettings.md).

## How it works
The pipeline executes five main steps: covariance reconstruction, 2D projection, color evaluation, tile intersection, and alpha compositing [S5, unofficial]. It accepts Gaussian attributes (position, scale, rotation, opacity, SH coefficients) and camera parameters, and outputs a rendered image along with auxiliary channels (depth, normals, opacity) and visibility masks [S3, S5, unofficial].

### Inputs and Outputs
**Inputs:**
*   `means3D`: 3D positions, shape $[N, 3]$ [S5, S6, unofficial].
*   `quats`: Unit quaternions for rotation, shape $[N, 4]$ (w,x,y,z) [S5, unofficial].
*   `scales`: Log-scales, shape $[N, 3]$ [S6, unofficial].
*   `opacities`: Raw opacity values, shape $[N]$ [S6, unofficial].
*   `shs`: Spherical Harmonics coefficients, shape $[N, K, 3]$ where $K = (d_{max}+1)^2$ [S5, unofficial].
*   `viewpoint_camera`: Contains intrinsics ($fx, fy, cx, cy$), resolution ($H, W$), and extrinsics ($c2w$ matrix) [S6, unofficial].
*   `pipe`: Configuration flags including `convert_SHs_python`, `compute_cov3D_python`, `depth_ratio`, and `debug` [S3, S4, unofficial].
*   `bg`: Background color tensor, shape $[3]$ [S4, unofficial].

**Outputs:**
*   `render`: Image tensor, shape $[H, W, 3]$ for RGB or $[H, W, D]$ for multi-channel [S5, S6, unofficial].
*   `viewspace_points`: 2D mean positions with gradients enabled, shape $[N, 2]$ [S2, S4, unofficial].
*   `visibility_filter`: Boolean tensor $[N]$, true if Gaussian radius $> 0$ [S4, unofficial].
*   `radii`: Screen-space radii of visible Gaussians, shape $[N]$ [S4, unofficial].

### Data Structures
*   **GaussianRasterizationSettings**: Encapsulates camera intrinsics, extrinsics, and rasterization flags. Fields include `image_height`, `image_width`, `tanfovx`, `tanfovy`, `bg`, `scale_modifier`, `viewmatrix`, `projmatrix`, `sh_degree`, `campos`, `prefiltered`, and `debug` [S4, S7, unofficial].
*   **Covariance Representation**: Can be precomputed 3D covariance matrices ($\Sigma_{3D}$) or parametric scales/rotations passed to the rasterizer [S3, S4, unofficial].
*   **Color Representation**: Either precomputed RGB (from SH evaluation) or raw SH coefficients [S4, unofficial].

### Steps

1.  **Initialize Screenspace Points**: Create a zero tensor `screenspace_points` on the target device (e.g., CUDA) with `requires_grad=True` to capture 2D mean gradients. Attempt to retain gradients via `retain_grad()` for debugging [S2, S7, unofficial].

2.  **Compute Camera Parameters**: Calculate half-field-of-view tangents:
    $$ \text{tanfovx} = \tan(\text{FoVx} \times 0.5) $$
    $$ \text{tanfovy} = \tan(\text{FoVy} \times 0.5) $$
    Extract principal point $(cx, cy)$ from camera intrinsics [S4, S7, unofficial].

3.  **Select Covariance Representation**:
    *   If `pipe.compute_cov3D_python` is true: Compute $\Sigma_{3D}$ from scales and rotations in Python using `pc.get_covariance(scaling_modifier)` [S3, S4, unofficial].
    *   Else: Pass `scales` and `rotations` directly to the rasterizer to compute covariances internally [S3, S4, unofficial].

4.  **Select Color Representation**:
    *   If `override_color` is None:
        *   If `pipe.convert_SHs_python` is true:
            1.  Extract SH view: `shs_view = pc.get_features.transpose(1, 2).view(-1, 3, (pc.max_sh_degree+1)**2)` [S4, unofficial].
            2.  Compute direction: `dir_pp = pc.get_xyz - viewpoint_camera.camera_center.repeat(pc.get_features.shape[0], 1)` [S4, unofficial].
            3.  Normalize: `dir_pp_normalized = dir_pp / dir_pp.norm(dim=1, keepdim=True)` [S4, unofficial].
            4.  Evaluate SH: `sh2rgb = eval_sh(pc.active_sh_degree, shs_view, dir_pp_normalized)` [S4, unofficial].
            5.  Clamp: `colors_precomp = torch.clamp_min(sh2rgb + 0.5, 0.0)` [S4, unofficial].
        *   Else: Set `shs = pc.get_features` [S4, unofficial].
    *   If `override_color` is not None: Set `colors_precomp = override_color` [S4, unofficial].

5.  **Construct Settings and Rasterizer**: Instantiate `GaussianRasterizationSettings` with image dimensions, FOV tangents, background, scale modifier, view/projection matrices, SH degree, camera position, and debug flags [S4, S7, unofficial]. Instantiate `GaussianRasterizer` with these settings [S7, unofficial].

6.  **Invoke Rasterizer**: Call the rasterizer with:
    *   `means3D`: `pc.get_xyz`
    *   `means2D`: `screenspace_points`
    *   `shs`: (if using SH)
    *   `colors_precomp`: (if precomputed)
    *   `opacity`: `pc.get_opacity`
    *   `scales`/`rotations`/`cov3D_precomp`: Based on step 3 selection [S4, S7, unofficial].
    
    The rasterizer performs:
    *   **Projection**: Transform 3D Gaussians to camera space and project to 2D [S6, unofficial].
    *   **Covariance Projection**: Project 3D covariance $\Sigma_{3D}$ to 2D using the Jacobian $J$ of the affine approximation of the projective transformation:
        $$ \Sigma_{2D} = J \Sigma_{3D}^{\text{camera}} J^T $$
        where $W$ is the view transformation matrix and $J$ is the Jacobian [S1][S6, unofficial].
    *   **Tiling**: Divide the image into tiles (default size $16 \times 16$ pixels) for efficient parallel processing [S6, unofficial].
    *   **Rasterization**: Evaluate Gaussian contributions per pixel within each tile [S6, unofficial].
    *   **Alpha Blending**: Composite Gaussians front-to-back. For a pixel, let $N$ be the set of contributing Gaussians sorted by depth. The color $C$ is:
        $$ C = \sum_{i \in N} c_i \alpha_i \prod_{j < i} (1 - \alpha_j) $$
        where $c_i$ is the view-dependent color and $\alpha_i$ is the opacity of Gaussian $i$ [S1][S6, unofficial].

7.  **Compute Depth (Optional)**: If `return_depth` is true:
    *   Extract view direction `projvect1` (first 3 elements of `world_view_transform[:,2]`) and offset `projvect2` (last element) [S7, unofficial].
    *   Compute depth values: `means3D_depth = dot(means3D, projvect1) + projvect2` [S7, unofficial].
    *   Repeat depth values 3 times to match RGB channel count [S7, unofficial].
    *   Rasterize again with `shs=None`, `colors_precomp=means3D_depth` [S7, unofficial].
    *   Average resulting depth image over channels: `render_depth = render_depth.mean(dim=0)` [S7, unofficial].

8.  **Return Results**: Construct a dictionary with keys: `render`, `viewspace_points`, `visibility_filter` (`radii > 0`), and `radii` [S4, S7, unofficial].

### Parameters and Defaults
*   `tile_size`: Tile side length. Default: 16 [S5, S6, unofficial].
*   `near_plane`: Near clipping plane. Default: 0.01 [S5, unofficial] or 2e-3 [S6, unofficial].
*   `far_plane`: Far clipping plane. Default: 1e10 [S5, unofficial] or 100 [S6, unofficial].
*   `eps2d`: Regularization added to 2D covariance diagonal. Default: 0.3 [S5, unofficial].
*   `chi_square_clip`: Chi-square clipping threshold for covariance regularization. Default: 9.21 (approx 3σ) [S6, unofficial].
*   `alpha_max`: Maximum alpha value to prevent numerical instability. Default: 0.99 [S6, unofficial].
*   `alpha_cutoff`: Alpha cutoff threshold for early termination. Default: 1/255 [S6, unofficial].
*   `scale_factor`: Factor to scale rendering resolution. Default: 2 (half resolution) [S6, unofficial].
*   `pix_guard`: Pixel guard band width for culling. Default: 64 [S6, unofficial].
*   `scaling_modifier`: Multiplier applied to Gaussian scales. Default: 1.0 [S3, unofficial].

### Edge Cases and Pitfalls
*   **Depth Sorting Swaps**: As the camera moves, overlapping Gaussians may swap sort order, causing popping artifacts (temporal instability) [S10]. Per-pixel depth comparison or ray-tracing can resolve this [S10].
*   **Near-Plane Culling**: Gaussians behind the camera near-plane must be filtered to reduce computation. Condition: $z_{\max} < z_{\text{near}}$, where $z_{\max} = z + \Delta z$ is the Gaussian center depth plus half-extent along the camera z-axis [S11].
*   **Precision**: Lower weight precision (e.g., 8-bit) may be required for inference compared to training [S1].

## Subtopics

### [Neural Gaussian Rasterization Pipeline](7419-neural-gaussian-rasterization-pipeline.md)
Replaces the standard exponential opacity formulation with direct neural network outputs for per-Gaussian opacity and color, retaining the standard 2D Gaussian rasterizer for compositing. It computes view-dependent color and density ($\sigma$) for each Gaussian center, maps $\sigma$ directly to opacity, and composites using standard alpha blending.

### Octree-GS Rendering
Uses an octree structure to manage Gaussian visibility. It initializes `GaussianModel` with dataset parameters, sets anchor masks based on camera position, prefilters voxels for visibility, and renders the scene. It clamps the rendered image to $[0.0, 1.0]$ and records visible Gaussian counts per view [S8, unofficial]. Parameter `ape_code` (default 10) is passed to the render function [S8, unofficial].

### Unity 3DGS Rendering Pipeline Integration
Integrates 3DGS into the Unity engine by inserting an additional sorting pass into each camera's command buffer and draw calls for 3DGS objects into the transparent mesh queue based on bounding boxes. It avoids intermediate texture and color space conversion passes by rendering directly to the final target [S9, unofficial].

### Per-pixel Depth Comparison for Transparency
Resolves popping artifacts caused by depth-sorting swaps by replacing global sorting with per-pixel depth comparison or ray-tracing [S10].

### Near-plane Culling
Filters out Gaussian points behind or too close to the near clipping plane using axis-aligned bounding boxes (AABB) before projection. The culling condition is $z_{\max} < z_{\text{near}}$. This hardware unit design uses a single MAC unit, achieving a 56% culling rate and increasing throughput from 15.81% to 35.94% (2.27× gain) at 1080p [S11].

### Radiance Field Reconstruction from Gaussians
Renders RGB, depth, alpha, and normal maps from Gaussian splats. It initializes a background color tensor, iterates over viewpoints, extracts RGB, alpha, depth, and normals from the render package, and estimates the scene's bounding sphere [S12, unofficial].

## Operating characteristics
*   **Time Complexity**: Dominated by projection and rasterization. Projection is $O(N)$ per Gaussian. Rasterization is $O(N \cdot T)$ where $T$ is the number of tiles, but optimized via tile-Gaussian intersection to only process visible Gaussians [S5, unofficial].
*   **Memory Complexity**: $O(N)$ for Gaussian attributes. The `screenspace_points` tensor requires additional memory for gradients [S2, unofficial].
*   **Performance**: Real-time performance on high-end GPUs (e.g., 100+ FPS at 1080p with millions of Gaussians) [S1]. On Apple Silicon via MLX, forward pass for 100K Gaussians at 256x256 takes ~62.6ms for projection and ~29.8ms for covariance [S5, unofficial].
*   **Strengths**: High rendering speed, differentiable, explicit scene representation, view-dependent appearance via SH [S1].
*   **Limitations**: High memory usage for large scenes (gigabytes), potential popping artifacts from depth sorting, limited integration with standard hardware rasterization pipelines without custom shaders [S1, S10].
*   **When to Use**: Novel view synthesis, real-time rendering of dynamic scenes, applications requiring differentiable rendering for optimization [S1].
*   **When Not to Use**: Low-end devices with limited VRAM without compression, environments requiring seamless integration with polygon-based hardware rasterization without custom pipelines [S1].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S1]. It builds upon EWA volume splatting [34] to efficiently compute the footprint of projected Gaussians [S1]. The method models a scene as a set of 3D Gaussians with shape and appearance parameters, optimized via differentiable rendering to match recorded images [S1]. It has become a backbone for modern 3DGS research, powering projects like SplaTAM, MonoGS, WildGS-SLAM, and nerfstudio [S5, unofficial]. Variants include compressed 3DGS for accelerated synthesis [S1], neural Gaussian rasterization [S1], and hardware accelerators [S11].

## Sources
[S1] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
[S2] ldyken53/VEG, https://github.com/ldyken53/VEG, fetched 2026-10-26 (unofficial)
[S3] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S4] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S5] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
[S6] ashu1069/3D-Gaussian-Splatting-for-Novel-View-Synthesis, https://github.com/ashu1069/3D-Gaussian-Splatting-for-Novel-View-Synthesis, fetched 2026-10-27 (unofficial)
[S7] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
[S8] 3DAgentWorld/Advanced3DGS, https://github.com/3DAgentWorld/Advanced3DGS, fetched 2026-10-27 (unofficial)
[S9] wuyize25/gsplat-unity, https://github.com/wuyize25/gsplat-unity, fetched 2026-10-27 (unofficial)
[S10] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S11] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
[S12] Zhaoyibinn/VGVGGT, https://github.com/Zhaoyibinn/VGVGGT, fetched 2026-10-27 (unofficial)
