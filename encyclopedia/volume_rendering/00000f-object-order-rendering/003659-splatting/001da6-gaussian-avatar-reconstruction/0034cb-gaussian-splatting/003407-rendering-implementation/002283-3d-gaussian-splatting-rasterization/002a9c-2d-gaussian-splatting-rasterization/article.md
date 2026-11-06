# 2D Gaussian Splatting Rasterization

Also called: 2D Gaussian Splatting Surfel Rasterization, rasterize_to_pixels_2dgs [S1, unofficial]

## Overview
2D Gaussian Splatting Rasterization renders a scene represented by 2D Gaussian surfels by projecting them to screen space and performing ray-surfel intersection per pixel, accumulating colour, alpha, and normal maps via front-to-back alpha compositing [S1, unofficial]. It differs from the standard 3D Gaussian Splatting Rasterization [2812-3d-gaussian-splatting-rasterization.md] by treating primitives as planar surfels with 2D covariance structures rather than 3D ellipsoids, which simplifies the intersection geometry but requires specific handling of the surfel plane orientation relative to the view ray [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   `means`: 3D positions of surfels, shape `[N, 3]` [S1, unofficial].
*   `quats`: Rotation quaternions (w, x, y, z), shape `[N, 4]` [S1, unofficial].
*   `scales`: Scale factors, shape `[N, 3]` [S1, unofficial].
*   `opacities`: Per-surfel opacity, shape `[N]` [S1, unofficial].
*   `colors`: Per-surfel color coefficients. If `sh_degree` is not None, shape `[N, K, 3]` where $K$ is the number of SH coefficients [S1, unofficial]. If `sh_degree` is None, shape `[N, D]` where $D$ is the number of color channels (typically 3) [S1, unofficial].
*   `viewmats`: View matrices, shape `[C, 4, 4]` [S1, unofficial].
*   `Ks`: Camera intrinsic matrices, shape `[C, 3, 3]` [S1, unofficial].
*   `campos`: Camera positions, shape `[C, 3]` [S1, unofficial].
*   `width`, `height`: Image dimensions [S1, unofficial].
*   `near_plane`: Near clipping plane distance, default 0.01 [S1, unofficial].
*   `far_plane`: Far clipping plane distance, default 1e10 [S1, unofficial].
*   `tile_size`: Integer, default 16. Side length of tiles for rasterization [S1, unofficial].
*   `sh_degree`: Integer or None. If not None, spherical harmonics are evaluated [S1, unofficial].
*   `backgrounds`: Background color per camera, shape `[C, 3]` [S1, unofficial].

**Outputs:**
*   `render_colors`: Computed color per pixel, shape `[C, H, W, channels]` [S1, unofficial].
*   `render_alphas`: Computed alpha per pixel, shape `[C, H, W, 1]` [S1, unofficial].
*   `render_normals`: Computed normal per pixel, shape `[C, H, W, 3]` [S1, unofficial].

### Data Structures
*   **Surfel Properties**: Stored in contiguous arrays or tensors. `means` (float32, `[N, 3]`), `quats` (float32, `[N, 4]`), `scales` (float32, `[N, 3]`), `opacities` (float32, `[N]`) [S1, unofficial].
*   **Projection Outputs**:
    *   `radii`: Radius of the 2D Gaussian bounding circle per surfel per camera, shape `[C, N]` [S1, unofficial].
    *   `means2d`: Projected 2D center coordinates, shape `[C, N, 2]` [S1, unofficial].
    *   `depths`: Z-depth of surfel centers, shape `[C, N]` [S1, unofficial].
    *   `ray_transforms`: Transformation matrices for ray-surfel intersection, shape `[C, N, 3, 3]` (derived from view and projection) [S1, unofficial].
    *   `normals`: Surface normals in world space, shape `[C, N, 3]` [S1, unofficial].
*   **Tile Indices**:
    *   `isect_offsets`: Encoded intersection offsets, shape `[C, tile_height, tile_width]` [S1, unofficial].
    *   `flatten_ids`: Flattened list of surfel IDs sorted by depth within tiles [S1, unofficial].

### Steps

1.  **Projection**: Project 3D surfels to 2D screen space using `fully_fused_projection_2dgs`, passing `means`, `quats`, `scales`, `viewmats`, `Ks`, `width`, `height`, `near_plane`, and `far_plane` [S1, unofficial]. Outputs `radii`, `means2d`, `depths`, `ray_transforms`, and `normals` [S1, unofficial].
2.  **Color Preparation**:
    *   Broadcast scalar per-surfel opacities `[N]` to per-camera `[C, N]` using `mx.broadcast_to(mx.expand_dims(opacities, axis=0), (C, N))` [S1, unofficial].
    *   If `sh_degree` is not None, compute view directions `dirs = means[None, :, :] - campos[:, None, :]`, normalize `dirs`, broadcast color coefficients to `[C, N, K, 3]`, evaluate `spherical_harmonics(sh_degree, dirs, coeffs_broadcast)` to get `rgb` `[C, N, 3]`, and clamp with `mx.maximum(rgb + 0.5, 0.0)` [S1, unofficial].
    *   If `sh_degree` is None, broadcast colors `[N, D]` to `[C, N, D]` or use colors `[C, N, D]` directly as `colors_for_raster` [S1, unofficial].
3.  **Tile Assignment**:
    *   Compute tile dimensions `tile_width = ceil(width / tile_size)`, `tile_height = ceil(height / tile_size)` [S1, unofficial].
    *   Call `isect_tiles(means2d, radii, depths, tile_size, tile_width, tile_height, sort=True)` to get `tiles_per_gauss`, `isect_ids`, `flatten_ids` [S1, unofficial].
    *   Encode intersection offsets using `isect_offset_encode(isect_ids, C, tile_width, tile_height)` to get `isect_offsets` `[C, tile_height, tile_width]` [S1, unofficial].
4.  **Rasterization**:
    *   Call `rasterize_to_pixels_2dgs` with `means2d`, `ray_transforms`, `colors_for_raster`, `normals`, `opacities_cn`, `width`, `height`, `tile_size`, `isect_offsets`, `flatten_ids`, and `backgrounds` [S1, unofficial].
    *   This function iterates through surfels in depth order within each tile [S1, unofficial].

### Intersection and Compositing Formula

For each surfel `gid` in depth order and each pixel `(pxf, pyf)` it covers:

1.  **Ray-Surfel Intersection Vectors**:
    Let $Mg$ be the ray transform matrix for surfel $g$ (from `ray_transforms`) [S1, unofficial]. Let $p_x, p_y$ be the pixel coordinates (normalized or screen space as defined by the projection, typically normalized device coordinates or similar relative to the center) [S1, unofficial]. The source specifies:
    $$ h_u = -Mg[0, :3] + Mg[2, :3] \cdot p_x $$
    $$ h_v = -Mg[1, :3] + Mg[2, :3] \cdot p_y $$
    Where $Mg[0, :3]$ is the first row's x,y,z components, etc. [S1, unofficial].

2.  **Cross Product and Degeneracy Guard**:
    $$ tmp = h_u \times h_v $$
    If $|tmp[2]| < 10^{-10}$, skip this intersection (degenerate case where rays/surfels are parallel) [S1, unofficial].

3.  **Intersection Coordinates**:
    $$ u_s = \frac{tmp[0]}{tmp[2]} $$
    $$ v_s = \frac{tmp[1]}{tmp[2]} $$
    These $(u_s, v_s)$ represent the coordinates in the local 2D Gaussian space [S1, unofficial].

4.  **Sigma Calculation**:
    Compute 3D sigma from intersection:
    $$ \sigma_{3d} = u_s^2 + v_s^2 $$
    Compute 2D fallback sigma from distance to center (where $dx, dy$ are the 2D distance components from `means2d` to pixel center, or derived from $u_s, v_s$ depending on implementation specifics, but the source explicitly defines a fallback):
    $$ \sigma_{2d} = 2.0 \cdot (dx^2 + dy^2) $$
    Take the tighter bound:
    $$ \sigma = 0.5 \cdot \min(\sigma_{3d}, \sigma_{2d}) $$
    Negative sigma values are skipped to prevent invalid alpha computation [S1, unofficial].

5.  **Alpha Computation**:
    $$ \alpha = \min(\text{opacity} \cdot e^{-\sigma}, \text{MAX\_ALPHA}) $$
    Where `MAX_ALPHA` is a clamping value (e.g., 0.999 or similar, exact value not specified in source, but clamping is required) [S1, unofficial].

6.  **Front-to-Back Compositing**:
    Accumulate color and alpha for the pixel using standard front-to-back alpha blending:
    $$ C_{out} = C_{prev} \cdot (1 - \alpha) + C_{surf} \cdot \alpha $$
    $$ \alpha_{out} = \alpha_{prev} + (1 - \alpha_{prev}) \cdot \alpha $$
    Where $C_{surf}$ is the surfel color and $\alpha$ is the computed alpha. The process starts with background color and zero alpha [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def rasterize_to_pixels_2dgs(
    means2d: np.ndarray,      # [C, N, 2]
    ray_transforms: np.ndarray, # [C, N, 3, 3]
    colors: np.ndarray,       # [C, N, 3]
    normals: np.ndarray,      # [C, N, 3]
    opacities: np.ndarray,    # [C, N]
    radii: np.ndarray,        # [C, N]
    width: int,
    height: int,
    tile_size: int,
    isect_offsets: np.ndarray, # [C, tile_h, tile_w]
    flatten_ids: np.ndarray,   # [total_surfels_in_tiles]
    backgrounds: np.ndarray,   # [C, 3]
    MAX_ALPHA: float = 0.999   # [G] Standard clamping value; exact value not given in sources.
) -> tuple:
    """
    Rasterize 2D Gaussian surfels to pixels.
    Returns: (render_colors, render_alphas, render_normals)
    """
    C = means2d.shape[0]
    H, W = height, width
    tile_h = int(np.ceil(H / tile_size))
    tile_w = int(np.ceil(W / tile_size))
    
    render_colors = np.zeros((C, H, W, 3), dtype=np.float32)
    render_alphas = np.zeros((C, H, W, 1), dtype=np.float32)
    render_normals = np.zeros((C, H, W, 3), dtype=np.float32)
    
    for c in range(C):
        render_colors[c, :, :, :] = backgrounds[c]
        render_alphas[c, :, :, 0] = 0.0
        render_normals[c, :, :, :] = [0.0, 0.0, 1.0]

    for c in range(C):
        for ty in range(tile_h):
            for tx in range(tile_w):
                start_idx = isect_offsets[c, ty, tx]
                end_idx = isect_offsets[c, ty + 1, tx] if ty + 1 < tile_h else len(flatten_ids)
                
                for idx in range(start_idx, end_idx):
                    gid = flatten_ids[idx]
                    m2d = means2d[c, gid]
                    rt = ray_transforms[c, gid]
                    color = colors[c, gid]
                    normal = normals[c, gid]
                    alpha_surf = opacities[c, gid]
                    
                    r = radii[c, gid]
                    min_px = max(0, int(m2d[0] - r))
                    max_px = min(W, int(m2d[0] + r))
                    min_py = max(0, int(m2d[1] - r))
                    max_py = min(H, int(m2d[1] + r))
                    
                    for py in range(min_py, max_py):
                        for px in range(min_px, max_px):
                            pxf = float(px)
                            pyf = float(py)
                            
                            h_u = -rt[0, :3] + rt[2, :3] * pxf
                            h_v = -rt[1, :3] + rt[2, :3] * pyf
                            
                            tmp = np.cross(h_u, h_v)
                            if abs(tmp[2]) < 1e-10:
                                continue
                                
                            us = tmp[0] / tmp[2]
                            vs = tmp[1] / tmp[2]
                            
                            sigma_3d = us * us + vs * vs
                            dx = m2d[0] - pxf
                            dy = m2d[1] - pyf
                            sigma_2d = 2.0 * (dx * dx + dy * dy)
                            sigma = 0.5 * min(sigma_3d, sigma_2d)
                            
                            if sigma < 0:
                                continue
                                
                            alpha = min(alpha_surf * np.exp(-sigma), MAX_ALPHA)
                            
                            render_colors[c, py, px, :] = (
                                render_colors[c, py, px, :] * (1.0 - alpha) + color * alpha
                            )
                            render_alphas[c, py, px, 0] += alpha * (1.0 - render_alphas[c, py, px, 0])
                            render_normals[c, py, px, :] = normal

    return render_colors, render_alphas, render_normals
```

### Parameters
*   `tile_size`: Integer, default 16. Side length of tiles for rasterization [S1, unofficial].
*   `near_plane`: Float, default 0.01. Near clipping plane distance [S1, unofficial].
*   `far_plane`: Float, default 1e10. Far clipping plane distance [S1, unofficial].

### Pitfalls
*   **Degenerate Cross Product**: When `tmp[2]` is near zero, the rays and surfels are parallel, leading to division by zero. This is handled by skipping the intersection if `abs(tmp[2]) < 1e-10` [S1, unofficial].
*   **Negative Sigma**: Negative sigma values result in invalid alpha computation (exponential growth). These are skipped [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The 2D Gaussian Splatting Rasterization method is part of the 2D Gaussian Splatting (2DGS) approach, which represents scenes using 2D Gaussian surfels [S1, unofficial]. This method was introduced as an alternative to 3D Gaussian Splatting to improve rendering efficiency and quality for certain types of scenes [S1, unofficial]. The reference implementation is provided by the `gsplat-mlx` library [S1, unofficial].

## Sources

[S1] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
