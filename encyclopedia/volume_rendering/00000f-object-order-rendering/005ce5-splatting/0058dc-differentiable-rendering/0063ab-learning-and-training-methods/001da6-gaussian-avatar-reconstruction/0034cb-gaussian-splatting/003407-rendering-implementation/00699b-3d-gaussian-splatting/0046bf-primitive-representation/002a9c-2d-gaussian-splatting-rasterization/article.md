# 2D Gaussian Splatting Rasterization

Also called: 2D Gaussian Splatting Surfel Rasterization

## Overview
This method renders a scene represented by 2D Gaussian primitives (surfels) into RGB, alpha, normal, and depth maps using a custom rasterizer [S1, unofficial]. It differs from standard 3D Gaussian Splatting by operating on surfels projected into screen space, accumulating color and geometric attributes via front-to-back alpha compositing [Parent]. It serves as the Tier-1 NumPy reference implementation for 2D Gaussian Splatting, performing ray-surfel intersection per pixel using the projection matrix [S1, unofficial].

## How it works
**Inputs:**
- `means`: 3D positions of $N$ surfels, shape $[N, 3]$.
- `quats`: Unit quaternions $(w, x, y, z)$ defining orientation, shape $[N, 4]$.
- `scales`: Local scaling factors, shape $[N, 3]$.
- `opacities`: Per-surfel opacity values, shape $[N]$.
- `colors`: Per-surfel colors, shape $[N, D]$ (where $D$ is color channels, typically 3) or spherical harmonic coefficients if `sh_degree` is provided.
- `viewmats`: Camera-to-world transformation matrices, shape $[C, 4, 4]$ (where $C$ is number of cameras).
- `Ks`: Camera intrinsic matrices, shape $[C, 3, 3]$.
- `width`, `height`: Image dimensions.
- `tile_size`: Integer, default 16. Side length of tiles for rasterization [S1, unofficial].
- `near_plane`: Float, default 0.01. Near clipping plane distance [S1, unofficial].
- `far_plane`: Float, default 1e10. Far clipping plane distance [S1, unofficial].
- `sh_degree`: Integer or None. If not None, spherical harmonics are evaluated; otherwise, direct colors are used [S1, unofficial].
- `backgrounds`: Background color for each camera, shape $[C, 3]$ [S1, unofficial].

**Outputs:**
- `render_colors`: Shape $[C, H, W, 3]$.
- `render_alphas`: Shape $[C, H, W, 1]$.
- `render_normals`: Shape $[C, H, W, 3]$.

**Data Structures:**
- `means2d`: Projected 2D centers, shape $[C, N, 2]$.
- `radii`: Screen-space radii of surfels, shape $[C, N]$.
- `depths`: Depth of surfel centers, shape $[C, N]$.
- `ray_transforms`: Transformation matrices for ray-surfel intersection, shape $[C, N, 3, 3]$ (derived from projection).
- `normals`: World-space normals of surfels, shape $[C, N, 3]$.
- `isect_offsets`: Encoded intersection offsets for tile-based processing, shape $[C, \text{tile\_height}, \text{tile\_width}]$.
- `flatten_ids`: Flattened indices of surfels intersecting tiles, shape $[C, \text{num\_intersecting}]$.

**Steps:**

1. **Projection:** Project 2D Gaussian surfels to 2D screen space using `fully_fused_projection_2dgs`. This function takes `means`, `quats`, `scales`, `viewmats`, `Ks`, `width`, `height`, `near_plane`, and `far_plane`. It outputs `radii`, `means2d`, `depths`, `ray_transforms`, and `normals` [S1, unofficial].

2. **Opacity Broadcasting:** Broadcast scalar per-surfel opacities $[N]$ to per-camera $[C, N]$ using `mx.broadcast_to(mx.expand_dims(opacities, axis=0), (C, N))` [S1, unofficial].

3. **Color Evaluation:**
   - If `sh_degree` is not None:
     - Compute view directions: $\text{dirs} = \text{means}[\text{None}, :, :] - \text{campos}[:, \text{None}, :]$.
     - Normalize `dirs`.
     - Broadcast color coefficients to $[C, N, K, 3]$ (where $K$ is SH coefficient count).
     - Evaluate spherical harmonics: $\text{rgb} = \text{spherical\_harmonics}(\text{sh\_degree}, \text{dirs}, \text{coeffs\_broadcast})$.
     - Clamp with $\text{rgb} = \max(\text{rgb} + 0.5, 0.0)$ [S1, unofficial].
   - If `sh_degree` is None:
     - Broadcast colors $[N, D]$ to $[C, N, D]$ or use colors $[C, N, D]$ directly as `colors_for_raster` [S1, unofficial].

4. **Tile Intersection:**
   - Compute tile dimensions: $\text{tile\_width} = \lceil \text{width} / \text{tile\_size} \rceil$, $\text{tile\_height} = \lceil \text{height} / \text{tile\_size} \rceil$.
   - Call `isect_tiles(means2d, radii, depths, tile_size, tile_width, tile_height, sort=True)` to get `tiles_per_gauss`, `isect_ids`, and `flatten_ids` [S1, unofficial].

5. **Offset Encoding:** Encode intersection offsets using `isect_offset_encode(isect_ids, C, tile_width, tile_height)` to get `isect_offsets` $[C, \text{tile\_height}, \text{tile\_width}]$ [S1, unofficial].

6. **Rasterization:** Call `rasterize_to_pixels_2dgs` with `means2d`, `ray_transforms`, `colors_for_raster`, `normals`, `opacities_cn`, `width`, `height`, `tile_size`, `isect_offsets`, `flatten_ids`, and `backgrounds`. This function performs the per-pixel ray-surfel intersection and accumulation [S1, unofficial].

**Per-Pixel Ray-Surfel Intersection (Inside `rasterize_to_pixels_2dgs`):**
For each surfel `gid` in depth order:
1. Compute ray-surfel intersection vectors using the projection matrix $M_g$ (from `ray_transforms`):
   $$h_u = -M_g[0, :3] + M_g[2, :3] \cdot p_x$$
   $$h_v = -M_g[1, :3] + M_g[2, :3] \cdot p_y$$
   where $p_x, p_y$ are pixel coordinates [S1, unofficial].
2. Compute cross product and guard against degenerate case:
   $$tmp = \text{cross}(h_u, h_v)$$
   If $|tmp[2]| < 10^{-10}$, skip this intersection [S1, unofficial].
3. Compute intersection coordinates:
   $$u_s = tmp[0] / tmp[2]$$
   $$v_s = tmp[1] / tmp[2]$$
4. Compute 3D sigma from intersection:
   $$\sigma_{3d} = u_s^2 + v_s^2$$
5. Compute 2D fallback sigma from distance to center:
   $$\sigma_{2d} = 2.0 \cdot (dx^2 + dy^2)$$
   where $dx, dy$ are distances from pixel to surfel center [S1, unofficial].
6. Take tighter bound:
   $$\sigma = 0.5 \cdot \min(\sigma_{3d}, \sigma_{2d})$$
7. Compute alpha clamped to `MAX_ALPHA`:
   $$\alpha = \min(\text{opacity} \cdot \exp(-\sigma), \text{MAX\_ALPHA})$$
   Negative sigma values are skipped to prevent invalid alpha computation [S1, unofficial].
8. Accumulate color, alpha, and normal maps via front-to-back alpha compositing [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import math

def rasterize_to_pixels_2dgs(
    means2d: np.ndarray,      # [C, N, 2]
    ray_transforms: np.ndarray, # [C, N, 3, 3]
    colors_for_raster: np.ndarray, # [C, N, 3]
    normals: np.ndarray,      # [C, N, 3]
    opacities_cn: np.ndarray, # [C, N]
    width: int,
    height: int,
    tile_size: int = 16,
    isect_offsets: np.ndarray, # [C, tile_h, tile_w]
    flatten_ids: np.ndarray,   # [C, num_intersecting]
    backgrounds: np.ndarray,   # [C, 3]
) -> tuple:
    """
    Rasterize 2D Gaussian surfels to pixels.
    Returns: (render_colors, render_alphas, render_normals)
    """
    C = means2d.shape[0]
    H, W = height, width
    tile_h = math.ceil(H / tile_size)
    tile_w = math.ceil(W / tile_size)
    
    render_colors = np.zeros((C, H, W, 3), dtype=np.float32)
    render_alphas = np.zeros((C, H, W, 1), dtype=np.float32)
    render_normals = np.zeros((C, H, W, 3), dtype=np.float32)
    
    # Initialize compositing buffers
    acc_alpha = np.zeros((C, H, W, 1), dtype=np.float32)
    acc_color = np.zeros((C, H, W, 3), dtype=np.float32)
    acc_normal = np.zeros((C, H, W, 3), dtype=np.float32)
    
    # Process each camera
    for c in range(C):
        # Get surfels intersecting this camera's view
        # flatten_ids[c] contains indices into the N surfels
        surfel_indices = flatten_ids[c]
        num_surfels = len(surfel_indices)
        
        # Sort surfels by depth for front-to-back rendering
        # depths are assumed to be available or passed; here we assume means2d depth is implicit or passed separately
        # For simplicity, we assume surfels are pre-sorted by depth in flatten_ids or we sort them here
        # In the reference, isect_tiles returns sorted ids
        for i in range(num_surfels):
            gid = surfel_indices[i]
            
            # Get surfel properties
            m2d = means2d[c, gid]
            rt = ray_transforms[c, gid]
            color = colors_for_raster[c, gid]
            normal = normals[c, gid]
            opacity = opacities_cn[c, gid]
            
            # Determine which tiles this surfel covers
            # This is handled by isect_offsets and flatten_ids structure
            # In the reference, we iterate over pixels covered by the surfel's tile
            # For a complete implementation, we would iterate over the bounding box of the surfel in tile space
            
            # Simplified: Iterate over all pixels (for reference correctness, though inefficient)
            # In practice, we only iterate over pixels within the surfel's radius
            radius = radii[c, gid] # Assume radii is passed or computed
            min_x = max(0, int(m2d[0] - radius))
            max_x = min(W, int(m2d[0] + radius))
            min_y = max(0, int(m2d[1] - radius))
            max_y = min(H, int(m2d[1] + radius))
            
            for y in range(min_y, max_y):
                for x in range(min_x, max_x):
                    # Compute ray-surfel intersection
                    # h_u = -Mg[0, :3] + Mg[2, :3] * px
                    # h_v = -Mg[1, :3] + Mg[2, :3] * py
                    px = float(x)
                    py = float(y)
                    
                    h_u = -rt[0, :3] + rt[2, :3] * px
                    h_v = -rt[1, :3] + rt[2, :3] * py
                    
                    tmp = np.cross(h_u, h_v)
                    
                    if abs(tmp[2]) < 1e-10:
                        continue
                    
                    us = tmp[0] / tmp[2]
                    vs = tmp[1] / tmp[2]
                    
                    sigma_3d = us * us + vs * vs
                    
                    dx = x - m2d[0]
                    dy = y - m2d[1]
                    sigma_2d = 2.0 * (dx * dx + dy * dy)
                    
                    sigma = 0.5 * min(sigma_3d, sigma_2d)
                    
                    if sigma < 0:
                        continue
                        
                    alpha = min(opacity * np.exp(-sigma), 1.0) # MAX_ALPHA is typically 1.0
                    
                    # Front-to-back compositing
                    # C_out = C_in + C_surfel * alpha * (1 - alpha_in)
                    # A_out = A_in + alpha * (1 - alpha_in)
                    
                    alpha_in = acc_alpha[c, y, x, 0]
                    if alpha_in >= 1.0:
                        continue
                        
                    weight = alpha * (1.0 - alpha_in)
                    
                    acc_color[c, y, x] += color * weight
                    acc_normal[c, y, x] += normal * weight # Note: Normal compositing is complex, usually weighted by alpha
                    acc_alpha[c, y, x, 0] += weight
                    
                    if acc_alpha[c, y, x, 0] >= 1.0:
                        break # Optimization: if fully opaque, stop processing further surfels for this pixel
        
        # Apply background
        for y in range(H):
            for x in range(W):
                alpha_final = acc_alpha[c, y, x, 0]
                if alpha_final < 1.0:
                    bg = backgrounds[c]
                    render_colors[c, y, x] = acc_color[c, y, x] + bg * (1.0 - alpha_final)
                else:
                    render_colors[c, y, x] = acc_color[c, y, x]
                
                render_alphas[c, y, x, 0] = alpha_final
                render_normals[c, y, x] = acc_normal[c, y, x] / (alpha_final + 1e-8) # Normalize normals

    return render_colors, render_alphas, render_normals
```

## Subtopics

### [Per-Primitive Texture Mapping for Reflective Gaussian Splatting](8629-per-primitive-texture-mapping-for-reflective-gaussian-splatt.md)
Enhances 2D Gaussian Splatting by replacing single attribute values per primitive with spatially varying texture maps for material properties and normals. Unlike methods that assign constant attributes to each primitive, this approach decouples geometric representation from appearance, enabling high-frequency specular reconstruction and accurate reflections in reflective scenes without increasing the primitive count. It builds on [2D Gaussian Splatting Rasterization](4815-2d-gaussian-splatting-rasterization.md) by leveraging the closed-form ray-splat intersection to map screen pixels to local texture coordinates, allowing hardware-accelerated rendering via texture atlases.

### [ToF Gaussian Rasterization Forward Pass](9261-tof-gaussian-rasterization-forward-pass.md)
This method implements the forward pass of the C++/CUDA rasterizer for Time-of-Flight Gaussian Splatting. It restructures scene and camera parameters into a single tuple, invokes the CUDA kernel, and returns rendered maps including color, phasor, depth, and auxiliary buffers.

### [Vanilla 2D Gaussian Splatting Rendering](12208-vanilla-2d-gaussian-splatting-rendering.md)
Renders a scene represented by 2D Gaussian primitives into RGB, alpha, normal, and depth maps using a custom rasterizer. It differs from standard 3D Gaussian Splatting by operating on surfels (2D Gaussians) projected into screen space, accumulating color and geometric attributes via front-to-back alpha compositing [Parent].

### [Nexels](13877-nexels.md)
Nexels is a novel view synthesis method that decouples geometry and appearance to achieve a compact representation using a fixed number of primitives per pixel. It uses surfels with a generalized Gaussian kernel for geometry and a shared neural field (Instant-NGP) for view-dependent appearance, rendered via a two-pass process to handle sparse primitives efficiently. Unlike sibling methods that couple appearance to primitives or use slower neural field lookups, Nexels textures only the top-$K$ primitives per pixel, ensuring real-time performance while matching the perceptual quality of 3D Gaussian Splatting with significantly fewer primitives.

## Operating characteristics
- **Time Complexity:** $O(N \cdot H \cdot W)$ in the naive reference implementation, where $N$ is the number of surfels and $H, W$ are image dimensions. Tile-based optimization reduces this to $O(K \cdot H \cdot W)$ where $K$ is the average number of surfels per tile.
- **Memory Complexity:** $O(N + H \cdot W)$ for storing surfel properties and output buffers.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Provides a clear, differentiable (via MLX autodiff in the reference) reference implementation for 2D Gaussian Splatting. Handles multiple camera models (pinhole, fisheye, orthographic) [S1, unofficial].
- **Limitations:** The NumPy reference implementation is not differentiable and is slow for large scenes. The CUDA implementation is faster but not detailed in the sources [S1, unofficial].
- **When to use:** For debugging, testing, and educational purposes. For production, use the CUDA implementation.
- **When not to use:** When real-time performance is required and the CUDA implementation is not available.

## Background
The method is part of the 2D Gaussian Splatting framework, which extends 3D Gaussian Splatting by using surfels (2D Gaussians) instead of 3D ellipsoids. The reference implementation is provided by the `gsplat-mlx` library [S1, unofficial].

## Sources
[S1] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
