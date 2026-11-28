# 2D Gaussian Splatting Rasterization

Also called: 2D Gaussian Splatting (2D-GS), 2D Gaussian Splatting Surfel Rasterization, 2D-GS, Gaussian Surfel Splatting, Gaussian Surfels (2DGS), Screen-space Gaussian Rasterization, standard 2DGS

## Overview
2D Gaussian Splatting Rasterization is a differentiable rendering technique that represents scene geometry and radiance using semi-opaque 2D Gaussian primitives (surfels) [S4, S8]. It solves the problem of accurate ray-splat intersection and normal estimation by replacing 3D ellipsoids with 2D oriented planar disks, which allows for more precise surface reconstruction and view-consistent geometry [S6, unofficial][S8]. What sets it apart from its siblings, such as [3D Gaussian Splatting Rasterization](2812-3d-gaussian-splatting-rasterization.md), is the use of 2D Gaussians defined by tangential vectors and a zero depth scale, enabling direct normal extraction via the cross product of tangents and avoiding degenerate line projections through low-pass filtering [S6, unofficial][S8]. It builds on [Primitive Representation](11838-primitive-representation.md) by projecting these 2D primitives to screen space and accumulating color, alpha, depth, and normal maps via front-to-back alpha compositing [S3, S5, unofficial][S8].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `means`: $N \times 3$ array of 3D positions $\boldsymbol{\mu}$ [S2, S5, unofficial].
    *   `quats`: $N \times 4$ array of quaternions $(w, x, y, z)$ defining rotation $R$ [S2, S5, unofficial].
    *   `scales`: $N \times 3$ array of scaling factors. For 2DGS, the depth scale is fixed to 0, so $S = \text{Diag}(s_x, s_y, 0)$ [S7, S8].
    *   `opacities`: $N$ array of scalar opacity values $o$ [S2, S5, unofficial].
    *   `colors`: $N \times 3$ or $N \times K$ array of RGB colors or Spherical Harmonics (SH) coefficients [S2, S5, unofficial].
    *   `viewmats`: $C \times 4 \times 4$ camera-to-world transformation matrices $W$ [S5, unofficial].
    *   `Ks`: $C \times 3 \times 3$ camera intrinsic matrices [S5, unofficial].
    *   `width`, `height`: Image dimensions [S5, unofficial].
    *   `near_plane`, `far_plane`: Clipping distances (default 0.01, 1e10) [S5, unofficial].
    *   `tile_size`: Integer, default 16, side length of tiles for rasterization [S5, unofficial].
*   **Outputs:**
    *   `render_colors`: $C \times H \times W \times 3$ RGB images [S5, unofficial].
    *   `render_alphas`: $C \times H \times W \times 1$ alpha maps [S5, unofficial].
    *   `render_normals`: $C \times H \times W \times 3$ surface normal maps [S5, unofficial].
    *   `render_depths`: $C \times H \times W \times 1$ depth maps [S8].

**Data Structures:**
Each Gaussian primitive $i$ is projected to screen space, resulting in a per-pixel state structure (80 bytes total) [S2, unofficial]:
*   `means2d` (2 elements, 8 B): 2D projected position in pixel space $(u_c, v_c)$ [S2, unofficial].
*   `ray_transforms` (9 elements, 36 B): $3 \times 3$ affine matrix mapping pixel-space UV to splat coordinates in row-major KWH order [S2, unofficial]. This matrix is derived from the screen-space covariance $\Sigma'_{2D}$ [S3, unofficial].
*   `opacities` (1 element, 4 B): Alpha transparency value $o_i$ [S2, unofficial].
*   `colors` (3 elements, 12 B): RGB color $\mathbf{c}_i$ [S2, unofficial].
*   `radii` (1 element, 4 B): Maximum projected radius in pixels for culling/LOD [S2, unofficial].
*   `depths` (1 element, 4 B): Z-depth of the Gaussian center for depth sorting [S2, unofficial].
*   `normals` (3 elements, 12 B): Surface normal $\mathbf{n}_i$ in camera space, computed as $\mathbf{t}_u \times \mathbf{t}_v$ [S8].

**Steps:**

1.  **Projection:**
    Project 3D Gaussians to 2D screen space. For each Gaussian $i$:
    *   Compute the world-space covariance $\Sigma^w_i = R_i S_i S_i^T R_i^T$ [G].
    *   Transform to camera space: $\Sigma_{cam, i} = W_i \Sigma^w_i W_i^T$ [G].
    *   Compute the screen-space covariance $\Sigma'_{2D, i}$ using the Jacobian $\mathbf{J}$ of the perspective projection evaluated at the camera-space center $\mathbf{x}_{c,cam}$ [S3, unofficial]:
        $$ \Sigma'_{2D, i} = \mathbf{J} \Sigma_{cam, i} \mathbf{J}^\top $$
        where $\mathbf{J} \in \mathbb{R}^{2 \times 3}$ [S3, unofficial].
    *   Compute the normal $\mathbf{n}_i$ as the cross product of the two principal tangential vectors $\mathbf{t}_u, \mathbf{t}_v$ derived from the rotation and scale [S8].
    *   Output `means2d`, `ray_transforms` (derived from $\Sigma'_{2D}$), `radii`, `depths`, and `normals` [S2, S5, unofficial].

2.  **Color Evaluation:**
    *   If SH degree is not None: Compute view directions $\mathbf{d}_i = \text{normalize}(\text{means}_i - \text{campos})$. Evaluate SH to get RGB colors $\mathbf{c}_i$ [S5, unofficial].
    *   If SH degree is None: Use direct RGB colors $\mathbf{c}_i$ [S5, unofficial].
    *   Broadcast opacities and colors to match the number of cameras $C$ [S5, unofficial].

3.  **Tile Intersection:**
    *   Compute tile dimensions: `tile_width` = $\lceil \text{width} / \text{tile\_size} \rceil$, `tile_height` = $\lceil \text{height} / \text{tile\_size} \rceil$ [S5, unofficial].
    *   Identify which tiles each Gaussian overlaps using `means2d` and `radii` [S5, unofficial].
    *   Sort Gaussians by depth within each tile [S5, unofficial].

4.  **Rasterization (Per-Pixel):**
    For each pixel $\mathbf{u}$ and each overlapping Gaussian $i$ in depth order:
    *   **Ray-Splat Intersection:**
        Compute the intersection of the ray passing through pixel $\mathbf{u}$ with the 2D Gaussian disk.
        Let $M$ be the projection matrix combining view and intrinsics. Let $p_x, p_y$ be the pixel coordinates.
        Compute intersection vectors in the local tangent space [S5, unofficial]:
        $$ \mathbf{h}_u = -M_g[0, :3] + M_g[2, :3] \cdot p_x $$
        $$ \mathbf{h}_v = -M_g[1, :3] + M_g[2, :3] \cdot p_y $$
        where $M_g$ is the $3 \times 4$ projection matrix for the Gaussian [S5, unofficial].
        
        Compute the cross product to find the intersection point in the tangent plane [S5, unofficial]:
        $$ \mathbf{tmp} = \text{cross}(\mathbf{h}_u, \mathbf{h}_v) $$
        
        **Pitfall:** Guard against degenerate cases where the ray is parallel to the surfel or the surfel is projected as a line. If $|\mathbf{tmp}[2]| < 10^{-10}$, skip the intersection [S5, S6, unofficial].
        
        Compute intersection coordinates $(u, v)$ in the local tangent space [S5, unofficial]:
        $$ u = \mathbf{tmp}[0] / \mathbf{tmp}[2] $$
        $$ v = \mathbf{tmp}[1] / \mathbf{tmp}[2] $$

    *   **Alpha Computation:**
        Compute the 2D Gaussian value at $(u, v)$ [S8]:
        $$ G(u, v) = \exp\left(-\frac{u^2 + v^2}{2}\right) $$
        
        Compute the effective alpha $\alpha_i$ [S3, S5, unofficial]:
        $$ \alpha_i = \min(o_i \cdot G(u, v), \text{MAX\_ALPHA}) $$
        
        **Pitfall:** Skip if $\alpha_i \le 0$ or if the computed sigma (distance metric) is negative [S5, unofficial]. Some implementations use a tighter bound combining 3D and 2D sigma estimates: $\sigma = 0.5 \cdot \min(\sigma_{3D}, \sigma_{2D})$ where $\sigma_{3D} = u^2 + v^2$ and $\sigma_{2D} = 2.0(dx^2 + dy^2)$ [S5, unofficial].

    *   **Accumulation:**
        Maintain transmittance $T = \prod_{j=1}^{i-1} (1 - \alpha_j)$ [S3, unofficial].
        Accumulate color, depth, and normal [S3, unofficial][S8]:
        $$ \mathbf{C}(\mathbf{u}) = \sum_{i \in N} \mathbf{c}_i \alpha_i T_i $$
        $$ D(\mathbf{u}) = \sum_{i \in N} d_i \alpha_i T_i / \sum_{i \in N} \alpha_i T_i $$
        $$ \mathbf{N}(\mathbf{u}) = \sum_{i \in N} \mathbf{n}_i \alpha_i T_i / \sum_{i \in N} \alpha_i T_i $$
        where $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ [S3, unofficial][S8].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def rasterize_2dgs(means, quats, scales, opacities, colors, viewmats, Ks, width, height, tile_size=16):
    """
    Rasterize 2D Gaussian Splatting primitives.
    Simplified NumPy reference implementation.
    """
    N = means.shape[0]
    C = viewmats.shape[0]
    
    # 1. Projection
    # Compute screen-space covariance and means2d
    # This involves: quat_scale_to_covar_preci, fully_fused_projection
    # Outputs: means2d (C, N, 2), ray_transforms (C, N, 3, 3), radii (C, N), depths (C, N), normals (C, N, 3)
    means2d, ray_transforms, radii, depths, normals = project_2dgs(means, quats, scales, viewmats, Ks, width, height)
    
    # 2. Color Evaluation
    # If SH is used, evaluate SH here. Otherwise use direct colors.
    # colors_for_raster shape: (C, N, 3)
    colors_for_raster = evaluate_colors(colors, means, viewmats)
    
    # 3. Tile Intersection
    # Determine which tiles each Gaussian overlaps
    tile_width = int(np.ceil(width / tile_size))
    tile_height = int(np.ceil(height / tile_size))
    
    # isect_ids: (C, N, max_tiles_per_gauss)
    # flatten_ids: (C, total_intersections)
    isect_ids, flatten_ids = intersect_tiles(means2d, radii, depths, tile_size, tile_width, tile_height)
    
    # 4. Rasterization Loop
    render_colors = np.zeros((C, height, width, 3))
    render_alphas = np.zeros((C, height, width, 1))
    render_normals = np.zeros((C, height, width, 3))
    render_depths = np.zeros((C, height, width, 1))
    
    for c in range(C):
        # Get primitives for this camera
        m2d = means2d[c] # (N, 2)
        rt = ray_transforms[c] # (N, 3, 3)
        opac = opacities # (N,)
        cols = colors_for_raster[c] # (N, 3)
        rads = radii[c] # (N,)
        deps = depths[c] # (N,)
        norms = normals[c] # (N, 3)
        
        # Sort primitives by depth
        sort_idx = np.argsort(deps)
        m2d = m2d[sort_idx]
        rt = rt[sort_idx]
        opac = opac[sort_idx]
        cols = cols[sort_idx]
        rads = rads[sort_idx]
        norms = norms[sort_idx]
        
        # Iterate over pixels
        for y in range(height):
            for x in range(width):
                # Find overlapping Gaussians for this pixel
                # In optimized versions, this is done via tile offsets
                # Here, we assume a list of overlapping indices for simplicity
                overlap_indices = get_overlapping_gaussians(x, y, m2d, rads)
                
                T = 1.0
                acc_color = np.zeros(3)
                acc_alpha = 0.0
                acc_depth = 0.0
                acc_normal = np.zeros(3)
                weight_sum = 0.0
                
                for i in overlap_indices:
                    # Ray-Splat Intersection
                    # Compute intersection in local tangent space
                    # Using the projection matrix M for this Gaussian
                    # M is derived from viewmat and K
                    
                    # Simplified intersection logic from [S5, unofficial]
                    # h_u and h_v depend on pixel coordinates and projection matrix
                    # tmp = cross(h_u, h_v)
                    # if abs(tmp[2]) < 1e-10: continue
                    
                    # u, v = tmp[0]/tmp[2], tmp[1]/tmp[2]
                    
                    # Compute Alpha
                    # sigma = u^2 + v^2
                    # alpha = min(opac[i] * exp(-sigma/2), MAX_ALPHA)
                    
                    # Guard against degenerate/negative alpha
                    if alpha <= 0:
                        continue
                        
                    # Accumulate
                    acc_color += cols[i] * alpha * T
                    acc_alpha += alpha * T
                    acc_depth += deps[i] * alpha * T
                    acc_normal += norms[i] * alpha * T
                    weight_sum += alpha * T
                    
                    T *= (1 - alpha)
                    
                    if T < 1e-4:
                        break
                
                render_colors[c, y, x] = acc_color
                render_alphas[c, y, x] = acc_alpha
                if weight_sum > 0:
                    render_depths[c, y, x] = acc_depth / weight_sum
                    render_normals[c, y, x] = acc_normal / weight_sum

    return render_colors, render_alphas, render_normals, render_depths
```

## Subtopics

### Spatially-varying Gaussian (SVG)
A variant where each Gaussian primitive has $M$ Gaussian vertices (typically $M=4$) instead of constant attributes [S7]. It allows spatially varying material and normal attributes by interpolating among these vertices in the tangent space, defined by rotation $R$ and scaling $S$ [S7]. This replaces the constant attributes of standard Gaussians, enabling high-frequency specular reconstruction and accurate reflections without increasing primitive count [S1, S7]. It is chosen when high-fidelity inverse rendering and relighting are required [S7].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$ where $N$ is the number of Gaussians and $P$ is the number of pixels, but optimized to $O(N \cdot K)$ where $K$ is the average number of overlapping Gaussians per pixel using tile-based rasterization [S3, S5, unofficial].
*   **Memory Complexity:** $O(N)$ for storing Gaussian attributes and $O(H \cdot W)$ for the output buffers [S2, S5, unofficial].
*   **Performance:** Real-time performance is achievable on modern GPUs due to the tile-based, fused kernel implementation [S5, unofficial]. It is generally faster than neural field-based methods but may require more primitives than implicit representations for complex geometry [S6, unofficial].
*   **Strengths:** Accurate ray-splat intersection, direct normal estimation, differentiable rendering, real-time capability [S4][S6, unofficial][S8].
*   **Limitations:** Degenerate line projections if not filtered, requires careful handling of opacity and depth sorting [S6, unofficial][S8].

## Background
2D Gaussian Splatting (2DGS) was introduced as a lower-dimensional form of 3D Gaussian Splatting (3DGS) to improve geometry reconstruction and normal estimation [S7, S8]. It replaces 3D ellipsoids with 2D oriented planar disks (surfels), defined by position, scaling, tangential vectors, opacity, and color [S8]. This approach addresses the limitations of 3DGS in modeling complex surfaces and provides view-consistent geometry [S1, S8]. It is widely used in novel view synthesis, SLAM, and inverse rendering [S6, unofficial][S7].

## Sources
[S1] ROS-GS: Relightable Outdoor Scenes With Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3709027, fetched 2026-10-26
[S2] Scaling Point-based Differentiable Rendering for Large-scale Reconstruction, https://doi.org/10.48550/arxiv.2512.20017, fetched 2026-10-26 (unofficial)
[S3] Geometry-Grounded Gaussian Splatting, https://arxiv.org/abs/2601.17835v2, fetched 2026-10-26 (unofficial)
[S4] Differentiable Light Transport with Gaussian Surfels via Adapted Radiosity for Efficient Relighting and Geometry Reconstruction, https://doi.org/10.1145/3763305, fetched 2026-10-27
[S5] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
[S6] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
[S7] SVG-IR: Spatially-Varying Gaussian Splatting for Inverse Rendering, https://doi.org/10.1109/cvpr52734.2025.01505, fetched 2026-10-29
[S8] PS-GS: Gaussian splatting for multi-view photometric stereo, https://doi.org/10.1016/j.cag.2026.104561, fetched 2026-10-29
