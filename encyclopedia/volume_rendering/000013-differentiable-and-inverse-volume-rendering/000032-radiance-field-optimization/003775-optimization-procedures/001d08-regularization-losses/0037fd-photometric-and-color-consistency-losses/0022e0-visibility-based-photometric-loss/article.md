# Visibility-Based Photometric Loss

Also called: Visibility Masking for Photometric Loss

## Overview
Visibility-Based Photometric Loss computes photometric error only on pixels identified as visible across source and target views [S1]. It differs from standard photometric losses by filtering out pixels where the squared difference between reprojected source depth and directly rendered target depth exceeds a threshold, preventing the model from fitting to occluded or transient geometry [S1, S2].

## How it works
The method restricts the photometric error computation to pixels that are geometrically consistent across views [S1]. Visibility is determined by comparing the depth of a pixel reprojected from a source view into the target view against the directly rendered depth of that target view [S1].

### Inputs and Outputs
*   **Inputs:** Source view color image $I_s$ and depth map $D_s$ [S1]. Target view color image $I_t$ and rendered depth map $D_t$ [S1]. Camera poses for source and target views [S1]. Intrinsic camera parameters [G].
*   **Outputs:** Scalar loss value $L_{photo}$ [S1].

### Data Structures
*   **Visibility Mask ($M_v$):** A 2D tensor of shape $(H, W)$ with values in $\{0, 1\}$ [S1]. $1$ indicates a visible pixel, $0$ indicates occluded or transient [S1].
*   **Reprojected Depth ($D_{reproj}$):** A 2D tensor of shape $(H, W)$ containing the depth of the source view pixel projected into the target view coordinate system [S1].

### Steps
1.  Compute reprojected depth from source view to target view [S1].
2.  Compute directly rendered depth of target view [S1].
3.  Calculate difference $\Delta d$ between reprojected and rendered depth [S1].
4.  Mark pixel as visible if $\Delta d^2 < 0.001$ [S1].
5.  Compute photometric error only on visible pixels using L1 loss [S1].

### Formula
The visibility mask $M_v(u, v)$ is defined as:
$$
M_v(u, v) = \begin{cases} 
1 & \text{if } (D_{reproj}(u, v) - D_t(u, v))^2 < \tau \\
0 & \text{otherwise}
\end{cases}
$$
where $\tau = 0.001$ [S1].

The photometric loss $L_{photo}$ is:
$$
L_{photo} = \frac{1}{\sum M_v} \sum_{u,v} M_v(u, v) \cdot |I_s^{warped}(u, v) - I_t(u, v)|
$$
where $|\cdot|$ denotes the L1 norm [S1].

### Parameters
*   **Threshold ($\tau$):** $0.001$ [S1]. This value determines the tolerance for depth inconsistency [G].

### Edge Cases and Pitfalls
*   **Precision:** Depth values must be in consistent units; small numerical errors in reprojection can lead to false occlusions if $\tau$ is too small [G].
*   **Boundaries:** Pixels near object boundaries may have ambiguous depth due to anti-aliasing or discretization, potentially causing flickering visibility masks [G].
*   **Degenerate Input:** If depth maps are missing or invalid, reprojection fails [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def transform_points(pts, pose):
    """Transform 3D points (H, W, 3) by 4x4 pose matrix."""
    return pts @ pose[:3, :3].T + pose[:3, 3]

def sample_depth(D, u, v):
    """Bilinearly sample depth map D at coordinates u, v (H, W)."""
    # Clamp coordinates to valid range
    u = np.clip(u, 0, D.shape[1] - 1)
    v = np.clip(v, 0, D.shape[0] - 1)
    # Floor and ceil for bilinear interpolation
    u0, v0 = np.floor(u).astype(int), np.floor(v).astype(int)
    u1, v1 = u0 + 1, v0 + 1
    # Handle boundary cases
    u1 = np.clip(u1, 0, D.shape[1] - 1)
    v1 = np.clip(v1, 0, D.shape[0] - 1)
    # Interpolate
    wa = (u1 - u)[:, :, np.newaxis]
    wb = (v1 - v)[:, :, np.newaxis]
    wc = (u - u0)[:, :, np.newaxis]
    wd = (v - v0)[:, :, np.newaxis]
    return (wa * wb * D[v0, u0] + wa * wd * D[v0, u1] + 
            wc * wb * D[v1, u0] + wc * wd * D[v1, u1])

def warp_image(I, u, v):
    """Bilinearly warp image I (H, W, 3) to coordinates u, v."""
    # Clamp coordinates
    u = np.clip(u, 0, I.shape[1] - 1)
    v = np.clip(v, 0, I.shape[0] - 1)
    u0, v0 = np.floor(u).astype(int), np.floor(v).astype(int)
    u1, v1 = np.clip(u0 + 1, 0, I.shape[1] - 1), np.clip(v0 + 1, 0, I.shape[0] - 1)
    wa = (u1 - u)[:, :, np.newaxis]
    wb = (v1 - v)[:, :, np.newaxis]
    wc = (u - u0)[:, :, np.newaxis]
    wd = (v - v0)[:, :, np.newaxis]
    return (wa * wb * I[v0, u0] + wa * wd * I[v0, u1] + 
            wc * wb * I[v1, u0] + wc * wd * I[v1, u1])

def visibility_based_photometric_loss(I_s, D_s, I_t, D_t, pose_s, pose_t, K, tau=0.001):
    """
    Compute photometric loss only on visible pixels.
    
    Args:
        I_s: Source view color image (H, W, 3)
        D_s: Source view depth map (H, W)
        I_t: Target view color image (H, W, 3)
        D_t: Target view rendered depth map (H, W)
        pose_s: Source camera pose (4, 4)
        pose_t: Target camera pose (4, 4)
        K: Camera intrinsics matrix (3, 3)
        tau: Depth difference threshold (float)
        
    Returns:
        L_photo: Scalar photometric loss
    """
    H, W = I_t.shape[:2]
    
    # Create grid of pixel coordinates
    u, v = np.meshgrid(np.arange(W), np.arange(H), indexing='ij')
    
    # Back-project source pixels to 3D using D_s and K
    x_s = (u - K[0, 2]) / K[0, 0]
    y_s = (v - K[1, 2]) / K[1, 1]
    z_s = D_s
    
    # 3D points in source camera frame
    pts_s_cam = np.stack([x_s * z_s, y_s * z_s, z_s], axis=-1)
    
    # Transform to world frame
    pts_s_world = transform_points(pts_s_cam, pose_s)
    
    # Transform to target camera frame
    pts_t_cam = transform_points(pts_s_world, pose_t)
    
    # Project to target image plane
    u_reproj = (pts_t_cam[..., 0] / pts_t_cam[..., 2]) * K[0, 0] + K[0, 2]
    v_reproj = (pts_t_cam[..., 1] / pts_t_cam[..., 2]) * K[1, 1] + K[1, 2]
    
    # Sample depth from source depth map at reprojected coordinates
    D_reproj = sample_depth(D_s, u_reproj, v_reproj)
    
    # Compute depth difference
    delta_d_sq = (D_reproj - D_t) ** 2
    
    # Determine visibility
    M_v = (delta_d_sq < tau).astype(np.float32)
    
    # Warp source image to target view
    I_s_warped = warp_image(I_s, u_reproj, v_reproj)
    
    # Compute L1 loss on visible pixels
    diff = np.abs(I_s_warped - I_t)
    loss_per_pixel = np.mean(diff, axis=-1)
    
    # Masked mean
    visible_sum = np.sum(M_v)
    if visible_sum > 0:
        L_photo = np.sum(M_v * loss_per_pixel) / visible_sum
    else:
        L_photo = np.float32(0.0)
        
    return L_photo
```

## Subtopics

### Transient Object Handling via Visibility Map
This variant uses a learned 2D visibility map $V_M \in \mathbb{R}^{1 \times H \times W}$ generated by a U-Net model to segment transient objects (e.g., moving people) from static geometry [S2]. The loss between the rendered image $I_r$ and reference image $I_{gt}$ is weighted by $V_M$ via Hadamard product ($\odot$) [S2]. To prevent the network from suppressing all pixels to minimize loss, a regularization loss $L_{vm} = L_2(V_M, 1)$ is applied, encouraging the visibility map to remain close to ones [S2]. This method is chosen when transient objects are prevalent and geometric consistency alone is insufficient to exclude them [G].

## Operating characteristics
*   **Time Complexity:** $O(HW)$ per view pair for reprojection and masking [G]. The U-Net variant adds inference time for the segmentation network [S2].
*   **Memory Complexity:** $O(HW)$ for storing depth maps and visibility masks [G].
*   **Strengths:** Effectively removes artifacts from occlusions and transient objects [S1, S2]. Improves reconstruction quality in dynamic scenes [S2].
*   **Limitations:** Requires accurate depth maps for the geometric variant [S1]. The threshold $\tau$ is sensitive to noise [G]. The U-Net variant requires additional training and memory [S2].
*   **When to use:** Multi-view reconstruction with occlusions or dynamic scenes [S1, S2].
*   **When not to use:** Static scenes with no occlusions where simple photometric loss suffices [G].

## Background
The geometric visibility-based photometric loss was introduced in the context of differentiable volume rendering and signed distance function (SDF) optimization [S1]. It addresses the issue of photometric errors being computed on occluded pixels, which leads to incorrect gradient updates [S1]. The transient object handling variant was proposed for 3D Gaussian Splatting in unconstrained image collections, where moving objects cause artifacts in the reconstructed scene [S2].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections, https://doi.org/10.1007/978-3-031-73116-7_20, fetched 2026-10-27
