# MIP-Splatting

Also called: Multi-Scale 3D Gaussian Splatting

## Overview
MIP-Splatting is a rendering method for 3D Gaussian Splatting that eliminates aliasing and dilation artifacts when rendering at resolutions, focal lengths, or camera distances different from those encountered during training [S3][S9, unofficial]. It solves the mismatch between the discrete sampling of the image sensor and the continuous representation of 3D Gaussians by introducing a multi-scale representation and two specific filters: a 3D smoothing filter and a 2D Mip filter [S9, unofficial]. Unlike standard 3DGS which suffers from high-frequency artifacts when zooming out and dilation effects when zooming in, MIP-Splatting constrains the representation frequency to the Nyquist limit of the training data and approximates the physical box filter of a camera sensor [S4][S9, unofficial].

## How it works
**Inputs:** A set of 3D Gaussians $\{G_k\}$ parameterized by center $p_k$, covariance $\Sigma_k$, opacity $\alpha_k$, and spherical harmonics coefficients; a camera defined by projection matrix and intrinsics; and rendering resolution.
**Outputs:** A rendered image and visibility radii.

**Data Structures:**
Each Gaussian $G_k$ stores:
- $p_k \in \mathbb{R}^3$: Center position.
- $\Sigma_k \in \mathbb{R}^{3 \times 3}$: Covariance matrix, parameterized as $\Sigma_k = O_k s_k s_k^T O_k^T$, where $s \in \mathbb{R}^3$ is a scaling vector and $O \in \mathbb{R}^{3 \times 3}$ is a rotation matrix from a quaternion [S9, unofficial].
- $\alpha_k \in [0, 1]$: Opacity.
- $SH_k$: Spherical harmonics features.
- $l_k \in \{1, \dots, L_{max}\}$: Level of detail index.
- $S_{k}^{max}, S_{k}^{min} \in \mathbb{R}$: Maximum and minimum pixel coverage thresholds for selective rendering [S3].

**Steps:**

1.  **3D Smoothing Filter (Training):**
    To prevent the optimization from creating Gaussians with frequencies higher than the training data can support, a 3D low-pass filter is applied. The maximal sampling frequency $\hat{\nu}$ is derived from the training views. For a camera at depth $d$ with focal length $f$, the world space sampling interval is $\hat{T} = d/f$, so $\hat{\nu} = f/d$ [S9, unofficial].
    The regularized 3D Gaussian $G_k(x)_{reg}$ is the convolution of $G_k$ with a low-pass Gaussian $G_{low}$ with variance $\hat{\nu}_k^s \cdot I$, where $s$ is a scalar hyperparameter (typical value 0.2) [S9, unofficial]:
    $$ G_k(x)_{reg} = \frac{\sqrt{|\Sigma_k|}}{\sqrt{|\Sigma_k + \hat{\nu}_k^s \cdot I|}} e^{-\frac{1}{2}(x-p_k)^T (\Sigma_k + \hat{\nu}_k^s \cdot I)^{-1} (x-p_k)} $$
    This effectively increases the 3D covariance: $\Sigma_k' = \Sigma_k + \hat{\nu}_k^s \cdot I$ [S9, unofficial]. The maximal sampling rate is recomputed every $m$ iterations (typical $m=100$) [S9, unofficial].

2.  **Multi-Scale Aggregation (Training):**
    The scene is represented by Gaussians at $L_{max}=4$ levels (1x, 4x, 16x, 64x downsampled resolution) [S3].
    -   **Initialization:** All Gaussians start at level $l=1$ [S3].
    -   **Aggregation Trigger:** After a warm-up stage (first 1,000 iterations), coarse-level Gaussians are created [S3].
    -   **Selection:** For each coarse level $l_m$ ($2 \le l_m \le L_{max}$), render all Gaussians from levels $[1, l_m-1]$ at $4^{l_m-1}$ times downsampled resolution. Gaussians with pixel coverage $S_k < S_T$ (threshold $S_T=2$ pixels) are selected as "too small" [S3].
    -   **Voxelization:** Selected Gaussians are binned into a voxel grid of resolution $(400/l_m)^3$ [S3].
    -   **Pooling:** Within each voxel, attributes (position, rotation, SH, opacity, scaling) are averaged [S3].
    -   **Enlargement:** The new Gaussian's scaling is enlarged by factor $S_T / S_{avg}$, where $S_{avg}$ is the average pixel coverage of the aggregated Gaussians, targeting a pixel coverage of $S_T$ [S3]. The new Gaussian is assigned to level $l_m$ [S3].
    -   **Training:** The model is trained on both original and downsampled images [S3].

3.  **Selective Rendering (Inference):**
    At render time, Gaussians are selected based on their current pixel coverage $S_k$ relative to their stored bounds $S_k^{max}$ and $S_k^{min}$ [S3].
    -   **Update Bounds:** If the rendering resolution scale matches the scale at which $G_k$ was created, update bounds:
        $$ S_k^{max'} = \max(\lambda_1 S_k^{max}, S_k), \quad S_k^{min'} = \min(\lambda_2 S_k^{min}, S_k) $$
        where $\lambda_1 = 0.95$ and $\lambda_2 = 1.05$ [S3].
    -   **Selection Condition:** A Gaussian is rendered if:
        $$ \frac{S_k}{S_k^{max}} \le S_{rel}^{max} \quad \land \quad \left( \frac{S_k}{S_k^{min}} \ge S_{rel}^{min} \lor S_k \ge S_T \right) $$
        where $S_{rel}^{max} = 1.5$ and $S_{rel}^{min} = 0.5$ [S3]. This filters out Gaussians that are too large (causing blur) or too small (causing aliasing), except those larger than $S_T$ which are safe [S3].

4.  **2D Mip Filter (Rendering):**
    To handle dilation artifacts when zooming in (where projected Gaussians become small but the fixed dilation kernel remains large), the standard 2D dilation is replaced by a 2D Mip filter [S9, unofficial].
    The projected 2D covariance $\Sigma_{k2D}$ is regularized by adding a variance $sI$ covering a single pixel in screen space (typical $s=0.1$) [S9, unofficial]:
    $$ \Sigma_{k2D, mip} = \Sigma_{k2D} + sI $$
    The opacity is scaled to preserve the integral:
    $$ \alpha_{new} = \alpha_{old} \cdot \frac{\sqrt{|\Sigma_{k2D}|}}{\sqrt{|\Sigma_{k2D} + sI|}} $$
    This approximates the physical box filter of the camera sensor [S9, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def render_mip_splatting(viewpoint_camera, gaussians, pipe, bg_color, kernel_size=1.0, scaling_modifier=1.0, subpixel_offset=None):
    """
    Render the scene using MIP-Splatting logic.
    """
    # 1. Prepare 2D means for gradient tracking
    screenspace_points = torch.zeros_like(gaussians.get_xyz, dtype=gaussians.get_xyz.dtype, requires_grad=True, device="cuda") + 0
    try:
        screenspace_points.retain_grad()
    except:
        pass

    # 2. Camera parameters
    tanfovx = math.tan(viewpoint_camera.FoVx * 0.5)
    tanfovy = math.tan(viewpoint_camera.FoVy * 0.5)
    
    if subpixel_offset is None:
        subpixel_offset = torch.zeros((int(viewpoint_camera.image_height), int(viewpoint_camera.image_width), 2), dtype=torch.float32, device="cuda")

    # 3. Rasterization Settings
    raster_settings = GaussianRasterizationSettings(
        image_height=int(viewpoint_camera.image_height),
        image_width=int(viewpoint_camera.image_width),
        tanfovx=tanfovx,
        tanfovy=tanfovy,
        kernel_size=kernel_size,
        subpixel_offset=subpixel_offset,
        bg=bg_color,
        scale_modifier=scaling_modifier,
        viewmatrix=viewpoint_camera.world_view_transform,
        projmatrix=viewpoint_camera.full_proj_transform,
        sh_degree=gaussians.active_sh_degree,
        campos=viewpoint_camera.camera_center,
        prefiltered=False,
        debug=pipe.debug
    )
    rasterizer = GaussianRasterizer(raster_settings=raster_settings)

    # 4. Prepare Gaussian Attributes
    means3D = gaussians.get_xyz
    means2D = screenspace_points
    
    # Apply 3D filter if active (returns modified opacity/scaling/rotation)
    opacity = gaussians.get_opacity_with_3D_filter
    
    scales = None
    rotations = None
    cov3D_precomp = None
    
    if pipe.compute_cov3D_python:
        cov3D_precomp = gaussians.get_covariance(scaling_modifier)
    else:
        scales = gaussians.get_scaling_with_3D_filter
        rotations = gaussians.get_rotation

    # 5. Prepare Colors
    shs = None
    colors_precomp = None
    if override_color is None:
        if pipe.convert_SHs_python:
            shs_view = gaussians.get_features.transpose(1, 2).view(-1, 3, (gaussians.max_sh_degree+1)**2)
            dir_pp = (gaussians.get_xyz - viewpoint_camera.camera_center.repeat(gaussians.get_features.shape[0], 1))
            dir_pp_normalized = dir_pp / dir_pp.norm(dim=1, keepdim=True)
            sh2rgb = eval_sh(gaussians.active_sh_degree, shs_view, dir_pp_normalized)
            colors_precomp = torch.clamp_min(sh2rgb + 0.5, 0.0)
        else:
            shs = gaussians.get_features
    else:
        colors_precomp = override_color

    # 6. Rasterize
    # Note: The rasterizer internally handles the 2D Mip filter logic via the 
    # modified covariance/opacity passed or via internal settings if configured.
    # In the official implementation, the 2D Mip filter is often applied 
    # by modifying the covariance before passing to the rasterizer or 
    # via a specific kernel_size parameter that triggers the box filter approximation.
    rendered_image, radii = rasterizer(
        means3D=means3D,
        means2D=means2D,
        shs=shs,
        colors_precomp=colors_precomp,
        opacities=opacity,
        scales=scales,
        rotations=rotations,
        cov3D_precomp=cov3D_precomp
    )

    return {
        "render": rendered_image,
        "viewspace_points": screenspace_points,
        "visibility_filter": radii > 0,
        "radii": radii
    }
```

## Subtopics

### [2D Mip Filter](9593-2d-mip-filter.md)
The 2D Mip Filter mitigates aliasing artifacts when rendering from camera distances or focal lengths larger than training. It extends the standard 2D Gaussian filter by introducing a view-dependent opacity compensation factor that prevents excessive blurring as Gaussians become smaller relative to the pixel grid. Unlike the basic 2D Gaussian filter which dilates the covariance matrix uniformly, this method dynamically adjusts opacity based on the ratio of the original to the dilated covariance determinant.

### LinPrim Anti-Aliasing
An anti-aliasing strategy combining a 3D smoothing filter and a bounding box expansion technique [S13]. It applies a 3D smoothing strategy to limit primitive size based on visibility from training views [S13]. It also expands the bounding box vertices outward parallel to the screen, enlarging the footprint and increasing minimum appearance size, acting as a coarse band-limiting approximation [S13]. It cannot use MIP-Splatting's 2D Mip filter because LinPrim primitives do not rely on modifying the screen space Gaussian distribution [S13].

### Anti-Aliasing via Kernel Adaptation
Adapts the kernel bandwidth to prevent aliasing when the volume is undersampled relative to the screen [S14, unofficial]. It calculates the Jacobian $J$ of the local perspective distortion for each point and computes the ray-space ellipsoid $G_{JMV}$ using $J$ to stretch the kernel appropriately [S14, unofficial].

### Integration with Mip-Splatting and Scaffold-GS
Mip-Splatting and Scaffold-GS employ conventional projection of 3D-GS, resulting in larger approximation error compared to optimal projection methods [S15]. The proposed optimal projection method can be seamlessly integrated with them to reduce their projection error [S15].

## Operating characteristics
**Time and Memory Complexity:**
Rendering time is not linearly reduced at very low resolutions because all Gaussians must be splatted before selective filtering, creating overhead [S3]. However, it achieves 160%-2400% rendering speed improvements at 4x-128x scale compared to vanilla 3DGS [S3]. Memory usage increases slightly due to the storage of multi-scale Gaussians (typically <5% additional Gaussians) and bounds [S3].

**Measured Performance:**
-   **MipNeRF 360 (64x scale):** PSNR 25.35 vs 15.23 (3DGS), LPIPS 0.066 vs 0.256, Time 4.9ms vs 103.3ms [S3].
-   **Blender (Full Res):** PSNR 32.81 [S9, unofficial].
-   **Tank and Temples (64x scale):** PSNR 19.67 vs 13.88, LPIPS 0.034 vs 0.447, Time 5.9ms vs 82.6ms [S3].
-   **Deep Blending (64x scale):** PSNR 25.70 vs 17.75, LPIPS 0.036 vs 0.379, Time 3.4ms vs 59.7ms [S3].

**Strengths and Limitations:**
-   **Strengths:** Eliminates aliasing and dilation artifacts across a wide range of resolutions and focal lengths without requiring multi-scale ground truth images for supervision [S9, unofficial]. Improves rendering speed at low resolutions [S3].
-   **Limitations:** Naive filtering of small Gaussians causes missing parts (holes), addressed by aggregation but adding complexity [S3]. Performance overhead at very low resolutions due to splatting all primitives [S3].

**When to Use:**
Use when rendering 3DGS scenes at resolutions, zoom levels, or focal lengths significantly different from the training data, or when high-quality novel view synthesis is required for VR/robotics with varying camera parameters [S9, unofficial].

## Background
MIP-Splatting was introduced to address the aliasing artifacts inherent in 3D Gaussian Splatting when the sampling rate changes [S9, unofficial]. It builds on the observation that standard 3DGS systematically underestimates scale parameters, leading to erosion when zooming in and dilation when zooming out [S9, unofficial]. The method was developed to provide robust novel view synthesis for virtual reality, cinematography, and robotics, specifically for out-of-distribution generalization where training and testing sampling rates differ [S9, unofficial]. It is evaluated on the MipNeRF360, Tank and Temples, and Deep Blending datasets [S3].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S3] Multi-Scale 3D Gaussian Splatting for Anti-Aliased Rendering, https://doi.org/10.1109/cvpr52733.2024.01977, fetched 2026-10-26
[S4] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S5] autonomousvision/mip-splatting, https://github.com/autonomousvision/mip-splatting, fetched 2026-10-26 (unofficial)
[S6] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S7] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S8] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S9] Mip-Splatting, https://arxiv.org/abs/2311.16493, fetched 2026-10-27 (unofficial)
[S10] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S11] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
[S12] Gaussian Splats Part 2: From Photos to a Trained Model, https://alphapixeldev.com/gaussian-splats-part-2-creating-gaussian-splats/, fetched 2026-10-29 (unofficial)
[S13] LinPrim: Linear Primitives for Differentiable Volumetric Rendering, https://arxiv.org/abs/2501.16312, fetched 2026-10-26
[S14] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
[S15] On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy, https://doi.org/10.1007/978-3-031-72643-9_15, fetched 2026-10-29
