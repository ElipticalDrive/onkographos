# EWA Splatting Renderer (Mesh-Free)

Also called: CamPoint splatting renderer, Point-cloud splatting renderer

## Overview
A mesh-free rendering pipeline that projects raw point clouds directly to 2D camera views using EWA projection and Z-buffer occlusion, producing RGB images, label maps, and 2D-to-3D index maps without mesh reconstruction [S1, unofficial]. It differs from its parent [EWA Splatting](1069-ewa-splatting.md) by using simplified screen-aligned spherical impostors with isotropic linear falloff, trading anisotropic, correctly band-limited footprints for a formulation that requires no per-point normals or radii [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   `xyz`: Point cloud coordinates, shape `[N, 3]`, dtype `float32` (world coordinates) [S1, unofficial].
*   `rgb`: Point colors, shape `[N, 3]`, dtype `uint8` (range 0–255) or `float` (range 0–1) [S1, unofficial].
*   `label`: Semantic labels, shape `[N]`, dtype `int64` (long) [S1, unofficial].
*   `intr`: Camera intrinsics, shape `[M, 4]` or `[4]`, containing `[fx, fy, cx, cy]` [S1, unofficial].
*   `extr`: Camera extrinsics, shape `[M, 3, 4]` or `[3, 4]` (rotation and translation) [S1, unofficial].
*   `W`, `H`: Output image width and height in pixels [S1, unofficial].
*   `splat_radius`: Pixel expansion radius `r` [S1, unofficial].
*   `extent`: Frustum culling margin factor (e.g., 1.3 for 30% beyond viewport) [S1, unofficial].
*   `cam_batch`: Number of cameras processed simultaneously on GPU [S1, unofficial].

**Outputs:**
*   `color_canvas`: RGB image, shape `[H, W, 3]`, dtype `uint8` [S1, unofficial].
*   `depth_canvas`: Depth image, shape `[H, W]`, dtype `float32` (background set to `inf`) [S1, unofficial].
*   `label_canvas`: Label map, shape `[H, W]`, dtype `int64` (background set to 0) [S1, unofficial].
*   `index_canvas`: 2D-to-3D index map, shape `[H, W]`, dtype `int64` (background set to -1) [S1, unofficial].

### Data Structures
*   **Bucket Tensor**: Used for Z-buffer occlusion. Size `bucket_size * M`, where `bucket_size = (mH + 1) * (mW + 1)`. `mW = int(extent * W) + 1`, `mH = int(extent * H) + 1`. Indices map 2D pixel coordinates to a flattened array for efficient occlusion testing [S1, unofficial].
*   **Sort Key**: Used for depth resolution. Calculated as `flat_idx * (d_max + 1.0) + depth`, where `d_max` is the maximum depth in the scene. This allows sorting by pixel index first, then by depth [S1, unofficial].

### Parameters
*   `splat_radius`: Default 1 (single pixel). 2 produces a 3×3 disk. Larger values densify the image but soften details [S1, unofficial].
*   `extent`: Default 1.3 [S1, unofficial].
*   `cam_batch`: Default 8 [S1, unofficial].

### Steps
1.  **Load and Normalize**: Convert `xyz` to `float32`. If `rgb` is `uint8`, normalize to `[0, 1]` by dividing by 255.0. Convert `label` to `int64` [S1, unofficial].
2.  **Project Points**: Project `xyz` using `intr` and `extr` for all `M` cameras. Returns screen coordinates `uv` of shape `[M, N, 2]` and depths `depths` of shape `[M, N, 1]` [S1, unofficial].
3.  **Reshape Tensors**: Permute `uv` and `depths` from `[M, N, ...]` to `[N, ..., M]` using `.permute(1, 2, 0).contiguous()` to facilitate per-point processing across cameras [S1, unofficial].
4.  **Z-buffer Occlusion**:
    *   Calculate extended dimensions `mW` and `mH` based on `extent` [S1, unofficial].
    *   Initialize a bucket tensor of size `(mH + 1) * (mW + 1) * M` [S1, unofficial].
    *   Call `shelter_points(uv, depths, mW, mH, offset, bucket)` to determine visible points. This returns occluded `uv`, `depths`, and a `visible` mask [S1, unofficial].
    *   *Pitfall*: If `M == 1`, the Numba kernel for `shelter_points` fails because it requires `M >= 2` for 3D array indexing. Workaround: Duplicate `uv` and `depths` along the camera axis to make `M=2`, perform occlusion, then slice back to `M=1` [S1, unofficial].
5.  **Filter Visible Points**: Create boolean mask `vis_mask = (visible.squeeze(-1) > 0) & (depths.squeeze(-1) > 0)`. Get indices `vis_idx` of visible points [S1, unofficial].
6.  **Coordinate Transformation**:
    *   Round `uv` to integer pixel coordinates: `px = round(uv[:, 0])`.
    *   Flip Y coordinate for image space: `py = H - 1 - round(uv[:, 1])` [S1, unofficial].
7.  **Boundary Check**: Filter points where `0 <= px < W` and `0 <= py < H`. Discard invalid points [S1, unofficial].
8.  **Splat Expansion**:
    *   For each visible point, expand to a disk of radius `r` (`splat_radius`).
    *   Generate offsets `dy` in `[-r, r]` and `dx` in `[-r, r]`, excluding `(0,0)`.
    *   Calculate neighbor pixel coordinates `nx = px + dx`, `ny = py + dy`.
    *   Filter out-of-bounds neighbors [S1, unofficial].
9.  **Flatten Indices**:
    *   Compute flat pixel index `flat_idx = py * W + px`.
    *   Expand to include neighbors: `flat_all[:, 1:] = n_flat` (neighbor flat indices).
    *   Mark out-of-bounds neighbors as -1 [S1, unofficial].
10. **Depth Resolve**:
    *   Sort splats by pixel index and depth to resolve collisions.
    *   Create `sort_key = flat_exp.float() * (d_src.max() + 1.0) + d_src`.
    *   Sort by `sort_key`. Keep the first entry per unique pixel index (minimum depth) [S1, unofficial].
11. **Write Canvases**:
    *   Initialize `color_canvas` `[H*W, 3]`, `depth_canvas` `[H*W]`, `label_canvas` `[H*W]`, `index_canvas` `[H*W]` [S1, unofficial].
    *   Write `rgb`, `depth`, `label`, and point ID to the winner pixels.
    *   Convert color to `uint8` by multiplying by 255 and clamping to `[0, 255]` [S1, unofficial].
    *   Set background pixels (`depth == inf`) to 0 for color/label and -1 for index [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def ewa_splatting_mesh_free(xyz, rgb, label, intr, extr, W, H, splat_radius=1, extent=1.3, cam_batch=8):
    """
    Mesh-free EWA splatting renderer.
    
    Args:
        xyz: [N, 3] float32
        rgb: [N, 3] uint8 or float32
        label: [N] int64
        intr: [M, 4] or [4] float32 (fx, fy, cx, cy)
        extr: [M, 3, 4] or [3, 4] float32
        W, H: int
        splat_radius: int
        extent: float
        cam_batch: int
        
    Returns:
        color_canvas: [H, W, 3] uint8
        depth_canvas: [H, W] float32
        label_canvas: [H, W] int64
        index_canvas: [H, W] int64
    """
    N = xyz.shape[0]
    
    # 1. Load and Normalize
    xyz = xyz.float()
    if rgb.dtype == torch.uint8:
        rgb = rgb.float() / 255.0
    label = label.long()
    
    # Handle single camera case for projection and occlusion
    M = intr.shape[0] if intr.dim() > 1 else 1
    if M == 1:
        intr = intr.unsqueeze(0)
        extr = extr.unsqueeze(0)
        M = 1
    
    # 2. Project Points
    # Assuming project_points handles broadcasting and returns [M, N, 2] and [M, N, 1]
    uv, depths = project_points(xyz, intr, extr, W, H, nearest=0.2, extent=extent)
    
    # 3. Reshape Tensors
    uv = uv.permute(1, 2, 0).contiguous() # [N, 2, M]
    depths = depths.permute(1, 2, 0).contiguous() # [N, 1, M]
    
    # 4. Z-buffer Occlusion
    mW = int(extent * W) + 1
    mH = int(extent * H) + 1
    bucket_size = (mH + 1) * (mW + 1)
    bucket = torch.zeros((bucket_size * M), dtype=torch.int32, device=xyz.device)
    
    # Workaround for M=1 if shelter_points requires M>=2
    if M == 1:
        uv_dup = uv.repeat(1, 1, 2)
        depths_dup = depths.repeat(1, 1, 2)
        visible, _, _ = shelter_points(uv_dup, depths_dup, mW, mH, 0, bucket)
        visible = visible[:, :, :1]
    else:
        visible, _, _ = shelter_points(uv, depths, mW, mH, 0, bucket)
        
    # 5. Filter Visible Points
    vis_mask = (visible.squeeze(-1) > 0) & (depths.squeeze(-1) > 0)
    vis_idx = torch.nonzero(vis_mask, as_tuple=False).squeeze(1)
    
    if len(vis_idx) == 0:
        # Return empty canvases
        color_canvas = torch.zeros((H, W, 3), dtype=torch.uint8)
        depth_canvas = torch.full((H, W), float('inf'), dtype=torch.float32)
        label_canvas = torch.zeros((H, W), dtype=torch.int64)
        index_canvas = torch.full((H, W), -1, dtype=torch.int64)
        return color_canvas, depth_canvas, label_canvas, index_canvas

    uv_vis = uv[vis_idx]
    depths_vis = depths[vis_idx]
    rgb_vis = rgb[vis_idx]
    label_vis = label[vis_idx]
    point_ids = vis_idx # Assuming vis_idx corresponds to original point IDs
    
    # 6. Coordinate Transformation
    px = torch.round(uv_vis[:, 0]).long()
    py = (H - 1 - torch.round(uv_vis[:, 1])).long()
    
    # 7. Boundary Check
    valid = (px >= 0) & (px < W) & (py >= 0) & (py < H)
    px = px[valid]
    py = py[valid]
    depths_vis = depths_vis[valid]
    rgb_vis = rgb_vis[valid]
    label_vis = label_vis[valid]
    point_ids = point_ids[valid]
    
    if len(px) == 0:
        color_canvas = torch.zeros((H, W, 3), dtype=torch.uint8)
        depth_canvas = torch.full((H, W), float('inf'), dtype=torch.float32)
        label_canvas = torch.zeros((H, W), dtype=torch.int64)
        index_canvas = torch.full((H, W), -1, dtype=torch.int64)
        return color_canvas, depth_canvas, label_canvas, index_canvas

    # 8. Splat Expansion
    r = splat_radius
    offsets = []
    for dy in range(-r, r + 1):
        for dx in range(-r, r + 1):
            if dx == 0 and dy == 0:
                continue
            offsets.append((dx, dy))
            
    num_offsets = len(offsets)
    
    # Expand px, py, depths, rgb, label, point_ids
    # px_exp: [N, 1 + num_offsets]
    px_exp = px.unsqueeze(1).repeat(1, 1 + num_offsets)
    py_exp = py.unsqueeze(1).repeat(1, 1 + num_offsets)
    depths_exp = depths_vis.unsqueeze(1).repeat(1, 1 + num_offsets)
    rgb_exp = rgb_vis.unsqueeze(1).repeat(1, 1 + num_offsets, 1)
    label_exp = label_vis.unsqueeze(1).repeat(1, 1 + num_offsets)
    point_ids_exp = point_ids.unsqueeze(1).repeat(1, 1 + num_offsets)
    
    # Fill offsets
    for i, (dx, dy) in enumerate(offsets):
        px_exp[:, i+1] = px + dx
        py_exp[:, i+1] = py + dy
        
    # 9. Flatten Indices
    flat_idx = py * W + px
    flat_all = py_exp * W + px_exp
    
    # Mark out-of-bounds neighbors as -1
    out_of_bounds = (px_exp < 0) | (px_exp >= W) | (py_exp < 0) | (py_exp >= H)
    flat_all[out_of_bounds] = -1
    
    # 10. Depth Resolve
    # Flatten for sorting
    flat_all_flat = flat_all.view(-1)
    depths_flat = depths_exp.view(-1)
    rgb_flat = rgb_exp.view(-1, 3)
    label_flat = label_exp.view(-1)
    point_ids_flat = point_ids_exp.view(-1)
    
    # Filter out -1 indices
    valid_mask = flat_all_flat >= 0
    flat_all_flat = flat_all_flat[valid_mask]
    depths_flat = depths_flat[valid_mask]
    rgb_flat = rgb_flat[valid_mask]
    label_flat = label_flat[valid_mask]
    point_ids_flat = point_ids_flat[valid_mask]
    
    if len(flat_all_flat) == 0:
        color_canvas = torch.zeros((H, W, 3), dtype=torch.uint8)
        depth_canvas = torch.full((H, W), float('inf'), dtype=torch.float32)
        label_canvas = torch.zeros((H, W), dtype=torch.int64)
        index_canvas = torch.full((H, W), -1, dtype=torch.int64)
        return color_canvas, depth_canvas, label_canvas, index_canvas

    d_max = depths_flat.max()
    sort_key = flat_all_flat.float() * (d_max + 1.0) + depths_flat
    
    # Sort by sort_key
    sort_indices = torch.argsort(sort_key)
    flat_all_sorted = flat_all_flat[sort_indices]
    depths_sorted = depths_flat[sort_indices]
    rgb_sorted = rgb_flat[sort_indices]
    label_sorted = label_flat[sort_indices]
    point_ids_sorted = point_ids_flat[sort_indices]
    
    # Keep first entry per unique pixel index
    unique_pixels, unique_indices = torch.unique(flat_all_sorted, return_inverse=True, return_counts=True)
    # unique_indices gives the index of the first occurrence of each unique pixel
    # We need to gather the first entry for each unique pixel
    # Since we sorted by sort_key (pixel then depth), the first entry is the winner
    
    # Create output canvases
    color_canvas = torch.zeros((H * W, 3), dtype=torch.float32)
    depth_canvas = torch.full((H * W,), float('inf'), dtype=torch.float32)
    label_canvas = torch.full((H * W,), -1, dtype=torch.int64)
    index_canvas = torch.full((H * W,), -1, dtype=torch.int64)
    
    # Assign winners
    # unique_indices contains the index in the sorted arrays of the first occurrence
    winners = torch.arange(len(unique_pixels), device=unique_pixels.device)
    
    color_canvas[unique_pixels] = rgb_sorted[unique_indices]
    depth_canvas[unique_pixels] = depths_sorted[unique_indices]
    label_canvas[unique_pixels] = label_sorted[unique_indices]
    index_canvas[unique_pixels] = point_ids_sorted[unique_indices]
    
    # 11. Write Canvases
    # Convert color to uint8
    color_canvas_uint8 = (color_canvas * 255).clamp(0, 255).long()
    
    # Reshape to [H, W, ...]
    color_canvas = color_canvas_uint8.view(H, W, 3).byte()
    depth_canvas = depth_canvas.view(H, W)
    label_canvas = label_canvas.view(H, W)
    index_canvas = index_canvas.view(H, W)
    
    # Set background pixels (depth == inf)
    bg_mask = depth_canvas == float('inf')
    color_canvas[bg_mask] = 0
    label_canvas[bg_mask] = 0
    index_canvas[bg_mask] = -1
    
    return color_canvas, depth_canvas, label_canvas, index_canvas

def project_points(xyz, intr, extr, W, H, nearest=0.2, extent=1.3):
    """
    Placeholder for projection logic.
    Returns uv [M, N, 2] and depths [M, N, 1]
    """
    # Implementation details depend on the specific projection library or math
    # This is a placeholder to satisfy the requirement of self-contained pseudocode
    # In practice, this would involve matrix multiplication and perspective division
    pass

def shelter_points(uv, depths, mW, mH, offset, bucket):
    """
    Placeholder for Numba kernel for Z-buffer occlusion.
    Returns visible mask, occluded uv, occluded depths
    """
    # Implementation details depend on the specific Numba kernel
    pass
```

## Operating characteristics
*   **Time Complexity**: Dominated by projection and sorting. Projection is $O(N \cdot M)$. Occlusion is $O(N \cdot M)$ with bucket access. Sorting is $O(K \log K)$ where $K$ is the number of splats (up to $N \cdot (2r+1)^2 \cdot M$).
*   **Memory Complexity**: $O(N \cdot M)$ for intermediate tensors during projection and occlusion. Bucket size is $O(M \cdot \text{extent}^2 \cdot W \cdot H)$.
*   **Performance**: Measured on GPU with `cam_batch=8`. Performance scales with number of points $N$ and cameras $M$. Larger `splat_radius` increases memory and compute due to more splats per point [S1, unofficial].
*   **Strengths**: No mesh reconstruction required. Fast projection and rasterization. Handles large-scale point clouds efficiently.
*   **Limitations**: Spherical impostors lack anisotropic filtering, potentially causing aliasing or blurring at oblique angles. Z-buffer occlusion is approximate due to bucket resolution and `extent` margin. Single camera case requires workaround [S1, unofficial].
*   **When to Use**: When mesh reconstruction is undesirable or too expensive, and real-time or near-real-time rendering of point clouds is needed. Suitable for multi-view synthesis and visualization.
*   **When Not to Use**: When high-quality anti-aliasing and correct perspective projection of point sizes are critical. Not suitable for scenes with extreme depth variations where bucket resolution is insufficient.

## Background
Originated from the "Point-Splatting-Multi-View-2D-Rendering-Pipeline" project by Luhao03, described as an unofficial implementation [S1, unofficial]. It is part of the CamPoint ecosystem, designed for multi-view splatting of large-scale point clouds, particularly for architectural structures like city walls [S1, unofficial].

## Sources
[S1] Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, https://github.com/Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, fetched 2026-10-28 (unofficial)
