# EWA Splatting with kNN Covariance Estimation

Also called: Elliptical Weighted Average splatting

## Overview
EWA Splatting with kNN Covariance Estimation is a rendering method that treats volumetric data as a set of point primitives, using per-point 3D covariance matrices estimated from k-nearest neighbours to project elliptical weighted average (EWA) shapes onto the 2D image plane [S1, unofficial]. It solves the problem of sparse coverage and aliasing in raw point cloud rendering by providing denser, smoother coverage than simple disk splatting, without requiring explicit mesh reconstruction [S1, unofficial]. It differs from standard [EWA Splatting](1069-ewa-splatting.md) by deriving the kernel shape from local point cloud geometry (kNN) rather than assuming a fixed or isotropic kernel, and from [Splat Radius Adjustment Based on Normal Angle](1264-splat-radius-adjustment-based-on-normal-angle.md) by using covariance-based elliptical footprints instead of angle-dependent radius scaling.

## How it works
**Inputs:**
- Point cloud: positions $\mathbf{xyz} \in \mathbb{R}^{N \times 3}$, colors $\mathbf{rgb} \in \mathbb{R}^{N \times 3}$ (normalized to $[0,1]$), and semantic labels $\mathbf{label} \in \mathbb{Z}^N$.
- Camera parameters: intrinsic matrices $\mathbf{K} \in \mathbb{R}^{M \times 4}$ (containing $f_x, f_y, c_x, c_y$), extrinsic matrices $\mathbf{E} \in \mathbb{R}^{M \times 4 \times 4}$ (world-to-camera), viewport width $W$, and height $H$.
- Parameters: $k$ (number of neighbours for covariance estimation, default 16), `splat_radius` (pixel radius for splatting, default 1), `extent` (frustum culling margin, default 1.3).

**Outputs:**
- `color_img_np`: $[H, W, 3]$ uint8 RGB image.
- `label_img_np`: $[H, W]$ uint8 label map.
- `index_map_np`: $[H, W]$ int32 map storing the 3D point index for each pixel (filled with -1 if no point covers the pixel).

**Data Structures:**
- KDTree: Built on $\mathbf{xyz}$ for $O(\log N)$ nearest neighbour queries.
- Covariance Matrix: For each point $i$, a $3 \times 3$ symmetric matrix $\mathbf{C}_i$ computed from its $k$ nearest neighbours.
- Conic Matrix: The 2D projection of $\mathbf{C}_i$, used to define the elliptical footprint.

**Steps:**

1.  **Build KDTree:** Convert $\mathbf{xyz}$ to a numpy array and construct a KDTree.
    $$ \text{xyz\_np} = \mathbf{xyz}.cpu().numpy() $$
    $$ \text{kdt} = \text{KDTree}(\text{xyz\_np}) $$

2.  **Query Neighbours:** For each point, find the indices of its $k$ nearest neighbours.
    $$ \_, \text{knn\_idx} = \text{kdt.query}(\text{xyz\_np}, k=k) $$
    Result shape: $[N, k]$.

3.  **Gather Neighbour Coordinates:** Retrieve the 3D coordinates of the $k$ neighbours for each point.
    $$ \text{xyz\_padded} = \mathbf{xyz}[\text{knn\_idx}] $$
    Result shape: $[N, k, 3]$.

4.  **Compute 3D Covariance:** Calculate the covariance matrix for each point based on its neighbours. Let $\mathbf{p}_j$ be the coordinates of the $j$-th neighbour of point $i$. The covariance $\mathbf{C}_i$ is:
    $$ \mathbf{C}_i = \frac{1}{k-1} \sum_{j=1}^{k} (\mathbf{p}_{i,j} - \bar{\mathbf{p}}_i)(\mathbf{p}_{i,j} - \bar{\mathbf{p}}_i)^T $$
    where $\bar{\mathbf{p}}_i$ is the mean position of the $k$ neighbours.
    Result shape: $[N, 3, 3]$.

5.  **Project to 2D (EWA):** For each camera view, project the 3D covariance to 2D screen space to obtain the conic matrix, radius, and tile information. This accounts for perspective distortion.
    $$ [\text{conic}, \text{radius}, \text{tile}] = \text{ewa\_project}(\mathbf{xyz}_b, \mathbf{cov3d}_b, \mathbf{intr}_b, \mathbf{extr}_b, \mathbf{uv}_j, W, H, \text{vis}_j) $$
    The projection uses the camera intrinsics and extrinsics to map the 3D ellipsoid to a 2D ellipse.

6.  **Filter Visible Points:** Identify points that are visible and in front of the camera.
    $$ \text{visible} = (\text{vis\_idx} > 0) \land (\text{depths} > 0) $$
    If no points are visible, return zero-filled canvases with `index_map` filled with -1.

7.  **Calculate Pixel Coordinates and Radius:** Compute integer pixel coordinates $(px, py)$ and the integer splat radius $r_{int}$. The Y-axis is flipped to match image coordinates (world Y up, image Y down).
    $$ px = \lfloor uv_{vis}[:, 0] \rfloor $$
    $$ py = H - 1 - \lfloor uv_{vis}[:, 1] \rfloor $$
    $$ r_{int} = \text{clamp}(\text{radius\_vis}, 0, 32) $$

8.  **Sort by Depth:** Sort visible points by depth in ascending order (front-to-back).
    $$ d_{sorted}, \text{sort\_idx} = \text{sort}(d_{flat}) $$

9.  **Rasterize (Splat):** Iterate through points in depth order. For each point, iterate over the bounding box defined by $r_{int}$:
    $$ \text{for } dy \in [-r_{int}, r_{int}] $$
    $$ \text{for } dx \in [-r_{int}, r_{int}] $$
    Check if the pixel $(px+dx, py+dy)$ is within bounds and if the current point's depth $d_k$ is less than the existing depth in the z-buffer (`depth_canvas`). If so, write the color, label, and point index to the respective output arrays.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
from scipy.spatial import KDTree

def ewa_splatting_knn(xyz, rgb, label, intr, extr, W, H, k=16, splat_radius=1, extent=1.3):
    """
    Render point cloud using EWA splatting with kNN covariance estimation.
    
    Args:
        xyz: [N, 3] float32, world coordinates.
        rgb: [N, 3] float32, colors in [0, 1].
        label: [N] int64, semantic labels.
        intr: [M, 4] float32, intrinsics (fx, fy, cx, cy).
        extr: [M, 4, 4] float32, world-to-camera extrinsics.
        W: int, viewport width.
        H: int, viewport height.
        k: int, number of neighbours for covariance.
        splat_radius: int, base pixel radius.
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
    
    # Step 2: Query k-nearest neighbours
    _, knn_idx = kdt.query(xyz_np, k=k)
    
    # Step 3: Gather neighbour coordinates
    xyz_padded = xyz[knn_idx] # [N, k, 3]
    
    # Step 4: Compute 3D covariance for each point
    # Mean of neighbours
    mean_p = xyz_padded.mean(dim=1, keepdim=True) # [N, 1, 3]
    diff = xyz_padded - mean_p # [N, k, 3]
    # Covariance: sum of outer products / (k-1)
    cov3d = torch.bmm(diff.transpose(1, 2), diff) / (k - 1) # [N, 3, 3]
    
    # Step 5: Project to 2D (EWA)
    # This step typically involves projecting the 3D covariance to 2D screen space
    # using the camera matrices. For implementation, we assume a helper function
    # that computes the 2D conic matrix and radius.
    # uv: [N, 2, M], depths: [N, 1, M], visible: [N, 1, M]
    # Note: project_points and shelter_points are assumed to be available
    # as per the source code structure.
    
    # Simplified projection logic for clarity (assuming helper functions exist):
    # uv, depths = project_points(xyz, intr, extr, W, H, extent=extent)
    # visible = shelter_points(uv, depths, W, H)
    
    # For the purpose of this article, we describe the logical flow:
    # 1. Project all points to all cameras.
    # 2. Apply Z-buffer occlusion to determine visibility.
    # 3. Compute 2D footprint parameters (conic, radius) for visible points.
    
    # Step 6: Filter visible points
    # Assume vis_idx, depths, visible are computed from projection/occlusion
    # vis_idx: [N, M] indices of visible cameras
    # depths: [N, M] depth values
    # visible: [N, M] boolean mask
    
    # Step 7: Calculate pixel coordinates and radius
    # Flip Y axis: py = H - 1 - round(uv[:, 1])
    # r_int = clamp(radius, 0, 32)
    
    # Step 8: Sort visible points by depth
    # d_sorted, sort_idx = torch.sort(depths)
    
    # Step 9: Rasterize
    # Initialize canvases
    color_img = np.zeros((H, W, 3), dtype=np.uint8)
    label_img = np.zeros((H, W), dtype=np.uint8)
    index_map = np.full((H, W), -1, dtype=np.int32)
    depth_canvas = np.full((H, W), np.inf, dtype=np.float32)
    
    # Iterate through sorted visible points
    for point_idx in sort_idx:
        if not visible[point_idx]:
            continue
            
        # Get pixel coordinates
        px = int(round(uv[point_idx, 0]))
        py = H - 1 - int(round(uv[point_idx, 1]))
        
        # Get radius
        r_int = min(max(int(radius[point_idx]), 0), 32)
        
        # Iterate over footprint
        for dy in range(-r_int, r_int + 1):
            for dx in range(-r_int, r_int + 1):
                x, y = px + dx, py + dy
                if 0 <= x < W and 0 <= y < H:
                    if depths[point_idx] < depth_canvas[y, x]:
                        depth_canvas[y, x] = depths[point_idx]
                        color_img[y, x] = (rgb[point_idx] * 255).astype(np.uint8)
                        label_img[y, x] = label[point_idx]
                        index_map[y, x] = point_idx
                        
    return color_img, label_img, index_map
```

**Edge Cases and Pitfalls:**
- **Y-axis Orientation:** World camera Y points up, but image v points down. The code must flip Y by calculating $py = H - 1 - \lfloor uv_{vis}[:, 1] \rfloor$ [S1, unofficial].
- **Empty Visible Set:** If no points are visible (`vis_idx.numel() == 0`), return zero-filled canvases with `index_map` filled with -1 [S1, unofficial].
- **Radius Clamping:** The integer radius $r_{int}$ must be clamped between 0 and 32 to prevent excessive memory access and ensure bounded footprint [S1, unofficial].
- **Precision:** Use float32 for coordinates and depths to maintain precision during projection and comparison [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Building the KDTree is $O(N \log N)$. Querying neighbours is $O(N k \log N)$. Covariance computation is $O(N k)$. Projection and rasterization are $O(N \cdot M \cdot r^2)$, where $r$ is the average splat radius.
- **Memory Complexity:** $O(N k)$ for storing neighbour indices and coordinates. $O(N \cdot M)$ for storing projection results.
- **Performance:** The method is hardware-accelerated via GPU rasterization, enabling real-time interactivity [S1, unofficial]. Performance scales with the number of points $N$, cameras $M$, and splat radius $r$.
- **Strengths:** Provides denser, smoother coverage than simple disk splatting. No mesh reconstruction required. Works directly on raw point clouds.
- **Limitations:** Computationally more expensive than simple splatting due to covariance estimation. Sensitive to the choice of $k$ and splat radius.

## Background
EWA Splatting with kNN Covariance Estimation is a modern approach to point cloud rendering, leveraging the efficiency of splatting with the geometric accuracy of local covariance estimation. It is used in applications requiring real-time visualization of large-scale point clouds, such as autonomous driving simulations and 3D reconstruction pipelines [S1, unofficial]. The method builds on the principles of EWA splatting, which was originally introduced for volume rendering, and adapts it to point clouds by estimating the local surface orientation and scale from k-nearest neighbours.

## Sources
[S1] Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, https://github.com/Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, fetched 2026-10-28 (unofficial)
