# 3D Gaussian Splatting Rasterization

Also called: 3D Gaussian Splatting (3DGS), 3DGS Rasterization, Differentiable Gaussian Rasterization, TrimGS, gsplat.rasterization

## Overview
3D Gaussian Splatting Rasterization is a differentiable rendering technique that represents a scene as a collection of anisotropic 3D Gaussians and projects them onto a 2D image plane for real-time rendering [S1][S3, S4, unofficial]. It solves the problem of efficiently rendering continuous volumetric representations by converting each primitive into a 2D splat, sorting them by depth, and accumulating color via alpha compositing [S4, S5, unofficial]. Unlike traditional triangle rasterization or quad-based approximations, this method preserves high visual fidelity and supports gradient-based optimization of scene parameters [S4, S5, unofficial]. It builds on [Primitive Representation](11838-primitive-representation.md) and [Projection Geometry and Coordinate Systems](9313-projection-geometry-and-coordinate-systems.md).

## How it works
**Inputs:**
- $N$ 3D Gaussians defined by mean position $\mu \in \mathbb{R}^3$, scale $s \in \mathbb{R}^3$, rotation $q \in \mathbb{R}^4$ (quaternion), opacity $\alpha \in \mathbb{R}$, and spherical harmonics (SH) coefficients $c_{\ell m}$ or direct RGB color [S1][S4, unofficial].
- Camera parameters: view matrix $V \in \mathbb{R}^{4 \times 4}$, intrinsic matrix $K \in \mathbb{R}^{3 \times 3}$, image dimensions $H \times W$ [S2, S3, unofficial].
- Rendering parameters: `render_mode` (e.g., "RGB", "D"), `rasterize_mode` ("classic", "antialiased"), `tile_size` (default 16), `near_plane` (default 0.01), `far_plane` (default $10^{10}$) [S2, unofficial].

**Outputs:**
- Rendered image $I \in \mathbb{R}^{H \times W \times 3}$ (and optionally depth $D \in \mathbb{R}^{H \times W}$, normal $N \in \mathbb{R}^{H \times W \times 3}$, or mask $M \in \mathbb{R}^{H \times W}$) [S2, S3, unofficial].

**Data Structures:**
- **Gaussian Attributes:** Stored in contiguous arrays or tensors.
  - `means`: $[N, 3]$ float32.
  - `quats`: $[N, 4]$ float32 (normalized).
  - `scales`: $[N, 3]$ float32.
  - `opacities`: $[N]$ float32.
  - `colors`: $[N, 3]$ float32 (if SH degree is 0) or $[N, K, 3]$ where $K = (sh\_degree+1)^2$ (if SH degree > 0) [S2, unofficial].
- **Projected Attributes (per camera):**
  - `means2d`: $[N, 2]$ float32.
  - `conics`: $[N, 3]$ float32, storing the upper triangle of the inverse 2D covariance $\Sigma_{2D}^{-1}$ as $[C_{00}, C_{01}, C_{11}]$ [S2, S3, unofficial].
  - `radii`: $[N, 2]$ int32, bounding box in pixels [S2, unofficial].
  - `depths`: $[N]$ float32, camera-space Z-depth [S2, unofficial].
- **Tile Intersection:**
  - `isect_offsets`: $[C, T_h, T_w]$ int32, encoding the start index of Gaussians intersecting each tile [S2, unofficial].
  - `flatten_ids`: $[N_{intersected}]$ int32, sorted list of Gaussian indices [S2, unofficial].

**Steps:**

1. **Covariance Construction:** Compute the 3D covariance matrix $\Sigma_{3D}$ from scale $S = \text{diag}(s_x, s_y, s_z)$ and rotation $R$ (derived from quaternion $q$):
   $$ \Sigma_{3D} = R S S^T R^T $$
   [S1][S3, unofficial]

2. **Projection:** Transform means and covariance to camera space using $V$. Project to 2D screen space using the local affine approximation of the projective transformation. Compute the 2D covariance $\Sigma_{2D}$:
   $$ \Sigma_{2D} = J \Sigma' J^T $$
   where $J$ is the Jacobian of the projection function [S3, unofficial]. Compute the conic representation $C = [\Sigma_{2D}^{-1}[0,0], \Sigma_{2D}^{-1}[0,1], \Sigma_{2D}^{-1}[1,1]]$ [S2, S3, unofficial].

3. **Filtering and Sorting:** Filter Gaussians behind the camera ($z < 0$) and those outside the frustum. Sort the remaining Gaussians by camera-space depth $z$ to ensure correct front-to-back compositing [S3, S4, unofficial].

4. **Tile Intersection:** Divide the image into tiles of size `tile_size` $\times$ `tile_size`. For each Gaussian, compute its 2D bounding box (radius). Identify which tiles each Gaussian intersects. Encode these intersections into `isect_offsets` and `flatten_ids` to enable efficient access during rasterization [S2, unofficial].

5. **Color Evaluation:** If SH degree is enabled, compute view directions $d = \mu - \text{campos}$, normalize, and evaluate SH coefficients to get RGB color. Otherwise, use direct color [S2, unofficial].

6. **Rasterization (Compositing):** For each pixel in each tile:
   - Initialize transmittance $T = 1.0$ and accumulated color $C_{acc} = 0.0$.
   - Iterate over Gaussians intersecting the tile (from `flatten_ids`).
   - For each Gaussian, compute the 2D Gaussian value at the pixel center using the conic representation:
     $$ \alpha_{pixel} = \alpha \cdot \exp\left(-\frac{1}{2} \mathbf{x}^T \Sigma_{2D}^{-1} \mathbf{x}\right) $$
     where $\mathbf{x}$ is the vector from the pixel center to the Gaussian mean [S3, unofficial].
   - Compute weight $w = \alpha_{pixel} \cdot T$.
   - Update accumulated color: $C_{acc} \leftarrow C_{acc} + w \cdot C_{gaussian}$.
   - Update transmittance: $T \leftarrow T \cdot (1 - \alpha_{pixel})$.
   - If $T$ is negligible, break early [S3, unofficial].

7. **Background Compositing:** For pixels where $T > 0$ (or mask is zero), composite the background color $C_{bg}$:
   $$ C_{final} = C_{acc} + T \cdot C_{bg} $$
   [S3, unofficial]

8. **Depth/Normal Output:** If requested, accumulate depth/normal similarly using weighted sums, normalized by the accumulated alpha/mask [S2, S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterize_gaussians(means_3d, quats, scales, opacities, colors, viewmats, Ks, width, height, tile_size=16, near_plane=0.01, far_plane=1e10, sh_degree=None):
    """
    Rasterize 3D Gaussians to 2D image.
    Returns: render_colors [C, H, W, D], render_alphas [C, H, W, 1]
    """
    C = viewmats.shape[0] # Number of cameras
    N = means_3d.shape[0] # Number of Gaussians
    
    # 1. Compute 3D Covariance
    # cov_3d: [N, 3, 3]
    cov_3d = quat_scale_to_covar(quats, scales) 
    
    # 2. Project to 2D
    # means_2d: [C, N, 2], conics: [C, N, 3], radii: [C, N, 2], depths: [C, N]
    means_2d, conics, radii, depths = project_gaussians(
        means_3d, cov_3d, viewmats, Ks, width, height, near_plane, far_plane
    )
    
    # 3. Filter and Sort
    # Filter Gaussians behind camera and outside frustum
    valid_mask = depths >= near_plane
    # Sort by depth for each camera
    sorted_indices = argsort_depth(depths, valid_mask)
    
    # 4. Tile Intersection
    # Compute tile dimensions
    tile_width = ceil(width / tile_size)
    tile_height = ceil(height / tile_size)
    
    # Find intersecting tiles for each Gaussian
    # Returns: isect_offsets [C, tile_height, tile_width], flatten_ids [N_intersected]
    isect_offsets, flatten_ids = intersect_tiles(
        means_2d, radii, depths, tile_size, tile_width, tile_height
    )
    
    # 5. Color Evaluation
    # If SH degree > 0, compute colors based on view direction
    if sh_degree is not None:
        dirs = means_3d[None, :, :] - viewmats[:, None, :3, 3] # [C, N, 3]
        dirs = normalize(dirs)
        colors = evaluate_spherical_harmonics(sh_degree, dirs, sh_coeffs)
    else:
        colors = colors # Direct color [C, N, 3]
        
    # 6. Rasterize
    # Initialize output buffers
    render_colors = zeros((C, H, W, 3))
    render_alphas = zeros((C, H, W, 1))
    
    # Iterate over tiles
    for c in range(C):
        for ty in range(tile_height):
            for tx in range(tile_width):
                # Get start index for this tile
                start_idx = isect_offsets[c, ty, tx]
                end_idx = isect_offsets[c, ty+1, tx] if ty < tile_height-1 else len(flatten_ids)
                
                # Iterate over Gaussians in this tile
                for idx in range(start_idx, end_idx):
                    gauss_idx = flatten_ids[idx]
                    
                    # Get projected attributes
                    m2d = means_2d[c, gauss_idx]
                    conic = conics[c, gauss_idx]
                    alpha = opacities[gauss_idx]
                    color = colors[c, gauss_idx]
                    
                    # Iterate over pixels in the tile
                    for py in range(ty * tile_size, min((ty+1) * tile_size, height)):
                        for px in range(tx * tile_size, min((tx+1) * tile_size, width)):
                            # Compute vector from pixel to gaussian mean
                            dx = px - m2d[0]
                            dy = py - m2d[1]
                            
                            # Evaluate 2D Gaussian using conic
                            # alpha_pixel = alpha * exp(-0.5 * (conic[0]*dx^2 + 2*conic[1]*dx*dy + conic[2]*dy^2))
                            exponent = -0.5 * (conic[0]*dx*dx + 2*conic[1]*dx*dy + conic[2]*dy*dy)
                            alpha_pixel = alpha * exp(exponent)
                            
                            # Alpha compositing
                            # T is transmittance, accumulated in a separate buffer or computed on fly
                            # For simplicity, assuming we have T from previous gaussians
                            # w = alpha_pixel * T
                            # render_colors[c, py, px] += w * color
                            # T *= (1 - alpha_pixel)
                            pass # Implementation detail: efficient accumulation requires careful memory layout
    
    # 7. Background Compositing
    # For pixels with T > 0, add background color
    # render_colors = render_colors + (1 - render_alphas) * background_color
    
    return render_colors, render_alphas
```

**Parameters:**
- `tile_size`: Integer, default 16. Side length of tiles for rasterization [S2, unofficial].
- `near_plane`: Float, default 0.01. Near clipping plane distance [S2, unofficial].
- `far_plane`: Float, default $10^{10}$. Far clipping plane distance [S2, unofficial].
- `sh_degree`: Integer or None. If not None, enables spherical harmonics color computation [S2, unofficial].
- `render_mode`: String specifying output channels. Valid values: "RGB", "D", "ED", "RGB+D", "RGB+ED", "d", "Ed", "RGB+d", "RGB+Ed" [S2, unofficial].
- `rasterize_mode`: String specifying rendering style. Valid values: "classic", "antialiased" [S2, unofficial].
- `camera_model`: String specifying camera projection. Valid values: "pinhole", "ortho", "fisheye" [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Sorting Bottleneck:** Sorting millions of Gaussians by depth is a primary bottleneck. The original implementation uses CUB device radix sort, which is highly optimized but CUDA-specific [S4, unofficial].
- **Precision:** Floating-point precision issues can arise when evaluating the Gaussian exponent for very small or large distances. Clamping or compensation terms may be needed for antialiased mode [S2, unofficial].
- **Degenerate Gaussians:** Gaussians with zero or near-zero scale can lead to numerical instability in covariance inversion. Regularization or clamping of scales is required [G].
- **Memory Constraints:** High VRAM usage (4GB to view, 12GB to train) and large disk sizes (1GB+ for a scene) are common [S4, unofficial]. Splitting Gaussians into mini-batches can help manage memory [S3, unofficial].

## Subtopics

### [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md)
The 3D Gaussian Splatting Differentiable Rasterizer provides the core forward and backward passes for 3D Gaussians, allowing gradients to flow back to Gaussian parameters and camera view matrices. It differs from siblings by focusing on differentiability, often implemented as CUDA kernels for performance.

### [Perspective-Correct 3D Gaussian Splatting via Plücker Coordinates](3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md)
This method computes the value of a 3D Gaussian primitive for a given viewing ray in screen space without performing matrix inversion, utilizing Plücker coordinates. It sets itself apart by providing a closed-form, numerically stable solution for perspective-correct splat evaluation.

### [Speedy-Splat Rendering Pipeline](3543-speedy-splat-rendering-pipeline.md)
The Speedy-Splat Rendering Pipeline replaces coarse bounding-box tile intersection tests with precise Gaussian-tile intersection checks and sorting-based depth ordering. It addresses the inefficiency of standard 3D-GS rasterization by generating a sparse set of Gaussian-tile pairs.

### [Ghost Ellipsoids for Chunked Training](3559-ghost-ellipsoids-for-chunked-training.md)
Ghost Ellipsoids for Chunked Training resolves boundary artifacts in chunked volume training by including ellipsoids from neighboring chunks in the loss computation without updating their parameters. It provides boundary context without requiring a global volume.

### [2D Gaussian Splatting Rasterization](4815-2d-gaussian-splatting-rasterization.md)
2D Gaussian Splatting Rasterization renders a scene represented by 3D Gaussian surfels by projecting them to 2D screen-space ellipses. It differs from its parent by treating primitives as planar surfels with explicit normal vectors, enabling the accumulation of normal maps alongside color and depth.

### [Hardware-Accelerated Gaussian Rasterization](7329-hardware-accelerated-gaussian-rasterization.md)
Hardware-Accelerated Gaussian Rasterization is a hardware-algorithm co-design that augments standard triangle rasterizers with dedicated units for Gaussian projection and blending. It solves the bottleneck of processing millions of 3D Gaussians on general-purpose hardware.

### [AdR-Gaussian](8365-adr-gaussian.md)
AdR-Gaussian is a rendering acceleration method for 3D Gaussian Splatting that optimizes the rasterization pipeline by moving serial culling operations to a parallel preprocessing stage and balancing computational loads across pixel threads.

### [QuadBox](9231-quadbox.md)
QuadBox is a Gaussian-tile intersection method that uses four axis-aligned bounding boxes (AABBs) to tightly encapsulate projected 2D Gaussians, reducing tile over-approximation and redundant computations in the rasterizer preprocess stage.

### [Projection Geometry and Coordinate Systems](9313-projection-geometry-and-coordinate-systems.md)
This article defines the mathematical framework for transforming 3D Gaussian primitives from world space into the 2D image plane for rasterization in 3D Gaussian Splatting. It solves the problem of accurately projecting 3D ellipsoids onto a 2D sensor plane.

### [Point-Based Ray Tracing for Visibility](10934-point-based-ray-tracing-for-visibility.md)
Point-Based Ray Tracing for Visibility serves as a sparse visibility oracle that evaluates transmittance along sampled light directions using a bounding volume hierarchy (BVH) of Gaussian proxies. It decouples visibility estimation from screen-space projection.

### [Primitive Representation](11838-primitive-representation.md)
Primitive Representation defines the geometric and photometric data structures used to approximate a 3D scene in Gaussian Splatting methods. It explicitly defines the kernel shape and the associated attribute storage.

### [Shading and Lighting Models](11924-shading-and-lighting-models.md)
Shading and Lighting Models extend the standard 3D Gaussian Splatting rasterization pipeline to support physically based rendering (PBR), self-occlusion, and anti-aliased forward shading. It introduces mechanisms for deferred shading pipelines and light-space rasterization.

### [Reflective Gaussian Splatting](12321-reflective-gaussian-splatting.md)
Reflective Gaussian Splatting (Ref-Gaussian) is a framework for the real-time, high-quality rendering of reflective objects with inter-reflection effects. It distinguishes itself by utilizing 2D Gaussian primitives for view-consistent geometry and employing physically based deferred rendering.

### Temporal Marginal Probability Filtering
Pre-rasterization culling step that removes 4D Gaussians with negligible temporal support at the current frame timestamp, stabilizing rendering speed across video length. It computes the marginal probability $p_i(t)$ for each 4D Gaussian $i$ at the current frame's timestamp $t$ and discards Gaussians with negligible $p_i(t)$ before executing standard frustum culling. Unlike methods where Gaussian count scales with video length, this filtering keeps active Gaussians constant, making the approach friendly to long videos in terms of rendering speed [S6, unofficial].

### t-distribution truncation for splatting
A projection truncation technique that replaces the fixed three-sigma rule used in vanilla 3DGS with a $\nu$-dependent critical value interpolation to handle fat-tailed Student's t-distributions. For t-distributions with thin tails (high $\nu$), it applies the standard three-sigma rule directly. For fat-tailed distributions (low $\nu$), it obtains critical values for different $\nu$ degrees and interpolates between them to compute the appropriate truncation threshold. Using a fixed three-sigma threshold on fat-tailed t-distributions causes excessive truncation or leakage, which interpolation resolves [S7].

### Gaussian Opacity Fields (GOF)
Uses ray-tracing-style intersection for accurate depth and derives a level-set surface from accumulated opacity [S5, unofficial].

### Progressive Normal Propagation
Uses progressive propagation along normals to fill texture-less regions [S5, unofficial].

### Splat TBO Upload & Layout
Packs per-splat data (center, covariance, DC color/alpha, optional SH) into Texture Buffer Objects (TBOs) for GPU consumption. It iterates over splats, copying center.xyz as raw float bits to `d[0..2]`, packing covariance upper-triangle to `d[3..5]` using `packHalf2x16`, and packing DC color/alpha to `d[6]` using `packDCColor`. The base TBO uses 2 uvec4 texels per splat (32 B). The SH TBO uses GL_RGBA16F format, with 8 B per non-DC SH coefficient per splat. Memory usage is 32 B/splat for base TBO + 8 B $\times$ sh_coeffs_per_splat_ $\times$ splat_count_ for SH TBO [S8, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by sorting $N$ Gaussians by depth, which is $O(N \log N)$ or $O(N)$ with radix sort. Rasterization is $O(P \cdot G_{tile})$ where $P$ is the number of pixels and $G_{tile}$ is the average number of Gaussians per tile [S4, unofficial].
- **Memory Complexity:** $O(N)$ for storing Gaussian attributes. High VRAM usage (4GB to view, 12GB to train) is common due to the large number of Gaussians [S4, unofficial].
- **Measured Performance:** Real-time rendering rates are achievable with efficient CUDA implementations [S1]. The primary bottleneck is sorting millions of Gaussians, which is done efficiently using CUB device radix sort in the original implementation [S4, unofficial].
- **Strengths:** High-quality, photorealistic scenes; fast, real-time rasterization; relatively fast to train [S4, unofficial].
- **Limitations:** High VRAM usage; large disk size (1GB+ for a scene); incompatible with existing rendering pipelines (Vulkan, DirectX, WebGPU) without adaptation; static scenes (for now) [S4, unofficial].
- **When to Use:** Real-time rendering of photorealistic scenes learned from small samples of images.
- **When Not to Use:** When compatibility with existing rendering pipelines is required, or when memory/disk constraints are tight.

## Background
3D Gaussian Splatting was introduced in "3D Gaussian Splatting for Real-Time Radiance Field Rendering" [S4, unofficial]. It represents a breakthrough in static scene rendering by representing scenes as collections of 3D Gaussian primitives, enabling real-time rendering rates while maintaining high visual fidelity [S1]. The method is analogous to triangle rasterization but uses Gaussians instead of triangles [S4, unofficial]. It has been extended to 4D (space+time) and 6D (space+direction) Gaussian Splatting to address dynamic scenes and view-dependent effects [S1].

## Sources
[S1] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
[S2] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
[S3] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S4] Introduction to 3D Gaussian Splatting - Hugging Face, https://huggingface.co/blog/gaussian-splatting, fetched 2026-10-27 (unofficial)
[S5] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S6] “Real-time Photorealistic Dynamic Scene Representation and Rendering with 4D Gaussian Splatting”, https://radiancefields.com/papers/real-time-photorealistic-dynamic-scene-representation-and-rendering-with-4d-gaussian-splatting, fetched 2026-10-30 (unofficial)
[S7] 3D Student Splatting and Scooping, https://doi.org/10.1109/cvpr52734.2025.01960, fetched 2026-10-29
[S8] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
