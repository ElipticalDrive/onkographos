# 3D Gaussian Splatting Rasterization

Also called: 3D Gaussian Splatting (3DGS), 3DGS Rasterization, Differentiable Gaussian Rasterization, TrimGS, gsplat.rasterization

## Overview
3D Gaussian Splatting Rasterization is a differentiable rendering technique that represents a scene as a collection of anisotropic 3D Gaussians and projects them onto a 2D image plane for real-time, photorealistic rendering [S1][S3, S4, unofficial]. It solves the problem of efficiently rendering continuous volumetric representations by converting each primitive into a 2D splat, sorting them by depth, and accumulating color via alpha compositing [S3, S4, unofficial]. Unlike traditional triangle rasterization or quad-based approximations, this method directly rasterizes Gaussian kernels, enabling high-fidelity results and gradient flow to Gaussian parameters (position, scale, rotation, opacity, color) for training [S4, S5, unofficial]. It builds on [Rendering Implementation](7155-rendering-implementation.md) by implementing the specific projection, sorting, and compositing steps for Gaussian primitives.

## How it works
**Inputs:**
- $N$ 3D Gaussians, each defined by mean position $\mu \in \mathbb{R}^3$, scale $s \in \mathbb{R}^3$, rotation $q \in \mathbb{R}^4$ (quaternion), opacity $\alpha \in \mathbb{R}$, and spherical harmonics (SH) coefficients $c_{\ell m}$ or direct RGB color [S1][S2, S4, unofficial].
- Camera parameters: view matrix $V \in \mathbb{R}^{4 \times 4}$, intrinsic matrix $K \in \mathbb{R}^{3 \times 3}$, image dimensions $H \times W$, near/far planes [S2, unofficial].
- Configuration: `render_mode` (e.g., "RGB", "D", "ED"), `rasterize_mode` ("classic", "antialiased"), `camera_model` ("pinhole", "ortho", "fisheye"), `sh_degree`, `tile_size` (default 16), `near_plane` (default 0.01), `far_plane` (default $10^{10}$) [S2, unofficial].

**Outputs:**
- Rendered image tensor of shape $[C, H, W, D]$ (where $C$ is number of cameras, $D$ depends on mode) and alpha mask $[C, H, W, 1]$ [S2, unofficial].

**Data Structures:**
- **Gaussian Attributes:** Stored in contiguous arrays or tensors.
  - `means`: $[N, 3]$ float32.
  - `quats`: $[N, 4]$ float32 (normalized).
  - `scales`: $[N, 3]$ float32.
  - `opacities`: $[N]$ float32.
  - `colors`: $[N, K, 3]$ float32 for SH, or $[N, 3]$ for direct color, where $K = (sh\_degree + 1)^2$ [S2, unofficial].
- **2D Projection Attributes (per Gaussian per Camera):**
  - `means2d`: $[C, N, 2]$ float32, screen-space center.
  - `conics`: $[C, N, 3]$ float32, storing the upper triangle of the inverse 2D covariance $\Sigma_{2D}^{-1}$ as $[C_{00}, C_{01}, C_{11}]$ [S2, S3, unofficial].
  - `radii`: $[C, N, 2]$ int32, bounding box in pixels.
  - `depths`: $[C, N]$ float32, camera-space depth.
- **Tile Intersection Data:**
  - `isect_offsets`: $[C, \lceil H/T \rceil, \lceil W/T \rceil]$ int32, encoding the start index of Gaussians intersecting each tile [S2, unofficial].
  - `flatten_ids`: $[M]$ int32, flattened list of Gaussian indices sorted by tile and depth, where $M$ is the total number of tile-Gaussian intersections [S2, unofficial].

**Steps:**

1. **Covariance Construction:** Compute the 3D covariance matrix $\Sigma_{3D}$ from scale $S = \text{diag}(s_x, s_y, s_z)$ and rotation $R$ derived from quaternion $q$:
   $$ \Sigma_{3D} = R S S^T R^T $$
   [S1][S3, unofficial].

2. **Projection:** Transform means to camera space and project to 2D screen space using the local affine approximation of the projective transformation. Compute the 2D covariance $\Sigma_{2D}$ using the Jacobian $J$ of the projection function:
   $$ \Sigma_{2D} = J \Sigma'_{3D} J^T $$
   where $\Sigma'_{3D}$ is the upper-left $3 \times 3$ block of the transformed 3D covariance [S3, unofficial]. Compute the conic vector $C = [\Sigma_{2D}^{-1}[0,0], \Sigma_{2D}^{-1}[0,1], \Sigma_{2D}^{-1}[1,1]]$ for efficient evaluation [S2, S3, unofficial].

3. **Filtering and Sorting:** Filter Gaussians behind the camera ($z < 0$ or outside near/far planes) [S3, unofficial]. Sort the remaining Gaussians by camera-space depth to ensure correct front-to-back alpha compositing order [S2, S3, S4, unofficial]. The primary bottleneck is sorting millions of Gaussians, typically handled by CUB device radix sort in CUDA implementations [S4, unofficial].

4. **Tile Intersection:** Divide the image into tiles of size $T \times T$ (default 16). For each Gaussian, compute its bounding box in screen space. Identify which tiles each Gaussian intersects. Encode these intersections into `isect_offsets` and `flatten_ids` to allow efficient parallel access during rasterization [S2, unofficial].

5. **Color Computation:** If `sh_degree` is not None, compute view directions $d = \mu - \text{campos}$, normalize them, and evaluate spherical harmonics to get RGB color. Otherwise, use direct color [S2, unofficial]. Broadcast opacities to $[C, N]$ and apply compensation factors if `rasterize_mode` is "antialiased" [S2, unofficial].

6. **Rasterization (Compositing):** For each pixel, iterate over the sorted Gaussians that cover it. Compute the alpha value $\alpha_i$ for each Gaussian at the pixel using the 2D Gaussian PDF and the conic representation. Compute transmittance $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ and weight $w_i = \alpha_i T_i$. Accumulate color:
   $$ C_{pixel} = \sum_{i} w_i C_i + \left(1 - \sum_{i} w_i\right) C_{bg} $$
   [S2, S3, unofficial]. Handle memory constraints by splitting Gaussians into mini-batches if necessary, passing `start_transmittance` between batches [S3, unofficial].

7. **Post-processing:** If depth or expected depth is requested, compute weighted sums of z-values or hit distances. Normalize expected depth by accumulated alpha. Composite background color where alpha is zero [S2, S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def rasterize_gaussians(
    means_3d: np.ndarray,      # [N, 3]
    quats: np.ndarray,         # [N, 4]
    scales: np.ndarray,        # [N, 3]
    opacities: np.ndarray,     # [N]
    colors: np.ndarray,        # [N, 3] or [N, K, 3]
    viewmats: np.ndarray,      # [C, 4, 4]
    Ks: np.ndarray,            # [C, 3, 3]
    width: int,
    height: int,
    tile_size: int = 16,
    near_plane: float = 0.01,
    far_plane: float = 1e10,
    sh_degree: int = None
) -> tuple:
    """
    Rasterize 3D Gaussians to 2D image.
    Returns: (render_colors [C, H, W, 3], render_alphas [C, H, W, 1])
    """
    C = viewmats.shape[0]
    N = means_3d.shape[0]
    
    # 1. Covariance Construction
    # Compute rotation matrices from quaternions
    Rs = quat_to_rotmat(quats) # [N, 3, 3]
    Ss = np.diag_embed(scales) # [N, 3, 3]
    covs_3d = Rs @ Ss @ Ss.transpose(0, 2, 1) # [N, 3, 3]
    
    # 2. Projection
    # Transform means to camera space
    # means_c = R_cam @ means_3d + t_cam
    R_cam = viewmats[:, :3, :3] # [C, 3, 3]
    t_cam = viewmats[:, :3, 3]  # [C, 3]
    means_c = np.einsum('cij,nj->cni', R_cam, means_3d) + t_cam[:, None, :] # [C, N, 3]
    
    # Project to 2D
    # z_vals = means_c[:, :, 2]
    # x_ndc = (K_cam @ means_c[:, :, :3].T).T / z_vals
    # ... (simplified linearized projection)
    means_2d, covs_2d, depths, radii, conics = linearized_project(
        means_c, covs_3d, Ks, width, height, near_plane, far_plane
    )
    
    # 3. Filtering and Sorting
    # Filter behind camera
    valid_mask = depths > 0
    # Sort by depth
    # idxs = argsort(depths[valid_mask])
    # ... (reorder means_2d, conics, opacities, colors, depths)
    
    # 4. Tile Intersection
    tile_width = math.ceil(width / tile_size)
    tile_height = math.ceil(height / tile_size)
    isect_offsets, flatten_ids = isect_tiles(
        means_2d, radii, depths, tile_size, tile_width, tile_height
    )
    
    # 5. Color Computation
    if sh_degree is not None:
        dirs = means_3d[None, :, :] - viewmats[:, :3, 3][:, None, :]
        dirs = dirs / np.maximum(np.linalg.norm(dirs, axis=-1, keepdims=True), 1e-8)
        colors = spherical_harmonics(sh_degree, dirs, sh_coeffs)
    colors = np.clip(colors + 0.5, 0.0, 1.0) # Clamp
    
    # Broadcast opacities
    opacities_cn = np.broadcast_to(opacities[None, :], (C, N))
    
    # 6. Rasterization
    render_colors = np.zeros((C, height, width, 3))
    render_alphas = np.zeros((C, height, width, 1))
    
    # Parallel loop over tiles/pixels would happen here in GPU kernel
    # Pseudocode for single pixel accumulation:
    for c in range(C):
        for y in range(height):
            for x in range(width):
                # Get Gaussians covering this pixel from flatten_ids/isect_offsets
                gaussians_for_pixel = get_gaussians_for_pixel(x, y, isect_offsets, flatten_ids, c)
                
                transmittance = 1.0
                accumulated_color = np.zeros(3)
                accumulated_alpha = 0.0
                
                for g in gaussians_for_pixel:
                    # Compute alpha at pixel (x,y) using conic
                    alpha = compute_alpha_pixel(x, y, means_2d[c, g], conics[c, g])
                    alpha *= opacities_cn[c, g]
                    
                    weight = alpha * transmittance
                    accumulated_color += weight * colors[c, g]
                    accumulated_alpha += weight
                    transmittance *= (1.0 - alpha)
                    
                    if transmittance < 1e-4:
                        break
                
                render_colors[c, y, x] = accumulated_color + transmittance * background_color
                render_alphas[c, y, x] = accumulated_alpha
                
    return render_colors, render_alphas

def quat_to_rotmat(quats):
    # Standard quaternion to rotation matrix conversion
    # Returns [N, 3, 3]
    pass

def linearized_project(means_c, covs_3d, Ks, width, height, near, far):
    # Compute Jacobian J of projection
    # Project means and covs
    # Return means_2d, covs_2d, depths, radii, conics
    pass

def compute_alpha_pixel(x, y, mean_2d, conic):
    # Evaluate 2D Gaussian PDF at (x,y)
    # conic = [C00, C01, C11]
    dx = x - mean_2d[0]
    dy = y - mean_2d[1]
    exponent = -0.5 * (conic[0]*dx*dx + 2*conic[1]*dx*dy + conic[2]*dy*dy)
    return np.exp(exponent)
```

## Subtopics

### [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md)
Provides the core differentiable forward and backward passes for 3D Gaussians, often implemented as CUDA kernels. Choose when training is required.

### [Perspective-Correct 3D Gaussian Splatting via Plücker Coordinates](3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md)
Uses Plücker coordinates for numerically stable, closed-form perspective-correct splat evaluation without matrix inversion. Choose when handling degenerate ellipsoids or avoiding inversion costs.

### [Speedy-Splat Rendering Pipeline](3543-speedy-splat-rendering-pipeline.md)
Replaces coarse bounding-box tile intersection with precise Gaussian-tile intersection checks and sorting-based depth ordering. Choose for improved efficiency over standard rasterization.

### [Ghost Ellipsoids for Chunked Training](3559-ghost-ellipsoids-for-chunked-training.md)
Includes ellipsoids from neighboring chunks in loss computation without updating parameters to resolve boundary artifacts. Choose for chunked volume training.

### [2D Gaussian Splatting Rasterization](4815-2d-gaussian-splatting-rasterization.md)
Renders planar surfels with explicit normals, enabling normal map accumulation. Choose when surface normals are needed.

### [Hardware-Accelerated Gaussian Rasterization](7329-hardware-accelerated-gaussian-rasterization.md)
Augments triangle rasterizers with dedicated units for Gaussian projection and blending. Choose for order-of-magnitude speedup on specialized hardware.

### [AdR-Gaussian](8365-adr-gaussian.md)
Optimizes rasterization by moving serial culling to parallel preprocessing. Choose for significant speedups without quality degradation.

### [QuadBox](9231-quadbox.md)
Uses four AABBs to tightly encapsulate projected 2D Gaussians, reducing over-approximation. Choose for tighter culling than single AABB.

### [Projection Geometry and Coordinate Systems](9313-projection-geometry-and-coordinate-systems.md)
Defines coordinate transformations for various camera models (panoramic, equidistant). Choose for non-standard camera projections.

### [Point-Based Ray Tracing for Visibility](10934-point-based-ray-tracing-for-visibility.md)
Sparse visibility oracle using BVH of Gaussian proxies. Choose for visibility estimation decoupled from screen-space projection.

### [Primitive Representation](11838-primitive-representation.md)
Defines geometric and photometric data structures for Gaussian primitives. Choose for understanding kernel shapes and attribute storage.

### [Shading and Lighting Models](11924-shading-and-lighting-models.md)
Defines radiance computation and compositing for color, transparency, and illumination. Choose for advanced shading beyond baked color.

### Temporal Marginal Probability Filtering
Pre-rasterization culling step that removes 4D Gaussians with negligible temporal support at the current frame timestamp, stabilizing rendering speed across video length [S6, unofficial]. It computes the marginal probability $p_i(t)$ for each 4D Gaussian $i$ at the current frame's timestamp $t$ and discards Gaussians with negligible $p_i(t)$ before frustum culling [S6, unofficial]. Unlike methods where Gaussian count scales with video length, this filtering keeps active Gaussians constant, making the approach friendly to long videos in terms of rendering speed [S6, unofficial].

### t-distribution truncation for splatting
A projection truncation technique that replaces the fixed three-sigma rule used in vanilla 3DGS with a $\nu$-dependent critical value interpolation to handle fat-tailed Student's t-distributions [S7]. For t-distributions with thin tails (high $\nu$), it applies the standard three-sigma rule directly. For fat-tailed distributions (low $\nu$), it obtains critical values for different $\nu$ degrees from reference [9] and interpolates between them to compute the appropriate truncation threshold [S7]. Using a fixed three-sigma threshold on fat-tailed t-distributions causes excessive truncation or leakage, which interpolation resolves [S7].

### Gaussian Opacity Fields (GOF)
Uses ray-tracing-style intersection for accurate depth and derives a level-set surface from accumulated opacity [S5, unofficial].

### Progressive Normal Propagation
Uses progressive propagation along normals to fill texture-less regions [S5, unofficial].

### Splat TBO Upload & Layout
Packs per-splat data (center, covariance, DC color/alpha, optional SH) into Texture Buffer Objects (TBOs) for GPU consumption [S8, unofficial]. It iterates over splats, copying `center.xyz` as raw float bits to `d[0..2]`, packing covariance upper-triangle to `d[3..5]` using `packHalf2x16`, packing DC color/alpha to `d[6]` using `packDCColor`, and setting `d[7]=0`. It creates a `GL_TEXTURE_BUFFER` with `GL_RGBA32UI` format. For non-DC SH coefficients, it packs each coefficient's RGB into `RGBA16F` texels (B=0), with layout `[splat0_c1, splat0_c2, …, splat1_c1, splat1_c2, …]` in a `GL_RGBA16F` SH TBO [S8, unofficial]. The memory complexity is 32 B/splat for base TBO + 8 B $\times$ `sh_coeffs_per_splat_` $\times$ `splat_count_` for SH TBO, where `sh_coeffs_per_splat_` = $(active\_sh\_degree\_ + 1)^2 - 1$ [S8, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by sorting $N$ Gaussians by depth, typically $O(N \log N)$ or $O(N)$ with radix sort [S4, unofficial]. Rasterization is $O(P \cdot K)$ where $P$ is pixels and $K$ is average Gaussians per pixel.
- **Memory Complexity:** $O(N)$ for Gaussian attributes. High VRAM usage: 4GB to view, 12GB to train [S4, unofficial]. Large disk size: 1GB+ for a scene [S4, unofficial].
- **Measured Performance:** Real-time rendering rates (30+ FPS) on modern GPUs for scenes with millions of Gaussians [S1][S4, unofficial].
- **Strengths:** High visual fidelity, real-time rendering, differentiable for training [S4, unofficial].
- **Limitations:** High VRAM and disk usage, static scenes (without extensions), sorting bottleneck in non-CUDA environments [S4, unofficial].
- **When to use:** Real-time rendering of high-quality static or dynamic scenes from learned representations.
- **When not to use:** When memory is severely constrained, or when integration with existing triangle-based pipelines is required without adaptation [S4, unofficial].

## Background
3D Gaussian Splatting was introduced in "3D Gaussian Splatting for Real-Time Radiance Field Rendering" (Kerbl et al., SIGGRAPH 2023) [S4, unofficial]. It builds on the idea of representing scenes as collections of 3D Gaussians, enabling real-time rendering through efficient rasterization [S1]. The method uses Structure from Motion (SfM) to initialize Gaussians from point clouds, followed by differentiable rasterization and Stochastic Gradient Descent (SGD) to optimize Gaussian parameters [S4, unofficial]. It has inspired numerous variants for dynamic scenes (4DGS, 6DGS), surface alignment (2DGS, SuGaR), and acceleration techniques [S1][S5, unofficial].

## Sources
[S1] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
[S2] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
[S3] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S4] Introduction to 3D Gaussian Splatting - Hugging Face, https://huggingface.co/blog/gaussian-splatting, fetched 2026-10-27 (unofficial)
[S5] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S6] “Real-time Photorealistic Dynamic Scene Representation and Rendering with 4D Gaussian Splatting”, https://radiancefields.com/papers/real-time-photorealistic-dynamic-scene-representation-and-rendering-with-4d-gaussian-splatting, fetched 2026-10-30 (unofficial)
[S7] 3D Student Splatting and Scooping, https://doi.org/10.1109/cvpr52734.2025.01960, fetched 2026-10-29
[S8] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
