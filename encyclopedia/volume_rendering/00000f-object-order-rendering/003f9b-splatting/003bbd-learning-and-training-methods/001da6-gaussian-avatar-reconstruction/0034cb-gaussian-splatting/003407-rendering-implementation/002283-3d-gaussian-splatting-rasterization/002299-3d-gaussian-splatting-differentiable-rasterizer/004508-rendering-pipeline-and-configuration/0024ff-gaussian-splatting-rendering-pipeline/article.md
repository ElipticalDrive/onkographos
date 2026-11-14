# Gaussian Splatting Rendering Pipeline

Also called: 3D Gaussian Splatting Rendering Pipeline, Gaussian Splatting Rasterization and Rendering Pipeline

## Overview
The Gaussian Splatting Rendering Pipeline orchestrates the differentiable rasterization of a scene represented as a set of 3D Gaussians into a 2D image for a specific camera viewpoint [S1][S2, S3, S4, unofficial]. It solves the problem of efficiently rendering millions of adaptive volumetric primitives by projecting 3D covariances to 2D, evaluating view-dependent colors via Spherical Harmonics (SH), and compositing the results using alpha blending [S1][S6, unofficial]. It sets itself apart from standard polygon rasterization by handling soft, overlapping volumetric primitives with depth-sorted alpha compositing, and from NeRF-style volume rendering by using explicit, sparse Gaussian kernels that allow for real-time performance on consumer GPUs [S1].

## How it works
The pipeline takes a set of 3D Gaussian parameters and camera extrinsics/intrinsics, and produces a rendered image along with auxiliary buffers (depth, normals, opacity) and visibility masks.

### Inputs and Outputs
**Inputs:**
*   `pc`: A Gaussian model object containing:
    *   `xyz`: 3D positions, shape $(N, 3)$ [S6, unofficial].
    *   `features_dc`, `features_rest`: SH coefficients, shape $(N, 3)$ and $(N, 45)$ for degree 3, or combined as `features` shape $(N, K, 3)$ where $K=(D+1)^2$ [S6, S5, unofficial].
    *   `opacity`: Raw opacity values, shape $(N)$ [S6, unofficial].
    *   `scaling`: Log-scale parameters, shape $(N, 3)$ [S6, unofficial].
    *   `rotation`: Quaternions $(w, x, y, z)$, shape $(N, 4)$ [S6, unofficial].
    *   `active_sh_degree`: The maximum SH degree currently active for training [S4, unofficial].
*   `viewpoint_camera`: Camera object containing:
    *   `world_view_transform`: $4 \times 4$ view matrix $W$ [S4, unofficial].
    *   `full_proj_transform`: $4 \times 4$ projection matrix [S4, unofficial].
    *   `camera_center`: Camera position in world space [S4, unofficial].
    *   `FoVx`, `FoVy`: Field of view angles in radians [S4, unofficial].
    *   `image_width`, `image_height`: Output resolution [S4, unofficial].
*   `pipe`: Pipeline configuration object [S4, unofficial]:
    *   `convert_SHs_python`: Boolean. If true, compute SH colors in Python before rasterization [S3, unofficial].
    *   `compute_cov3D_python`: Boolean. If true, compute 3D covariance matrices in Python [S3, unofficial].
    *   `depth_ratio`: Float. Weight for blending expected and median depth [S3, unofficial].
    *   `debug`: Boolean. Enables gradient retention checks [S4, unofficial].
*   `bg_color`: Background color tensor, shape $(3)$, typically white $(1,1,1)$ or black $(0,0,0)$ [S4, unofficial].
*   `override_color`: Optional tensor $(N, 3)$ to bypass SH evaluation [S4, unofficial].

**Outputs:**
*   `render`: Rendered image tensor, shape $(3, H, W)$ or $(C, H, W)$ depending on mode [S4, S5, unofficial].
*   `viewspace_points`: Tensor of 2D screen-space means, shape $(N, 3)$, with gradients enabled [S4, unofficial].
*   `visibility_filter`: Boolean tensor $(N)$, true if Gaussian radius $> 0$ [S4, unofficial].
*   `radii`: Integer tensor $(N)$, screen-space radius of each Gaussian [S4, unofficial].
*   Optional: `render_depth`, `rend_alpha`, `rend_normal`, `surf_normal` [S3, S12, unofficial].

### Data Structures
*   **Gaussian Attributes**: Stored in contiguous CUDA tensors. `xyz` $(N, 3)$, `quats` $(N, 4)$, `scales` $(N, 3)$, `opacities` $(N)$ [S5, unofficial].
*   **Covariance Representation**:
    *   Parametric: `scales` $(N, 3)$ and `rotations` $(N, 4)$ [S2, unofficial].
    *   Precomputed: `cov3D_precomp` $(N, 9)$, storing the upper triangle of the symmetric $3 \times 3$ covariance matrix in row-major order [S3, unofficial].
*   **Color Representation**:
    *   SH: `shs` $(N, K, 3)$ [S5, unofficial].
    *   Precomputed RGB: `colors_precomp` $(N, 3)$ [S4, unofficial].
*   **Rasterization Settings**:
    *   `image_height`, `image_width` [S4, unofficial].
    *   `tanfovx`, `tanfovy` [S4, unofficial].
    *   `viewmatrix`, `projmatrix` [S4, unofficial].
    *   `campos` [S4, unofficial].
    *   `sh_degree` [S4, unofficial].
    *   `scale_modifier`: Default $1.0$ [S3, unofficial].
    *   `near_plane`: Default $0.01$ [S5, unofficial].
    *   `far_plane`: Default $10^{10}$ [S5, unofficial].
    *   `tile_size`: Default $16$ [S5, unofficial].
    *   `eps2d`: Regularization for 2D covariance diagonal, default $0.3$ [S5, unofficial].

### Steps

1.  **Initialize Screenspace Points**:
    Create a zero tensor `screenspace_points` with shape $(N, 3)$, same dtype and device as `xyz`, with `requires_grad=True` [S2, S4, unofficial]. Call `retain_grad()` if `pipe.debug` is true to ensure gradients are retained for this tensor during backpropagation [S4, S7, unofficial].

2.  **Compute Camera Parameters**:
    Calculate half-field-of-view tangents:
    $$ \text{tanfovx} = \tan(\text{viewpoint\_camera.FoVx} / 2) $$
    $$ \text{tanfovy} = \tan(\text{viewpoint\_camera.FoVy} / 2) $$
    [S4, S7, unofficial].
    Extract camera position `campos` from the view matrix $W = [R | t; 0 \ 1]$ as $\text{campos} = -R^T t$ [S5, unofficial].

3.  **Select Covariance Representation**:
    *   If `pipe.compute_cov3D_python` is true:
        Compute the 3D covariance matrix $\Sigma_{3D}$ from `scales` and `rotations` in Python. Transform to pixel space using the Jacobian of the projection and the view matrix. The resulting `cov3D_precomp` is reshaped to $(N, 9)$ [S3, unofficial].
    *   Else:
        Set `cov3D_precomp` to `None`. Assign `scales` from `pc.get_scaling()` and `rotations` from `pc.get_rotation()` to be processed by the CUDA rasterizer [S3, S4, unofficial].

4.  **Select Color Representation**:
    *   If `override_color` is not `None`:
        Set `colors_precomp` = `override_color` and `shs` = `None` [S4, unofficial].
    *   Else if `pipe.convert_SHs_python` is true:
        Extract SH coefficients `shs_view` from `pc.get_features`.
        Compute view directions `dir_pp` from each Gaussian center to the camera center:
        $$ \text{dir\_pp} = \text{pc.xyz} - \text{viewpoint\_camera.camera\_center.repeat}(N, 1) $$
        Normalize: $\text{dir\_pp\_normalized} = \text{dir\_pp} / \|\text{dir\_pp}\|_2$ [S4, unofficial].
        Evaluate SH: $\text{sh2rgb} = \text{eval\_sh}(\text{pc.active\_sh\_degree}, \text{shs\_view}, \text{dir\_pp\_normalized})$ [S4, unofficial].
        Clamp to valid range: $\text{colors\_precomp} = \max(\text{sh2rgb} + 0.5, 0.0)$ [S4, unofficial].
        Set `shs` = `None` [S4, unofficial].
    *   Else:
        Set `shs` = `pc.get_features()` and `colors_precomp` = `None` [S4, unofficial].

5.  **Configure Rasterization Settings**:
    Construct `GaussianRasterizationSettings` with:
    *   `image_height`, `image_width`
    *   `tanfovx`, `tanfovy`
    *   `bg` (background color)
    *   `scale_modifier` (default 1.0)
    *   `viewmatrix`, `projmatrix`
    *   `sh_degree`
    *   `campos`
    *   `prefiltered=False`
    *   `debug` flag [S4, unofficial].

6.  **Instantiate Rasterizer**:
    Create the `GaussianRasterizer` instance using the configured `raster_settings` [S4, S7, unofficial].

7.  **Invoke Rasterizer**:
    Call the rasterizer with the following arguments:
    *   `means3D`: `pc.get_xyz()`
    *   `means2D`: `screenspace_points`
    *   `shs`: (from step 4)
    *   `colors_precomp`: (from step 4)
    *   `opacities`: `pc.get_opacity()`
    *   `scales`: (from step 3, if not precomputed)
    *   `rotations`: (from step 3, if not precomputed)
    *   `cov3D_precomp`: (from step 3, if precomputed)
    [S4, S7, unofficial].

    The rasterizer performs:
    *   **Projection**: Transforms 3D Gaussians to camera space and projects to 2D. The 2D covariance is computed as $\Sigma_{2D} = J \Sigma_{3D}^{camera} J^T$, where $J$ is the Jacobian of the affine approximation of the projective transformation [S1][S6, unofficial].
    *   **Tiling**: Divides the image into tiles (default $16 \times 16$) for efficient parallel processing [S6, unofficial].
    *   **Intersection**: Computes tile-Gaussian intersections [S5, unofficial].
    *   **Rasterization**: Evaluates Gaussian contributions per pixel within each tile, sorting them by depth [S6, unofficial].
    *   **Alpha Blending**: Composites Gaussians front-to-back. For a pixel, the final color $C$ is:
        $$ C = \sum_{i \in N} \alpha_i c_i \prod_{j < i} (1 - \alpha_j) $$
        where $N$ is the set of 2D Gaussians contributing to the pixel, sorted by depth, $c_i$ is the view-dependent color, and $\alpha_i$ is the opacity [S1][S6, unofficial].

8.  **Post-Processing (Optional Depth/Normals)**:
    *   If depth is requested:
        Extract view direction `projvect1` (first 3 elements of `world_view_transform[:,2]`) and offset `projvect2` (last element) [S7, unofficial].
        Compute depth values for means: $\text{means3D\_depth} = \text{dot}(\text{means3D}, \text{projvect1}) + \text{projvect2}$ [S7, unofficial].
        Repeat depth values 3 times to match RGB channels.
        Rasterize again with `shs=None`, `colors_precomp=means3D_depth` [S7, unofficial].
        Average the resulting depth image over channels: $\text{render\_depth} = \text{render\_depth.mean(dim=0)}$ [S7, unofficial].
    *   If `pipe.depth_ratio` is used, blend expected and median depth:
        $$ \text{surf\_depth} = \text{render\_depth\_expected} \times (1 - \text{pipe.depth\_ratio}) + \text{pipe.depth\_ratio} \times \text{render\_depth\_median} $$
        [S3, unofficial].

9.  **Return Results**:
    Construct a dictionary containing:
    *   `render`: The rendered image.
    *   `viewspace_points`: The `screenspace_points` tensor.
    *   `visibility_filter`: Boolean tensor `radii > 0` [S4, unofficial].
    *   `radii`: Screen-space radii of visible Gaussians [S4, unofficial].
    *   Additional buffers if requested (alpha, normal, depth) [S3, S12, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def render_gaussian_splatting(viewpoint_camera, pc, pipe, bg_color, override_color=None):
    """
    Renders a scene of 3D Gaussians to a 2D image.
    
    Args:
        viewpoint_camera: Object with FoVx, FoVy, world_view_transform, full_proj_transform, 
                          camera_center, image_width, image_height.
        pc: GaussianModel object with get_xyz, get_opacity, get_scaling, get_rotation, 
            get_features, active_sh_degree, max_sh_degree.
        pipe: Pipeline config with convert_SHs_python, compute_cov3D_python, depth_ratio, debug.
        bg_color: Tensor (3,) background color.
        override_color: Optional Tensor (N, 3) to bypass SH.
        
    Returns:
        dict: 'render', 'viewspace_points', 'visibility_filter', 'radii', and optionally 'render_depth'.
    """
    N = pc.get_xyz().shape[0]
    
    # 1. Initialize screenspace points
    screenspace_points = torch.zeros(N, 3, dtype=pc.get_xyz().dtype, device=pc.get_xyz().device, requires_grad=True)
    if pipe.debug:
        screenspace_points.retain_grad()
        
    # 2. Compute camera parameters
    tanfovx = math.tan(viewpoint_camera.FoVx * 0.5)
    tanfovy = math.tan(viewpoint_camera.FoVy * 0.5)
    
    # Extract camera position from view matrix W = [R|t; 0 1]
    R = viewpoint_camera.world_view_transform[:3, :3]
    t = viewpoint_camera.world_view_transform[:3, 3]
    campos = -R.T @ t
    
    # 3. Select Covariance Representation
    if pipe.compute_cov3D_python:
        # Compute 3D covariance in Python
        # This involves converting scales/rotations to Sigma_3D and transforming to pixel space
        # Placeholder for complex matrix math defined in pc.get_covariance
        cov3D_precomp = pc.get_covariance(scaling_modifier=1.0) 
        # Reshape to (N, 9) if necessary, assuming get_covariance returns upper triangle
        cov3D_precomp = cov3D_precomp.reshape(N, 9)
        scales = None
        rotations = None
    else:
        cov3D_precomp = None
        scales = pc.get_scaling()
        rotations = pc.get_rotation()
        
    # 4. Select Color Representation
    if override_color is not None:
        colors_precomp = override_color
        shs = None
    else:
        if pipe.convert_SHs_python:
            # Extract SH coefficients
            # Features shape: (N, 3) for DC + (N, 45) for rest -> (N, 48, 3) or similar
            # View depends on implementation, assuming standard 3DGS layout
            shs_view = pc.get_features().transpose(1, 2).view(-1, 3, (pc.max_sh_degree + 1)**2)
            
            # Compute view directions
            dir_pp = pc.get_xyz() - viewpoint_camera.camera_center.repeat(N, 1)
            dir_pp_normalized = dir_pp / dir_pp.norm(dim=1, keepdim=True)
            
            # Evaluate SH
            sh2rgb = eval_sh(pc.active_sh_degree, shs_view, dir_pp_normalized)
            
            # Clamp to [0, 1]
            colors_precomp = torch.clamp_min(sh2rgb + 0.5, 0.0)
            shs = None
        else:
            shs = pc.get_features()
            colors_precomp = None
            
    # 5. Configure Rasterization Settings
    raster_settings = GaussianRasterizationSettings(
        image_height=viewpoint_camera.image_height,
        image_width=viewpoint_camera.image_width,
        tanfovx=tanfovx,
        tanfovy=tanfovy,
        bg=bg_color,
        scale_modifier=1.0,
        viewmatrix=viewpoint_camera.world_view_transform,
        projmatrix=viewpoint_camera.full_proj_transform,
        sh_degree=pc.active_sh_degree,
        campos=viewpoint_camera.camera_center,
        prefiltered=False,
        debug=pipe.debug
    )
    
    # 6. Instantiate Rasterizer
    rasterizer = GaussianRasterizer(raster_settings=raster_settings)
    
    # 7. Invoke Rasterizer
    means3D = pc.get_xyz()
    opacity = pc.get_opacity()
    
    rendered_image, radii = rasterizer(
        means3D=means3D,
        means2D=screenspace_points,
        shs=shs,
        colors_precomp=colors_precomp,
        opacities=opacity,
        scales=scales,
        rotations=rotations,
        cov3D_precomp=cov3D_precomp
    )
    
    # 8. Post-Processing (Depth)
    result = {
        'render': rendered_image,
        'viewspace_points': screenspace_points,
        'visibility_filter': radii > 0,
        'radii': radii
    }
    
    # Optional: Render Depth if requested by caller or pipe config
    # This requires a second rasterization pass or specific output flags
    # Not included in standard minimal pipeline but supported in extended versions [S7, unofficial]
    
    return result

def eval_sh(deg, shs, dirs):
    """
    Evaluates Spherical Harmonics.
    deg: active SH degree
    shs: Tensor (N, K, 3)
    dirs: Tensor (N, 3) normalized view directions
    """
    # Implementation depends on SH basis evaluation
    # Standard 3DGS uses degree 3 (16 coefficients)
    # This is a placeholder for the actual SH evaluation math
    pass
```

### Parameters and Typical Values
*   `tile_size`: 16 pixels [S5, S6, unofficial].
*   `near_plane`: 0.01 [S5, unofficial] or 0.002 [S6, unofficial].
*   `far_plane`: $10^{10}$ [S5, unofficial] or 100 [S6, unofficial].
*   `eps2d`: 0.3 (regularization for 2D covariance diagonal) [S5, unofficial].
*   `alpha_max`: 0.99 (maximum alpha to prevent instability) [S6, unofficial].
*   `alpha_cutoff`: $1/255$ (early termination threshold) [S6, unofficial].
*   `chi_square_clip`: 9.21 (approx 3$\sigma$ for covariance regularization) [S6, unofficial].
*   `pix_guard`: 64 pixels (culling guard band) [S6, unofficial].
*   `scale_modifier`: 1.0 [S3, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: Lower precision (e.g., 8-bit) may be required for inference on low-end devices, but training typically uses 32-bit for stable gradients [S1].
*   **Empty Tensors**: `None` inputs must be converted to empty tensors `torch.Tensor([])` or shape $(N, 0)$ to prevent `None` propagation into CUDA kernels [Parent].
*   **Mutual Exclusivity**: Exactly one of `shs` or `colors_precomp` must be provided. Exactly one of (`scales`, `rotations`) or `cov3D_precomp` must be provided. Violating this raises an exception [Parent].
*   **Device Consistency**: All input tensors must reside on the same device (typically CUDA) [Parent].
*   **Popping Artifacts**: Depth-sorting swaps can cause temporal instability. Per-pixel depth comparison or ray-tracing can resolve this [S10].
*   **Near-plane Culling**: Gaussians behind the near plane should be culled to save computation. Condition: $z_{\max} < z_{\text{near}}$ [S11].

## Subtopics

### [Neural Gaussian Rasterization Pipeline](7419-neural-gaussian-rasterization-pipeline.md)
Replaces the standard exponential opacity formulation with direct neural network outputs for per-Gaussian opacity and color, while retaining the standard 2D Gaussian rasterizer for compositing. It computes view-dependent color and density ($\sigma$) for each Gaussian center, maps $\sigma$ directly to opacity, and composites the result using standard alpha blending.

### Octree-GS Rendering
Uses an octree structure to manage Gaussian visibility. It initializes a `GaussianModel` with dataset parameters, sets an anchor mask for the current view, prefilters voxels to determine visibility, and renders only the visible Gaussians [S8, unofficial].

### Unity 3DGS Rendering Pipeline Integration
Integrates 3DGS into the Unity engine by inserting an additional sorting pass into each camera's command buffer and drawing calls for 3DGS objects into the transparent mesh queue based on bounding boxes. It avoids intermediate texture and color space conversion passes [S9, unofficial].

### Per-pixel Depth Comparison for Transparency
Resolves popping artifacts caused by depth-sorting swaps by replacing global sorting with per-pixel depth comparison or ray-tracing [S10].

### Near-plane Culling
A hardware-accelerated culling method using axis-aligned bounding boxes (AABB) to check depth and remove Gaussian points behind the camera near-plane. It checks $z_{\max} < z_{\text{near}}$ where $z_{\max} = z + \Delta z$ [S11].

### Radiance Field Reconstruction from Gaussians
Renders RGB, depth, alpha, and normal maps from a set of Gaussian splats. It extracts surface normals from rendered normals or depth, and estimates the scene bounding sphere [S12, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ for projection and SH evaluation, where $N$ is the number of Gaussians. Rasterization is $O(N \cdot T)$ where $T$ is the number of tiles, but effectively linear with respect to visible Gaussians per pixel due to early termination [S5, unofficial].
*   **Memory Complexity**: $O(N)$ to store Gaussian parameters. The output image is $O(H \cdot W)$ [S1].
*   **Performance**: Real-time performance on high-end GPUs (e.g., 100+ FPS at 1080p) is achievable with millions of Gaussians [S1]. On Apple Silicon via MLX, 100K Gaussians at 256x256 resolution take ~62.6ms for projection and ~29.8ms for covariance [S5, unofficial].
*   **Strengths**: High rendering speed, differentiable, supports view-dependent appearance via SH [S1].
*   **Limitations**: High memory usage for large scenes (gigabytes for millions of Gaussians), which can be problematic on low-end devices or VR/AR headsets [S1]. Sensitive to depth-sorting artifacts [S10].
*   **When to Use**: Novel view synthesis, real-time rendering of static or dynamic scenes where high fidelity and speed are required.
*   **When Not to Use**: Scenes with extremely high Gaussian counts on memory-constrained devices without compression, or when hardware rasterization of polygons is strictly required without integration [S1].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S1, S13]. It builds upon EWA volume splatting [S1] to efficiently compute the 2D footprint of projected Gaussians. The method models a scene as a set of 3D Gaussians with shape and appearance parameters, optimized via differentiable rendering to match recorded images [S1]. The rendering pipeline was designed to enable real-time performance on GPUs, contrasting with slower NeRF-based methods [S1]. Variants and ports include gsplat-mlx for Apple Silicon [S5, unofficial], Unity integration [S9, unofficial], and hardware accelerators [S11].

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
