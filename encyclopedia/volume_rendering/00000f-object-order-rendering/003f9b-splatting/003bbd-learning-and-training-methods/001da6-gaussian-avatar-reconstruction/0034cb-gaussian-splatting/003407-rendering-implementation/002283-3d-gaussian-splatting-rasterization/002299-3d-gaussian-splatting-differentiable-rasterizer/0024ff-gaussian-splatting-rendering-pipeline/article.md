# Gaussian Splatting Rendering Pipeline

Also called: 2D Gaussian Splatting Forward Pass, 2D Gaussian Splatting Renderer, 3D Gaussian Splatting Rendering Pipeline, Gaussian Rasterization Rendering, Gaussian Splatting Rasterization and Rendering Pipeline, GaussianRasterizer, gaussian_renderer/__init__.py render, gaussian_renderer/__init__3dgs.py render

## Overview
The Gaussian Splatting Rendering Pipeline orchestrates the final stage of 3D Gaussian Splatting, converting a set of learned 3D Gaussian primitives into a 2D image for a specific camera view [S1, S3, S4, unofficial]. It solves the problem of differentiable rasterization by projecting 3D Gaussians to screen space, evaluating view-dependent appearance, and compositing them front-to-back to produce color, depth, and auxiliary maps [S2, S4, unofficial]. It differs from its siblings by providing the complete high-level orchestration (camera setup, SH evaluation, feature assembly, and rasterization invocation) rather than just the low-level kernel logic [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
The pipeline takes a set of $N$ 3D Gaussians and camera parameters for $C$ views [S3, S4, unofficial].
*   **Gaussian Parameters:**
    *   `means3D`: Position tensor, shape $[N, 3]$ [S3, S4, unofficial].
    *   `scales`: Log-scale tensor, shape $[N, 3]$ [S4, unofficial].
    *   `rotations`: Quaternion tensor, shape $[N, 4]$ (w, x, y, z) [S3, S4, unofficial].
    *   `opacities`: Scalar tensor, shape $[N]$ [S3, S4, unofficial].
    *   `features`: Spherical Harmonics (SH) coefficients tensor, shape $[N, K, 3]$ where $K=(d+1)^2$ for degree $d$, or direct RGB colors $[N, 3]$ [S3, S4, unofficial].
*   **Camera Parameters:**
    *   `viewmats`: World-to-camera transformation matrices, shape $[C, 4, 4]$ [S3, unofficial].
    *   `Ks`: Camera intrinsic matrices, shape $[C, 3, 3]$ [S3, unofficial].
    *   `image_dims`: Height $H$ and Width $W$ [S2, unofficial].
*   **Outputs:**
    *   `rendered_image`: Color tensor, shape $[C, H, W, 3]$ [S3, unofficial].
    *   `rendered_opacity`: Alpha tensor, shape $[C, H, W, 1]$ [S3, unofficial].
    *   `rendered_depth`: Accumulated projection depth or hit distance [S3, unofficial].
    *   `rendered_feature`: Concatenated feature maps (normals, depth components, etc.) [S1, S2, unofficial].

**Data Structures:**
The pipeline uses `GaussianRasterizationSettings` to bundle camera and rendering configuration [S2, unofficial]. Key fields include:
*   `image_width`, `image_height`: Resolution [S2, unofficial].
*   `tanfovx`, `tanfovy`: Tangents of horizontal and vertical field of view [S1, S2, unofficial].
*   `cx`, `cy`: Principal point coordinates [S1, unofficial].
*   `background`: Background color tensor $[3]$ [S2, unofficial].
*   `scale_modifier`: Multiplier for Gaussian scales (default 1.0) [S2, unofficial].
*   `viewmat`, `projmat`: View and projection matrices [S2, unofficial].
*   `sh_degree`: Active SH degree (None for direct color) [S3, unofficial].
*   `campos`: Camera position in world space [S2, unofficial].
*   `near_plane`, `far_plane`: Clipping distances (default 0.01, 1e10) [S3, unofficial].
*   `tile_size`: Tile side length for rasterization (default 16) [S3, unofficial].
*   `eps2d`: Regularization added to 2D covariance diagonal (default 0.3) [S3, unofficial].
*   `pix_guard`: Pixel guard band width for culling (default 64) [S4, unofficial].
*   `alpha_max`: Maximum alpha value to prevent instability (default 0.99) [S4, unofficial].
*   `alpha_cutoff`: Alpha cutoff for early termination (default 1/255) [S4, unofficial].
*   `chi_square_clip`: Chi-square clipping threshold for covariance regularization (default 9.21) [S4, unofficial].

**Steps:**

1.  **Covariance Reconstruction:**
    Convert log-scales and quaternions into 3D covariance matrices $\Sigma_{3D}$ [S3, S4, unofficial].
    $$ \Sigma_{3D} = R \cdot S \cdot S^T \cdot R^T $$
    Where $R$ is the rotation matrix derived from the quaternion and $S$ is the diagonal scale matrix [G].
    *   *Implementation Detail:* If `pipe.compute_cov3D_python` is true, this is computed in Python using `pc.get_covariance(scaling_modifier)` [S1, S2, unofficial]. Otherwise, scales and rotations are passed directly to the rasterizer [S2, unofficial].

2.  **Projection to 2D:**
    Transform 3D Gaussians to camera space and project to 2D screen space [S4, unofficial].
    *   Transform mean $\mu_{3D}$ to camera space: $\mu_{cam} = R_{w2c} \cdot (\mu_{3D} - t_{w2c})$ [S4, unofficial].
    *   Project the 3D covariance to 2D using the Jacobian $J$ of the perspective projection transformation:
        $$ \Sigma_{2D} = J \cdot \Sigma_{3D_{cam}} \cdot J^T $$
        [S5, unofficial].
    *   Add regularization: $\Sigma_{2D} \leftarrow \Sigma_{2D} + \epsilon_{2d} I$ [S3, unofficial].
    *   Compute 2D means (screen coordinates) from $\mu_{cam}$ [S4, unofficial].

3.  **Spherical Harmonics Evaluation (Optional):**
    If SH degree > 0 and `pipe.convert_SHs_python` is false, the rasterizer handles SH evaluation. If `pipe.convert_SHs_python` is true, compute colors in Python [S2, unofficial]:
    *   Compute view directions: $v = \frac{\mu_{3D} - \text{campos}}{\|\mu_{3D} - \text{campos}\|}$ [S2, unofficial].
    *   Evaluate SH: $c = \text{eval\_sh}(d, \text{features}, v)$ [S2, unofficial].
    *   Clamp: $\text{colors\_precomp} = \max(c + 0.5, 0.0)$ [S2, unofficial].
    *   *Note:* If `override_color` is provided, use it instead [S1, unofficial].

4.  **Tile-Gaussian Intersection:**
    Divide the image into tiles of size `tile_size` x `tile_size` [S3, unofficial].
    *   The rasterizer determines which tiles intersect with each Gaussian's projected footprint [S3, unofficial].
    *   Sort Gaussians within each tile by depth (z-coordinate in camera space) for front-to-back rendering [S4, unofficial].

5.  **Rasterization and Alpha Compositing:**
    For each pixel in each tile, composite the contributions of overlapping Gaussians [S4, unofficial].
    *   For each Gaussian $i$ in the tile (sorted front-to-back):
        *   Compute 2D Gaussian value at pixel $(u, v)$:
            $$ \alpha_i = \exp\left(-\frac{1}{2} (\mathbf{x} - \mu_i)^T \Sigma_{2D_i}^{-1} (\mathbf{x} - \mu_i)\right) $$
        *   Apply opacity: $\alpha_i \leftarrow \alpha_i \cdot \text{opacity}_i$ [S4, unofficial].
        *   Clamp $\alpha_i$ to $[0, \alpha_{max}]$ [S4, unofficial].
        *   Composite:
            $$ C_{final} = \sum_i \alpha_i c_i \prod_{j<i} (1 - \alpha_j) $$
            $$ \alpha_{final} = \sum_i \alpha_i \prod_{j<i} (1 - \alpha_j) $$
            [S5, unofficial].
    *   Early termination: Stop compositing if accumulated alpha exceeds $1 - \alpha_{cutoff}$ [S4, unofficial].
    *   Add background color for pixels with $\alpha_{final} < 1$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class GaussianRasterizationSettings:
    def __init__(self, image_width, image_height, tanfovx, tanfovy, cx, cy,
                 background, scale_modifier, viewmat, projmat, sh_degree,
                 campos, near_plane, far_plane, eps2d, tile_size, alpha_max,
                 alpha_cutoff, chi_square_clip, pix_guard):
        self.image_width = image_width
        self.image_height = image_height
        self.tanfovx = tanfovx
        self.tanfovy = tanfovy
        self.cx = cx
        self.cy = cy
        self.background = background
        self.scale_modifier = scale_modifier
        self.viewmat = viewmat
        self.projmat = projmat
        self.sh_degree = sh_degree
        self.campos = campos
        self.near_plane = near_plane
        self.far_plane = far_plane
        self.eps2d = eps2d
        self.tile_size = tile_size
        self.alpha_max = alpha_max
        self.alpha_cutoff = alpha_cutoff
        self.chi_square_clip = chi_square_clip
        self.pix_guard = pix_guard

def quaternion_to_rotation_matrix(quats):
    """Convert batch of quaternions [N, 4] (w,x,y,z) to rotation matrices [N, 3, 3]."""
    w, x, y, z = quats[:, 0], quats[:, 1], quats[:, 2], quats[:, 3]
    R = math.zeros((quats.shape[0], 3, 3))
    R[:, 0, 0] = 1 - 2*(y*y + z*z)
    R[:, 0, 1] = 2*(x*y - w*z)
    R[:, 0, 2] = 2*(x*z + w*y)
    R[:, 1, 0] = 2*(x*y + w*z)
    R[:, 1, 1] = 1 - 2*(x*x + z*z)
    R[:, 1, 2] = 2*(y*z - w*x)
    R[:, 2, 0] = 2*(x*z - w*y)
    R[:, 2, 1] = 2*(y*z + w*x)
    R[:, 2, 2] = 1 - 2*(x*x + y*y)
    return R

def eval_sh(degree, features, directions):
    """Evaluate spherical harmonics for given directions [N, 3] and coefficients [N, K, 3]."""
    # Standard SH basis evaluation up to degree `degree`
    # Returns RGB colors [N, 3]
    # Implementation follows standard SH basis polynomials [G]
    return features @ directions[:, None, :] # Simplified placeholder for standard SH eval [G]

def render_pipeline(means3D, scales, rotations, opacities, features, 
                    viewmats, Ks, image_dims, pipe, background):
    C, H, W = len(viewmats), image_dims[0], image_dims[1]
    N = len(means3D)
    
    # 1. Setup Rasterization Settings
    # tanfovx/tanfovy computed from FOV, not intrinsics directly [S1, S3, S6, unofficial]
    settings = GaussianRasterizationSettings(
        image_width=W, image_height=H,
        tanfovx=math.tan(pipe.FoVx * 0.5), tanfovy=math.tan(pipe.FoVy * 0.5),
        cx=Ks[0, 0, 2], cy=Ks[0, 1, 2],
        background=background, scale_modifier=pipe.scale_modifier,
        viewmat=viewmats[0], projmat=Ks[0] @ viewmats[0],
        sh_degree=pipe.sh_degree,
        campos=-viewmats[0, :3, :3].T @ viewmats[0, :3, 3], # Camera position [S4, unofficial]
        near_plane=pipe.near_plane, far_plane=pipe.far_plane,
        eps2d=pipe.eps2d, tile_size=pipe.tile_size,
        alpha_max=pipe.alpha_max, alpha_cutoff=pipe.alpha_cutoff,
        chi_square_clip=pipe.chi_square_clip, pix_guard=pipe.pix_guard
    )
    
    # 2. Precompute 3D Covariances if needed
    if pipe.compute_cov3D_python:
        cov3D_precomp = compute_cov3D_python(means3D, scales, rotations, pipe.scale_modifier)
    else:
        cov3D_precomp = None
        
    # 3. Precompute Colors if SH evaluation is in Python
    if pipe.convert_SHs_python and features.shape[1] > 1:
        R_w2c = viewmats[0, :3, :3]
        t_w2c = viewmats[0, :3, 3]
        campos = -R_w2c.T @ t_w2c
        
        dir_pp = means3D - campos
        dir_pp_normalized = dir_pp / math.norm(dir_pp, axis=1, keepdims=True)
        
        shs_view = features.transpose(1, 2).reshape(-1, 3, features.shape[1])
        colors_precomp = eval_sh(pipe.sh_degree, shs_view, dir_pp_normalized)
        colors_precomp = math.clamp_min(colors_precomp + 0.5, 0.0)
    else:
        colors_precomp = None
        
    # 4. Initialize screenspace points for gradient storage
    screenspace_points = math.zeros_like(means3D, requires_grad=True, device=means3D.device)
    try:
        screenspace_points.retain_grad()
    except:
        pass # Framework-specific gradient retention [S1, S2, S6, unofficial]
        
    # 5. Invoke Rasterizer (Tiling, Intersection, Sorting, Compositing)
    # The rasterizer handles projection, tiling, intersection, and compositing [S3, S4, unofficial]
    num_rendered, num_contrib, rendered_image, rendered_opacity, rendered_depth, \
        rendered_feature, rendered_pseudo_normal, rendered_surface_xyz, radii = \
        rasterizer(
            means3D=means3D, means2D=screenspace_points,
            shs=features if colors_precomp is None else None,
            colors_precomp=colors_precomp, opacities=opacities,
            scales=scales, rotations=rotations, cov3D_precomp=cov3D_precomp,
            settings=settings
        )
        
    # 6. Post-process features if needed
    if rendered_feature is not None:
        # Split feature tensor based on channel widths [S1, unofficial]
        rendered_phong, rendered_normal, rendered_offset_color, \
        rendered_diffuse_factor, rendered_shininess, rendered_ambient_factor, \
        rendered_specular_factor, rendered_offset_color_norm = \
            split_features(rendered_feature, [3, 3, 3, 1, 1, 1, 1, 1])
            
    # 7. Depth Rendering (Optional) [S6, unofficial]
    # If return_depth is true, compute depth by rasterizing means3D_depth
    # means3D_depth = dot(means3D, projvect1) + projvect2, repeated 3 times
    # Rasterize again with shs=None, colors_precomp=means3D_depth
    # Average resulting depth image over channels
    pass 

def compute_cov3D_python(means3D, scales, rotations, scale_modifier):
    linear_scales = math.exp(scales) * scale_modifier
    rotations_mat = quaternion_to_rotation_matrix(rotations)
    S = math.diag_embed(linear_scales)
    cov3D = rotations_mat @ S @ S.transpose(1, 2) @ rotations_mat.transpose(1, 2)
    return cov3D

def split_features(feature_tensor, channel_widths):
    start = 0
    components = []
    for width in channel_widths:
        end = start + width
        components.append(feature_tensor[..., start:end])
        start = end
    return components

def rasterizer(means3D, means2D, shs, colors_precomp, opacities, scales, rotations, cov3D_precomp, settings):
    # Implements tile-Gaussian intersection, depth sorting, and front-to-back alpha compositing [S3, S4, unofficial]
    # Returns rendered_image, rendered_opacity, rendered_depth, rendered_feature, radii, etc.
    pass # Core CUDA/MLX kernel implementation [S4, unofficial]
```

## Subtopics

### [Neural Gaussian Rasterization Pipeline](7419-neural-gaussian-rasterization-pipeline.md)
The Neural Gaussian Rasterization Pipeline replaces the standard exponential opacity formulation of 3D Gaussian Splatting with direct neural network outputs for per-Gaussian opacity and color, while retaining the standard 2D Gaussian rasterizer for compositing. It computes view-dependent color and density ($\sigma$) for each Gaussian center, maps $\sigma$ directly to opacity, and composites the result using standard alpha blending.

### Octree-GS Rendering
Initializes `GaussianModel` with dataset parameters including `feat_dim`, `n_offsets`, `fork`, `use_feat_bank`, `appearance_dim`, `add_opacity_dist`, `add_cov_dist`, `add_color_dist`, `add_level`, `visible_threshold`, `dist2level`, `base_layer`, `progressive`, `extend` [S7, unofficial]. Applies a compatibility patch to `gaussian_renderer.generate_neural_gaussians` to replace `ape_code[0]` with `ape_code` [S7, unofficial]. Sets the anchor mask for the current view using `gaussians.set_anchor_mask(view.camera_center, scene.loaded_iter, view.resolution_scale)` [S7, unofficial]. Prefilters voxels to determine visibility using `prefilter_voxel(view, gaussians, pipeline, background)` [S7, unofficial]. Renders the scene using `render(view, gaussians, pipeline, background, visible_mask=visible, ape_code=args.ape)` [S7, unofficial]. Clamps the rendered image to `[0.0, 1.0]` and saves it [S7, unofficial]. Records the number of visible Gaussians per view in `per_view_count.json` [S7, unofficial]. The `ape_code` integer argument passed to the render function defaults to 10 [S7, unofficial].

### Unity 3DGS Rendering Pipeline Integration
Inserts an additional sorting pass into each camera's command buffer [S8, unofficial]. Inserts draw calls for 3DGS objects into the existing rendering queue for transparent meshes based on their bounding boxes [S8, unofficial]. Differs from separate render texture approaches by avoiding an intermediate texture and color space conversion pass after alpha blending [S8, unofficial].

### Per-pixel Depth Comparison for Transparency
A technique to resolve popping artifacts (temporal instability) caused by depth-sorting swaps by replacing global sorting with per-pixel depth comparison or ray-tracing [S9]. Popping artifacts arise from the depth-sorting step; as the camera moves, two overlapping Gaussians may swap their sort order, causing a sudden discontinuity in the alpha-blending accumulation [S9].

### Near-plane Culling
A culling method using axis-aligned bounding boxes (AABB) to check depth and remove Gaussian points that are behind the camera near-plane, reducing unnecessary computations and memory accesses [S10]. Filters out Gaussian points that lie behind or too close to the near clipping plane before they enter the rendering pipeline [S10]. Checks each Gaussian's position relative to the camera near plane; discards or skips points that fail the test before projection [S10]. Culling condition: $z_{\max} < z_{\text{near}}$, where $z$ is the Gaussian center depth in camera coordinates, $\Delta z \ge 0$ is the half-extent of the AABB along the camera z-axis, $z_{\min} = z - \Delta z$, $z_{\max} = z + \Delta z$, and $z_{\text{near}}$ is the camera near-plane depth [S10]. Hardware unit design operates with a single MAC unit, latency of 4 cycles, processing inputs (x, y, depth) and view matrix row [3:0], using D flip-flops between operations for timing stability at 800 MHz FP16 [S10]. Achieves a 56% culling rate on the compressed model, saving 56% of unnecessary Gaussian point computations and memory accesses [S10]. Increases throughput from 15.81% to 35.94% (2.27× gain), raising FPS from 20.4 to 46.4 at 1080p [S10].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot T \cdot P)$ where $N$ is the number of Gaussians, $T$ is the number of tiles, and $P$ is the number of pixels per tile. In practice, culling and early termination reduce this significantly [S3, S4, unofficial].
*   **Memory Complexity:** $O(N + H \cdot W)$ for storing Gaussian parameters and output buffers [S3, unofficial].
*   **Performance:** On Apple Silicon (M1/M2/M3/M4) with MLX, the full pipeline (forward pass) for 1000 Gaussians at 128x128 resolution takes approximately 73ms [S4, unofficial]. The CUDA implementation (gsplat) is generally faster due to optimized kernels [S4, unofficial].
*   **Strengths:** Differentiable end-to-end, supports view-dependent appearance via SH, efficient tiling-based rasterization [S2, S3, unofficial].
*   **Limitations:** High memory usage for large scenes, artifacts from overlapping Gaussians, sensitivity to hyperparameters like `eps2d` and `alpha_cutoff` [S3, S4, unofficial].
*   **When to use:** Novel view synthesis, real-time rendering of Gaussian splats, differentiable rendering for optimization [S2, S3, unofficial].
*   **When not to use:** Scenes with extremely high Gaussian counts where memory is constrained, or when non-differentiable rendering is sufficient [S3, unofficial].

## Background
The Gaussian Splatting Rendering Pipeline is based on the 3D Gaussian Splatting method introduced by Kerbl et al. in 2023 [S2, S4, unofficial]. It builds on the concept of splatting, where 3D primitives are projected to 2D and accumulated [S4, unofficial]. The pipeline has been implemented in various frameworks, including the original CUDA-based `diff-gaussian-rasterization` [S2, unofficial], the optimized `gsplat` library [S3, unofficial], and the MLX port `gsplat-mlx` for Apple Silicon [S3, unofficial]. It is widely used in 3D reconstruction, novel view synthesis, and SLAM systems [S3, unofficial].

## Sources
[S1] ldyken53/VEG, https://github.com/ldyken53/VEG, fetched 2026-10-26 (unofficial)
[S2] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S3] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S4] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
[S5] ashu1069/3D-Gaussian-Splatting-for-Novel-View-Synthesis, https://github.com/ashu1069/3D-Gaussian-Splatting-for-Novel-View-Synthesis, fetched 2026-10-27 (unofficial)
[S6] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
[S7] 3DAgentWorld/Advanced3DGS, https://github.com/3DAgentWorld/Advanced3DGS, fetched 2026-10-27 (unofficial)
[S8] wuyize25/gsplat-unity, https://github.com/wuyize25/gsplat-unity, fetched 2026-10-27 (unofficial)
[S9] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S10] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
