# Geometry Pseudo Label via Image Warping

Also called: Image Warping for Geometry Pseudo Labels, Self-supervised Geometry Pseudo Labeling

## Overview
This method reconstructs reasonable 3D geometry from a single reference view by propagating depth information to unseen views through image warping, serving as pseudo ground truth labels to prevent overfitting and collapse in single-view NeRF training [S1]. It sets itself apart from standard NeRF [4972-neural-radiance-fields.md] by introducing explicit geometric consistency constraints derived from multi-view projection, rather than relying solely on photometric reconstruction loss.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A single reference view image $I_{ref}$ with known camera intrinsics $K_{ref}$ and extrinsics; a set of unseen view images $I_{unseen}$ with known intrinsics $K_{unseen}$ and extrinsics; the current predicted depth map $Z_{ref}$ for the reference view.
*   **Outputs:** Pseudo depth labels for the unseen views; a geometry consistency loss $\mathcal{L}_{geo}$ used to update the radiance field.

**Data Structures:**
*   **Depth Map:** A 2D array of floating-point values representing depth $Z$ for each pixel in the reference view.
*   **Camera Matrices:** $K$ ($3 \times 3$ intrinsic matrix), $T$ ($4 \times 4$ transformation matrix representing the relative pose between reference and unseen views).

**Numbered Steps:**

1.  **Forward Warping:** For each pixel $p_i(x_i, y_i)$ in the reference view $I_{ref}$ with depth $Z_i$, compute the corresponding pixel $p_j(x_j, y_j)$ in the $j$-th unseen view $I_{unseen}$ using the projection equation [S1]:
    $$
    p_j = K_{\text{unseen}} T (K^{-1}_{\text{ref}} Z_i p_i)
    $$
    Here, $p_i$ and $p_j$ are treated as homogeneous coordinates. $T$ is the transformation matrix from the reference camera frame to the unseen camera frame.

2.  **Occlusion Handling (Painter’s Algorithm):** If multiple points from the reference view project to the same pixel location in the unseen view, select the point with the smallest depth value as the valid warping result [S1]. This handles occlusions by prioritizing closer geometry.

3.  **Backward Warping:** Reproject the unseen views back to the reference view using the warped depth information to enforce geometry consistency [S1]. This creates a bidirectional consistency check.

4.  **Smoothness Regularization:** Calculate the self-supervised inverse depth smoothness loss $\mathcal{L}_{\text{smooth}}$ to regularize uncertain regions in the warped results [S1]. This loss is calculated on a downscaled resolution [S1].

5.  **Loss Computation:** Combine the forward and backward warping errors with the smoothness term to form the total geometry pseudo label loss $\mathcal{L}_{geo}$ [S1].

**Formulas:**

*   **Projection:**
    $$
    p_j = K_{\text{unseen}} T (K^{-1}_{\text{ref}} Z_i p_i)
    $$
    Where:
    *   $p_i, p_j$: Homogeneous pixel coordinates in reference and unseen views.
    *   $Z_i$: Depth value at pixel $p_i$ in the reference view.
    *   $K_{ref}, K_{unseen}$: Intrinsic camera matrices.
    *   $T$: Extrinsic transformation matrix from reference to unseen view.

*   **Smoothness Loss:**
    $$
    \mathcal{L}_{\text{smooth}}(d_i) = e^{-\nabla^2 \mathcal{I}(\mathbf{x}_i)} \left( |\partial_{xx} d_i| + |\partial_{xy} d_i| + |\partial_{yy} d_i| \right)
    $$
    Where:
    *   $d_i$: The depth map.
    *   $\mathcal{I}(\mathbf{x}_i)$: The RGB pixel value at location $\mathbf{x}_i$.
    *   $\nabla^2 \mathcal{I}(\mathbf{x}_i)$: The Laplacian of the pixel value at location $\mathbf{x}_i$.
    *   $\partial_{xx}, \partial_{xy}, \partial_{yy}$: Second-order spatial derivatives of the depth map.

*   **Total Geometry Loss:**
    $$
    \mathcal{L}_{geo} = \mathcal{L}_1(d_1, f(d_2)) + \mathcal{L}_1(f(d_1), d_2) + \lambda_4 \mathcal{L}_{smooth}
    $$
    Where:
    *   $d_1, d_2$: Depth maps of two views (reference and unseen).
    *   $f(\cdot)$: The image warping function using the current view's depth information.
    *   $\lambda_4$: Weight for the smoothness term, empirically set to 0.1 [S1].
    *   $\mathcal{L}_1$: L1 loss function.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_geometry_pseudo_label_loss(ref_depth, unseen_depths, K_ref, K_unseen, T_ref_to_unseen, lambda_4=0.1):
    """
    Computes the geometry pseudo label loss using image warping.
    
    Args:
        ref_depth: 2D array, depth map of the reference view.
        unseen_depths: List of 2D arrays, depth maps of unseen views.
        K_ref: 3x3 matrix, intrinsic matrix of reference view.
        K_unseen: 3x3 matrix, intrinsic matrix of unseen view.
        T_ref_to_unseen: 4x4 matrix, extrinsic transformation from ref to unseen.
        lambda_4: Float, weight for smoothness loss.
        
    Returns:
        total_loss: Float, the computed geometry loss.
    """
    total_loss = 0.0
    
    # Step 1 & 2: Forward Warping (Ref -> Unseen)
    # Warp ref_depth to get pseudo depth for unseen view
    warped_depth_unseen = warp_depth(ref_depth, K_ref, K_unseen, T_ref_to_unseen)
    
    # Compute L1 loss between predicted unseen depth and warped reference depth
    # Note: unseen_depths here refers to the depth predicted by the network for the unseen view
    # if available, or we compare consistency. Based on [S1], d1 and d2 are depths of two views.
    # In single-view training, we often compare the warped reference depth against the 
    # network's prediction for the unseen view, or vice versa.
    # The formula L1(d1, f(d2)) implies comparing view 1's depth with warped view 2's depth.
    
    for unseen_depth in unseen_depths:
        # Forward term: L1(d_unseen, f(d_ref))
        loss_forward = l1_loss(unseen_depth, warped_depth_unseen)
        
        # Step 3: Backward Warping (Unseen -> Ref)
        # Warp unseen_depth back to reference frame
        T_unseen_to_ref = inverse(T_ref_to_unseen)
        warped_depth_ref = warp_depth(unseen_depth, K_unseen, K_ref, T_unseen_to_ref)
        
        # Backward term: L1(f(d_unseen), d_ref)
        loss_backward = l1_loss(warped_depth_ref, ref_depth)
        
        # Step 4: Smoothness Loss
        # Calculate on downscaled resolution
        downscaled_depth = downscale(unseen_depth)
        downscaled_image = downscale(unseen_image) # Assuming unseen_image is available for Laplacian
        smooth_loss = compute_smoothness_loss(downscaled_depth, downscaled_image)
        
        # Step 5: Combine
        view_loss = loss_forward + loss_backward + lambda_4 * smooth_loss
        total_loss += view_loss
        
    return total_loss

def warp_depth(depth_map, K_src, K_dst, T_src_to_dst):
    """
    Warps depth map from source view to destination view.
    Uses Painter's Algorithm for occlusion handling.
    """
    H, W = depth_map.shape
    # Create grid of pixel coordinates
    y, x = np.meshgrid(np.arange(H), np.arange(W), indexing='ij')
    # Homogeneous coordinates
    p_src = np.stack([x, y, np.ones((H, W))], axis=-1).reshape(-1, 3)
    
    # Get depth values
    z = depth_map.flatten()
    
    # Convert to 3D points in source camera frame
    # p_3d = K^{-1} * z * p_src
    K_inv_src = np.linalg.inv(K_src)
    points_3d_src = np.einsum('ij,nj->ni', K_inv_src, p_src) * z[:, np.newaxis]
    
    # Transform to destination camera frame
    # Add homogeneous coordinate 1
    points_3d_src_h = np.hstack([points_3d_src, np.ones((len(points_3d_src), 1))])
    points_3d_dst_h = (T_src_to_dst @ points_3d_src_h.T).T
    
    # Project to destination image plane
    K_dst_inv = np.linalg.inv(K_dst) # Not needed directly, we use K_dst
    # p_dst = K_dst * points_3d_dst
    points_3d_dst = points_3d_dst_h[:, :3]
    p_dst_h = (K_dst @ points_3d_dst.T).T
    
    # Normalize
    u = p_dst_h[:, 0] / p_dst_h[:, 2]
    v = p_dst_h[:, 1] / p_dst_h[:, 2]
    z_dst = points_3d_dst_h[:, 2]
    
    # Handle invalid projections (outside image or negative depth)
    valid_mask = (u >= 0) & (u < W) & (v >= 0) & (v < H) & (z_dst > 0)
    
    # Painter's Algorithm: For each pixel in dest, keep the one with smallest depth
    # Initialize with infinity
    warped_depth = np.full((H, W), np.inf)
    
    # We need to iterate or use scatter operations. 
    # Since multiple src pixels can map to same dst pixel, we keep min depth.
    # This is complex to vectorize efficiently without libraries, but conceptually:
    for idx in np.where(valid_mask)[0]:
        ui, vi = int(u[idx]), int(v[idx])
        zi = z_dst[idx]
        if zi < warped_depth[vi, ui]:
            warped_depth[vi, ui] = zi
            
    return warped_depth

def compute_smoothness_loss(depth, image):
    """
    Computes self-supervised inverse depth smoothness loss.
    """
    # Calculate Laplacian of image
    laplacian_image = calculate_laplacian(image)
    
    # Calculate second order derivatives of depth
    dxx = second_derivative_x(depth)
    dxy = second_derivative_xy(depth)
    dyy = second_derivative_y(depth)
    
    # Compute loss
    # exp(-Laplacian) * (|dxx| + |dxy| + |dyy|)
    weight = np.exp(-laplacian_image)
    smooth_term = np.abs(dxx) + np.abs(dxy) + np.abs(dyy)
    
    loss = np.mean(weight * smooth_term)
    return loss
```

**Parameters:**
*   $\lambda_4$: 0.1 [S1].

**Edge Cases and Pitfalls:**
*   **Occlusion:** When multiple points project to the same pixel, the Painter’s Algorithm selects the smallest depth [S1]. Failure to handle this leads to ghosting artifacts.
*   **Uncertain Regions:** Projected results may contain uncertain regions due to occlusion or depth prediction errors. The smoothness loss helps regularize these [S1].
*   **Misalignment:** Small misalignments in the predicted depth map can cause large errors when projected to other views [S1].
*   **Resolution:** The smoothness loss is calculated on a downscaled resolution [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of unseen views and the resolution of the depth maps. Warping is $O(N)$ where $N$ is the number of pixels.
*   **Memory Complexity:** Requires storing depth maps for reference and unseen views.
*   **Strengths:** Provides explicit geometric supervision in single-view settings, preventing collapse and improving 3D shape reconstruction [S1].
*   **Limitations:** Relies on accurate camera calibration and depth prediction. Uncertain regions due to occlusion can still introduce errors [S1].
*   **When to use:** Single-view NeRF training where depth supervision is needed but ground truth depth is unavailable [S1].

## Background
Originated in the SinNeRF paper, which proposes training NeRF from a single image by propagating depth information to unseen views [S1]. The method addresses the issue that simple depth supervision from a single view cannot produce reasonable 3D shapes without additional consistency constraints [S1].

## Sources
[S1] SinNeRF: Training Neural Radiance Fields on Complex Scenes from a Single Image, https://doi.org/10.1007/978-3-031-20047-2_42, fetched 2026-10-26
