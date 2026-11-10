# 3D Gaussian Splatting Rasterization

Also called: 3D Gaussian Splatting (3DGS), 3DGS Rasterization, Differentiable Gaussian Rasterization, TrimGS, gsplat.rasterization

## Overview
3D Gaussian Splatting Rasterization is a differentiable rendering technique that represents a scene as a collection of anisotropic 3D Gaussians and projects them onto a 2D image plane for real-time rendering [S1][S3, S4, unofficial]. It solves the problem of efficiently rendering continuous volumetric representations by converting each primitive into a 2D splat, sorting them by depth, and accumulating color via alpha compositing [S4, S5, unofficial]. It differs from its siblings by providing the core differentiable forward and backward passes for 3D Gaussians, often implemented as optimized GPU kernels (e.g., CUDA or MLX), whereas alternatives like [Perspective-Correct 3D Gaussian Splatting via Plücker Coordinates](3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md) focus on specific projection corrections or [Analytic-Splatting Forward Shading](3459-analytic-splatting-forward-shading.md) on shading models.

## How it works
**Inputs:**
*   $N$ Gaussians defined by mean position $\mu \in \mathbb{R}^3$, scale $s \in \mathbb{R}^3$, rotation $q \in \mathbb{R}^4$ (quaternion), opacity $\alpha \in \mathbb{R}$, and spherical harmonics (SH) coefficients $C_{SH}$ or direct color $C_{RGB} \in \mathbb{R}^3$ [S1][S2, S4, unofficial].
*   Camera parameters: View matrix $V \in \mathbb{R}^{4 \times 4}$, projection matrix $K \in \mathbb{R}^{3 \times 3}$, image dimensions $W \times H$ [S2, S3, unofficial].
*   Parameters: `render_mode` (e.g., "RGB", "D"), `rasterize_mode` ("classic", "antialiased"), `camera_model` ("pinhole", "ortho", "fisheye"), `tile_size` (default 16), `near_plane` (default 0.01), `far_plane` (default $10^{10}$) [S2, unofficial].

**Outputs:**
*   Rendered image $I \in \mathbb{R}^{H \times W \times 3}$ (and optionally depth/mask) [S2, S3, unofficial].

**Data Structures:**
*   **Gaussian Attributes:** Stored in contiguous arrays or tensors.
    *   `means`: $[N, 3]$ float32.
    *   `quats`: $[N, 4]$ float32 (normalized).
    *   `scales`: $[N, 3]$ float32.
    *   `opacities`: $[N]$ float32.
    *   `colors`: $[N, 3]$ or $[N, K]$ if SH is used.
*   **Projected Attributes (per camera):**
    *   `means_2d`: $[N, 2]$ float32.
    *   `conics`: $[N, 3]$ float32, storing the upper triangle of $\Sigma_{2D}^{-1}$ as $[C_{00}, C_{01}, C_{11}]$ [S2, S3, unofficial].
    *   `radii`: $[N, 2]$ int32, bounding box radius in pixels.
    *   `depths`: $[N]$ float32, camera-space z-depth.

**Steps:**

1.  **Covariance Construction:** Compute the 3D covariance matrix $\Sigma_{3D}$ from scale $S = \text{diag}(s_x, s_y, s_z)$ and rotation $R$ (derived from quaternion $q$) [S1][S2, unofficial]:
    $$ \Sigma_{3D} = R S S^T R^T $$
    [S2, unofficial]

2.  **Projection:** Transform means to camera space and project to 2D screen space using the local affine approximation of the projective transformation [S3, unofficial]. Compute the 2D covariance $\Sigma_{2D}$ using the Jacobian $J$ of the projection function:
    $$ \Sigma_{2D} = J \Sigma'_{3D} J^T $$
    where $\Sigma'_{3D}$ is the projected 3D covariance [S3, unofficial]. Compute the conic vector $C = [\Sigma_{2D}^{-1}[0,0], \Sigma_{2D}^{-1}[0,1], \Sigma_{2D}^{-1}[1,1]]$ for efficient evaluation [S2, S3, unofficial].

3.  **Culling and Sorting:** Filter Gaussians behind the camera ($z < \text{near\_plane}$) [S3, unofficial]. Sort the remaining Gaussians by camera-space depth to ensure correct alpha compositing order [S2, S3, S4, unofficial].

4.  **Tile Intersection:** Divide the image into tiles of size `tile_size` $\times$ `tile_size`. For each Gaussian, determine which tiles it intersects using its 2D bounding box (radii) [S2, unofficial].

5.  **Rasterization (Splatting):** For each pixel in each tile, iterate over the intersecting Gaussians in front-to-back order. Compute the alpha value $\alpha_i$ for each Gaussian at the pixel using the 2D Gaussian PDF and the Gaussian's opacity [S3, unofficial]:
    $$ \alpha_i = \alpha_{\text{gauss}} \cdot \exp\left(-\frac{1}{2} \mathbf{d}^T \Sigma_{2D}^{-1} \mathbf{d}\right) $$
    where $\mathbf{d}$ is the vector from the pixel center to the Gaussian mean [S3, unofficial]. Compute the weight $w_i = \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)$ [S2, S3, unofficial].

6.  **Compositing:** Accumulate the color $C_i$ (evaluated from SH or direct color) weighted by $w_i$. The final pixel color is:
    $$ C_{\text{pixel}} = \sum_{i} w_i C_i + \left(1 - \sum_{i} w_i\right) C_{\text{bg}} $$
    [S2, S3, unofficial]

7.  **Antialiasing (Optional):** If `rasterize_mode` is "antialiased", apply compensation factors to the opacities to account for sub-pixel sampling [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterize_gaussians(means_3d, quats, scales, opacities, colors, viewmats, Ks, width, height, tile_size=16, near_plane=0.01, far_plane=1e10, sh_degree=None):
    """
    Rasterize 3D Gaussians to 2D image.
    Args:
        means_3d: [N, 3]
        quats: [N, 4]
        scales: [N, 3]
        opacities: [N]
        colors: [N, 3] or [N, K] if SH
        viewmats: [C, 4, 4]
        Ks: [C, 3, 3]
        width, height: int
        tile_size: int
        near_plane, far_plane: float
        sh_degree: int or None
    Returns:
        render_colors: [C, H, W, 3]
        render_alphas: [C, H, W, 1]
    """
    C = viewmats.shape[0]
    N = means_3d.shape[0]
    render_colors = mx.zeros((C, height, width, 3))
    render_alphas = mx.zeros((C, height, width, 1))

    for c in range(C):
        # 1. Project to 2D
        # Compute 3D covariance from quat and scale
        cov_3d = compute_cov_3d(quats, scales) # [N, 3, 3]
        
        # Linearized projection
        means_2d, cov_2d, radii, depths, conics = linearized_project(
            means_3d, cov_3d, viewmats[c], Ks[c], width, height, 
            near_plane, far_plane, calc_compensations=True
        ) # [N, 2], [N, 3, 3], [N, 2], [N], [N, 3]
        
        # 2. Compute SH colors if needed
        if sh_degree is not None:
            campos = viewmats[c, :3, 3]
            dirs = means_3d - campos
            dirs = dirs / mx.maximum(mx.linalg.norm(dirs, axis=1, keepdims=True), 1e-8)
            # Broadcast SH coefficients and evaluate
            sh_colors = evaluate_spherical_harmonics(sh_degree, dirs, colors) # [N, 3]
            colors_for_raster = sh_colors
        else:
            colors_for_raster = colors # [N, 3]
            
        # 3. Broadcast opacities and apply compensations
        opacities_cn = mx.broadcast_to(mx.expand_dims(opacities, 0), (C, N))[c]
        # Apply compensation if antialiased (not shown in detail, assumed in conics/alphas)
        
        # 4. Tile Intersection
        tile_width = mx.ceil(width / tile_size)
        tile_height = mx.ceil(height / tile_size)
        tiles_per_gauss, isect_ids, flatten_ids = isect_tiles(
            means_2d, radii, depths, tile_size, tile_width, tile_height, sort=True
        )
        
        # 5. Encode offsets
        isect_offsets = isect_offset_encode(isect_ids, C, tile_width, tile_height)
        
        # 6. Rasterize to pixels
        # This step iterates over tiles and pixels, compositing front-to-back
        render_colors_c, render_alphas_c = rasterize_to_pixels(
            means_2d, conics, colors_for_raster, opacities_cn, 
            width, height, tile_size, isect_offsets, flatten_ids, 
            mx.ones((3)) # background
        )
        
        render_colors[c] = render_colors_c
        render_alphas[c] = render_alphas_c

    return render_colors, render_alphas
```

## Subtopics

### [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md)
Provides the core differentiable forward and backward passes for 3D Gaussians, allowing gradients to flow back to Gaussian parameters and camera view matrices. It differs from siblings by focusing on the differentiable kernel implementation, often in CUDA.

### [Perspective-Correct 3D Gaussian Splatting via Plücker Coordinates](3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md)
Computes Gaussian values using Plücker coordinates for a numerically stable, closed-form perspective-correct solution without matrix inversion.

### [Speedy-Splat Rendering Pipeline](3543-speedy-splat-rendering-pipeline.md)
Replaces coarse bounding-box tile intersection with precise Gaussian-tile intersection checks and sorting-based depth ordering to reduce redundant computations.

### [Ghost Ellipsoids for Chunked Training](3559-ghost-ellipsoids-for-chunked-training.md)
Resolves boundary artifacts in chunked training by including ellipsoids from neighboring chunks in loss computation without updating their parameters.

### [2D Gaussian Splatting Rasterization](4815-2d-gaussian-splatting-rasterization.md)
Renders planar surfels with explicit normal vectors, enabling normal map accumulation alongside color and depth.

### [Hardware-Accelerated Gaussian Rasterization](7329-hardware-accelerated-gaussian-rasterization.md)
Augments standard triangle rasterizers with dedicated units for Gaussian projection and blending for an order-of-magnitude speedup.

### [AdR-Gaussian](8365-adr-gaussian.md)
Optimizes the rasterization pipeline by moving serial culling to a parallel preprocessing stage and balancing loads across pixel threads.

### [QuadBox](9231-quadbox.md)
Uses four axis-aligned bounding boxes to tightly encapsulate projected 2D Gaussians, reducing tile over-approximation.

### [Projection Geometry and Coordinate Systems](9313-projection-geometry-and-coordinate-systems.md)
Defines coordinate transformations and projection mappings for various camera models (panoramic, equidistant) within the 3DGS framework.

### [Point-Based Ray Tracing for Visibility](10934-point-based-ray-tracing-for-visibility.md)
Serves as a sparse visibility oracle using a BVH of Gaussian proxies, decoupling visibility from screen-space projection.

### [Primitive Representation](11838-primitive-representation.md)
Defines the geometric and photometric data structures (kernel shape, attributes) for Gaussian primitives.

### [Shading and Lighting Models](11924-shading-and-lighting-models.md)
Extends standard alpha blending to support dynamic lighting, self-occlusion, and order-independent compositing.

### Temporal Marginal Probability Filtering
Pre-rasterization culling step that removes 4D Gaussians with negligible temporal support at the current frame timestamp, stabilizing rendering speed across video length [S6, unofficial]. It computes the marginal probability $p_i(t)$ for each 4D Gaussian $i$ at the current frame's timestamp $t$ and discards those with negligible $p_i(t)$ before frustum culling [S6, unofficial]. Unlike methods where Gaussian count scales with video length, this keeps active Gaussians constant, making it friendly to long videos [S6, unofficial].

### t-distribution truncation for splatting
A projection truncation technique that replaces the fixed three-sigma rule used in vanilla 3DGS with a $\nu$-dependent critical value interpolation to handle fat-tailed Student's t-distributions [S7]. For thin tails (high $\nu$), it applies the standard three-sigma rule; for fat tails (low $\nu$), it interpolates critical values from reference [9] to compute the truncation threshold [S7]. Using a fixed threshold on fat-tailed distributions causes excessive truncation or leakage, which this method resolves [S7].

### Gaussian Opacity Fields (GOF)
Uses ray-tracing-style intersection for accurate depth and derives a level-set surface from accumulated opacity [S5, unofficial].

### Progressive Normal Propagation
Uses progressive propagation along normals to fill texture-less regions [S5, unofficial].

### Splat TBO Upload & Layout
Packs per-splat data into Texture Buffer Objects (TBOs) for GPU consumption [S8, unofficial]. It iterates over splats, copying `center.xyz` as raw float bits to `d[0..2]`, packing covariance upper-triangle to `d[3..5]` using `packHalf2x16`, and packing DC color/alpha to `d[6]` using `packDCColor` [S8, unofficial]. The base TBO uses `GL_RGBA32UI` format with 2 uvec4 texels per splat (32 B) [S8, unofficial]. SH coefficients are packed into a `GL_RGBA16F` TBO, with layout `[splat0_c1, splat0_c2, …, splat1_c1, splat1_c2, …]` [S8, unofficial]. Memory usage is 32 B/splat for base TBO + 8 B $\times$ sh_coeffs_per_splat_ $\times$ splat_count_ for SH TBO, where sh_coeffs_per_splat_ = $(active\_sh\_degree\_ + 1)^2 - 1$ [S8, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by sorting $N$ Gaussians by depth, typically $O(N \log N)$ or $O(N)$ with radix sort [S4, unofficial]. Rasterization is $O(N \cdot A)$ where $A$ is the average number of Gaussians per pixel/tile.
*   **Memory Complexity:** $O(N)$ for Gaussian attributes. High VRAM usage: ~4GB to view, ~12GB to train [S4, unofficial]. Large disk size: 1GB+ for a scene [S4, unofficial].
*   **Performance:** Enables real-time rendering (60+ FPS) on modern GPUs [S1][S4, unofficial]. The primary bottleneck is sorting millions of Gaussians, efficiently handled by CUB device radix sort in CUDA [S4, unofficial].
*   **Strengths:** High visual fidelity, real-time performance, differentiable training [S4, unofficial].
*   **Limitations:** High memory consumption, static scenes (without extensions), incompatibility with some existing rendering pipelines (e.g., WebGL requires quad-based approximations) [S4, unofficial].

## Background
3D Gaussian Splatting was introduced in "3D Gaussian Splatting for Real-Time Radiance Field Rendering" [S4, unofficial]. It builds on the idea of representing scenes as discrete primitives, analogous to triangle rasterization but using Gaussians [S4, unofficial]. The method enables real-time rendering of photorealistic scenes learned from small samples of images [S1][S4, unofficial]. It differs from traditional quad-based rasterization approaches used in some viewers (WebGL, Unity) by using direct Gaussian rasterization, which may result in lower quality/performance in those contexts but is native to the 3DGS pipeline [S4, unofficial].

## Sources
[S1] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
[S2] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
[S3] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S4] Introduction to 3D Gaussian Splatting - Hugging Face, https://huggingface.co/blog/gaussian-splatting, fetched 2026-10-27 (unofficial)
[S5] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S6] “Real-time Photorealistic Dynamic Scene Representation and Rendering with 4D Gaussian Splatting”, https://radiancefields.com/papers/real-time-photorealistic-dynamic-scene-representation-and-rendering-with-4d-gaussian-splatting, fetched 2026-10-30 (unofficial)
[S7] 3D Student Splatting and Scooping, https://doi.org/10.1109/cvpr52734.2025.01960, fetched 2026-10-29
[S8] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
