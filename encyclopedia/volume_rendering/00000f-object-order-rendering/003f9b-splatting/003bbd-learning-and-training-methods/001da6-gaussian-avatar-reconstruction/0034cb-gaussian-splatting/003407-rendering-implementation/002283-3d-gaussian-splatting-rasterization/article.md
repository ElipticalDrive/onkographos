# 3D Gaussian Splatting Rasterization

Also called: 3D Gaussian Splatting (3DGS), 3DGS Rasterization, Differentiable Gaussian Rasterization, TrimGS, gsplat.rasterization

## Overview
3D Gaussian Splatting Rasterization is a screen-space rendering technique that projects a set of 3D Gaussian primitives into 2D ellipses, sorts them by depth, and alpha-composites them to produce an RGB image [S2, S6, unofficial][S9, unofficial]. It solves the problem of efficiently rendering large-scale point-cloud-based scene representations while maintaining full differentiability with respect to all Gaussian parameters (position, scale, rotation, opacity, and color) and camera poses [S2, S6, unofficial][S9, unofficial]. It differs from volumetric ray-marching methods like NeRF by using discrete rasterization of 2D kernels rather than continuous integral evaluation, and from standard OpenGL rasterization by operating on non-polygonal, probabilistic primitives with a custom front-to-back blending order [S2, S6, unofficial][S9, unofficial].

## How it works
**Inputs:**
*   `means_3D`: $N \times 3$ tensor of Gaussian mean positions in world space.
*   `quats`: $N \times 4$ tensor of unit quaternions $(w, x, y, z)$ representing rotation.
*   `scales`: $N \times 3$ tensor of scaling factors $(s_x, s_y, s_z)$.
*   `opacities`: $N$ tensor of scalar opacity values $\alpha_i \in [0, 1]$.
*   `colors`: $N \times D$ tensor of color attributes. If `sh_degree` is None, $D=3$ (RGB). If `sh_degree` > 0, $D = 3 \times (sh\_degree + 1)^2$ (SH coefficients).
*   `viewmats`: $C \times 4 \times 4$ camera-to-world transformation matrices (or world-to-camera depending on convention; standard is World-to-Camera for projection math, but inputs often provided as View Matrices $V$). Let $V$ be the $4 \times 4$ matrix transforming world to camera space.
*   `Ks`: $C \times 3 \times 3$ camera intrinsic matrices.
*   `width`, `height`: Image dimensions in pixels.
*   `tile_size`: Integer, typically 16. Side length of tiles for tile-based rasterization.
*   `near_plane`, `far_plane`: Floats, typically 0.01 and $10^{10}$ respectively.
*   `sh_degree`: Integer or None. If not None, enables spherical harmonics color computation.
*   `camera_model`: String, typically "pinhole", "ortho", or "fisheye".
*   `render_mode`: String specifying output channels: "RGB", "D" (depth), "ED" (expected depth), "RGB+D", "RGB+ED", "d" (hit distance), "Ed", "RGB+d", "RGB+Ed".
*   `rasterize_mode`: String, "classic" or "antialiased".
*   `backgrounds`: $C \times 3$ tensor of background colors.

**Outputs:**
*   `render_colors`: $C \times H \times W \times D'$ tensor of rendered colors (and optionally depth/distance channels).
*   `render_alphas`: $C \times H \times W \times 1$ tensor of accumulated alpha values.

**Data Structures:**
*   **Conic Representation**: The inverse 2D covariance $\Sigma_{2D}^{-1}$ is stored as a 3-element vector $C = [\Sigma_{2D}^{-1}[0,0], \Sigma_{2D}^{-1}[0,1], \Sigma_{2D}^{-1}[1,1]]$ to avoid repeated matrix inversions during rasterization [S2, S3, unofficial].
*   **Tile Intersection Data**: `tiles_per_gauss`: $N$ tensor indicating which tiles each Gaussian intersects. `isect_ids`: Flattened list of Gaussian IDs sorted by tile. `isect_offsets`: $C \times H_{tile} \times W_{tile}$ tensor encoding the start index in `isect_ids` for each tile.

**Steps:**

1.  **Input Validation and Parameter Activation**:
    Validate `render_mode`, `rasterize_mode`, and `camera_model` against allowed sets [S2, unofficial]. Apply activation functions to raw parameters to ensure valid ranges: quaternions are normalized, scales are exponentiated ($s = \exp(\text{raw\_s})$), and opacities are sigmoided ($\alpha = \sigma(\text{raw\_\alpha})$) [S3, unofficial].

2.  **Covariance Construction**:
    Compute the 3D covariance matrix $\Sigma_{3D}$ from quaternions and scales. First, convert quaternions to rotation matrices $R$. Construct diagonal scaling matrix $S = \text{diag}(s_x, s_y, s_z)$. Compute:
    $$ \Sigma_{3D} = R S S^T R^T $$
    [S1][S3, unofficial]. This results in a symmetric $3 \times 3$ matrix.

3.  **Linearized Projection**:
    Transform the 3D means to camera space: $\mu' = V \mu$. Transform the 3D covariance to camera space: $\Sigma' = V_{1:3, 1:3} \Sigma_{3D} V_{1:3, 1:3}^T$. Compute the 2D screen-space covariance $\Sigma_{2D}$ using the Jacobian $J$ of the projective projection:
    $$ \Sigma_{2D} = J \Sigma' J^T $$
    where $J = \begin{pmatrix} \frac{f_x}{\mu'_z} & 0 & -\frac{f_x \mu'_x}{\mu'_z^2} \\ 0 & \frac{f_y}{\mu'_z} & -\frac{f_y \mu'_y}{\mu'_z^2} \end{pmatrix}$ [S2, S3, unofficial].
    *Note*: For orthographic or fisheye models, $J$ is adjusted accordingly [S2, unofficial].

4.  **Conic Computation and Culling**:
    Compute the conic vector $C = [\Sigma_{2D}^{-1}[0,0], \Sigma_{2D}^{-1}[0,1], \Sigma_{2D}^{-1}[1,1]]$ [S2, S3, unofficial].
    Filter Gaussians:
    *   Remove those behind the near plane ($\mu'_z < z_{near}$) [S3, S4, unofficial].
    *   Remove those outside the view frustum (screen-space bounds) [S3, S4, unofficial].
    *   If `rasterize_mode` is "antialiased", compute compensation factors to account for sub-pixel aliasing [S2, unofficial].

5.  **Color Computation (Spherical Harmonics)**:
    If `sh_degree` is not None:
    *   Compute view directions: $\mathbf{d}_i = \text{normalize}(\text{campos} - \mu_i)$ [S2, unofficial].
    *   Evaluate spherical harmonics: $c_i = \sum_{\ell=0}^{N} \sum_{m=-\ell}^{\ell} \beta_{\ell m} Y_{\ell m}(\mathbf{d}_i)$ [S1].
    *   Clamp color values: $c_i = \max(c_i + 0.5, 0.0)$ [S2, unofficial].
    If `sh_degree` is None, use direct RGB colors [S2, unofficial].

6.  **Tile Intersection and Sorting**:
    *   Compute bounding boxes for each Gaussian in screen space using the 2D covariance eigenvalues or a fixed sigma multiplier (typically 3$\sigma$) [S4, unofficial][S7].
    *   Determine which tiles each Gaussian intersects [S2, unofficial].
    *   Sort Gaussians by camera-space depth $z$ within each tile to ensure correct front-to-back compositing [S3, S4, unofficial]. The original implementation uses CUB device radix sort for efficiency [S4, unofficial].

7.  **Rasterization (Pixel-wise Compositing)**:
    For each pixel $(u, v)$ and each camera $c$:
    *   Initialize accumulated color $C_{acc} = 0$ and transmittance $T = 1.0$ [S3, unofficial].
    *   Iterate over Gaussians $i$ that cover pixel $(u, v)$ in depth order:
        *   Compute the 2D offset vector $\mathbf{x} = [u - \mu'_{x,i}, v - \mu'_{y,i}]^T$ [S3, unofficial].
        *   Compute the alpha contribution:
            $$ \alpha_{pixel, i} = \alpha_i \cdot \exp\left(-\frac{1}{2} \mathbf{x}^T \Sigma_{2D, i}^{-1} \mathbf{x}\right) $$
            [S3, unofficial].
        *   Update accumulation:
            $$ C_{acc} \leftarrow C_{acc} + T \cdot \alpha_{pixel, i} \cdot c_i $$
            $$ T \leftarrow T \cdot (1 - \alpha_{pixel, i}) $$
            [S3, unofficial].
    *   If `render_mode` includes depth, compute weighted depth: $D_{acc} = \sum T_i \cdot \alpha_{pixel, i} \cdot z_i$ and expected depth $D_{exp} = D_{acc} / \max(\sum T_i \cdot \alpha_{pixel, i}, \epsilon)$ [S2, unofficial].
    *   Final color: $C_{final} = C_{acc} + T \cdot C_{bg}$ [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterize_gaussians(
    means_3d, quats, scales, opacities, colors,
    viewmats, Ks, width, height,
    tile_size=16, near_plane=0.01, far_plane=1e10,
    sh_degree=None, background=None
):
    N = means_3d.shape[0]
    C = viewmats.shape[0]
    
    # 1. Activation (simplified)
    scales = torch.exp(scales)
    opacities = torch.sigmoid(opacities)
    quats = torch.nn.functional.normalize(quats, dim=-1)
    
    # 2. Covariance Construction
    # R is rotation matrix from quats
    R = quat_to_rotmat(quats) # Helper defined in standard libraries
    S = torch.diag_embed(scales)
    cov_3d = torch.bmm(torch.bmm(R, S), torch.bmm(S, R.transpose(1, 2)))
    
    # 3. Projection
    # Transform means to camera space
    # viewmat is 4x4, means is Nx3
    means_cam = torch.einsum('cij,nj->cni', viewmats[:, :3, :3], means_3d) + viewmats[:, :3, 3, None, :]
    
    # Transform covariance
    cov_cam = torch.einsum('cij,njk,clk->cnil', viewmats[:, :3, :3], cov_3d, viewmats[:, :3, :3])
    
    # Jacobian J
    fx = Ks[:, 0, 0][:, None, None]
    fy = Ks[:, 1, 1][:, None, None]
    cx = Ks[:, 0, 2][:, None, None]
    cy = Ks[:, 1, 2][:, None, None]
    
    z = means_cam[:, :, 2]
    # Avoid division by zero
    eps = 1e-6
    
    # 2D Covariance Sigma_2D
    # J = [[fx/z, 0, -fx*x/z^2], [0, fy/z, -fy*y/z^2]]
    # Sigma_2D = J * Sigma_cam * J^T
    # Simplified calculation for symmetric Sigma_cam
    # ... (Standard linearized projection math) ...
    cov_2d = compute_cov_2d(cov_cam, means_cam, fx, fy, eps)
    
    # 4. Conics and Culling
    # Compute conics C = [Sigma^-1[0,0], Sigma^-1[0,1], Sigma^-1[1,1]]
    conics = compute_conics(cov_2d)
    
    # Filter by near plane and frustum
    mask = (z > near_plane) & (z < far_plane)
    # Further frustum culling based on screen bounds
    # ...
    
    # 5. Color (SH)
    if sh_degree is not None:
        dirs = means_3d[None, :, :] - viewmats[:, 0:3, 3, None, :] # Approx cam pos
        dirs = torch.nn.functional.normalize(dirs, dim=-1)
        colors = eval_sh(sh_degree, dirs, colors)
        colors = torch.clamp(colors + 0.5, 0.0, 1.0)
    
    # 6. Tile Intersection and Sorting
    # Compute radii and bounding boxes
    radii = compute_radii(cov_2d)
    tiles_per_gauss, isect_ids, flatten_ids = intersect_tiles(
        means_cam[:, :, :2], radii, width, height, tile_size
    )
    
    # Sort by depth within tiles
    sorted_gaussians = sort_gaussians_by_depth(means_cam, flatten_ids)
    
    # 7. Rasterization
    # Initialize output buffers
    out_color = torch.zeros(C, height, width, 3, device=means_3d.device)
    out_alpha = torch.zeros(C, height, width, 1, device=means_3d.device)
    
    # Iterate tiles and pixels
    for c in range(C):
        for tile_y in range(0, height, tile_size):
            for tile_x in range(0, width, tile_size):
                # Get gaussians for this tile
                tile_gaussians = get_tile_gaussians(c, tile_x, tile_y, isect_ids, flatten_ids)
                
                for u in range(tile_x, min(tile_x + tile_size, width)):
                    for v in range(tile_y, min(tile_y + tile_size, height)):
                        T = 1.0
                        C_acc = torch.zeros(3, device=means_3d.device)
                        
                        for g in tile_gaussians:
                            # Check if pixel is inside gaussian bounding box
                            if not pixel_in_gauss(u, v, g):
                                continue
                                
                            # Compute alpha
                            x = u - g.mean_2d[0]
                            y = v - g.mean_2d[1]
                            # Alpha = alpha_i * exp(-0.5 * x^T Sigma^-1 x)
                            alpha = g.opacity * torch.exp(-0.5 * (
                                g.conic[0] * x*x + 2*g.conic[1] * x*y + g.conic[2] * y*y
                            ))
                            
                            if alpha > 1e-4: # Threshold for optimization
                                C_acc = C_acc + T * alpha * g.color
                                T = T * (1 - alpha)
                        
                        out_color[c, v, u] = C_acc + T * background[c]
                        out_alpha[c, v, u] = 1 - T
                        
    return out_color, out_alpha
```

## Subtopics

### [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md)
The 3D Gaussian Splatting Differentiable Rasterizer is a screen-space rendering method that projects a set of 3D Gaussian primitives into 2D ellipses, sorts them by depth, and alpha-composites them to produce an RGB image. It solves the problem of efficiently rendering large-scale point-cloud-based scene representations while maintaining full differentiability with respect to all Gaussian parameters (position, scale, rotation, opacity, and color) and camera poses. It differs from volumetric ray-marching methods like NeRF by using discrete rasterization of 2D kernels rather than continuous integral evaluation, and from standard OpenGL rasterization by operating on non-polygonal, probabilistic primitives with a custom front-to-back blending order.

### [Speedy-Splat Rendering Pipeline](3543-speedy-splat-rendering-pipeline.md)
The Speedy-Splat Rendering Pipeline is a GPU-accelerated rasterization method for 3D Gaussian Splatting that replaces coarse bounding-box tile intersection tests with precise Gaussian-tile intersection checks and sorting-based depth ordering. It addresses the inefficiency of standard 3D-GS rasterization, which overestimates Gaussian extents and processes non-intersecting pixels, by generating a sparse set of Gaussian-tile pairs and sorting them for efficient pixel-wise compositing.

### [Ghost Ellipsoids for Chunked Training](3559-ghost-ellipsoids-for-chunked-training.md)
Ghost Ellipsoids for Chunked Training resolves boundary artifacts in chunked volume training by including ellipsoids from neighboring chunks in the loss computation without updating their parameters, providing boundary context without requiring a global volume or merged model. Unlike chunk-based 3DGS methods that resolve boundary inconsistencies by merging chunks into a global coarse model or through post-training stages, this method requires boundary support during training itself without access to a global volume.

### [2D Gaussian Splatting Rasterization](4815-2d-gaussian-splatting-rasterization.md)
2D Gaussian Splatting Rasterization renders a scene represented by 3D Gaussian surfels by projecting them to 2D screen-space ellipses and performing tile-based, front-to-back alpha compositing. It differs from its parent [3D Gaussian Splatting Rasterization](2812-3d-gaussian-splatting-rasterization.md) by treating primitives as planar surfels with explicit normal vectors, enabling the accumulation of normal maps alongside color and depth, and by using a specific ray-surfel intersection logic to determine the contribution of each surfel to a pixel. It serves as a differentiable rendering kernel for 2D Gaussian Splatting, allowing gradients to flow from the rendered image back to the Gaussian parameters [G].

### [Hardware-Accelerated Gaussian Rasterization](7329-hardware-accelerated-gaussian-rasterization.md)
Hardware-Accelerated Gaussian Rasterization is a hardware-algorithm co-design that augments standard triangle rasterizers with dedicated units for Gaussian projection and blending, achieving an order of magnitude speedup in the rasterization stage. It solves the bottleneck of processing millions of 3D Gaussians on general-purpose hardware by isolating specific computational phases into fixed-function pipelines. Unlike software-based differentiable rasterizers [Sibling: 3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md) which rely on GPU kernels, this method uses a four-stage hardware pipeline to handle culling, projection, sorting, and blending in parallel.

### [AdR-Gaussian](8365-adr-gaussian.md)
AdR-Gaussian is a rendering acceleration method for 3D Gaussian Splatting that optimizes the rasterization pipeline by moving serial culling operations to a parallel preprocessing stage and balancing computational loads across pixel threads. It achieves significant speedups without reducing the total number of Gaussians, thereby avoiding the quality degradation associated with pruning-based acceleration methods.

### [Projection Geometry and Coordinate Systems](9313-projection-geometry-and-coordinate-systems.md)
This article defines the mathematical framework for transforming 3D Gaussian primitives from world space into the 2D image plane for rasterization in 3D Gaussian Splatting. It solves the problem of accurately projecting 3D ellipsoids onto a 2D sensor plane while preserving geometric fidelity and enabling differentiable rendering. It sets apart from standard perspective projection by supporting optimal tangent-plane projections [3293-optimal-gaussian-splatting.md] and specialized geometries such as cone-beam X-ray scanning [3458-3d-gaussian-splatting-rasterization-cone-beam-scanner-space.md].

### [Point-Based Ray Tracing for Visibility](10934-point-based-ray-tracing-for-visibility.md)
Point-Based Ray Tracing for Visibility serves as a sparse visibility oracle that evaluates transmittance along sampled light directions using a bounding volume hierarchy (BVH) of Gaussian proxies, enabling evaluation from any point in space in any direction. Unlike the camera-tied rasterizer, it decouples visibility estimation from screen-space projection, allowing it to act as a drop-in companion to standard 3D Gaussian Splatting (3DGS) by reusing the same EWA-splatting formulation for consistent blending without requiring training mismatch.

### [Primitive Representation](11838-primitive-representation.md)
Primitive Representation defines the geometric and photometric data structures used to approximate a 3D scene in Gaussian Splatting methods. It solves the problem of representing continuous geometry and appearance with discrete, differentiable primitives that can be efficiently rasterized. It sets itself apart from parent methods by explicitly defining the kernel shape (e.g., isotropic, half-space, or hybrid Bézier) and the associated attribute storage (e.g., opacity, normals, control points) that determine how light interacts with the primitive during rendering.

### [Shading and Lighting Models](11924-shading-and-lighting-models.md)
Shading and Lighting Models extend the standard 3D Gaussian Splatting rasterization pipeline to support physically based rendering (PBR), self-occlusion, and anti-aliased forward shading. While standard splatting assumes a fixed, uniform lighting environment and simple alpha blending, this family introduces mechanisms for deferred shading pipelines, light-space rasterization for shadows, and analytic integration approximations for multiscale fidelity. It differs from sibling methods like [Analytic-Splatting Forward Shading](3459-analytic-splatting-forward-shading.md) by focusing on material attributes and light interaction rather than just pixel-area integration, and from [Transparency and Blending Models](9274-transparency-and-blending-models.md) by explicitly handling light source geometry and PBR material buffers.

### [Reflective Gaussian Splatting](12321-reflective-gaussian-splatting.md)
Reflective Gaussian Splatting (Ref-Gaussian) is a framework for the real-time, high-quality rendering of reflective objects with inter-reflection effects, addressing the limitations of standard 3D Gaussian Splatting in modeling complex surfaces and multi-view inconsistencies. It distinguishes itself from sibling methods like [3D Gaussian Splatting Rasterization](2812-3d-gaussian-splatting-rasterization.md) by utilizing 2D Gaussian primitives for view-consistent geometry and employing physically based deferred rendering with split-sum approximation to stabilize optimization and enable efficient inter-reflection modeling without heavy Monte Carlo sampling.

### [Ray-Splat Intersection Algorithms](13401-ray-splat-intersection-algorithms.md)
Ray-splat intersection algorithms determine the contribution of 3D Gaussian primitives to a pixel by evaluating the Gaussian density along the viewing ray rather than projecting the 3D covariance to 2D screen space. This approach solves the numerical instability and projection errors inherent in standard rasterization, particularly for degenerate splats or extreme perspective. It sets itself apart from sibling methods like [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md) by avoiding the explicit inversion of the 4x4 projection matrix during per-pixel evaluation, trading slightly higher per-pixel cost for robustness to degeneracy [3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md].

### Temporal Marginal Probability Filtering
Pre-rasterization culling step that removes 4D Gaussians with negligible temporal support at the current frame timestamp, stabilizing rendering speed across video length [S6, unofficial]. Compute the marginal probability $p_i(t)$ for each 4D Gaussian $i$ at the current frame's timestamp $t$. Discard Gaussians with negligible $p_i(t)$ before executing standard frustum culling. Unlike methods where Gaussian count scales with video length, this filtering keeps active Gaussians constant, making the approach friendly to long videos in terms of rendering speed [S6, unofficial].

### t-distribution truncation for splatting
A projection truncation technique that replaces the fixed three-sigma rule used in vanilla 3DGS with a $\nu$-dependent critical value interpolation to handle fat-tailed Student's t-distributions [S7]. For t-distributions with thin tails (high $\nu$), apply the standard three-sigma rule directly. For fat-tailed distributions (low $\nu$), obtain critical values for different $\nu$ degrees from reference [9] and interpolate between them to compute the appropriate truncation threshold. Using a fixed three-sigma threshold on fat-tailed t-distributions causes excessive truncation or leakage. Interpolation across $\nu$ degrees resolves this [S7].

### Progressive Normal Propagation
Uses progressive propagation along normals to fill texture-less regions [S5, unofficial].

### Splat TBO Upload & Layout
Packs per-splat data (center, covariance, DC color/alpha, optional SH) into Texture Buffer Objects (TBOs) for GPU consumption [S8, unofficial]. Iterate over splats. For each splat, copy center.xyz as raw float bits to d[0..2]. Pack covariance upper-triangle to d[3..5] using packHalf2x16: d[3]=covA[0],covA[1]; d[4]=covA[2],covB[0]; d[5]=covB[1],covB[2]. Pack DC color/alpha to d[6] using packDCColor. Set d[7]=0. Create GL_TEXTURE_BUFFER buffer with GL_RGBA32UI format. Bind to tbo_tex_. For non-DC SH coefficients, pack each coefficient's RGB into RGBA16F texels (B=0), layout is [splat0_c1, splat0_c2, …, splat1_c1, splat1_c2, …]. Create SH TBO with GL_RGBA16F format. Memory: 32 B/splat for base TBO + 8 B × sh_coeffs_per_splat_ × splat_count_ for SH TBO. sh_coeffs_per_splat_ = (active_sh_degree_ + 1)^2 - 1 (excludes DC) [S8, unofficial].

### 3D Gaussian Splatting for X-ray Projection Rasterization
A rendering technique that leverages 3D Gaussian Splatting (3DGS) to rasterize X-ray projections by modeling X-ray attenuation using the Beer-Lambert law, treating the volume as a collection of 3D Gaussians with density and isotropic absorption [S9, unofficial]. The exponential transmittance model $T(t)$ is defined as $T(t) = \exp\left(-\int_{t_0}^t \sigma(\mathbf{r}(s)) ds\right)$, where $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ represents a 3D point along a ray with origin $\mathbf{o}$ and direction $\mathbf{d}$, and $\sigma(\mathbf{p})$ denotes the density at point $\mathbf{p}$ [S9, unofficial]. Color attributes are optional and can be excluded from the rendering model, accounting solely for isotropic absorption [S9, unofficial].

### Ray-Gaussian Intersection
Project the 3D Gaussian's covariance and center onto the camera ray to convert the 3D influence into a 1D Gaussian function along that specific ray [S10].

## Operating characteristics
**Time Complexity:** $O(N \cdot T \cdot P)$ where $N$ is the number of Gaussians, $T$ is the number of tiles, and $P$ is the average number of Gaussians per tile. The dominant cost is sorting $N$ Gaussians, which is $O(N \log N)$ or $O(N)$ with radix sort [S4, unofficial].
**Memory Complexity:** $O(N)$ for storing Gaussian parameters and intermediate buffers. High VRAM usage: 4GB to view, 12GB to train [S4, unofficial]. Large disk size: 1GB+ for a scene [S4, unofficial].
**Performance:** Real-time rendering rates (60+ FPS) are achievable on modern GPUs for scenes with millions of Gaussians [S1]. The primary bottleneck is sorting millions of gaussians, which is done efficiently in the original implementation using CUB device radix sort, a highly optimized sort only available in CUDA [S4, unofficial].
**Strengths:** High-quality, photorealistic scenes; fast, real-time rasterization; relatively fast to train [S4, unofficial].
**Limitations:** High VRAM usage; large disk size; incompatible with existing rendering pipelines (Vulkan, DirectX, WebGPU) without adaptation; static (for now) [S4, unofficial]. Unlike traditional quad-based rasterization approaches used in some viewers (WebGL, Unity), 3D Gaussian Splatting uses direct gaussian rasterization; quad-based approaches may result in lower quality/performance but are compatible with existing pipelines [S4, unofficial].

## Background
3D Gaussian Splatting was introduced in the paper "3D Gaussian Splatting for Real-Time Radiance Field Rendering" [S4, unofficial]. It represents a breakthrough in static scene rendering by representing scenes as collections of 3D Gaussian primitives, enabling real-time rendering rates while maintaining high visual fidelity [S1]. The method is analogous to triangle rasterization but uses Gaussians instead of triangles [S4, unofficial]. It is typically initialized using Structure from Motion (SfM) methods like COLMAP to estimate a point cloud, which is then converted to Gaussians [S4, unofficial]. Training uses Stochastic Gradient Descent with automated densification and pruning [S4, unofficial].

## Sources
[S1] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
[S2] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
[S3] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S4] Introduction to 3D Gaussian Splatting - Hugging Face, https://huggingface.co/blog/gaussian-splatting, fetched 2026-10-27 (unofficial)
[S5] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S6] “Real-time Photorealistic Dynamic Scene Representation and Rendering with 4D Gaussian Splatting”, https://radiancefields.com/papers/real-time-photorealistic-dynamic-scene-representation-and-rendering-with-4d-gaussian-splatting, fetched 2026-10-30 (unofficial)
[S7] 3D Student Splatting and Scooping, https://doi.org/10.1109/cvpr52734.2025.01960, fetched 2026-10-29
[S8] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
[S9] X-GRM: Large Gaussian Reconstruction Model for Sparse-view X-rays to Computed Tomography, https://doi.org/10.48550/arXiv.2505.15235, fetched 2026-10-27 (unofficial)
[S10] GVKF: Gaussian Voxel Kernel Functions for Highly Efficient Surface Reconstruction in Open Scenes, https://doi.org/10.52202/079017-3328, fetched 2026-10-26
