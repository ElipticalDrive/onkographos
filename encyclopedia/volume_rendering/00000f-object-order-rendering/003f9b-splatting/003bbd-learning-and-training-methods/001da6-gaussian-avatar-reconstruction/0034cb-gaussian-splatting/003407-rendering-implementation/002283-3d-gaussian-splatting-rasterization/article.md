# 3D Gaussian Splatting Rasterization

Also called: 3DGS Rasterization, Differentiable Gaussian Rasterization, TrimGS, gsplat.rasterization

## Overview
3D Gaussian Splatting Rasterization is a differentiable rendering technique that represents a scene as a set of 3D Gaussians and rasterizes them to 2D images by projecting 3D covariances to 2D screen covariances, sorting by depth, and compositing colors based on alpha and transmittance [S2, S3, unofficial]. It solves the problem of real-time, photorealistic rendering of learned radiance fields by replacing triangle rasterization with Gaussian splatting, enabling gradient-based optimization of Gaussian parameters (position, covariance, opacity, color) and camera poses [S4, S5, unofficial]. It differs from its siblings by providing the specific differentiable forward and backward passes required for training; for instance, [Optimal Gaussian Splatting](3293-optimal-gaussian-splatting.md) focuses on sampling efficiency, while [Hybrid Transparency](3294-hybrid-transparency.md) modifies the blending model, and [Perspective-Correct 3D Gaussian Splatting via Plücker Coordinates](3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md) alters the projection geometry.

## How it works
The method projects a set of 3D Gaussian primitives into a 2D image plane and accumulates color via alpha compositing [S4, S5, unofficial]. The process assumes the scene is initialized via Structure from Motion (SfM) [S3, unofficial], where points are converted to Gaussians with position, color, and initial covariance [S3, unofficial].

### Inputs and Outputs
**Inputs:**
*   `means_3D`: $[N, 3]$ tensor of 3D Gaussian centers.
*   `quats`: $[N, 4]$ tensor of unit quaternions $(w, x, y, z)$ representing rotation.
*   `scales`: $[N, 3]$ tensor of Gaussian scales.
*   `opacities`: $[N]$ tensor of scalar opacities.
*   `colors`: $[N, D]$ or $[N, K, 3]$ tensor of color coefficients (direct RGB or Spherical Harmonics coefficients).
*   `viewmats`: $[C, 4, 4]$ camera-to-world transformation matrices.
*   `Ks`: $[C, 3, 3]$ camera intrinsic matrices.
*   `width`, `height`: Integer image dimensions.
*   `tile_size`: Integer, default 16. Side length of tiles for rasterization [S1, unofficial].
*   `near_plane`, `far_plane`: Float, default 0.01 and 1e10. Clipping planes [S1, unofficial].
*   `render_mode`: String specifying output channels. Valid values: "RGB", "D", "ED", "RGB+D", "RGB+ED", "d", "Ed", "RGB+d", "RGB+Ed" [S1, unofficial].
*   `rasterize_mode`: String specifying rendering style. Valid values: "classic", "antialiased" [S1, unofficial].
*   `camera_model`: String specifying camera projection. Valid values: "pinhole", "ortho", "fisheye" [S1, unofficial].
*   `sh_degree`: Integer or None. If not None, enables spherical harmonics color computation; otherwise uses direct color mode [S1, unofficial].
*   `backgrounds`: $[C, D]$ tensor of background colors.

**Outputs:**
*   `render_colors`: $[C, H, W, D]$ tensor of rendered colors.
*   `render_alphas`: $[C, H, W, 1]$ tensor of accumulated alpha.

### Data Structures
*   **Covariance**: The 3D covariance matrix $\Sigma_{3D}$ is computed from quaternions and scales [S2, S3, unofficial]. It is a $3 \times 3$ symmetric positive-definite matrix [S3, unofficial].
*   **Conic Representation**: To avoid repeated matrix inversions during rasterization, the inverse 2D covariance $\Sigma_{2D}^{-1}$ is precomputed and stored as a "conic" vector containing the upper triangle elements: $[\Sigma_{2D}^{-1}[0,0], \Sigma_{2D}^{-1}[0,1], \Sigma_{2D}^{-1}[1,1]]$ [S1, unofficial].
*   **Bounding Box**: The spatial extent is determined by the 3-sigma rule. The bounding box size in pixels is derived from the diagonal elements of $\Sigma_{2D}$:
    $$ \text{bboxsize\_cam} = [3.0 \sqrt{\Sigma_{2D}[0,0]}, 3.0 \sqrt{\Sigma_{2D}[1,1]}] $$
    This is converted to screen coordinates to define the rasterization region [S1, unofficial].

### Steps

1.  **Input Validation**: Check `render_mode`, `rasterize_mode`, and `camera_model` against valid sets [S1, unofficial].
2.  **3D Covariance Computation**: Compute 3D covariance matrices from quaternions and scales using `quat_scale_to_covar_preci` with `compute_covar=True`, `compute_preci=False`, `triu=False`, resulting in shape $[N, 3, 3]$ [S1, S2, unofficial].
3.  **2D Projection**: Linearize projection of 3D means and 3D covariance to 2D screen space using camera intrinsics/extrinsics and image size [S2, unofficial]. This involves:
    *   Transforming 3D means to camera space.
    *   Projecting using the Jacobian $J$ of the projective transformation $\phi(\mathbf{x}') = \mathbf{x}' (\mathbf{x}_0^\top \mathbf{x}')^{-1}$ evaluated at the camera-space mean $\mu'$ [Parent].
    *   Computing $\Sigma_{2D} = J \Sigma' J^T$ [Parent].
    *   Computing radii, 2D means, depths, and conics [S1, unofficial].
4.  **Spherical Harmonics Evaluation** (if `sh_degree` is not None):
    *   Compute view directions: $\text{dirs} = \text{means}[None, :, :] - \text{campos}[:, None, :]$ [S1, unofficial].
    *   Normalize directions: $\text{dirs} = \text{dirs} / \max(\|\text{dirs}\|, 1e-8)$ [S1, unofficial].
    *   Broadcast color coefficients to $[C, N, K, 3]$ [S1, unofficial].
    *   Evaluate spherical harmonics to get RGB $[C, N, 3]$ and clamp with $\max(\text{rgb} + 0.5, 0.0)$ [S1, unofficial].
5.  **Opacity and Compensation**:
    *   Broadcast scalar per-Gaussian opacities $[N]$ to per-camera $[C, N]$ [S1, unofficial].
    *   If `rasterize_mode` is "antialiased", multiply opacities by compensations [S1, unofficial].
6.  **Tile Intersection**:
    *   Compute tile dimensions: $\text{tile\_width} = \lceil \text{width} / \text{tile\_size} \rceil$, $\text{tile\_height} = \lceil \text{height} / \text{tile\_size} \rceil$ [S1, unofficial].
    *   Call `isect_tiles` with `means2d`, `radii`, `depths`, `tile_size`, `tile_width`, `tile_height`, `sort=True` to get `tiles_per_gauss`, `isect_ids`, `flatten_ids` [S1, unofficial].
7.  **Sorting**:
    *   Filter Gaussians behind the camera ($z \ge 0$) [S2, unofficial].
    *   Sort remaining Gaussians by depth using a device radix sort (e.g., CUB) [S2, S3, unofficial].
8.  **Rasterization to Pixels**:
    *   Encode intersection offsets using `isect_offset_encode` [S1, unofficial].
    *   Call `rasterize_to_pixels` (or `rasterize_to_pixels_mlx` for differentiable mode) with `means2d`, `conics`, `colors_for_raster`, `opacities_cn`, `width`, `height`, `tile_size`, `isect_offsets`, `flatten_ids`, and `backgrounds` [S1, unofficial].
    *   Inside the kernel, for each pixel in a tile:
        *   Iterate over Gaussians in the sorted list.
        *   Compute alpha: $\alpha_i = \text{opacity}_i \cdot G_{2D}(\mathbf{x}_i, \Sigma_{2D,i})$ [Parent, S2].
        *   Compute transmittance: $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ [Parent].
        *   Compute weight: $w_i = \alpha_i T_i$ [Parent].
        *   Accumulate color: $\text{color} += w_i \cdot \text{color}_i$ [S2, unofficial].
        *   Accumulate alpha: $\text{alpha} += w_i$ [S2, unofficial].
        *   Break if $T_i < \epsilon$ or all Gaussians processed.
9.  **Post-Processing**:
    *   Composite background color where mask is zero: $\text{image} = \text{mask} \cdot \text{image} + (1.0 - \text{mask}) \cdot \text{bg\_colour}$ [S2, unofficial].
    *   If depth is requested, normalize accumulated depth by alpha: $\text{depth\_channel} = \text{depth\_channel} / \max(\text{render\_alphas}, 1e-10)$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import mlx.core as mx
import numpy as np

def rasterize_gaussians(
    means_3d: mx.array,       # [N, 3]
    quats: mx.array,          # [N, 4]
    scales: mx.array,         # [N, 3]
    opacities: mx.array,      # [N]
    colors: mx.array,         # [N, D] or [N, K, 3]
    viewmats: mx.array,       # [C, 4, 4]
    Ks: mx.array,             # [C, 3, 3]
    width: int,
    height: int,
    tile_size: int = 16,
    sh_degree: int = None,
    backgrounds: mx.array = None,
    render_mode: str = "RGB"
) -> tuple:
    """
    Rasterize 3D Gaussians to 2D images.
    Returns (render_colors, render_alphas).
    """
    C = viewmats.shape[0]
    N = means_3d.shape[0]
    
    # 1. Compute 3D Covariance
    # cov_3d shape: [N, 3, 3]
    cov_3d = quat_scale_to_covar(quats, scales)
    
    # 2. Project to 2D
    # means_2d: [C, N, 2], cov_2d: [C, N, 2, 2], depths: [C, N], conics: [C, N, 3]
    means_2d, cov_2d, depths, conics = linearized_project(
        means_3d, cov_3d, viewmats, Ks, width, height
    )
    
    # 3. Compute Radii and Bounding Boxes
    # radii: [C, N, 2]
    radii = compute_radii(cov_2d)
    
    # 4. Spherical Harmonics (if applicable)
    if sh_degree is not None:
        campos = viewmats[:, :3, 3] # [C, 3]
        dirs = means_3d[None, :, :] - campos[:, None, :] # [C, N, 3]
        dirs_norm = mx.maximum(mx.linalg.norm(dirs, axis=-1, keepdims=True), 1e-8)
        dirs = dirs / dirs_norm
        # Broadcast colors to [C, N, K, 3]
        colors_broadcast = mx.broadcast_to(colors[None, :, :, :], (C, N, colors.shape[0], 3))
        # Evaluate SH
        rgb = spherical_harmonics(sh_degree, dirs, colors_broadcast)
        rgb = mx.maximum(rgb + 0.5, 0.0)
        colors_for_raster = rgb
    else:
        # Broadcast colors [N, D] to [C, N, D]
        colors_for_raster = mx.broadcast_to(colors[None, :, :], (C, N, colors.shape[1]))
        
    # 5. Opacity and Compensation
    opacities_cn = mx.broadcast_to(mx.expand_dims(opacities, axis=0), (C, N))
    # If antialiased, multiply by compensations (not shown in detail here)
    
    # 6. Tile Intersection
    tile_width = mx.ceil(width / tile_size).astype(mx.int32)
    tile_height = mx.ceil(height / tile_size).astype(mx.int32)
    tiles_per_gauss, isect_ids, flatten_ids = isect_tiles(
        means_2d, radii, depths, tile_size, tile_width, tile_height, sort=True
    )
    
    # 7. Encode Offsets
    isect_offsets = isect_offset_encode(isect_ids, C, tile_width, tile_height)
    
    # 8. Rasterize
    # Backgrounds default to black if not provided
    if backgrounds is None:
        backgrounds = mx.zeros((C, colors_for_raster.shape[-1]))
        
    render_colors, render_alphas = rasterize_to_pixels_mlx(
        means_2d=means_2d,
        conics=conics,
        colors_for_raster=colors_for_raster,
        opacities_cn=opacities_cn,
        width=width,
        height=height,
        tile_size=tile_size,
        isect_offsets=isect_offsets,
        flatten_ids=flatten_ids,
        backgrounds=backgrounds
    )
    
    return render_colors, render_alphas

def quat_scale_to_covar(quats: mx.array, scales: mx.array) -> mx.array:
    """
    Compute 3D covariance from quaternion and scale.
    quats: [N, 4] (w, x, y, z)
    scales: [N, 3]
    Returns: [N, 3, 3]
    """
    # Construct rotation matrix from quaternion
    R = quat_to_rot_matrix(quats) # [N, 3, 3]
    # Construct scale matrix
    S = mx.diag_embed(scales) # [N, 3, 3]
    # Covariance = R * S * S^T * R^T
    # Since S is diagonal, S * S^T is diag(s^2)
    cov = R @ (S @ S.T) @ R.T
    return cov

def linearized_project(means_3d: mx.array, cov_3d: mx.array, viewmats: mx.array, Ks: mx.array, width: int, height: int) -> tuple:
    """
    Project 3D Gaussians to 2D.
    """
    # Transform means to camera space
    # means_c = R * means + t
    R = viewmats[:, :3, :3] # [C, 3, 3]
    t = viewmats[:, :3, 3]  # [C, 3]
    means_c = mx.einsum("cij,nj->cni", R, means_3d) + t[:, None, :] # [C, N, 3]
    
    # Transform covariance to camera space
    cov_c = mx.einsum("cij,njk,clk->cnil", R, cov_3d, R) # [C, N, 3, 3]
    
    # Project to 2D using Jacobian J
    # J is derived from K and means_c
    # This step computes Sigma_2D and means_2D
    # For implementation, see Parent article for J calculation
    # Here we assume a helper function exists
    means_2d, cov_2d = project_covariance(means_c, cov_c, Ks)
    
    # Compute conics (inverse of 2D covariance upper triangle)
    # Inverse of 2x2 matrix [a b; b c] is 1/(ac-b^2) * [c -b; -b a]
    # Conic = [a_inv, b_inv, c_inv]
    conics = compute_conics(cov_2d)
    
    # Compute depths (z-coordinate in camera space)
    depths = means_c[:, :, 2]
    
    return means_2d, cov_2d, depths, conics

def compute_radii(cov_2d: mx.array) -> mx.array:
    """
    Compute bounding box radii using 3-sigma rule.
    cov_2d: [C, N, 2, 2]
    Returns: [C, N, 2]
    """
    diag = mx.diagonal(cov_2d, offset1=0, offset2=0) # [C, N, 2]
    radii = 3.0 * mx.sqrt(diag)
    return radii

def rasterize_to_pixels_mlx(
    means_2d: mx.array,
    conics: mx.array,
    colors_for_raster: mx.array,
    opacities_cn: mx.array,
    width: int,
    height: int,
    tile_size: int,
    isect_offsets: mx.array,
    flatten_ids: mx.array,
    backgrounds: mx.array
) -> tuple:
    """
    GPU kernel for rasterization.
    """
    # This function is implemented in C++/CUDA or MLX custom ops
    # Pseudocode for the logic inside the kernel:
    
    # Initialize output buffers
    render_colors = mx.zeros((backgrounds.shape[0], height, width, colors_for_raster.shape[-1]))
    render_alphas = mx.zeros((backgrounds.shape[0], height, width, 1))
    
    # Iterate over tiles
    for tile_y in range(mx.ceil(height / tile_size)):
        for tile_x in range(mx.ceil(width / tile_size)):
            # Get Gaussians intersecting this tile
            # flatten_ids contains the list of Gaussian indices for each tile
            # isect_offsets helps index into flatten_ids
            gaussians_in_tile = get_gaussians_for_tile(tile_x, tile_y, isect_offsets, flatten_ids)
            
            # Sort gaussians by depth (already sorted in isect_tiles if sort=True)
            
            # Iterate over pixels in tile
            for py in range(tile_y * tile_size, min((tile_y + 1) * tile_size, height)):
                for px in range(tile_x * tile_size, min((tile_x + 1) * tile_size, width)):
                    transmittance = 1.0
                    pixel_color = backgrounds[0] # Assuming C=1 for simplicity
                    
                    for g in gaussians_in_tile:
                        # Compute 2D Gaussian value at (px, py)
                        # G_2D = exp(-0.5 * (x^T * Sigma_inv * x))
                        # x = [px - means_2d[g, 0], py - means_2d[g, 1]]
                        # Sigma_inv is encoded in conics[g]
                        
                        dx = px - means_2d[0, g, 0]
                        dy = py - means_2d[0, g, 1]
                        
                        # Conic: [a, b, c] corresponds to inverse matrix [[a, b], [b, c]]
                        a, b, c = conics[0, g]
                        
                        exponent = -0.5 * (a * dx * dx + 2 * b * dx * dy + c * dy * dy)
                        
                        # Clamp exponent to avoid overflow
                        exponent = mx.minimum(exponent, 0.0)
                        
                        gaussian_alpha = mx.exp(exponent)
                        alpha = opacities_cn[0, g] * gaussian_alpha
                        
                        weight = alpha * transmittance
                        pixel_color += weight * colors_for_raster[0, g]
                        transmittance *= (1.0 - alpha)
                        
                        if transmittance < 1e-4:
                            break
                    
                    render_colors[0, py, px] = pixel_color
                    render_alphas[0, py, px] = 1.0 - transmittance
                    
    return render_colors, render_alphas
```

### Parameters
*   `tile_size`: Integer, default 16 [S1, unofficial].
*   `near_plane`: Float, default 0.01 [S1, unofficial].
*   `far_plane`: Float, default 1e10 [S1, unofficial].
*   `sh_degree`: Integer or None [S1, unofficial].
*   `render_mode`: String, e.g., "RGB", "D", "ED" [S1, unofficial].
*   `rasterize_mode`: String, "classic" or "antialiased" [S1, unofficial].
*   `camera_model`: String, "pinhole", "ortho", "fisheye" [S1, unofficial].

### Edge Cases and Pitfalls
*   **Sorting Bottleneck**: The primary bottleneck is sorting millions of Gaussians, which is done efficiently in the original implementation using CUB device radix sort, a highly optimized sort only available in CUDA [S3, unofficial].
*   **Memory Constraints**: High VRAM usage (4GB to view, 12GB to train) and large disk size (1GB+ for a scene) [S3, unofficial].
*   **Precision**: Matrix inversions for conics can be numerically unstable for degenerate ellipsoids; the conic representation avoids repeated inversions [Parent].
*   **Boundary Artifacts**: Standard rasterization may suffer from popping artifacts due to order-dependent alpha blending; variants like StopThePop address this [S4, unofficial].

## Subtopics

### [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md)
The core rendering engine enabling gradient-based optimization.

### [Perspective-Correct 3D Gaussian Splatting via Plücker Coordinates](3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md)
Provides a closed-form, numerically stable solution for perspective-correct splat evaluation without matrix inversion.

### [Analytic-Splatting Forward Shading](3459-analytic-splatting-forward-shading.md)
Computes color contribution by analytically integrating the Gaussian PDF over the pixel area to solve aliasing.

### [Speedy-Splat Rendering Pipeline](3543-speedy-splat-rendering-pipeline.md)
Replaces coarse bounding-box tile intersection with precise Gaussian-tile intersection checks and sorting-based depth ordering.

### [Ghost Ellipsoids for Chunked Training](3559-ghost-ellipsoids-for-chunked-training.md)
Resolves boundary artifacts in chunked volume training by including ellipsoids from neighboring chunks in the loss computation.

### [Ref-Gaussian Deferred Surfel Rendering with Split-Sum IBL](4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md)
A real-time deferred shading method for relighting 3DGS models on mobile hardware.

### [3D Gaussian Splatting Self-Shadowing (T_light)](4487-3d-gaussian-splatting-self-shadowing-t-light.md)
Calculates self-shadowing transmittance for each Gaussian along a directional sun direction.

### [Light Space Rasterization for T_light](4489-light-space-rasterization-for-t-light.md)
Computes front-facing occlusion transmittance for each 3D Gaussian along a directional sun direction.

### [2D Gaussian Splatting Rasterization](4815-2d-gaussian-splatting-rasterization.md)
Renders 3D Gaussian surfels as planar disks, enabling normal map accumulation.

### [t-distribution truncation for splatting](5811-t-distribution-truncation-for-splatting.md)
Modifies fragment culling logic to enable opacity-independent truncation at fewer standard deviations.

### [Hardware-Accelerated Gaussian Rasterization](7329-hardware-accelerated-gaussian-rasterization.md)
Augments standard triangle rasterizers with dedicated units for Gaussian projection and blending.

### [AdR-Gaussian](8365-adr-gaussian.md)
Optimizes the rasterization pipeline by moving serial culling operations to a parallel preprocessing stage.

### [3D Half-Gaussian Splatting](9011-3d-half-gaussian-splatting.md)
Replaces the standard isotropic Gaussian with a half-space truncated Gaussian to resolve shape discontinuities.

### [QuadBox](9231-quadbox.md)
Uses four AABBs to tightly encapsulate projected 2D Gaussians, reducing tile over-approximation.

### [Transparency and Blending Models](9274-transparency-and-blending-models.md)
Defines alternatives to standard alpha blending, such as Order-Independent Transparency.

### [Projection Geometry and Coordinate Systems](9313-projection-geometry-and-coordinate-systems.md)
Defines coordinate transformations and projection mappings for various camera models.

### Temporal Marginal Probability Filtering
Pre-rasterization culling step that removes 4D Gaussians with negligible temporal support at the current frame timestamp, stabilizing rendering speed across video length [S5, unofficial]. It computes the marginal probability $p_i(t)$ for each 4D Gaussian $i$ at the current frame's timestamp $t$ and discards Gaussians with negligible $p_i(t)$ before executing standard frustum culling [S5, unofficial]. Unlike methods where Gaussian count scales with video length, this filtering keeps active Gaussians constant, making the approach friendly to long videos in terms of rendering speed [S5, unofficial].

### Gaussian Opacity Fields (GOF)
Uses ray-tracing-style intersection for accurate depth and derives a level-set surface from accumulated opacity [S4, unofficial].

### Progressive Normal Propagation
Uses progressive propagation along normals to fill texture-less regions [S4, unofficial].

### Gaussian Surfels
Uses zero-thickness oriented surfels [S4, unofficial].

## Operating characteristics
*   **Time Complexity**: Dominated by sorting $N$ Gaussians, typically $O(N \log N)$ or $O(N)$ with radix sort [S3, unofficial]. Rasterization is $O(P \cdot G_{tile})$ where $P$ is pixels and $G_{tile}$ is Gaussians per tile.
*   **Memory Complexity**: $O(N)$ for Gaussian parameters and $O(P)$ for output buffers [S3, unofficial]. High VRAM usage (4GB to view, 12GB to train) [S3, unofficial].
*   **Measured Performance**: Real-time rendering (60+ FPS) on modern GPUs for scenes with millions of Gaussians [S3, unofficial].
*   **Strengths**: Photorealistic quality, real-time performance, differentiable [S3, unofficial].
*   **Limitations**: High VRAM usage, large disk size, static scenes (without extensions), sorting bottleneck on non-CUDA hardware [S3, unofficial].
*   **When to Use**: Real-time rendering of learned radiance fields, VR/AR applications, neural rendering training.
*   **When Not to Use**: When strict triangle-based pipeline compatibility is required, or when memory is severely constrained.

## Background
3D Gaussian Splatting was introduced in the paper "3D Gaussian Splatting for Real-Time Radiance Field Rendering" [S3, unofficial]. It allows real-time rendering of photorealistic scenes learned from small samples of images [S3, unofficial]. The method is analogous to triangle rasterization but uses Gaussians instead of triangles [S3, unofficial]. The primary bottleneck is sorting millions of Gaussians, which is done efficiently in the original implementation using CUB device radix sort, a highly optimized sort only available in CUDA [S3, unofficial].

## Sources
[S1] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
[S2] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S3] Introduction to 3D Gaussian Splatting - Hugging Face, https://huggingface.co/blog/gaussian-splatting, fetched 2026-10-27 (unofficial)
[S4] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S5] “Real-time Photorealistic Dynamic Scene Representation and Rendering with 4D Gaussian Splatting”, https://radiancefields.com/papers/real-time-photorealistic-dynamic-scene-representation-and-rendering-with-4d-gaussian-splatting, fetched 2026-10-30 (unofficial)
