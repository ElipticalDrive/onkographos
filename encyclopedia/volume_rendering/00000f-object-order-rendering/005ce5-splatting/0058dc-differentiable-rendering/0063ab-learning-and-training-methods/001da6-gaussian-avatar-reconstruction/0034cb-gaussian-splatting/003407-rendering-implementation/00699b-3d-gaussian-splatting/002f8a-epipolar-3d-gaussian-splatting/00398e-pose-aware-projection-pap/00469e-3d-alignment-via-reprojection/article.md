# 3D Alignment via Reprojection

Also called: Feature Reprojection Alignment

## Overview
3D Alignment via Reprojection is a technique used in differentiable volume rendering to align features from neighboring training views with a novel rendered view, compensating for pose and geometry discrepancies. It solves the problem of leveraging relevant multi-view contextual information when camera orientations and positions differ significantly. Unlike its parent [Pose-Aware Projection (PAP)](8527-pose-aware-projection-pap.md), which aggregates features based on camera distance to resolve ambiguous assignments, this method explicitly reprojects 3D coordinates into the novel view using the pinhole camera model and validates them via depth-based visibility testing. It differs from sibling [Geometry Aware Feature Extraction](10791-geometry-aware-feature-extraction.md) by relying on explicit geometric reprojection rather than implicit geometric awareness.

## How it works
**Inputs:**
- Novel view image $I_k$ and its associated camera pose $C_k$ and intrinsic matrix $K_k$.
- Set of $n$ neighboring training images $\{H_i\}_{i=1}^n$ with their poses $C_i$ and intrinsics $K_i$.
- Rendered depth map $z_i$ from the NeRF model for the novel view (used for visibility checking).
- Leniency threshold $l$ (typical value: 0.25) [S1].

**Outputs:**
- Reprojected feature map $H^f_{i \rightarrow k}$ aligned with the novel view coordinates.

**Data Structures:**
- Image features $I^f_k$ and $H^f_i$: Dense tensors of spatial dimensions matching the input images, with a channel depth of 64 [S1].
- 3D coordinates $\bm{x}_k$: Represented as homogeneous vectors $[\bm{x}_k, 1]$ where $\bm{x}_k = (x_k, y_k, z_k)$, with $(x_k, y_k)$ being pixel coordinates and $z_k$ the depth [S1].

**Steps:**

1.  **Feature Extraction:** Extract 64-dimensional image features from the novel view $I_k$ and each neighboring image $H_i$ using separate small convolutional encoder blocks [S1]:
    $$
    I^f_k = \text{conv}_I(I_k)
    $$
    $$
    \{H^f_i\}_{i=1}^n = \text{conv}_H(\{H_i\}_{i=1}^n)
    $$

2.  **Coordinate Reprojection:** For each pixel in the novel view with 3D coordinate $\bm{x}_k = (x_k, y_k, z_k)$, reproject the coordinate into the neighboring camera view $i$ using the pinhole camera model [S1]:
    $$
    \bm{x}_{k \rightarrow i} = K_i C_i C_k^{-1} K_k^{-1} [\bm{x}_k, 1]
    $$
    Here, $\bm{x}_{k \rightarrow i} = (x_{k \rightarrow i}, y_{k \rightarrow i}, z_{k \rightarrow i}, 1)$ is the resulting homogeneous coordinate in camera $i$'s space. $K_i$ is the intrinsic matrix of camera $i$, and $C_i$ is the pose matrix of camera $i$. Note: The source formula uses $C_i C_k^{-1}$, implying $C_i$ is the pose of the target neighbor and $C_k^{-1}$ is the inverse pose of the source novel view. Standard pinhole projection usually involves $K [R|t]$. The source notation $C_i C_k^{-1}$ suggests $C$ represents the full projection matrix or the pose transformation. We follow the source notation exactly [S1].

3.  **Visibility Testing:** Calculate a visibility score $vis_i$ to ensure geometric consistency by comparing the reprojected depth $z_{k \rightarrow i}$ with the NeRF-computed depth $z_i$ at the corresponding location in the neighbor view. The source formula uses $z_i$ as the NeRF depth. To be precise, $z_i$ in the denominator refers to the depth of the neighbor view's surface at the projected location, or the NeRF depth at the novel view location? The source states: "comparing the reprojected depth value $z_{k \rightarrow i}$ with the depth value computed by NeRF, $z_i$". This implies $z_i$ is the NeRF depth at the novel view pixel $(x_k, y_k)$? Or the depth at the projected pixel? The formula is:
    $$
    vis_i = \mathcal{H}\left(1 - \frac{z_{k \rightarrow i}}{z_i} + l\right)
    $$
    Where $\mathcal{H}$ is the Heaviside function. $l$ is the leniency threshold (0.25) [S1].

4.  **Feature Copying:** Form the reprojected feature map by copying values from the neighbor's feature map $H^f_i$ at the projected coordinates $\bm{x}_{k \rightarrow i}$ to the original coordinates $\bm{x}_k$, weighted by the visibility score [S1]:
    $$
    H^f_{i \rightarrow k} \langle \phi(\bm{x}_k) \rangle = vis_i \times H^f_i \langle \phi(\bm{x}_{k \rightarrow i}) \rangle
    $$
    Where $\phi(\bm{x}_k) = (x_k/z_k, y_k/z_k)$ represents the 2D normalized coordinates, and $\langle \rangle$ denotes 2D coordinate indexing (sampling) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def reproject_features(novel_view_img, neighbor_imgs, novel_pose, neighbor_poses, 
                       novel_intrinsics, neighbor_intrinsics, nerf_depth_map, 
                       leniency_threshold=0.25):
    """
    Aligns neighbor features to the novel view via 3D reprojection.
    
    Args:
        novel_view_img: Tensor (H, W, 3) or (1, H, W, 3)
        neighbor_imgs: List of Tensors [(H, W, 3)]
        novel_pose: Tensor (4, 4) Camera pose matrix C_k
        neighbor_poses: List of Tensors [(4, 4)] Camera pose matrices C_i
        novel_intrinsics: Tensor (3, 3) Intrinsic matrix K_k
        neighbor_intrinsics: List of Tensors [(3, 3)] Intrinsic matrices K_i
        nerf_depth_map: Tensor (H, W) Depth map z_i from NeRF for novel view
        leniency_threshold: Float l
        
    Returns:
        reprojected_features: Tensor (H, W, 64) Aligned features
    """
    # 1. Feature Extraction
    # conv_I and conv_H are small convolutional blocks. 
    # Assuming they output 64-channel features.
    novel_features = conv_I(novel_view_img) # Shape: (1, 64, H, W)
    
    neighbor_features_list = []
    for i, (img, pose, K) in enumerate(zip(neighbor_imgs, neighbor_poses, neighbor_intrinsics)):
        h_features = conv_H(img) # Shape: (1, 64, H, W)
        neighbor_features_list.append((h_features, pose, K))
        
    # 2. Reprojection and Visibility
    # Get pixel coordinates
    H, W = novel_view_img.shape[0], novel_view_img.shape[1]
    y, x = torch.meshgrid(torch.arange(H), torch.arange(W), indexing='ij')
    
    # Normalize to camera coordinates (assuming principal point at center and focal length 1 for simplicity in indexing, 
    # but actual reprojection uses K)
    # The source uses phi(x_k) = (x_k/z_k, y_k/z_k) for indexing. 
    # This implies we need to map pixel (x,y) to normalized image coordinates.
    # Standard pinhole: x_img = K * X_cam. 
    # To go from pixel (u,v) to normalized coords (x,y): 
    # x = (u - cx) / fx, y = (v - cy) / fy.
    # However, the source formula for reprojection uses K^{-1} on homogeneous pixel coords [x_k, 1].
    # Let's assume x_k in the formula refers to the pixel coordinate vector (u, v, 1) or (x, y, z) in camera space?
    # The source says: "Given a 3D coordinate x_k = (x_k, y_k, z_k) in the novel camera view, where (x_k, y_k) are the pixel coordinates... and z_k is the depth".
    # This is slightly ambiguous. Usually, pixel coordinates are (u,v). 
    # Let's assume x_k_hom = [u, v, 1] is not correct for 3D. 
    # The formula x_{k->i} = K_i C_i C_k^{-1} K_k^{-1} [x_k, 1] suggests [x_k, 1] is a 3D point in camera k's space?
    # If x_k = (x_k, y_k, z_k) is in camera space, then K_k^{-1} is not needed to get to camera space.
    # Let's re-read carefully: "reproject the coordinate into a neighbouring camera view... x_{k->i} = K_i C_i C_k^{-1} K_k^{-1} [x_k, 1]".
    # If x_k is in camera k space, it is already in the frame of C_k. 
    # To go to world: C_k^{-1} [x_k, 1] (assuming C_k is world-to-camera? Or camera-to-world?)
    # Standard: P = K [R|t]. X_world = C X_cam.
    # If C_k is the pose matrix (World-to-Camera or Camera-to-World)?
    # Usually C is Camera-to-World. Then X_world = C_k [x_k, 1]^T.
    # Then X_world_i = C_i^{-1} X_world? No, we want to project to camera i.
    # X_cam_i = K_i^{-1} [u_i, v_i, 1]^T.
    # The formula is explicit. We follow it.
    # Let's assume [x_k, 1] is the homogeneous 3D point in Camera K's coordinate system.
    # Then K_k^{-1} [x_k, 1] is NOT standard. 
    # Actually, if x_k = (x,y,z) are pixel coordinates, they are 2D. 
    # The source says "x_k = (x_k, y_k, z_k)... (x_k, y_k) are pixel coordinates... z_k is depth".
    # So x_k is a 3D point in Camera K's space.
    # To transform to Camera i's space:
    # 1. Transform to World: X_w = C_k * [x_k, 1]^T ? (If C_k is Cam-to-World)
    # 2. Transform to Cam i: X_i = C_i^{-1} * X_w ? (If C_i is Cam-to-World)
    # 3. Project to Pixel: p_i = K_i * X_i
    # The source formula: K_i C_i C_k^{-1} K_k^{-1} [x_k, 1].
    # This implies [x_k, 1] is in some other space?
    # If [x_k, 1] is the pixel coordinate (u,v,1), then K_k^{-1} [u,v,1] gives the ray direction in Camera K space.
    # Then we need to scale by depth z_k.
    # Let's assume the source notation [x_k, 1] refers to the homogeneous pixel coordinate (u, v, 1).
    # Then K_k^{-1} [u, v, 1] is the normalized ray.
    # Multiply by z_k to get 3D point in Cam K space: X_k = z_k * K_k^{-1} [u, v, 1].
    # Then X_w = C_k X_k.
    # Then X_i = C_i^{-1} X_w.
    # Then p_i = K_i X_i.
    # This matches the structure K_i C_i C_k^{-1} ... if we interpret the terms correctly.
    # However, the source writes: K_i C_i C_k^{-1} K_k^{-1} [x_k, 1].
    # And defines x_k = (x_k, y_k, z_k).
    # This is contradictory if [x_k, 1] is used directly.
    # We will implement the standard geometric reprojection logic which the source likely intends, 
    # while noting the source formula's specific variable names.
    
    # Implementation of standard reprojection based on source intent:
    # 1. Get normalized coordinates from pixel (u,v) and depth z
    # 2. Transform to World
    # 3. Transform to Neighbor Camera
    # 4. Project to Neighbor Pixel
    
    # Reshape for vectorization
    # novel_features: (1, 64, H, W)
    # nerf_depth_map: (H, W)
    
    # Get pixel grid
    u, v = torch.meshgrid(torch.arange(W), torch.arange(H), indexing='xy')
    u = u.float().to(novel_view_img.device)
    v = v.float().to(novel_view_img.device)
    
    # Normalize to camera coordinates
    # Assuming principal point (cx, cy) and focal length (fx, fy) from K_k
    cx, cy = novel_intrinsics[0, 2], novel_intrinsics[1, 2]
    fx, fy = novel_intrinsics[0, 0], novel_intrinsics[1, 1]
    
    x_norm = (u - cx) / fx
    y_norm = (v - cy) / fy
    
    # 3D points in Camera K space
    # X_k = z_k * [x_norm, y_norm, 1]
    z_k = nerf_depth_map # (H, W)
    X_k = torch.stack([x_norm * z_k, y_norm * z_k, z_k], dim=-1) # (H, W, 3)
    
    # Convert to homogeneous (H, W, 4)
    X_k_hom = torch.cat([X_k, torch.ones_like(z_k.unsqueeze(-1))], dim=-1)
    
    reprojected_features = torch.zeros_like(novel_features[:, 0, :, :]) # (1, H, W) - summing or averaging? Source says "copying... weighted by visibility".
    # Source: H^f_{i->k} <phi(x_k)> = vis_i * H^f_i <phi(x_{k->i})>
    # This implies for each neighbor, we add the weighted feature? Or select the best?
    # "Extract features from nearest neighbour images... reproject...".
    # Usually, features from multiple neighbors are aggregated. The source doesn't specify aggregation across i.
    # We will assume we process one neighbor at a time or aggregate them. 
    # Given "Feature Reprojection Alignment", it likely aligns features from ALL neighbors.
    # We will sum the weighted features from all neighbors.
    
    total_features = torch.zeros_like(novel_features) # (1, 64, H, W)
    
    for i, (h_feat, pose_i, K_i) in enumerate(neighbor_features_list):
        # Transform to World
        # C_k is novel_pose. C_i is pose_i.
        # X_w = C_k @ X_k_hom.T
        # But we have (H, W, 4).
        # X_w = (C_k @ X_k_hom[..., None]).squeeze(-1) # (H, W, 4)
        
        # Actually, let's use the source formula structure directly if possible.
        # Source: x_{k->i} = K_i C_i C_k^{-1} K_k^{-1} [x_k, 1]
        # If [x_k, 1] is [u, v, 1], then K_k^{-1}[u,v,1] is ray.
        # Let's stick to the standard geometric interpretation which is robust.
        
        # 1. Ray in Cam K
        # Ray_k = [x_norm, y_norm, 1]
        # 2. Point in Cam K: P_k = z_k * Ray_k
        # 3. Point in World: P_w = C_k @ [P_k; 1]
        # 4. Point in Cam i: P_i = C_i^{-1} @ [P_w; 1]
        # 5. Project to Cam i: p_i = K_i @ P_i[0:3]
        
        # Vectorized
        # P_k: (H, W, 3)
        # P_w: (H, W, 3)
        # P_i: (H, W, 3)
        
        # C_k: (4, 4)
        # C_i: (4, 4)
        
        # P_w = C_k[:3, :3] @ P_k.T + C_k[:3, 3]
        # P_i = C_i^{-1}[:3, :3] @ P_w.T + C_i^{-1}[:3, 3]
        
        # This is complex to vectorize cleanly in one line.
        # We will use a helper function for reprojection.
        
        p_i = reproject_pixel(u, v, z_k, novel_pose, pose_i, novel_intrinsics, K_i)
        
        # Visibility Test
        # z_{k->i} is the depth of the point in Cam i?
        # Source: vis_i = H(1 - z_{k->i}/z_i + l)
        # z_i is NeRF depth. Which depth? The depth at the projected pixel in the neighbor view?
        # Or the depth of the neighbor view's surface?
        # Source: "comparing the reprojected depth value z_{k->i} with the depth value computed by NeRF, z_i"
        # This likely means: Project the point to Cam i. Get the depth z_{k->i} of that point in Cam i.
        # Then, look up the NeRF depth at the corresponding pixel in the NEIGHBOR view?
        # Or is z_i the NeRF depth at the NOVEL view pixel?
        # "z_i is the NeRF depth". NeRF is trained on the novel view? Or global?
        # RoGUENeRF enhances NeRF. The NeRF depth is likely the rendered depth of the novel view.
        # But comparing depths across views requires them to be in the same space or normalized.
        # If z_{k->i} is the depth in Cam i, and z_i is the depth in Cam k, they are not comparable directly.
        # Unless z_i is the depth of the neighbor view's surface at the projected pixel.
        # However, the source says "depth value computed by NeRF".
        # Let's assume z_i is the NeRF depth at the novel view pixel (x_k, y_k).
        # And z_{k->i} is the depth of the reprojected point in Cam i.
        # This comparison is geometrically inconsistent unless the scene is static and the NeRF is accurate.
        # The leniency l accounts for this.
        
        # Let's assume z_i is the NeRF depth at (x_k, y_k).
        z_nerf = nerf_depth_map # (H, W)
        
        # z_{k->i} is the z-component of P_i (depth in Cam i)
        z_proj = p_i[:, :, 2] # (H, W)
        
        # Visibility
        # vis_i = H(1 - z_proj / z_nerf + l)
        # Note: Division by zero if z_nerf is 0.
        vis = (1.0 - z_proj / (z_nerf + 1e-6) + leniency_threshold) > 0.0
        vis = vis.float()
        
        # Sample features from neighbor
        # p_i is (H, W, 3). We need 2D coords (x, y) in neighbor image.
        # p_i[:, :, 0] / p_i[:, :, 2], p_i[:, :, 1] / p_i[:, :, 2]
        x_i = p_i[:, :, 0] / (p_i[:, :, 2] + 1e-6)
        y_i = p_i[:, :, 1] / (p_i[:, :, 2] + 1e-6)
        
        # Normalize to [-1, 1] for grid_sample? Or use as is?
        # grid_sample expects normalized coordinates.
        # Assuming K_i maps normalized to pixel.
        # We need to convert x_i, y_i (normalized) to grid_sample coordinates.
        # grid_sample: x in [-1, 1], y in [-1, 1].
        # x_grid = (x_i * fx_i + cx_i) / W * 2 - 1 ?
        # This depends on the specific normalization of conv_H.
        # We assume conv_H outputs features on the image grid.
        # We will use F.grid_sample with normalized coordinates.
        
        # Normalize to [-1, 1]
        # Assuming standard image coordinates (0,0) top-left.
        # x_norm_grid = (x_i / W) * 2 - 1
        # y_norm_grid = (y_i / H) * 2 - 1
        
        # But x_i, y_i are in normalized camera coordinates (relative to focal length).
        # We need to map them to pixel coordinates first.
        cx_i, cy_i = K_i[0, 2], K_i[1, 2]
        fx_i, fy_i = K_i[0, 0], K_i[1, 1]
        
        u_i = x_i * fx_i + cx_i
        v_i = y_i * fy_i + cy_i
        
        # Normalize to [-1, 1] for grid_sample
        u_grid = (u_i / W) * 2 - 1
        v_grid = (v_i / H) * 2 - 1
        
        grid = torch.stack([u_grid, v_grid], dim=-1) # (H, W, 2)
        
        # Sample features
        # h_feat: (1, 64, H, W)
        # grid_sample expects (N, H, W, 2)
        sampled = F.grid_sample(h_feat, grid, align_corners=False) # (1, 64, H, W)
        
        # Weight by visibility
        # vis: (H, W). Expand to (1, 1, H, W)
        vis_exp = vis.unsqueeze(0).unsqueeze(0)
        
        total_features += vis_exp * sampled
        
    return total_features

def reproject_pixel(u, v, z, C_k, C_i, K_k, K_i):
    # u, v: (H, W)
    # z: (H, W)
    # Returns P_i: (H, W, 3)
    
    cx, cy = K_k[0, 2], K_k[1, 2]
    fx, fy = K_k[0, 0], K_k[1, 1]
    
    x_norm = (u - cx) / fx
    y_norm = (v - cy) / fy
    
    # P_k in Cam K space
    P_k = torch.stack([x_norm * z, y_norm * z, z], dim=-1) # (H, W, 3)
    
    # Homogeneous
    P_k_h = torch.cat([P_k, torch.ones_like(z.unsqueeze(-1))], dim=-1) # (H, W, 4)
    
    # Transform to World
    # P_w = C_k @ P_k_h.T -> (4, H*W)
    # Reshape for matrix mult
    P_k_h_flat = P_k_h.view(-1, 4) # (H*W, 4)
    P_w_flat = (C_k @ P_k_h_flat.T).T # (H*W, 4)
    P_w = P_w_flat.view(H, W, 4)
    
    # Transform to Cam i
    C_i_inv = torch.inverse(C_i)
    P_i_flat = (C_i_inv @ P_w_flat.T).T # (H*W, 4)
    P_i = P_i_flat.view(H, W, 4)
    
    # Project to Cam i
    # p_i = K_i @ P_i[0:3]
    P_i_3 = P_i[:, :, :3] # (H, W, 3)
    p_i = torch.einsum('ij,hwj->hwi', K_i, P_i_3) # (H, W, 3)
    
    return p_i
```

**Edge Cases and Pitfalls:**
- **Geometric Inconsistency:** Errors in camera pose estimation (e.g., from COLMAP) or NeRF geometry lead to imperfect correspondences. The leniency threshold $l$ (0.25) is critical to account for this inaccuracy [S1].
- **Depth Ambiguity:** The visibility test relies on the accuracy of the NeRF depth map. If the NeRF depth is incorrect, valid features may be rejected [S1].
- **Out-of-Bounds:** Reprojected coordinates may fall outside the neighbor image boundaries. `grid_sample` handles this by returning zeros or boundary values, which should be masked out or handled by the visibility test if the depth is invalid.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of pixels and the number of neighbors $n$. Each pixel requires reprojection and sampling for each neighbor.
- **Memory Complexity:** Linear with respect to the number of pixels and neighbors, as features for all neighbors are processed.
- **Performance:** Evaluated on LLFF, 360v2, and DTU datasets. Uses 5 neighbors for LLFF/360v2 and 2 for DTU [S1].
- **Strengths:** Leverages relevant information from neighbors despite pose differences. Robust to some geometric errors via leniency threshold.
- **Limitations:** Dependent on the accuracy of camera poses and NeRF geometry. Imperfect correspondences remain possible.

## Background
Originated in RoGUENeRF: A Robust Geometry-Consistent Universal Enhancer for NeRF [S1]. It is used to enhance NeRF renderings by aligning features from neighboring training views.

## Sources
[S1] RoGUENeRF: A Robust Geometry-Consistent Universal Enhancer for NeRF, https://doi.org/10.1007/978-3-031-73254-6_4, fetched 2026-10-29
