# EWA Splatting with kNN Covariance Estimation

Also called: Elliptical Weighted Average splatting

## Overview
A rendering method that uses per-point 3D covariance estimated from k-nearest neighbours to project an elliptical weighted average (EWA) shape onto the 2D image plane for denser coverage than simple disk splatting [S1, unofficial]. It works directly on raw point clouds without mesh reconstruction, addressing the need for high-quality, dense point-cloud rendering in multi-view pipelines [S1, unofficial].

## How it works
**Inputs:**
- Point cloud: positions `xyz` $[N, 3]$, colors `rgb` $[N, 3]$, labels `label` $[N]$.
- Cameras: intrinsics `intr` $[M, 4]$ (fx, fy, cx, cy), extrinsics `extr` $[M, 4, 4]$ (world-to-camera), viewport width $W$, height $H$.
- Parameters: $k$ (number of neighbours, default 16), `splat_radius` (pixel radius, default 1), `extent` (frustum culling margin, default 1.3) [S1, unofficial].

**Outputs:**
- `color_img_np`: $[H, W, 3]$ uint8.
- `label_img_np`: $[H, W]$ uint8.
- `index_map_np`: $[H, W]$ int32.

**Data Structures:**
- KDTree built from `xyz` for nearest-neighbour queries.
- 3D covariance matrix `cov3d` $[N, 3, 3]$ per point.
- 2D conic matrix, radius, and tile information per visible point.
- Depth canvas for Z-buffer occlusion.

**Steps:**

1. **Build KDTree:** Convert `xyz` to numpy and build a KDTree:
   ```python
   xyz_np = self.xyz.cpu().numpy()
   kdt = KDTree(xyz_np)
   ```
   [S1, unofficial]

2. **Query k-NN:** Query $k$-nearest neighbours for all points to get indices:
   ```python
   _, knn_idx = kdt.query(xyz_np, k=k)
   ```
   Result shape: $[N, k]$ [S1, unofficial].

3. **Gather Neighbours:** Gather 3D coordinates of the $k$-nearest neighbours for each point:
   ```python
   xyz_padded = self.xyz[knn_idx]
   ```
   Result shape: $[N, k, 3]$ [S1, unofficial].

4. **Compute 3D Covariance:** Compute the 3D covariance matrix for each point using the neighbour positions:
   ```python
   cov3d = compute_cov3d(xyz_padded)
   ```
   Result shape: $[N, 3, 3]$ [S1, unofficial].

5. **Project to 2D:** For each camera view, project the 3D covariance to 2D using `ewa_project` to obtain the conic matrix, radius, and tile information. This involves projecting points and applying Z-buffer occlusion via `shelter_points` [S1, unofficial].

6. **Filter Visible Points:** In `_assemble_view_ewa`, filter visible points where `visible.squeeze(-1) > 0` and `depths.squeeze(-1) > 0` [S1, unofficial].

7. **Calculate Pixel Coordinates:** Calculate pixel coordinates `px` and `py` (flipped Y axis) and integer radius `r_int` clamped between 0 and 32:
   ```python
   r_int = radius_vis.long().clamp(0, 32)
   py = H - 1 - torch.round(uv_vis[:, 1]).long()
   ```
   [S1, unofficial]

8. **Sort by Depth:** Sort visible points by depth ascending:
   ```python
   d_sorted, sort_idx = torch.sort(d_flat)
   ```
   [S1, unofficial]

9. **Splat Points:** Iterate through points in depth order. For each point, iterate `dy` from `-r_k` to `r_k` and `dx` from `-r_k` to `r_k`. If the pixel is within bounds and the current depth `d_k` is less than the existing `depth_canvas`, write the color, label, and index [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
from sklearn.neighbors import KDTree

def ewa_splatting_knn(xyz, rgb, label, intr, extr, W, H, k=16, splat_radius=1, extent=1.3):
    """
    EWA Splatting with kNN Covariance Estimation.
    
    Args:
        xyz: [N, 3] float32, world coordinates.
        rgb: [N, 3] float32, colors (0-1).
        label: [N] int64, semantic labels.
        intr: [M, 4] float32, intrinsics (fx, fy, cx, cy).
        extr: [M, 4, 4] float32, world-to-camera extrinsics.
        W: int, viewport width.
        H: int, viewport height.
        k: int, number of neighbours for covariance estimation.
        splat_radius: int, pixel radius for point splatting.
        extent: float, frustum culling margin.
        
    Returns:
        color_img: [H, W, 3] uint8
        label_img: [H, W] uint8
        index_map: [H, W] int32
    """
    N = xyz.shape[0]
    M = intr.shape[0]
    
    # Step 1: Build KDTree
    xyz_np = xyz.cpu().numpy()
    kdt = KDTree(xyz_np)
    
    # Step 2: Query k-NN
    _, knn_idx = kdt.query(xyz_np, k=k)
    
    # Step 3: Gather Neighbours
    xyz_padded = xyz[knn_idx] # [N, k, 3]
    
    # Step 4: Compute 3D Covariance
    # compute_cov3d is assumed to be defined elsewhere or implemented as:
    # cov3d = mean((xyz_padded - mean(xyz_padded, dim=1)) @ (xyz_padded - mean(xyz_padded, dim=1)).transpose(1,2))
    cov3d = compute_cov3d(xyz_padded) # [N, 3, 3]
    
    # Initialize output canvases
    color_img = np.zeros((H, W, 3), dtype=np.uint8)
    label_img = np.zeros((H, W), dtype=np.uint8)
    index_map = np.full((H, W), -1, dtype=np.int32)
    
    # Step 5: Project to 2D and Filter Visible Points
    # This step involves projecting points and applying Z-buffer occlusion.
    # For simplicity, we assume project_points and shelter_points are available.
    # uv: [N, 2, M], depths: [N, 1, M], visible: [N, 1, M]
    uv, depths, visible = project_and_shelter(xyz, intr, extr, W, H, extent)
    
    # Step 6: Filter Visible Points
    # Flatten visible points for processing
    visible_flat = visible.squeeze(-1) > 0 # [N, M]
    depths_flat = depths.squeeze(-1) # [N, M]
    
    # Step 7: Calculate Pixel Coordinates and Radius
    # py = H - 1 - torch.round(uv_vis[:, 1]).long()
    # r_int = radius_vis.long().clamp(0, 32)
    
    # Step 8: Sort by Depth
    # d_sorted, sort_idx = torch.sort(d_flat)
    
    # Step 9: Splat Points
    # Iterate through points in depth order and splat onto canvases
    for m in range(M):
        # Get visible points for camera m
        vis_mask = visible_flat[:, m]
        if not torch.any(vis_mask):
            continue
            
        vis_idx = torch.where(vis_mask)[0]
        vis_uv = uv[vis_idx, :, m] # [N_vis, 2]
        vis_depths = depths_flat[vis_idx, m] # [N_vis]
        vis_rgb = rgb[vis_idx] # [N_vis, 3]
        vis_labels = label[vis_idx] # [N_vis]
        
        # Sort by depth
        sorted_idx = torch.argsort(vis_depths)
        vis_idx_sorted = vis_idx[sorted_idx]
        vis_uv_sorted = vis_uv[sorted_idx]
        vis_depths_sorted = vis_depths[sorted_idx]
        vis_rgb_sorted = vis_rgb[sorted_idx]
        vis_labels_sorted = vis_labels[sorted_idx]
        
        # Initialize depth canvas for this view
        depth_canvas = np.full((H, W), np.inf, dtype=np.float32)
        
        # Splat points
        for i in range(len(vis_idx_sorted)):
            idx = vis_idx_sorted[i].item()
            uv_pixel = vis_uv_sorted[i].cpu().numpy()
            depth = vis_depths_sorted[i].item()
            rgb_val = vis_rgb_sorted[i].cpu().numpy()
            label_val = vis_labels_sorted[i].item()
            
            # Calculate pixel coordinates
            px = int(np.round(uv_pixel[0]))
            py = H - 1 - int(np.round(uv_pixel[1]))
            
            # Calculate radius
            # r_int = radius_vis.long().clamp(0, 32)
            # Assuming radius is derived from cov3d or splat_radius
            r_int = min(splat_radius, 32)
            
            # Iterate over splat area
            for dy in range(-r_int, r_int + 1):
                for dx in range(-r_int, r_int + 1):
                    y = py + dy
                    x = px + dx
                    
                    if 0 <= y < H and 0 <= x < W:
                        if depth < depth_canvas[y, x]:
                            depth_canvas[y, x] = depth
                            color_img[y, x] = (rgb_val * 255).astype(np.uint8)
                            label_img[y, x] = label_val
                            index_map[y, x] = idx
                            
    return color_img, label_img, index_map

def compute_cov3d(xyz_padded):
    """
    Compute 3D covariance matrix for each point using its k-nearest neighbours.
    
    Args:
        xyz_padded: [N, k, 3]
        
    Returns:
        cov3d: [N, 3, 3]
    """
    N, k, _ = xyz_padded.shape
    # Compute mean for each point
    mean = torch.mean(xyz_padded, dim=1, keepdim=True) # [N, 1, 3]
    # Compute deviations
    deviations = xyz_padded - mean # [N, k, 3]
    # Compute covariance
    # cov3d = (deviations.transpose(1, 2) @ deviations) / (k - 1)
    cov3d = torch.bmm(deviations.transpose(1, 2), deviations) / (k - 1)
    return cov3d

def project_and_shelter(xyz, intr, extr, W, H, extent):
    """
    Project points and apply Z-buffer occlusion.
    
    Args:
        xyz: [N, 3]
        intr: [M, 4]
        extr: [M, 4, 4]
        W: int
        H: int
        extent: float
        
    Returns:
        uv: [N, 2, M]
        depths: [N, 1, M]
        visible: [N, 1, M]
    """
    # This function is assumed to be implemented using project_points and shelter_points
    # from the CamPoint repo or similar.
    # For the purpose of this article, we assume it returns the required tensors.
    pass
```

**Edge Cases and Pitfalls:**
- **Y-axis orientation:** World camera Y points up, but image v points down. The code flips Y by calculating `py = H - 1 - torch.round(uv_vis[:, 1]).long()` [S1, unofficial].
- **Empty visible set:** If no points are visible (`vis_idx.numel() == 0`), return zero-filled canvases with index map filled with -1 [S1, unofficial].
- **Radius clamping:** Integer radius `r_int` is clamped between 0 and 32 to prevent excessive memory usage and processing time [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by KDTree construction $O(N \log N)$, k-NN query $O(N k \log N)$, and splatting $O(N_{vis} \cdot r^2)$, where $N_{vis}$ is the number of visible points and $r$ is the splat radius.
- **Memory Complexity:** $O(N k)$ for storing k-NN indices and $O(N)$ for covariance matrices. Output images are $O(W \cdot H)$.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with the number of points $N$ and cameras $M$. Performance may degrade with large $k$ or large splat radii.
- **Strengths:** Dense coverage, no mesh reconstruction required, works directly on raw point clouds.
- **Limitations:** Computationally intensive for large point clouds, sensitive to $k$ and splat radius parameters.
- **When to use:** When high-quality, dense point-cloud rendering is needed without mesh reconstruction.
- **When not to use:** When computational resources are limited or real-time performance is critical.

## Background
Originated in the context of point-cloud rendering, specifically in the CamPoint repository and related multi-view 2D rendering pipelines [S1, unofficial]. It builds on the EWA splatting technique, adapting it for point clouds by estimating covariance from k-nearest neighbours.

## Sources
[S1] Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, https://github.com/Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, fetched 2026-10-28 (unofficial)
