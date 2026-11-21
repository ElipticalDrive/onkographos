# Geometry Pseudo Label via Image Warping

Also called: Image Warping for Geometry Pseudo Labels, Self-supervised Geometry Pseudo Labeling

## Overview
This method reconstructs reasonable 3D geometry from a single reference view by propagating depth information to unseen views through image warping, serving as pseudo ground truth labels to prevent overfitting and collapse in single-view NeRF training [S1]. It sets itself apart from standard NeRF [4972-neural-radiance-fields.md] by introducing explicit geometric consistency constraints derived from multi-view projection, rather than relying solely on photometric reconstruction loss [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A single reference view image $I_{ref}$ with known camera intrinsics $K_{ref}$ and extrinsics; a set of unseen view images $I_{unseen}$ with known intrinsics $K_{unseen}$ and extrinsics; the current predicted depth map $Z_{ref}$ for the reference view [S1].
*   **Outputs:** Pseudo depth labels for the unseen views; a geometry consistency loss $\mathcal{L}_{geo}$ used to update the radiance field [S1].

**Data Structures:**
*   **Depth Map:** A 2D array of floating-point values representing depth $Z$ for each pixel in the reference view [S1].
*   **Camera Matrices:** $K$ ($3 \times 3$ intrinsic matrix), $T$ ($4 \times 4$ transformation matrix representing the relative pose between reference and unseen views) [S1].

**Numbered Steps:**

1.  **Forward Warping:** For each pixel $p_i(x_i, y_i)$ in the reference view $I_{ref}$ with depth $Z_i$, compute the corresponding pixel $p_j(x_j, y_j)$ in the $j$-th unseen view $I_{unseen}$ using the projection equation [S1]:
    $$
    p_j = K_{\text{unseen}} T (K^{-1}_{\text{ref}} Z_i p_i)
    $$
    Here, $p_i$ and $p_j$ are treated as homogeneous coordinates. $T$ is the transformation matrix from the reference camera frame to the unseen camera frame [S1].

2.  **Occlusion Handling (Painter’s Algorithm):** If multiple points from the reference view project to the same pixel location in the unseen view, select the point with the smallest depth value as the valid warping result [S1]. This handles occlusions by prioritizing closer geometry [S1].

3.  **Backward Warping:** Reproject the unseen views back to the reference view using the warped depth information to enforce geometry consistency [S1]. This creates a bidirectional consistency check [S1].

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
import math

def compute_geometry_pseudo_label_loss(depth_1, depth_2, K_1, K_2, T_1_to_2, lambda_4=0.1, image_1=None, image_2=None):
    """
    Computes the geometry pseudo label loss using image warping.
    Args:
        depth_1, depth_2: 2D lists of floats, depth maps for two views.
        K_1, K_2: 3x3 lists of lists, intrinsic matrices.
        T_1_to_2: 4x4 list of lists, extrinsic transformation from view 1 to view 2.
        lambda_4: Float, weight for smoothness loss.
        image_1, image_2: 2D lists of floats, RGB images for Laplacian computation.
    Returns:
        total_loss: Float, the computed geometry loss.
    """
    # Forward warp: depth_1 -> depth_2 frame
    warped_depth_2 = warp_depth(depth_1, K_1, K_2, T_1_to_2)
    # Backward warp: depth_2 -> depth_1 frame
    T_2_to_1 = inverse_matrix(T_1_to_2)
    warped_depth_1 = warp_depth(depth_2, K_2, K_1, T_2_to_1)

    loss_forward = mean_absolute_error(depth_2, warped_depth_2)
    loss_backward = mean_absolute_error(depth_1, warped_depth_1)

    smooth_loss = 0.0
    if image_1 is not None and image_2 is not None:
        # Downscale for smoothness loss as per [S1]
        d1_ds = downscale_2d(depth_1, factor=2)
        d2_ds = downscale_2d(depth_2, factor=2)
        img1_ds = downscale_2d(image_1, factor=2)
        img2_ds = downscale_2d(image_2, factor=2)
        smooth_loss = compute_smoothness_loss(d1_ds, img1_ds) + compute_smoothness_loss(d2_ds, img2_ds)

    return loss_forward + loss_backward + lambda_4 * smooth_loss

def warp_depth(depth_map, K_src, K_dst, T_src_to_dst):
    """
    Warps depth map from source view to destination view.
    Uses Painter's Algorithm for occlusion handling.
    """
    H, W = len(depth_map), len(depth_map[0])
    K_inv_src = inverse_matrix(K_src)
    warped = [[float('inf')] * W for _ in range(H)]

    for y in range(H):
        for x in range(W):
            z = depth_map[y][x]
            if z <= 0: continue
            # Homogeneous pixel
            p_src = [float(x), float(y), 1.0]
            # 3D point in src camera: K_inv * [x,y,1] * Z
            ray = matrix_vector_multiply(K_inv_src, p_src)
            point_3d = [ray[0]*z, ray[1]*z, ray[2]*z]
            # Transform to dst camera
            point_3d_h = point_3d + [1.0]
            point_dst_h = matrix_vector_multiply(T_src_to_dst, point_3d_h)
            point_dst = point_dst_h[:3]
            # Project to dst image: K_dst * point_dst
            p_dst_h = matrix_vector_multiply(K_dst, point_dst)
            if p_dst_h[2] <= 0: continue
            u = p_dst_h[0] / p_dst_h[2]
            v = p_dst_h[1] / p_dst_h[2]
            if 0 <= u < W and 0 <= v < H:
                if point_dst_h[2] < warped[int(v)][int(u)]:
                    warped[int(v)][int(u)] = point_dst_h[2]
    return warped

def compute_smoothness_loss(depth, image):
    """
    Computes self-supervised inverse depth smoothness loss.
    """
    H, W = len(depth), len(depth[0])
    laplacian = compute_laplacian(image)
    total = 0.0
    count = 0
    for y in range(1, H-1):
        for x in range(1, W-1):
            dxx = depth[y][x+1] - 2*depth[y][x] + depth[y][x-1]
            dyy = depth[y+1][x] - 2*depth[y][x] + depth[y-1][x]
            dxy = 0.25 * (depth[y-1][x-1] - depth[y-1][x+1] - depth[y+1][x-1] + depth[y+1][x+1])
            weight = math.exp(-laplacian[y][x])
            total += weight * (abs(dxx) + abs(dxy) + abs(dyy))
            count += 1
    return total / count if count > 0 else 0.0

def inverse_matrix(M):
    """Computes inverse of a 4x4 matrix using standard Gaussian elimination."""
    n = len(M)
    aug = [row[:] + [1.0 if i == j else 0.0 for j in range(n)] for i, row in enumerate(M)]
    for i in range(n):
        pivot = aug[i][i]
        for j in range(2*n): aug[i][j] /= pivot
        for k in range(n):
            if k != i:
                factor = aug[k][i]
                for j in range(2*n): aug[k][j] -= factor * aug[i][j]
    return [row[n:] for row in aug]

def matrix_vector_multiply(M, v):
    return [sum(M[i][j] * v[j] for j in range(len(v))) for i in range(len(M))]

def downscale_2d(arr, factor):
    H, W = len(arr), len(arr[0])
    return [[arr[i*factor][j*factor] for j in range(W//factor)] for i in range(H//factor)]

def compute_laplacian(image):
    H, W = len(image), len(image[0])
    lap = [[0.0]*W for _ in range(H)]
    for y in range(1, H-1):
        for x in range(1, W-1):
            lap[y][x] = (image[y-1][x] + image[y+1][x] + image[y][x-1] + image[y][x+1] - 4*image[y][x])
    return lap

def mean_absolute_error(pred, target):
    H, W = len(pred), len(pred[0])
    total = 0.0
    count = 0
    for y in range(H):
        for x in range(W):
            if target[y][x] != float('inf'):
                total += abs(pred[y][x] - target[y][x])
                count += 1
    return total / count if count > 0 else 0.0
```

**Parameters:**
*   $\lambda_4$: 0.1 [S1].

**Edge Cases and Pitfalls:**
*   **Occlusion:** When multiple points project to the same pixel, the Painter’s Algorithm selects the smallest depth [S1].
*   **Uncertain Regions:** Projected results may contain uncertain regions due to occlusion or depth prediction errors. The smoothness loss helps regularize these [S1].
*   **Misalignment:** Small misalignments in the predicted depth map can cause large errors when projected to other views [S1].
*   **Resolution:** The smoothness loss is calculated on a downscaled resolution [S1].

## Subtopics

### Self-supervised Visibility Task
A self-supervised task to classify points as visible or occluded along a ray to enhance 3D geometry prediction, without requiring multi-view geometry ground truth [S2]. Along a camera ray trajectory, points sampled prior to reaching the first surface point are categorized as visible, whereas points sampled beyond the first surface point are classified as occluded [S2]. To find the first surface point, the multiplication of SRDF values between adjacent sampled points is computed; when $\tilde{d}_{\Omega}^i \cdot \tilde{d}_{\Omega}^{i+1} \le 0$, the first surface lies within that interval, labeling points before it as visible (1) and after as occluded (0) [S2]. The same logic is applied to SDF predictions. Labels are only assigned if SRDF and SDF agree; points with divergent classifications are omitted [S2]. A visibility probability $V_{pred}$ is predicted from the SRDF branch, and binary cross-entropy loss $L_{vis}$ is computed between $V_{pred}$ and the ground truth $V_{gt}$ [S2]. The visibility ground truth is defined as:
$$
V_{gt}= \begin{cases} 1, & \text { if } (V^{\text{SRDF}} = 1) \& (V^{\text{SDF}} = 1) \\ 0, & \text { if } (V^{\text{SRDF}} = 0) \& (V^{\text{SDF}} = 0) \end{cases}
$$
This provides 3D space priors that directly influence SRDF and SDF learning [S2].

## Operating characteristics
*   **Strengths:** Provides explicit geometric supervision in single-view settings, preventing collapse and improving 3D shape reconstruction [S1].
*   **Limitations:** Relies on accurate camera calibration and depth prediction. Uncertain regions due to occlusion can still introduce errors [S1].
*   **When to use:** Single-view NeRF training where depth supervision is needed but ground truth depth is unavailable [S1].

## Background
Originated in the SinNeRF paper, which proposes training NeRF from a single image by propagating depth information to unseen views [S1]. The method addresses the issue that simple depth supervision from a single view cannot produce reasonable 3D shapes without additional consistency constraints [S1].

## Sources
[S1] SinNeRF: Training Neural Radiance Fields on Complex Scenes from a Single Image, https://doi.org/10.1007/978-3-031-20047-2_42, fetched 2026-10-26
[S2] Ray-Distance Volume Rendering for Neural Scene Reconstruction, https://doi.org/10.1007/978-3-031-72630-9_22, fetched 2026-10-26
