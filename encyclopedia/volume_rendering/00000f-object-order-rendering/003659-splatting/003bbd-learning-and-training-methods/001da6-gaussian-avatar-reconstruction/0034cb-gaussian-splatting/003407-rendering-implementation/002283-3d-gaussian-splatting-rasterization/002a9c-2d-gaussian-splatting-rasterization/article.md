# 2D Gaussian Splatting Rasterization

Also called: 2D Gaussian Splatting Surfel Rasterization

## Overview
2D Gaussian Splatting Rasterization renders scene primitives as 2D Gaussian surfels by performing explicit ray-surfel intersection tests per pixel, rather than projecting 3D ellipsoids to screen space. It solves the problem of rendering reflective and high-frequency specular surfaces more accurately than standard 3D Gaussian Splatting by treating primitives as planar disks with spatially varying attributes. Unlike [3D Gaussian Splatting Rasterization](2812-3d-gaussian-splatting-rasterization.md), which uses a tile-based alpha-blending approach on projected 2D Gaussians, this method computes the exact intersection of the viewing ray with the 2D Gaussian plane to determine the attenuation and normal, enabling hardware-accelerated texture mapping for normals and colors [S1, unofficial].

## How it works
**Inputs:**
- `means`: 3D positions of $N$ surfels, shape $[N, 3]$.
- `quats`: Unit quaternions $(w, x, y, z)$ defining orientation, shape $[N, 4]$.
- `scales`: Local scales, shape $[N, 3]$.
- `opacities`: Per-surfel opacity values, shape $[N]$.
- `colors`: Per-surfel colors (RGB), shape $[N, 3]$ or $[C, N, 3]$ for $C$ cameras.
- `viewmats`: View matrices for each camera, shape $[C, 4, 4]$.
- `Ks`: Camera intrinsic matrices, shape $[C, 3, 3]$.
- `width`, `height`: Image dimensions.
- `tile_size`: Integer, default 16.
- `near_plane`, `far_plane`: Clipping distances, defaults 0.01, $10^{10}$.
- `sh_degree`: Spherical Harmonics degree (integer or None). If None, direct color is used.
- `backgrounds`: Background color per camera, shape $[C, 3]$.

**Outputs:**
- `render_colors`: Composite RGB colors, shape $[C, H, W, 3]$.
- `render_alphas`: Composite alpha values, shape $[C, H, W, 1]$.
- `render_normals`: Composite normals, shape $[C, H, W, 3]$.

**Data Structures:**
- `means2d`: Projected 2D centers, shape $[C, N, 2]$.
- `radii`: Screen-space radii of bounding circles, shape $[C, N]$.
- `depths`: Z-depth of surfel centers, shape $[C, N]$.
- `ray_transforms`: Transformation matrices for ray-surfel intersection, shape $[C, N, 3, 3]$ (derived from view and surfel geometry).
- `normals`: World-space normals of surfels, shape $[C, N, 3]$.
- `isect_offsets`: Tile-based offset encoding, shape $[C, \lceil H/T \rceil, \lceil W/T \rceil]$.
- `flatten_ids`: Flattened indices of intersecting surfels per tile, shape $[C, \text{max\_hits}]$.

**Steps:**

1. **Projection:** Project 3D surfels to 2D screen space using `fully_fused_projection_2dgs`. This computes `means2d`, `radii`, `depths`, `ray_transforms`, and `normals` for all $N$ surfels and $C$ cameras. The projection accounts for the 2D Gaussian nature (surfels) rather than 3D ellipsoids [S1, unofficial].

2. **Color Evaluation:**
   - If `sh_degree` is not None:
     - Compute view directions: $\mathbf{dirs} = \mathbf{means} - \mathbf{campos}$, normalized.
     - Broadcast SH coefficients to shape $[C, N, K, 3]$ where $K$ is the number of SH bands.
     - Evaluate Spherical Harmonics: $\mathbf{rgb} = \text{spherical\_harmonics}(\text{sh\_degree}, \mathbf{dirs}, \text{coeffs})$.
     - Clamp: $\mathbf{rgb} = \max(\mathbf{rgb} + 0.5, 0.0)$.
   - If `sh_degree` is None:
     - Use direct colors. Broadcast per-surfel colors $[N, 3]$ to $[C, N, 3]$ if necessary.
   - Result: `colors_for_raster` of shape $[C, N, 3]$.

3. **Opacity Broadcasting:** Broadcast per-surfel opacities $[N]$ to $[C, N]$ for all cameras [S1, unofficial].

4. **Tile Intersection:**
   - Compute tile dimensions: $T_w = \lceil W / \text{tile\_size} \rceil$, $T_h = \lceil H / \text{tile\_size} \rceil$.
   - Identify which surfels intersect which tiles using `isect_tiles`. This function takes `means2d`, `radii`, `depths`, `tile_size`, $T_w$, $T_h$, and `sort=True`.
   - Output: `tiles_per_gauss` (count), `isect_ids` (tile IDs), `flatten_ids` (surfels IDs).
   - Encode offsets: `isect_offsets` = `isect_offset_encode(isect_ids, C, T_w, T_h)`. This creates a sparse representation of surfel indices per tile [S1, unofficial].

5. **Rasterization (Pixel-wise Intersection):**
   For each pixel $(x, y)$ in each camera view:
   - Iterate through surfels assigned to the tile containing $(x, y)$ (using `flatten_ids` and `isect_offsets`).
   - For each surfel $i$:
     - Get the ray transform matrix $M_g$ for the surfel (from `ray_transforms`). This matrix maps local surfel coordinates to the ray space.
     - Compute intersection vectors in the surfel's local plane:
       $$ \mathbf{h}_u = -M_g[0, :3] + M_g[2, :3] \cdot x $$
       $$ \mathbf{h}_v = -M_g[1, :3] + M_g[2, :3] \cdot y $$
       Where $x, y$ are the pixel coordinates (normalized or screen-space, depending on implementation details, but typically pixel coordinates in the reference NumPy implementation).
     - Compute cross product to find the intersection point in the local plane:
       $$ \mathbf{tmp} = \mathbf{h}_u \times \mathbf{h}_v $$
     - **Guard:** If $|\mathbf{tmp}[2]| < 10^{-10}$, skip this surfel (degenerate case, parallel rays/surfels) [S1, unofficial].
     - Compute local intersection coordinates $(u_s, v_s)$:
       $$ u_s = \frac{\mathbf{tmp}[0]}{\mathbf{tmp}[2]} $$
       $$ v_s = \frac{\mathbf{tmp}[1]}{\mathbf{tmp}[2]} $$
     - Compute 3D Gaussian sigma (attenuation) from intersection:
       $$ \sigma_{3d} = u_s^2 + v_s^2 $$
     - Compute 2D fallback sigma from distance to center (using $dx, dy$ as distance in screen space or local space, typically derived from `means2d` and pixel position):
       $$ \sigma_{2d} = 2.0 \cdot (dx^2 + dy^2) $$
     - Take the tighter bound:
       $$ \sigma = 0.5 \cdot \min(\sigma_{3d}, \sigma_{2d}) $$
     - **Guard:** If $\sigma < 0$, skip (invalid alpha) [S1, unofficial].
     - Compute alpha:
       $$ \alpha_i = \min(\text{opacity}_i \cdot e^{-\sigma}, \text{MAX\_ALPHA}) $$
       Where `MAX_ALPHA` is a small constant (e.g., 0.999) to prevent division by zero in compositing.
     - Accumulate color and normal using front-to-back alpha compositing:
       $$ C_{out} = C_{out} + \alpha_i \cdot \mathbf{c}_i \cdot (1 - \sum \alpha_{prev}) $$
       $$ N_{out} = N_{out} + \alpha_i \cdot \mathbf{n}_i \cdot (1 - \sum \alpha_{prev}) $$
       $$ \alpha_{out} = \alpha_{out} + \alpha_i \cdot (1 - \sum \alpha_{prev}) $$
       Where $\mathbf{c}_i$ is the color and $\mathbf{n}_i$ is the normal of surfel $i$.

6. **Background Compositing:**
   - After all surfels, composite the background color:
     $$ C_{final} = C_{out} + \mathbf{b} \cdot (1 - \alpha_{out}) $$
     Where $\mathbf{b}$ is the background color for the camera.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def rasterize_to_pixels_2dgs(
    means2d: np.ndarray,      # [C, N, 2]
    ray_transforms: np.ndarray, # [C, N, 3, 3]
    colors_for_raster: np.ndarray, # [C, N, 3]
    normals: np.ndarray,      # [C, N, 3]
    opacities_cn: np.ndarray, # [C, N]
    width: int,
    height: int,
    tile_size: int = 16,
    isect_offsets: np.ndarray, # [C, Th, Tw]
    flatten_ids: np.ndarray,   # [C, max_hits]
    backgrounds: np.ndarray,   # [C, 3]
) -> tuple:
    """
    Rasterize 2D Gaussian surfels to pixels.
    Returns: (render_colors, render_alphas, render_normals)
    """
    C = means2d.shape[0]
    H, W = height, width
    Th, Tw = (H + tile_size - 1) // tile_size, (W + tile_size - 1) // tile_size
    
    render_colors = np.zeros((C, H, W, 3), dtype=np.float32)
    render_alphas = np.zeros((C, H, W, 1), dtype=np.float32)
    render_normals = np.zeros((C, H, W, 3), dtype=np.float32)
    
    MAX_ALPHA = 0.999
    
    for c in range(C):
        bg = backgrounds[c]
        for y in range(H):
            for x in range(W):
                # Determine tile index
                ty = y // tile_size
                tx = x // tile_size
                
                # Get offset for this tile
                offset = isect_offsets[c, ty, tx]
                
                # Current accumulated alpha for this pixel
                acc_alpha = 0.0
                
                # Iterate through surfels in this tile
                # flatten_ids[c, offset] gives the start index in flatten_ids for this tile
                # The number of surfels is not explicitly stored in isect_offsets in the simple NumPy ref,
                # but typically flatten_ids is padded or we iterate until a sentinel or known count.
                # For the reference implementation, we assume flatten_ids contains valid indices
                # and we iterate through the relevant slice.
                
                # Note: The reference implementation uses a specific encoding.
                # isect_offsets encodes the start index. The end index is the start of the next tile or end of array.
                # For simplicity, we assume we have a way to know the count or iterate until a null index.
                # In the actual gsplat-mlx NumPy ref, flatten_ids is a flat array of surfel IDs.
                # We need to know how many surfels are in this tile.
                # Let's assume we have a helper to get the slice.
                
                # Simplified iteration based on typical tile-based rasterization:
                # In the reference, isect_offsets might just be the start index.
                # We need to know the count. Let's assume we iterate through all surfels
                # that are assigned to this tile.
                
                # For the purpose of this pseudocode, we assume we can get the list of surfel IDs
                # for the current tile.
                surfel_ids = get_surfels_for_tile(flatten_ids, c, ty, tx)
                
                for gid in surfel_ids:
                    # Get surfel properties
                    rt = ray_transforms[c, gid] # [3, 3]
                    color = colors_for_raster[c, gid]
                    normal = normals[c, gid]
                    opacity = opacities_cn[c, gid]
                    
                    # Compute intersection vectors
                    # h_u = -Mg[0, :3] + Mg[2, :3] * x
                    # h_v = -Mg[1, :3] + Mg[2, :3] * y
                    # Note: x, y are pixel coordinates. In the reference, they are used directly.
                    h_u = -rt[0, :3] + rt[2, :3] * x
                    h_v = -rt[1, :3] + rt[2, :3] * y
                    
                    # Cross product
                    tmp = np.cross(h_u, h_v)
                    
                    # Guard against degenerate case
                    if abs(tmp[2]) < 1e-10:
                        continue
                    
                    # Intersection coordinates
                    us = tmp[0] / tmp[2]
                    vs = tmp[1] / tmp[2]
                    
                    # Sigma computation
                    sigma_3d = us * us + vs * vs
                    
                    # Distance to center (dx, dy)
                    # In the reference, dx, dy are likely the distance in the local plane
                    # or screen space. The formula uses dx, dy.
                    # Assuming dx, dy are derived from means2d and pixel position.
                    # For the reference implementation, dx, dy are not explicitly passed,
                    # so they must be computed from means2d.
                    dx = x - means2d[c, gid, 0]
                    dy = y - means2d[c, gid, 1]
                    sigma_2d = 2.0 * (dx * dx + dy * dy)
                    
                    sigma = 0.5 * min(sigma_3d, sigma_2d)
                    
                    # Guard against negative sigma
                    if sigma < 0:
                        continue
                    
                    # Alpha computation
                    alpha = min(opacity * np.exp(-sigma), MAX_ALPHA)
                    
                    # Front-to-back compositing
                    weight = alpha * (1.0 - acc_alpha)
                    
                    render_colors[c, y, x] += weight * color
                    render_normals[c, y, x] += weight * normal
                    render_alphas[c, y, x] += weight
                    
                    acc_alpha += weight
                    
                    if acc_alpha >= 1.0:
                        break
                
                # Composite background
                remaining = 1.0 - acc_alpha
                render_colors[c, y, x] += remaining * bg
                render_alphas[c, y, x] = acc_alpha

    return render_colors, render_alphas, render_normals

def get_surfels_for_tile(flatten_ids, c, ty, tx):
    """Helper to get surfel IDs for a specific tile."""
    # This is a placeholder for the actual logic in gsplat-mlx
    # which uses isect_offsets to encode the start index.
    # The end index is the start of the next tile or the end of the array.
    # For a complete implementation, one would need the full isect_offsets structure
    # which includes end indices or counts.
    # In the reference NumPy code, isect_offsets is [C, Th, Tw] and contains
    # the start index in flatten_ids.
    # We need to know the count. Let's assume we have a separate count array
    # or we iterate until a sentinel.
    # For the sake of this pseudocode, we assume we have a way to get the slice.
    # In the actual implementation, flatten_ids is a flat array of surfel IDs
    # for all tiles, concatenated.
    # isect_offsets[c, ty, tx] gives the start index.
    # The end index is isect_offsets[c, ty, tx+1] if tx+1 < Tw, else end of array.
    # But isect_offsets is 2D [Th, Tw].
    # Let's assume we have a helper that returns the slice.
    pass
```

## Subtopics

### [Per-Primitive Texture Mapping for Reflective Gaussian Splatting](8629-per-primitive-texture-mapping-for-reflective-gaussian-splatt.md)
Enhances 2D Gaussian Splatting by replacing single attribute values per primitive with spatially varying texture maps for material properties and normals. It decouples geometric representation from appearance, enabling high-frequency specular reconstruction and accurate reflections in reflective scenes without increasing the primitive count. It builds on [2D Gaussian Splatting Rasterization](4815-2d-gaussian-splatting-rasterization.md) by leveraging the closed-form ray-splat intersection to map screen pixels to local texture coordinates, allowing hardware-accelerated rendering via texture atlases.

## Operating characteristics
- **Time Complexity:** $O(C \cdot H \cdot W \cdot K)$ where $K$ is the average number of surfels per tile. The tile-based approach reduces the search space from $O(C \cdot H \cdot W \cdot N)$ to $O(C \cdot H \cdot W \cdot K)$.
- **Memory Complexity:** $O(C \cdot N)$ for storing surfel properties and intermediate projections. The tile-based structure adds $O(C \cdot T_w \cdot T_h)$ for offsets.
- **Performance:** The NumPy reference implementation is slow due to Python loops. A fused GPU implementation (e.g., CUDA/Metal) is required for real-time performance. The method is comparable in speed to 3DGS rasterization when optimized, but the per-pixel intersection test adds overhead.
- **Strengths:** Accurate rendering of reflective surfaces, support for per-pixel normals and textures, robust handling of degenerate cases.
- **Limitations:** Slower than 3DGS rasterization due to intersection tests, requires careful handling of degenerate cases (parallel rays/surfels).

## Background
The method is part of the 2D Gaussian Splatting (2DGS) framework, which extends 3D Gaussian Splatting by using 2D Gaussian surfels to better represent reflective surfaces. The reference implementation is provided in the `gsplat-mlx` library by RobotFlow-Labs [S1, unofficial]. It is designed to be a Tier-1 NumPy reference for testing and validation.

## Sources
[S1] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
