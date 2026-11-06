# 3D Gaussian Splatting Rasterization

Also called: 3DGS Rasterization, Differentiable Gaussian Rasterization, TrimGS, gsplat.rasterization

## Overview
3D Gaussian Splatting Rasterization is a differentiable rendering technique that represents a scene as a set of 3D Gaussians and rasterizes them to 2D images by projecting 3D covariances to 2D screen covariances, sorting by depth, and compositing colors based on alpha and transmittance [S2, S3, unofficial]. It is analogous to triangle rasterization but uses Gaussians instead of triangles, enabling real-time rendering of photorealistic scenes learned from small samples of images [S3, unofficial]. It differs from its siblings by providing the core differentiable kernel that bridges the 3D scene representation and the 2D image plane, whereas methods like [Optimal Gaussian Splatting](3293-optimal-gaussian-splatting.md) focus on training efficiency and [Hybrid Transparency](3294-hybrid-transparency.md) on specific blending artifacts [S2, unofficial].

## How it works
**Inputs:**
*   `means_3D`: $[N, 3]$ tensor of 3D Gaussian centers.
*   `quats`: $[N, 4]$ tensor of unit quaternions $(w, x, y, z)$ representing rotation.
*   `scales`: $[N, 3]$ tensor of scale factors.
*   `opacities`: $[N]$ tensor of scalar opacity values.
*   `colors`: $[N, D]$ or $[N, K, 3]$ tensor of color coefficients (direct RGB or Spherical Harmonics coefficients).
*   `viewmats`: $[C, 4, 4]$ camera-to-world transformation matrices.
*   `Ks`: $[C, 3, 3]$ camera intrinsic matrices.
*   `width`, `height`: Integer image dimensions.
*   `tile_size`: Integer, default 16.
*   `near_plane`, `far_plane`: Float clipping distances, default 0.01 and $10^{10}$ respectively [S1, unofficial].
*   `render_mode`: String specifying output channels: "RGB", "D", "ED", "RGB+D", "RGB+ED", "d", "Ed", "RGB+d", "RGB+Ed" [S1, unofficial].
*   `rasterize_mode`: String, "classic" or "antialiased" [S1, unofficial].
*   `camera_model`: String, "pinhole", "ortho", or "fisheye" [S1, unofficial].
*   `sh_degree`: Integer or None. If not None, enables spherical harmonics color computation [S1, unofficial].
*   `backgrounds`: $[C, D]$ tensor of background colors.

**Outputs:**
*   `render_colors`: $[C, H, W, D]$ tensor of rendered colors.
*   `render_alphas`: $[C, H, W, 1]$ tensor of accumulated alpha.
*   `info`: Dictionary containing `radii`, `means2D`, `depths`, `conics`, `compensations` [S1, unofficial].

**Data Structures:**
*   **Conic**: A vector of 3 floats containing the upper triangle of the inverse 2D covariance $\Sigma_{2D}^{-1}$: $[\Sigma_{2D}^{-1}[0,0], \Sigma_{2D}^{-1}[0,1], \Sigma_{2D}^{-1}[1,1]]$ [S1, unofficial].
*   **Tile Intersection**: `isect_offsets` $[C, \text{tile\_height}, \text{tile\_width}]$ and `flatten_ids` mapping Gaussians to tiles [S1, unofficial].

**Steps:**

1.  **Input Validation**: Check `render_mode`, `rasterize_mode`, and `camera_model` against valid sets [S1, unofficial].
2.  **3D Covariance Construction**: Compute the 3D covariance matrix $\Sigma_{3D}$ from `quats` and `scales`. This is typically done via `quat_scale_to_covar_preci` with `compute_covar=True`, `compute_preci=False`, `triu=False`, resulting in shape $[N, 3, 3]$ [S1, unofficial].
3.  **Projection to 2D**:
    *   Transform `means_3D` to camera space: $\mu' = R \mu + t$, where $R$ and $t$ are from `viewmats` [S1, unofficial].
    *   Transform $\Sigma_{3D}$ to camera space: $\Sigma' = R \Sigma_{3D} R^T$ [S2, unofficial].
    *   Compute the Jacobian $J$ of the projective projection $\phi(\mathbf{x}') = \mathbf{x}' (\mathbf{x}_0^\top \mathbf{x}')^{-1}$ at $\mu'$ [S2, unofficial].
    *   Compute 2D covariance: $\Sigma_{2D} = J \Sigma' J^T$ [S2, unofficial].
    *   Compute the inverse 2D covariance (Conic) for efficient evaluation: $C = \Sigma_{2D}^{-1}$, stored as $[C[0,0], C[0,1], C[1,1]]$ [S1, unofficial].
    *   Compute 2D means `means2D` in screen coordinates [S1, unofficial].
    *   Compute radii and bounding boxes using the 3-sigma rule: $\text{bboxsize\_cam} = [3.0 \sqrt{\Sigma_{2D}[0,0]}, 3.0 \sqrt{\Sigma_{2D}[1,1]}]$ [S1, unofficial].
    *   If `rasterize_mode` is "antialiased", compute compensation factors [S1, unofficial].
4.  **Tile Intersection**:
    *   Calculate tile dimensions: $\text{tile\_width} = \lceil \text{width} / \text{tile\_size} \rceil$, $\text{tile\_height} = \lceil \text{height} / \text{tile\_size} \rceil$ [S1, unofficial].
    *   Identify which tiles each Gaussian intersects using `isect_tiles`, producing `tiles_per_gauss`, `isect_ids`, and `flatten_ids` [S1, unofficial].
    *   Encode intersections into `isect_offsets` using `isect_offset_encode` [S1, unofficial].
5.  **Color Computation**:
    *   If `sh_degree` is not None:
        *   Compute view directions: $\text{dirs} = \mu' - \text{campos}$, normalized [S1, unofficial].
        *   Broadcast color coefficients to $[C, N, K, 3]$ and evaluate Spherical Harmonics to get RGB colors [S1, unofficial].
        *   Clamp colors: $\max(\text{rgb} + 0.5, 0.0)$ [S1, unofficial].
    *   If `sh_degree` is None, use direct colors [S1, unofficial].
    *   Broadcast opacities from $[N]$ to $[C, N]$ [S1, unofficial].
    *   If antialiased, multiply opacities by compensations [S1, unofficial].
    *   Concatenate depth/hit-distance to colors if `render_mode` requires it (e.g., "RGB+D") [S1, unofficial].
6.  **Sorting**:
    *   Filter Gaussians behind the camera ($z < \text{near\_plane}$) [S2, unofficial].
    *   Sort remaining Gaussians by depth (camera-space Z) for each tile/camera [S1, S2, unofficial]. The original implementation uses CUB device radix sort for efficiency [S3, unofficial].
7.  **Rasterization (Pixel Compositing)**:
    *   For each pixel in each tile:
        *   Initialize transmittance $T = 1.0$ [S2, unofficial].
        *   Iterate through sorted Gaussians intersecting the pixel:
            *   Evaluate 2D Gaussian value $G_{2D}$ at pixel center using the Conic: $G_{2D} = \exp(-0.5 \mathbf{x}^T C \mathbf{x})$ [S2, unofficial].
            *   Compute alpha: $\alpha_i = \text{opacity}_i \cdot G_{2D}$ [S2, unofficial].
            *   Compute weight: $w_i = \alpha_i \cdot T$ [S2, unofficial].
            *   Accumulate color: $\text{color} += w_i \cdot \text{color}_i$ [S2, unofficial].
            *   Update transmittance: $T \leftarrow T \cdot (1 - \alpha_i)$ [S2, unofficial].
            *   Accumulate depth if requested: $\text{depth\_acc} += w_i \cdot \text{depth}_i$ [S1, unofficial].
        *   If depth is requested, normalize by accumulated alpha: $\text{depth} = \text{depth\_acc} / \max(\text{alpha\_acc}, 10^{-10})$ [S1, unofficial].
8.  **Background Compositing**:
    *   Composite background color where mask is zero: $\text{image} = \text{mask} \cdot \text{image} + (1.0 - \text{mask}) \cdot \text{background}$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import mlx.core as mx
import numpy as np

def rasterize_gaussians(
    means_3d: mx.array, # [N, 3]
    quats: mx.array,    # [N, 4]
    scales: mx.array,   # [N, 3]
    opacities: mx.array,# [N]
    colors: mx.array,   # [N, D] or [N, K, 3]
    viewmats: mx.array, # [C, 4, 4]
    Ks: mx.array,       # [C, 3, 3]
    width: int,
    height: int,
    tile_size: int = 16,
    near_plane: float = 0.01,
    far_plane: float = 1e10,
    sh_degree: int = None,
    render_mode: str = "RGB",
    backgrounds: mx.array = None
) -> tuple:
    """
    Rasterize 3D Gaussians to 2D images.
    Returns (render_colors, render_alphas, info_dict)
    """
    C = viewmats.shape[0]
    N = means_3d.shape[0]
    
    # 1. Compute 3D Covariance
    # Assuming a helper function exists for quat->covariance
    cov_3d = quat_scale_to_cov_3d(quats, scales) # [N, 3, 3]
    
    # 2. Project to 2D
    # Extract R and t from viewmats
    R = viewmats[:, :3, :3] # [C, 3, 3]
    t = viewmats[:, :3, 3]  # [C, 3]
    
    # Transform means to camera space
    # means_cam: [C, N, 3]
    means_cam = mx.einsum("cij,nj->cni", R, means_3d) + t[:, None, :]
    
    # Transform covariance to camera space
    # cov_cam: [C, N, 3, 3]
    cov_cam = mx.einsum("cij,njk,clk->cnik", R, cov_3d, R)
    
    # Compute Jacobian J for projection
    # Ks: [C, 3, 3]
    # Projective projection approximation
    # This step computes means_2d, cov_2d, and conics
    # conics are the upper triangle of inv(cov_2d)
    means_2d, conics, radii, depths = linearized_project(
        means_cam, cov_cam, Ks, width, height, near_plane, far_plane
    )
    
    # 3. Tile Intersection
    tile_width = (width + tile_size - 1) // tile_size
    tile_height = (height + tile_size - 1) // tile_size
    
    # Find which tiles each gaussian hits
    # Returns isect_ids, flatten_ids, isect_offsets
    isect_ids, flatten_ids, isect_offsets = isect_tiles(
        means_2d, radii, depths, tile_size, tile_width, tile_height, sort=True
    )
    
    # 4. Color and Opacity Preparation
    # Broadcast opacities to [C, N]
    opacities_cn = mx.broadcast_to(mx.expand_dims(opacities, 0), (C, N))
    
    # Compute colors
    if sh_degree is not None:
        # Compute view directions
        campos = -R[..., None, :] @ t[..., :, None] # [C, 3, 1] -> [C, 3]
        # Simplified direction calculation for Spherical Harmonics
        dirs = means_cam - campos[:, None, :]
        dirs_norm = mx.sqrt(mx.sum(dirs**2, axis=-1, keepdims=True))
        dirs = dirs / mx.maximum(dirs_norm, 1e-8)
        
        # Evaluate SH
        # Assuming coeffs are stored in colors [N, K, 3]
        # Broadcast coeffs to [C, N, K, 3]
        coeffs = mx.broadcast_to(colors[None, :, :, :], (C, N, colors.shape[1], 3))
        rgb = spherical_harmonics(sh_degree, dirs, coeffs) # [C, N, 3]
        rgb = mx.maximum(rgb + 0.5, 0.0)
    else:
        # Direct color
        rgb = mx.broadcast_to(colors[None, :, :], (C, N, colors.shape[-1]))
        
    # Prepare final color tensor for rasterization
    # If render_mode includes depth, concatenate depth
    has_depth = "D" in render_mode or "d" in render_mode
    if has_depth:
        # depths is [C, N]
        colors_for_raster = mx.concatenate([rgb, depths[..., None]], axis=-1)
    else:
        colors_for_raster = rgb
        
    # 5. Rasterize
    # Call fused kernel
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
    
    # 6. Post-processing
    # Normalize depth if requested
    if has_depth:
        depth_channel = render_colors[..., -1:]
        alpha_channel = render_alphas
        normalized_depth = depth_channel / mx.maximum(alpha_channel, 1e-10)
        if "RGB" in render_mode:
            render_colors = mx.concatenate([render_colors[..., :-1], normalized_depth], axis=-1)
        else:
            render_colors = normalized_depth
            
    return render_colors, render_alphas, {
        "radii": radii,
        "means2d": means_2d,
        "depths": depths,
        "conics": conics
    }

def quat_scale_to_cov_3d(quats, scales):
    """Convert quaternion and scale to 3D covariance matrix."""
    # Rotation matrix from quaternion
    # q = [w, x, y, z]
    w, x, y, z = quats[:, 0], quats[:, 1], quats[:, 2], quats[:, 3]
    
    # Construct rotation matrix R
    # R = I + 2 * [v]_x^2 + 2 * [v]_x (for unit quaternion)
    # Or standard conversion
    R = mx.zeros((quats.shape[0], 3, 3))
    R[:, 0, 0] = 1 - 2*(y*y + z*z)
    R[:, 0, 1] = 2*(x*y - z*w)
    R[:, 0, 2] = 2*(x*z + y*w)
    R[:, 1, 0] = 2*(x*y + z*w)
    R[:, 1, 1] = 1 - 2*(x*x + z*z)
    R[:, 1, 2] = 2*(y*z - x*w)
    R[:, 2, 0] = 2*(x*z - y*w)
    R[:, 2, 1] = 2*(y*z + x*w)
    R[:, 2, 2] = 1 - 2*(x*x + y*y)
    
    # Scale matrix S
    S = mx.zeros((quats.shape[0], 3, 3))
    S[:, 0, 0] = scales[:, 0]
    S[:, 1, 1] = scales[:, 1]
    S[:, 2, 2] = scales[:, 2]
    
    # Covariance = R * S * S^T * R^T
    # Sigma = R * diag(s^2) * R^T
    scales_sq = scales ** 2
    Sigma = R * scales_sq[:, None, :] @ R # Broadcasting: [N, 3, 3] * [N, 3, 3]
    # Correct matrix multiplication:
    # Sigma[i] = R[i] @ diag(s[i]^2) @ R[i].T
    Sigma = mx.einsum("nij,nj,nkj->nik", R, scales_sq, R)
    
    return Sigma
```

## Subtopics

### [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md)
The 3D Gaussian Splatting Differentiable Rasterizer projects 3D Gaussian primitives into 2D screen space and accumulates color per pixel using alpha compositing, enabling gradients to flow back to Gaussian parameters (position, scale, rotation, opacity, color). It differs from its siblings by providing the core differentiable kernel that bridges the 3D scene representation and the 2D image plane, whereas methods like [Optimal Gaussian Splatting](3293-optimal-gaussian-splatting.md) focus on training efficiency and [Hybrid Transparency](3294-hybrid-transparency.md) on specific blending artifacts.

### [Optimal Gaussian Splatting](3293-optimal-gaussian-splatting.md)
Optimal Gaussian Splatting is a rendering method that reduces projection error by radially projecting each 3D Gaussian onto a unique tangent plane perpendicular to the line connecting the Gaussian mean to the camera center, rather than projecting all Gaussians onto a single plane $z=1$. Unlike the original 3D Gaussian Splatting rasterization which uses a fixed perspective plane, this method treats the projection as a radial mapping onto a unit sphere, ensuring the projection of the Gaussian mean coincides with the ray direction from the camera center. It differs from NeRF by avoiding extensive ray sampling, instead determining which tangent-plane Gaussians intersect the ray on the unit sphere.

### [Hybrid Transparency](3294-hybrid-transparency.md)
Hybrid Transparency is a blending formulation that combines depth-ordered alpha blending for the first $K$ fragments (the 'core') with order-independent accumulation for the remaining fragments (the 'tail') to mitigate popping artifacts while maintaining real-time performance. It solves the inconsistency of splat shapes and popping artifacts across viewpoints in VR and motion by avoiding the performance cost of hierarchical sorting or ray tracing. Unlike standard 3D Gaussian Splatting which uses approximate per-primitive sorting, Hybrid Transparency guarantees correct depth ordering for the front-most contributions and uses fast, unsorted blending for the rest.

### [Perspective-Correct 3D Gaussian Splatting via Plücker Coordinates](3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md)
This method computes the value of a 3D Gaussian primitive for a given viewing ray in screen space without performing matrix inversion, utilizing Plücker coordinates to represent the ray and calculate the distance to the Gaussian's center. It sets itself apart from standard 3D Gaussian Splatting rasterization by providing a closed-form, numerically stable solution for perspective-correct splat evaluation that handles degenerate ellipsoids and avoids the computational cost of matrix inversions.

### [3D Gaussian Splatting Rasterization (Cone-Beam/Scanner Space)](3458-3d-gaussian-splatting-rasterization-cone-beam-scanner-space.md)
This method renders X-ray projections from a 3D Gaussian representation by integrating the 3D Gaussian density along cone-beam ray paths, resulting in a sum of 2D Gaussians rather than the alpha-compositing used in standard visible-light rendering. It solves the problem of simulating X-ray CT projections from learned 3D Gaussian fields by adapting the rasterization pipeline to a pinhole camera model, enabling analytical integration of radiative density. Unlike standard 3D Gaussian Splatting which composites opacities, this method sums projected 2D densities, fundamentally addressing integration bias issues where learning integrated density in 2D leads to inconsistencies in 3D density retrieval.

### [Analytic-Splatting Forward Shading](3459-analytic-splatting-forward-shading.md)
Analytic-Splatting Forward Shading computes the color contribution of 3D Gaussians to a pixel by analytically integrating the Gaussian probability density function over the pixel area, rather than sampling the center. It solves the aliasing artifacts inherent in point-sampling approaches by approximating the pixel area integral using a conditioned logistic function. It sets itself apart from vanilla 3DGS by providing anti-aliased results and from Mip-Splatting by using a different analytical approximation that maintains higher detail fidelity, albeit with a slight computational overhead.

### [Speedy-Splat Rendering Pipeline](3543-speedy-splat-rendering-pipeline.md)
The Speedy-Splat Rendering Pipeline is a GPU-accelerated rasterization method for 3D Gaussian Splatting that replaces coarse bounding-box tile intersection tests with precise Gaussian-tile intersection checks and sorting-based depth ordering. It addresses the inefficiency of standard 3D-GS rasterization, which overestimates Gaussian extents and processes non-intersecting pixels, by generating a sparse set of Gaussian-tile pairs and sorting them for efficient pixel-wise compositing.

### [Ghost Ellipsoids for Chunked Training](3559-ghost-ellipsoids-for-chunked-training.md)
Ghost Ellipsoids for Chunked Training resolves boundary artifacts in chunked volume training by including ellipsoids from neighboring chunks in the loss computation without updating their parameters, providing boundary context without requiring a global volume or merged model. Unlike chunk-based 3DGS methods that resolve boundary inconsistencies by merging chunks into a global coarse model or through post-training stages, this method requires boundary support during training itself without access to a global volume.

### [Analytic Transmittance 3DGS](4440-analytic-transmittance-3dgs.md)
Analytic Transmittance 3DGS is a rasterization-based rendering method that replaces the approximate alpha computation of standard 3D Gaussian Splatting (3DGS) with an analytic integration of the volume rendering equation along the camera ray. It computes physically accurate transmittance and alpha values without projecting 3D Gaussians to 2D splats, thereby avoiding the linearization of the exponential transmittance and the assumption of non-self-occlusion inherent in splatting. Unlike its sibling [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md) which relies on 2D Gaussian projection, this method integrates 3D Gaussians directly in ray-space to better represent opaque surfaces and improve view synthesis quality.

### [Ref-Gaussian Deferred Surfel Rendering with Split-Sum IBL](4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md)
Ref-Gaussian Deferred Surfel Rendering with Split-Sum IBL is a real-time deferred shading method for relighting 3D Gaussian Splatting (3DGS) models on mobile hardware (specifically iOS/Metal). It solves the problem of dynamic environment lighting in Gaussian splats by decomposing splats into physically-based rendering (PBR) materials (albedo, normal, roughness, reflectance) and computing lighting via a deferred pipeline with split-sum environment map integration. Unlike forward-shading Gaussian splatters that bake lighting into spherical harmonics or use analytic approximations, this method separates the material definition from the lighting environment, allowing the environment map to be swapped or rotated in real-time without retraining or re-rasterizing the splats. It differs from standard PBR deferred renderers by adapting the splat-based rasterization and surfel accumulation to the deferred G-buffer workflow.

### [3D Gaussian Splatting Self-Shadowing (T_light)](4487-3d-gaussian-splatting-self-shadowing-t-light.md)
Calculates the self-shadowing transmittance $T_{light}$ for each Gaussian along a directional sun direction, representing the 'front occlusion transmittance' to model self-shadowing effects in volume rendering. It differs from standard alpha blending by explicitly tracking accumulated occlusion from the light source's perspective rather than the camera's, enabling physically plausible shadows within the Gaussian volume.

### [Light Space Rasterization for T_light](4489-light-space-rasterization-for-t-light.md)
Light Space Rasterization computes $T_{light}$, the front-facing occlusion transmittance for each 3D Gaussian along a directional sun direction. It differs from voxel-based fallbacks by providing full continuous gradients and zero parameter expansion, enabling differentiable relighting without requiring specific training hyperparameters.

### [2D Gaussian Splatting Rasterization](4815-2d-gaussian-splatting-rasterization.md)
2D Gaussian Splatting Rasterization renders a scene represented by 2D Gaussian surfels by projecting them to screen space and performing ray-surfel intersection per pixel, accumulating colour, alpha, and normal maps via front-to-back alpha compositing. It differs from the standard 3D Gaussian Splatting Rasterization [2812-3d-gaussian-splatting-rasterization.md] by treating primitives as planar surfels with 2D covariance structures rather than 3D ellipsoids, which simplifies the intersection geometry but requires specific handling of the surfel plane orientation relative to the view ray.

### [SparseOIT Rasterizer](7562-sparseoit-rasterizer.md)
The SparseOIT Rasterizer is a rendering technique for 3D Gaussian Splatting that computes a blended image using Order-Independent Transparency (OIT) without sorting Gaussians by depth. It differs from standard 3DGS rasterization, which requires sorting by distance to the camera to correctly accumulate transmittance, by eliminating the sorting step entirely in favor of a weighted average that approximates the order-dependent result. It simultaneously produces a final OIT color image and a pre-rendered state image using Blend-and-Normalize (BAN) and Blend-and-Update (BAU) functions.

### [AdR-Gaussian](8365-adr-gaussian.md)
AdR-Gaussian is a rendering acceleration method for 3D Gaussian Splatting that optimizes the rasterization pipeline by moving serial culling operations to a parallel preprocessing stage and balancing computational loads across pixel threads. It achieves significant speedups without reducing the total number of Gaussians, thereby avoiding the quality degradation associated with pruning-based acceleration methods.

### Panorama Camera Adaptation
A method to adapt the rasterization projection for panoramic images by mapping pixel coordinates to the unit sphere [S5]. For a pixel $(u, v)$ on the image, the 2D coordinate vector $\mathbf{x}_{2D}$ is computed using the panoramic projection formula:
$$ \mathbf{x}_{2D} = \varphi_p \left( \begin{bmatrix} \sin \left( \frac{\pi(-W+2u)}{W} \right) \cos \left( \frac{\pi(-\frac{H}{2}+v)}{H} \right) \\ \sin \left( \frac{\pi(-\frac{H}{2}+v)}{H} \right) \\ \cos \left( \frac{\pi(-\frac{H}{2}+v)}{H} \right) \cos \left( \frac{\pi(-W+2u)}{W} \right) \end{bmatrix} \right) $$
where $H$ and $W$ are the height and width of the image [S5].

### Temporal Marginal Probability Filtering
Pre-rasterization culling step that removes 4D Gaussians with negligible temporal support at the current frame timestamp, stabilizing rendering speed across video length [S6, unofficial]. It computes the marginal probability $p_i(t)$ for each 4D Gaussian $i$ at the current frame's timestamp $t$ and discards Gaussians with negligible $p_i(t)$ before executing standard frustum culling [S6, unofficial]. Unlike methods where Gaussian count scales with video length, this filtering keeps active Gaussians constant, making the approach friendly to long videos in terms of rendering speed [S6, unofficial].

### t-distribution truncation for splatting
A projection truncation technique that replaces the fixed three-sigma rule used in vanilla 3DGS with a $\nu$-dependent critical value interpolation to handle fat-tailed Student's t-distributions [S7]. For t-distributions with thin tails (high $\nu$), the standard three-sigma rule is applied directly. For fat-tailed distributions (low $\nu$), critical values for different $\nu$ degrees are interpolated to compute the appropriate truncation threshold [S7]. Using a fixed three-sigma threshold on fat-tailed t-distributions causes excessive truncation or leakage, which interpolation resolves [S7].

### Gaussian Opacity Fields (GOF)
Uses ray-tracing-style intersection for accurate depth and derives a level-set surface from accumulated opacity [S4, unofficial].

### Progressive Normal Propagation
Uses progressive propagation along normals to fill texture-less regions [S4, unofficial].

### Gaussian Surfels
Uses zero-thickness oriented surfels [S4, unofficial].

### NeuSG
Combines NeuS-style SDF with a GS rasterizer [S4, unofficial].

### Hardware-Accelerated Gaussian Rasterization
A hardware-algorithm co-design that augments standard triangle rasterizers with dedicated units for Gaussian projection and blending, achieving an order of magnitude speedup in the rasterization stage [S8, unofficial].

## Operating characteristics
**Time Complexity:** Dominated by the sorting step, which is $O(N \log N)$ for $N$ Gaussians [S3, unofficial]. The original implementation uses CUB device radix sort, which is highly optimized for CUDA but requires adaptation for other pipelines [S3, unofficial]. Tile-based culling reduces the number of Gaussians processed per pixel, making the effective complexity closer to $O(P)$ where $P$ is the number of pixels, assuming sparse Gaussians [S1, unofficial].

**Memory Complexity:** High VRAM usage is a known characteristic: ~4GB to view and ~12GB to train for typical scenes [S3, unofficial]. Large disk size is also common, often exceeding 1GB per scene [S3, unofficial]. The rasterizer itself requires memory for intermediate tensors (conics, depths, tile intersections) proportional to $N \times C$ [S1, unofficial].

**Measured Performance:** Real-time rendering is achievable on modern GPUs (e.g., RTX 3090/4090) at 1080p or 4K resolutions [S3, unofficial]. Performance scales with the number of Gaussians and image resolution.

**Strengths:**
*   Real-time rendering speeds [S3, unofficial].
*   High-quality, photorealistic results [S3, unofficial].
*   Differentiable, enabling end-to-end training [S2, unofficial].
*   Efficient tile-based rasterization [S1, unofficial].

**Limitations:**
*   High VRAM and disk usage [S3, unofficial].
*   Sorting bottleneck on non-CUDA hardware [S3, unofficial].
*   Incompatible with existing rendering pipelines (Vulkan, DirectX, WebGPU) without significant adaptation [S3, unofficial].
*   Static representation (for now), though dynamic variants exist [S3, unofficial].

**When to use:** When real-time, high-quality novel view synthesis from sparse images is required, and GPU resources are sufficient.

**When not to use:** When hardware constraints are severe (low VRAM), when integration with existing triangle-based pipelines is mandatory without custom shaders, or when mesh extraction is the primary goal (consider [2D Gaussian Splatting](4815-2d-gaussian-splatting-rasterization.md) or [Gaussian Surfels](4815-2d-gaussian-splatting-rasterization.md) variants).

## Background
3D Gaussian Splatting was introduced in the paper "3D Gaussian Splatting for Real-Time Radiance Field Rendering" [S3, unofficial]. It was developed as a rasterization technique analogous to triangle rasterization but using Gaussians, allowing for real-time rendering of photorealistic scenes learned from small samples of images [S3, unofficial]. The method builds on earlier work in radiance fields and neural rendering but distinguishes itself by using explicit Gaussian primitives and a differentiable rasterizer [S3, unofficial]. The primary bottleneck identified in the original implementation is the sorting of millions of Gaussians, which relies on CUDA-specific optimizations like CUB radix sort [S3, unofficial].

## Sources
[S1] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
[S2] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S3] Introduction to 3D Gaussian Splatting - Hugging Face, https://huggingface.co/blog/gaussian-splatting, fetched 2026-10-27 (unofficial)
[S4] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S5] On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy, https://doi.org/10.1007/978-3-031-72643-9_15, fetched 2026-10-29
[S6] “Real-time Photorealistic Dynamic Scene Representation and Rendering with 4D Gaussian Splatting”, https://radiancefields.com/papers/real-time-photorealistic-dynamic-scene-representation-and-rendering-with-4d-gaussian-splatting, fetched 2026-10-30 (unofficial)
[S7] 3D Student Splatting and Scooping, https://doi.org/10.1109/cvpr52734.2025.01960, fetched 2026-10-29
[S8] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
