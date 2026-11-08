# EWA Splatting with kNN Covariance Estimation

Also called: Elliptical Weighted Average splatting

## Overview
EWA Splatting with kNN Covariance Estimation is a point-cloud rendering method that projects 3D points onto 2D image planes using elliptical footprints derived from local neighborhood statistics [S1, unofficial]. Unlike simple disk splatting, it estimates a 3D covariance matrix for each point using its $k$-nearest neighbors to account for local surface orientation and density, resulting in denser, less noisy coverage [S1, unofficial]. It builds on [Splatting](23-splatting.md) by replacing isotropic kernels with anisotropic EWA shapes determined by data-driven covariance [S1, unofficial].

## How it works
**Inputs:**
- Point cloud: positions $\mathbf{X} \in \mathbb{R}^{N \times 3}$, colors $\mathbf{C} \in \mathbb{R}^{N \times 3}$ (normalized to $[0,1]$), and semantic labels $\mathbf{L} \in \mathbb{Z}^N$.
- Camera batch: $M$ cameras, each with intrinsic matrix $\mathbf{K} \in \mathbb{R}^{3 \times 3}$ (encoded as $[f_x, f_y, c_x, c_y]$) and extrinsic matrix $\mathbf{T}_{wc} \in \mathbb{R}^{4 \times 4}$ (world-to-camera).
- Viewport dimensions: width $W$, height $H$.
- Parameter $k$: number of neighbors for covariance estimation (default 16) [S1, unofficial].
- Parameter $r_{max}$: maximum integer radius for splatting tiles (clamped to 32) [S1, unofficial].

**Outputs:**
- Color image: $\mathbf{I}_{color} \in \mathbb{Z}^{H \times W \times 3}$ (uint8).
- Label image: $\mathbf{I}_{label} \in \mathbb{Z}^{H \times W}$ (uint8).
- Index map: $\mathbf{I}_{idx} \in \mathbb{Z}^{H \times W}$ (int32, initialized to -1).

**Data Structures:**
- KDTree: Built on $\mathbf{X}$ for $O(\log N)$ neighbor queries [S1, unofficial].
- Covariance Matrix: $\mathbf{\Sigma}_3 \in \mathbb{R}^{3 \times 3}$ per point, computed from neighbor deviations [S1, unofficial].
- Conic Matrix: $\mathbf{C} \in \mathbb{R}^{3 \times 3}$ per point in screen space, derived from $\mathbf{\Sigma}_3$ and camera projection [S1, unofficial].

**Steps:**

1.  **Neighbor Query:** Convert $\mathbf{X}$ to a numpy array and build a KDTree. Query the $k$-nearest neighbors for all $N$ points to obtain indices $\mathbf{I}_{knn} \in \mathbb{Z}^{N \times k}$ [S1, unofficial].
2.  **Covariance Estimation:** Gather the 3D coordinates of the $k$ neighbors for each point $\mathbf{X}_{padded} \in \mathbb{R}^{N \times k \times 3}$. Compute the 3D covariance matrix $\mathbf{\Sigma}_3$ for each point using the positions of its neighbors [S1, unofficial].
3.  **EWA Projection:** For each camera view $j \in [0, M-1]$, project the 3D covariance $\mathbf{\Sigma}_3$ to 2D screen space using the camera intrinsics $\mathbf{K}_j$, extrinsics $\mathbf{T}_{wc,j}$, and point positions. This yields the conic matrix $\mathbf{C}_j$, the splat radius $r_j$, and visibility flags $\mathbf{V}_j$ [S1, unofficial].
4.  **Visibility Filtering:** Filter points where the visibility flag is true and depth $d > 0$. If no points are visible, return zero-filled canvases with $\mathbf{I}_{idx}$ set to -1 [S1, unofficial].
5.  **Coordinate Transformation:** Calculate pixel coordinates $(u, v)$ from the projected coordinates. Flip the Y-axis to match image space: $y_{pix} = H - 1 - \text{round}(v)$ [S1, unofficial].
6.  **Radius Calculation:** Compute the integer radius $r_{int} = \text{clamp}(\text{round}(r), 0, 32)$ [S1, unofficial].
7.  **Sorting:** Sort visible points by depth $d$ in ascending order (front-to-back) [S1, unofficial].
8.  **Rasterization:** Iterate through sorted points. For each point, iterate $dy \in [-r_{int}, r_{int}]$ and $dx \in [-r_{int}, r_{int}]$. If the pixel $(x+dx, y+dy)$ is within bounds and the current point's depth is less than the existing depth in the Z-buffer, write the color, label, and point index to the output canvases [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
from scipy.spatial import KDTree

def ewa_splatting_knn(xyz, rgb, label, cameras, W, H, k=16, r_max=32):
    """
    xyz: [N, 3] float32
    rgb: [N, 3] float32 (0-1)
    label: [N] int64
    cameras: list of dicts with 'K' (3x3), 'T_wc' (4x4)
    W, H: int
    k: int
    r_max: int
    """
    N = xyz.shape[0]
    M = len(cameras)
    
    # Step 1: Build KDTree and query neighbors
    xyz_np = xyz.cpu().numpy() if isinstance(xyz, torch.Tensor) else xyz
    kdt = KDTree(xyz_np)
    _, knn_idx = kdt.query(xyz_np, k=k) # [N, k]
    
    # Step 2: Compute 3D Covariance
    # Gather neighbors
    xyz_padded = xyz_np[knn_idx] # [N, k, 3]
    # Compute mean
    mean = np.mean(xyz_padded, axis=1, keepdims=True) # [N, 1, 3]
    # Compute covariance
    diff = xyz_padded - mean # [N, k, 3]
    cov3d = np.einsum('nki,nkj->nij', diff, diff) / (k - 1) # [N, 3, 3]
    
    # Initialize output canvases
    color_img = np.zeros((H, W, 3), dtype=np.uint8)
    label_img = np.zeros((H, W), dtype=np.uint8)
    index_map = np.full((H, W), -1, dtype=np.int32)
    depth_canvas = np.full((H, W), np.inf, dtype=np.float32)
    
    # Process each camera
    for j, cam in enumerate(cameras):
        K = cam['K']
        T_wc = cam['T_wc']
        
        # Step 3: Project to 2D (Simplified projection logic)
        # Transform points to camera space
        # xyz_cam = (T_wc[:3, :3] @ xyz.T + T_wc[:3, 3:4]).T # [N, 3]
        # Project
        # xyz_proj = K @ xyz_cam.T # [3, N]
        # depths = xyz_proj[2, :] # [N]
        # uv = xyz_proj[:2, :] / depths # [2, N]
        
        # Note: The source uses a batched CUDA kernel 'project_points'.
        # Here we simulate the result: uv [N, 2], depths [N]
        # We assume a helper function exists for the heavy lifting
        uv, depths = project_points_cuda(xyz, K, T_wc, W, H)
        
        # Step 4: Filter visible points
        visible = (depths > 0) & (uv[:, 0] >= 0) & (uv[:, 0] < W) & (uv[:, 1] >= 0) & (uv[:, 1] < H)
        
        if not np.any(visible):
            continue
            
        uv_vis = uv[visible]
        depths_vis = depths[visible]
        rgb_vis = rgb[visible]
        label_vis = label[visible]
        idx_vis = np.where(visible)[0]
        
        # Step 5: Y-flip
        py = H - 1 - np.round(uv_vis[:, 1]).astype(np.int32)
        px = np.round(uv_vis[:, 0]).astype(np.int32)
        
        # Step 6: Radius
        # In a real implementation, radius is derived from the conic matrix eigenvalues
        # Here we use a placeholder or fixed radius if not explicitly calculated from cov3d in this snippet
        # The source mentions 'ewa_project' returns radius. 
        # We assume radii are computed. For this pseudocode, we use a fixed radius or derived one.
        # Let's assume radii are computed and stored in 'radii_vis'
        radii_vis = compute_ewa_radius(cov3d[visible], K, T_wc) 
        
        r_int = np.clip(np.round(radii_vis).astype(np.int32), 0, r_max)
        
        # Step 7: Sort by depth
        sort_idx = np.argsort(depths_vis)
        px = px[sort_idx]
        py = py[sort_idx]
        depths_vis = depths_vis[sort_idx]
        rgb_vis = rgb_vis[sort_idx]
        label_vis = label_vis[sort_idx]
        idx_vis = idx_vis[sort_idx]
        r_int = r_int[sort_idx]
        
        # Step 8: Rasterize
        for i in range(len(px)):
            x, y = px[i], py[i]
            d = depths_vis[i]
            r = r_int[i]
            
            # Iterate over tile
            for dy in range(-r, r + 1):
                for dx in range(-r, r + 1):
                    x_tile, y_tile = x + dx, y + dy
                    if 0 <= x_tile < W and 0 <= y_tile < H:
                        if d < depth_canvas[y_tile, x_tile]:
                            depth_canvas[y_tile, x_tile] = d
                            color_img[y_tile, x_tile] = (rgb_vis[i] * 255).astype(np.uint8)
                            label_img[y_tile, x_tile] = label_vis[i]
                            index_map[y_tile, x_tile] = idx_vis[i]
                            
    return color_img, label_img, index_map

def project_points_cuda(xyz, K, T_wc, W, H):
    # Placeholder for the actual CUDA kernel call described in [S1, unofficial]
    # Returns uv [N, 2] and depths [N]
    pass

def compute_ewa_radius(cov3d, K, T_wc):
    # Placeholder for EWA projection logic
    # Projects 3D covariance to 2D conic and extracts radius
    pass
```

**Parameters:**
- $k$: Number of neighbors. Default 16 [S1, unofficial].
- $r_{max}$: Maximum splat radius. Default 32 [S1, unofficial].
- `extent`: Frustum culling margin. Default 1.3 (30% beyond viewport) [S1, unofficial].

**Edge Cases:**
- **Y-Axis Orientation:** World camera Y points up, but image v points down. The code flips Y by calculating $y_{pix} = H - 1 - \text{round}(v)$ [S1, unofficial].
- **Empty Visible Set:** If no points are visible, return zero-filled canvases with index map filled with -1 [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ for KDTree construction and neighbor queries. Projection is $O(N \cdot M)$. Rasterization is $O(N \cdot r^2)$ where $r$ is the average splat radius.
- **Memory Complexity:** $O(N \cdot k)$ for storing neighbor indices and covariance matrices. Output images are $O(W \cdot H)$.
- **Performance:** Enables real-time rendering on GPU via batched CUDA kernels [S1, unofficial].
- **Strengths:** No mesh reconstruction required; works directly on raw point clouds [S1, unofficial]. Denser coverage than disk splatting due to elliptical footprints [S1, unofficial].
- **Limitations:** Requires neighbor search which can be memory intensive for very large $N$. Quality depends on $k$ and local density.

## Background
Originating from the CamPoint point-cloud renderer, this method combines EWA splatting with k-nearest neighbor covariance estimation to render point clouds efficiently [S1, unofficial]. It is used in multi-view 2D rendering pipelines where explicit mesh reconstruction is undesirable [S1, unofficial].

## Sources
[S1] Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, https://github.com/Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, fetched 2026-10-28 (unofficial)
