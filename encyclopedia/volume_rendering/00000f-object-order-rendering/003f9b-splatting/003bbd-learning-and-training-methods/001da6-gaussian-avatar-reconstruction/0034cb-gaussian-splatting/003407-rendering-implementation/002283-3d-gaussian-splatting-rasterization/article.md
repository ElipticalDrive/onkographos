# 3D Gaussian Splatting Rasterization

Also called: 3D Gaussian Splatting (3DGS), 3DGS Rasterization, Differentiable Gaussian Rasterization, TrimGS, gsplat.rasterization

## Overview
3D Gaussian Splatting Rasterization is a differentiable rendering technique that represents a scene as a collection of anisotropic 3D Gaussians and projects them onto a 2D image plane for real-time rendering [S1][S3, S4, unofficial]. It solves the problem of efficiently rendering continuous volumetric representations by converting each 3D primitive into a 2D splat, sorting them by depth, and accumulating color via alpha compositing [S4, S5, unofficial]. Unlike traditional triangle rasterization, it uses direct Gaussian projection, and unlike its sibling [3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)](2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md), it typically relies on GPU-accelerated sorting (e.g., CUB radix sort) and tile-based processing to handle millions of primitives [S4, unofficial].

## How it works
**Inputs:**
*   $N$ Gaussians defined by mean position $\mu \in \mathbb{R}^3$, rotation quaternion $q \in \mathbb{R}^4$, scale $s \in \mathbb{R}^3$, opacity $\alpha \in \mathbb{R}$, and spherical harmonics (SH) coefficients $C_{SH}$ or direct color $C \in \mathbb{R}^3$ [S1][S2, S4, unofficial].
*   Camera parameters: View matrix $V \in \mathbb{R}^{4 \times 4}$, projection matrix $K \in \mathbb{R}^{3 \times 3}$, image width $W$, height $H$ [S2, S3, unofficial].
*   Parameters: `tile_size` (default 16), `near_plane` (default 0.01), `far_plane` (default $10^{10}$), `sh_degree` (default 3) [S2, unofficial].

**Outputs:**
*   Rendered image $I \in \mathbb{R}^{H \times W \times 3}$ (RGB) [S2, S3, unofficial].
*   Alpha mask $A \in \mathbb{R}^{H \times W \times 1}$ [S2, S3, unofficial].
*   Optional depth $D \in \mathbb{R}^{H \times W \times 1}$ [S2, unofficial].

**Data Structures:**
*   **Gaussian Attributes:** Stored in contiguous arrays or Texture Buffer Objects (TBOs). For TBO layout [S8, unofficial]:
    *   Base TBO: 2 `uvec4` texels per splat (32 bytes).
        *   Texel 0: `center.xyz` (raw float bits), `packHalf2x16(cov00, cov01)`.
        *   Texel 1: `packHalf2x16(cov02, cov11)`, `packHalf2x16(cov12, cov22)`, `packUnorm4x8(DC-baked RGBA)`, reserved.
    *   SH TBO: `GL_RGBA16F`, 8 bytes per non-DC SH coefficient per splat. Layout: `[splat0_c1, splat0_c2, …, splat1_c1, splat1_c2, …]`.
*   **Conic Representation:** The inverse 2D covariance $\Sigma_{2D}^{-1}$ is stored as a vector $C = [\Sigma_{2D}^{-1}[0,0], \Sigma_{2D}^{-1}[0,1], \Sigma_{2D}^{-1}[1,1]]$ [S2, S3, unofficial].

**Steps:**

1.  **Input Validation & Setup:**
    Validate `render_mode` (e.g., "RGB", "D", "RGB+D"), `rasterize_mode` ("classic", "antialiased"), and `camera_model` ("pinhole", "ortho", "fisheye") [S2, unofficial]. If `sh_degree` is not None, compute view directions $d_i = \text{normalize}(\mu_i - \text{campos})$ and evaluate SH coefficients to get RGB colors [S2, unofficial]. If `sh_degree` is None, use direct colors [S2, unofficial].

2.  **Covariance Construction:**
    Compute the 3D covariance matrix $\Sigma_{3D}$ from quaternions and scales:
    $$ \Sigma_{3D} = R S S^T R^T $$
    where $S = \text{diag}(s_x, s_y, s_z)$ and $R$ is the rotation matrix derived from $q$ [S1][S2, unofficial].

3.  **Projection:**
    Transform means and covariances to camera space. Project to 2D screen space using the local affine approximation of the projective transformation [S3, unofficial]:
    $$ \mu_{2D} = \text{project}(\mu_{3D}, V, K) $$
    $$ \Sigma_{2D} = J \Sigma'_{3D} J^T $$
    where $J$ is the Jacobian of the projection function [S3, unofficial]. Compute the conic vector $C = [\Sigma_{2D}^{-1}[0,0], \Sigma_{2D}^{-1}[0,1], \Sigma_{2D}^{-1}[1,1]]$ [S2, S3, unofficial].

4.  **Culling & Tile Intersection:**
    Filter Gaussians behind the camera ($z < \text{near\_plane}$) [S3, unofficial]. Compute bounding radii for each Gaussian in screen space. Determine which tiles (grid cells of size `tile_size` $\times$ `tile_size`) each Gaussian intersects [S2, unofficial]. This generates a list of `(tile_id, gaussian_id)` pairs [S2, unofficial].

5.  **Sorting:**
    Sort the intersecting Gaussians within each tile by their camera-space depth $z$ to ensure correct front-to-back compositing order [S2, S3, S4, unofficial]. This is typically the performance bottleneck, often solved using device radix sort (e.g., CUB) [S4, unofficial].

6.  **Alpha Compositing:**
    For each pixel, iterate over the sorted Gaussians. Compute the alpha contribution of each Gaussian at the pixel center:
    $$ \alpha_i = \alpha_{\text{primitive}} \cdot \exp\left(-\frac{1}{2} (\mathbf{x} - \mu_{2D,i})^T \Sigma_{2D,i}^{-1} (\mathbf{x} - \mu_{2D,i})\right) $$
    [S2, S3, unofficial]. Compute transmittance $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ [S3, unofficial]. The weight is $w_i = \alpha_i T_i$ [S3, unofficial]. Accumulate color:
    $$ C_{\text{pixel}} = \sum_{i} w_i C_i + \left(1 - \sum_{i} w_i\right) C_{\text{bg}} $$
    [S3, unofficial]. If `render_mode` includes depth, compute weighted depth: $D_{\text{pixel}} = \sum w_i z_i$ [S3, unofficial]. Normalize depth by accumulated alpha if required [S2, unofficial].

7.  **Background Handling:**
    For pixels with no intersecting Gaussians, fill with background color $C_{\text{bg}}$ [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterize_gaussians(means_3d, quats, scales, opacities, colors, viewmats, Ks, width, height, tile_size=16, sh_degree=None, near_plane=0.01, far_plane=1e10):
    """
    Renders 3D Gaussians to 2D images.
    Returns: render_colors [C, H, W, D], render_alphas [C, H, W, 1]
    """
    C = viewmats.shape[0] # Number of cameras
    N = means_3d.shape[0] # Number of Gaussians
    
    # 1. Compute 3D Covariance
    # cov_3D shape: [N, 3, 3]
    cov_3D = quat_scale_to_covar_preci(quats, scales, compute_covar=True, compute_preci=False)
    
    # 2. Project to 2D
    # means_2d: [C, N, 2], cov_2D: [C, N, 3, 3], conics: [C, N, 3], radii: [C, N, 2]
    means_2d, cov_2D, conics, radii, depths = fully_fused_projection(
        means_3d, cov_3D, viewmats, Ks, width, height, 
        near_plane, far_plane, calc_compensations=False
    )
    
    # 3. Compute Colors (if SH)
    if sh_degree is not None:
        campos = viewmats[:, :3, 3] # Camera position
        dirs = means_3d[None, :, :] - campos[:, None, :]
        dirs = dirs / mx.maximum(mx.linalg.norm(dirs, axis=-1, keepdims=True), 1e-8)
        # Evaluate SH to get rgb [C, N, 3]
        rgb = spherical_harmonics(sh_degree, dirs, colors)
        rgb = mx.maximum(rgb + 0.5, 0.0) # Clamp
    else:
        rgb = colors # Already [C, N, 3] or broadcasted
        
    # 4. Tile Intersection
    tile_width = mx.ceil(width / tile_size)
    tile_height = mx.ceil(height / tile_size)
    # Returns: tiles_per_gauss, isect_ids, flatten_ids
    tiles_per_gauss, isect_ids, flatten_ids = isect_tiles(
        means_2d, radii, depths, tile_size, tile_width, tile_height, sort=True
    )
    
    # 5. Encode Intersection Offsets for GPU Access
    isect_offsets = isect_offset_encode(isect_ids, C, tile_width, tile_height)
    
    # 6. Rasterize to Pixels
    # Opacities broadcast to [C, N]
    opacities_cn = mx.broadcast_to(mx.expand_dims(opacities, axis=0), (C, N))
    
    # Call fused kernel
    render_colors, render_alphas = rasterize_to_pixels_mlx(
        means_2d, conics, rgb, opacities_cn, 
        width, height, tile_size, isect_offsets, flatten_ids, 
        backgrounds=mx.ones((C, 3))
    )
    
    return render_colors, render_alphas
```

## Subtopics

### [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md)
Provides the core differentiable forward and backward passes for 3D Gaussians, allowing gradients to flow to Gaussian parameters and camera matrices. Chosen when training is required.

### [Perspective-Correct 3D Gaussian Splatting via Plücker Coordinates](3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md)
Uses Plücker coordinates for a closed-form, numerically stable perspective-correct evaluation without matrix inversion. Chosen when high geometric accuracy at extreme angles is needed.

### [Analytic-Splatting Forward Shading](3459-analytic-splatting-forward-shading.md)
Integrates the Gaussian PDF over the pixel area to reduce aliasing. Chosen when anti-aliasing is critical and point-sampling artifacts are visible.

### [Speedy-Splat Rendering Pipeline](3543-speedy-splat-rendering-pipeline.md)
Replaces coarse bounding-box tests with precise Gaussian-tile intersection checks. Chosen for improved performance on scenes with sparse Gaussians.

### [Ghost Ellipsoids for Chunked Training](3559-ghost-ellipsoids-for-chunked-training.md)
Includes ellipsoids from neighboring chunks in loss computation without updating parameters. Chosen for training large scenes in chunks without boundary artifacts.

### [2D Gaussian Splatting Rasterization](4815-2d-gaussian-splatting-rasterization.md)
Renders planar surfels with explicit normals. Chosen when normal maps and surface-aligned geometry are required.

### [Hardware-Accelerated Gaussian Rasterization](7329-hardware-accelerated-gaussian-rasterization.md)
Uses dedicated hardware units for projection and blending. Chosen for maximum throughput on specialized hardware.

### [AdR-Gaussian](8365-adr-gaussian.md)
Optimizes culling by moving serial operations to parallel preprocessing. Chosen for significant speedups without quality loss.

### [QuadBox](9231-quadbox.md)
Uses four AABBs to tightly encapsulate Gaussians. Chosen to reduce over-approximation in tile intersection.

### [Transparency and Blending Models](9274-transparency-and-blending-models.md)
Defines compositing strategies (e.g., OIT, depth correction). Chosen when standard alpha blending causes popping or inaccuracies.

### [Projection Geometry and Coordinate Systems](9313-projection-geometry-and-coordinate-systems.md)
Defines transformations for non-pinhole cameras. Chosen for panoramic or specialized camera models.

### [Point-Based Ray Tracing for Visibility](10934-point-based-ray-tracing-for-visibility.md)
Uses a BVH for visibility estimation. Chosen as a companion to rasterization for lighting/shadowing.

### [Primitive Representation](11838-primitive-representation.md)
Defines kernel shapes and attribute storage. Chosen to customize primitive behavior.

### [Lighting, Shadowing & Relighting](11894-lighting-shadowing-relighting.md)
Handles illumination. Chosen for dynamic lighting effects.

### Temporal Marginal Probability Filtering
Pre-rasterization culling that removes 4D Gaussians with negligible temporal support $p_i(t)$ at the current frame timestamp [S6, unofficial]. Discards Gaussians before frustum culling to keep active Gaussian count constant, stabilizing rendering speed for long videos [S6, unofficial].

### t-distribution truncation for splatting
Replaces the fixed three-sigma rule with $\nu$-dependent critical value interpolation for Student's t-distributions [S7]. Interpolates critical values for fat-tailed distributions (low $\nu$) to prevent excessive truncation or leakage [S7].

### Gaussian Opacity Fields (GOF)
Uses ray-tracing-style intersection for accurate depth and derives a level-set surface from accumulated opacity [S5, unofficial].

### Progressive Normal Propagation
Uses progressive propagation along normals to fill texture-less regions [S5, unofficial].

### Splat TBO Upload & Layout
Packs per-splat data into Texture Buffer Objects (TBOs) for GPU consumption [S8, unofficial]. Base TBO uses 32 B/splat (center, covariance, DC color/alpha). SH TBO uses `GL_RGBA16F` for non-DC coefficients [S8, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by sorting $N$ Gaussians, typically $O(N \log N)$ or $O(N)$ with radix sort [S4, unofficial]. Tile-based culling reduces the number of Gaussians processed per pixel [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for Gaussian attributes. High VRAM usage: ~4GB to view, ~12GB to train [S4, unofficial]. Large disk size: 1GB+ per scene [S4, unofficial].
*   **Performance:** Real-time rendering rates (60+ FPS) on modern GPUs for scenes with millions of Gaussians [S1][S4, unofficial]. The primary bottleneck is sorting millions of Gaussians, efficiently handled by CUDA CUB radix sort [S4, unofficial].
*   **Strengths:** Photorealistic quality, real-time performance, differentiable [S4, unofficial].
*   **Limitations:** High VRAM/disk usage, static scenes (without extensions), incompatibility with traditional pipelines without adaptation [S4, unofficial]. Sorting is CUDA-dependent in original implementations [S4, unofficial].

## Background
3D Gaussian Splatting was introduced in "3D Gaussian Splatting for Real-Time Radiance Field Rendering" [S4, unofficial]. It builds on the idea of representing scenes as discrete primitives, analogous to triangle rasterization but using Gaussians [S4, unofficial]. The method enables real-time rendering of photorealistic scenes learned from small samples of images [S1][S4, unofficial]. It differs from traditional quad-based rasterization approaches (e.g., WebGL, Unity viewers) by using direct Gaussian rasterization, which offers higher quality and performance but requires specialized rendering pipelines [S4, unofficial].

## Sources
[S1] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
[S2] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
[S3] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S4] Introduction to 3D Gaussian Splatting - Hugging Face, https://huggingface.co/blog/gaussian-splatting, fetched 2026-10-27 (unofficial)
[S5] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S6] “Real-time Photorealistic Dynamic Scene Representation and Rendering with 4D Gaussian Splatting”, https://radiancefields.com/papers/real-time-photorealistic-dynamic-scene-representation-and-rendering-with-4d-gaussian-splatting, fetched 2026-10-30 (unofficial)
[S7] 3D Student Splatting and Scooping, https://doi.org/10.1109/cvpr52734.2025.01960, fetched 2026-10-29
[S8] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
