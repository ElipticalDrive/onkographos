# Visibility-Based Photometric Loss

Also called: Visibility Masking for Photometric Loss

## Overview
Visibility-Based Photometric Loss computes the photometric error only on pixels where the squared depth difference between the reprojected source view depth and the directly rendered target view depth is below a fixed threshold ($\Delta d^2 < 0.001$) [S1]. It sets itself apart from standard photometric losses by explicitly masking out geometrically inconsistent pixels before applying the L1 loss [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Target view image $\mathbf{C}_{target}$, target view depth $d_{target}$, source view image $\mathbf{C}_{src}$, source view depth $d_{src}$, and camera intrinsics/extrinsics for both views [S1].
*   **Outputs:** A scalar photometric loss value $\mathcal{L}_{photo}$ [S1].

**Data Structures:**
*   **Depth Maps:** 2D arrays of float values representing metric depth from the camera center [G].
*   **Visibility Mask:** 2D boolean array of the same resolution as the image, where $1$ indicates a visible pixel and $0$ indicates an occluded or inconsistent pixel [G].

**Steps:**
1.  **Reprojection:** Unproject $d_{target}$ to 3D points in the target camera frame, transform them to the source camera frame, and project onto the source image plane to obtain reprojection coordinates $\mathbf{u}_{src}$ and the corresponding source depth $z_{src}$ [S1].
2.  **Depth Sampling:** Sample the source depth value $d_{src}(\mathbf{u}_{src})$ at $\mathbf{u}_{src}$ using bilinear interpolation [G].
3.  **Depth Difference Calculation:** Compute the squared depth difference $\Delta d^2 = (d_{target}(\mathbf{u}) - z_{src})^2$ [S1].
4.  **Visibility Masking:** Set $V(\mathbf{u}) = 1$ if $\Delta d^2 < 0.001$, otherwise $V(\mathbf{u}) = 0$ [S1].
5.  **Photometric Error Computation:** Compute the L1 photometric error between $\mathbf{C}_{target}(\mathbf{u})$ and the reprojected source color $\mathbf{C}_{src}(\mathbf{u}_{src})$, weighted by $V(\mathbf{u})$ [S1].

**Formulas:**
$$
\mathcal{L}_{photo} = \frac{1}{N} \sum_{\mathbf{u}} V(\mathbf{u}) \left\| \mathbf{C}_{target}(\mathbf{u}) - \mathbf{C}_{src}(\mathbf{u}_{src}) \right\|_1
$$
where $N$ is the total number of pixels in the image [G].

**Parameters:**
*   **Threshold ($\tau$):** The maximum allowed squared depth difference for a pixel to be considered visible. A typical value is $0.001$ [S1].

**Edge Cases and Pitfalls:**
*   **Occlusion Boundaries:** Pixels near occlusion boundaries yield large $\Delta d^2$ values and are masked out, which may leave object boundaries unreconstructed if $\tau$ is too strict [G].
*   **Depth Precision:** The method is sensitive to depth map precision, as noisy estimates produce incorrect $\Delta d$ calculations [G].
*   **Coordinate Alignment:** Mixing metric depth with normalized device coordinates or failing to align depth units breaks the $\Delta d$ computation [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def visibility_based_photometric_loss(target_color, target_depth, source_color, source_depth,
                                      target_K, target_Rt, source_K, source_Rt, threshold=0.001):
    """
    Computes photometric loss only on pixels visible in both views.
    Args:
        target_color: Array (H, W, 3), target view RGB.
        target_depth: Array (H, W), target view metric depth.
        source_color: Array (H, W, 3), source view RGB.
        source_depth: Array (H, W), source view metric depth.
        target_K: (3, 3) target camera intrinsics.
        target_Rt: (3, 4) target camera extrinsics [R|t].
        source_K: (3, 3) source camera intrinsics.
        source_Rt: (3, 4) source camera extrinsics [R|t].
        threshold: Float, squared depth difference threshold for visibility.
    Returns:
        Float, the weighted L1 photometric loss.
    """
    H, W = target_depth.shape
    
    # 1. Unproject target depth to 3D points in target camera frame
    u, v = np.meshgrid(np.arange(W), np.arange(H), indexing='ij')
    fx, fy = target_K[0, 0], target_K[1, 1]
    cx, cy = target_K[0, 2], target_K[1, 2]
    pts_tgt = np.stack([
        (u - cx) * target_depth / fx,
        (v - cy) * target_depth / fy,
        target_depth
    ], axis=-1)  # Shape (H, W, 3)

    # 2. Transform to source camera frame
    R_tgt, t_tgt = target_Rt[:, :3], target_Rt[:, 3]
    R_src, t_src = source_Rt[:, :3], source_Rt[:, 3]
    R_tgt_T = R_tgt.T
    pts_world = np.einsum('ij,hwj->hwi', R_tgt_T, pts_tgt) + t_tgt
    pts_src = np.einsum('ij,hwj->hwi', R_src, pts_world) + t_src
    z_src = pts_src[..., 2]

    # 3. Project to source image plane
    u_src = source_K[0, 0] * pts_src[..., 0] / z_src + source_K[0, 2]
    v_src = source_K[1, 1] * pts_src[..., 1] / z_src + source_K[1, 2]

    # 4. Bilinear sampling helper
    def bilinear_sample(grid, u, v):
        u0, v0 = np.floor(u).astype(int), np.floor(v).astype(int)
        u1, v1 = u0 + 1, v0 + 1
        u0 = np.clip(u0, 0, W-1); v0 = np.clip(v0, 0, H-1)
        u1 = np.clip(u1, 0, W-1); v1 = np.clip(v1, 0, H-1)
        w00 = (u1 - u) * (v1 - v)
        w01 = (u1 - u) * (v - v0)
        w10 = (u - u0) * (v1 - v)
        w11 = (u - u0) * (v - v0)
        return (w00 * grid[v0, u0] + w01 * grid[v0, u1] +
                w10 * grid[v1, u0] + w11 * grid[v1, u1])

    d_src_sampled = bilinear_sample(source_depth, u_src, v_src)
    c_src_sampled = bilinear_sample(source_color, u_src, v_src)

    # 5. Visibility mask and loss
    depth_diff_sq = (target_depth - z_src) ** 2
    visibility_mask = (depth_diff_sq < threshold).astype(float)
    color_diff = np.abs(target_color - c_src_sampled)
    loss_per_pixel = np.sum(color_diff, axis=-1)
    weighted_loss = loss_per_pixel * visibility_mask
    N = H * W
    # Normalize by total pixels N; normalizing by visible pixels is an alternative [G]
    return np.sum(weighted_loss) / N
```

## Subtopics

### [Transient Object Handling via Visibility Map](3789-transient-object-handling-via-visibility-map.md)
This method mitigates reconstruction artifacts caused by transient objects (e.g., moving pedestrians or cars) in unconstrained image collections by using a learned 2D visibility map to weight the photometric loss. It sets itself apart from static scene reconstruction by explicitly segmenting dynamic regions to prevent the model from attempting to reconstruct non-static geometry, which typically leads to floating artifacts or depth map inconsistencies. It builds on [Visibility-Based Photometric Loss](2917-visibility-based-photometric-loss.md) by introducing a learnable mask rather than relying solely on geometric consistency or fixed thresholds.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels, as it involves a single pass over the image for reprojection and loss computation [G].
*   **Memory Complexity:** $O(N)$ for storing intermediate depth and color maps [G].
*   **Strengths:** Effectively reduces noise and artifacts from occlusions and non-Lambertian surfaces by ignoring inconsistent pixels [G].
*   **Limitations:** Requires accurate depth estimation or implicit field queries, and may fail in textureless regions where depth estimation is unreliable [G].
*   **When to Use:** Multi-view reconstruction where geometric consistency is available and occlusions are present [G].
*   **When Not to Use:** Single-view reconstruction or scenarios where depth information is unavailable or highly noisy [G].

## Background
The method originates from work on differentiable rendering of implicit surfaces, specifically in the context of multi-view shape reconstruction [S1]. It was introduced in the DIST paper [S1] to improve the quality of reconstructed geometry by ensuring that photometric constraints are only applied where the geometry is consistent across views [S1].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
