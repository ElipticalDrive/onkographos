# Visibility-Based Photometric Loss

Also called: Visibility Masking for Photometric Loss

## Overview
Visibility-Based Photometric Loss restricts the photometric error computation to pixels that are geometrically consistent across multiple views, thereby preventing gradient updates from occluded or occluding regions. It sets itself apart from standard photometric losses by explicitly filtering out pixels where the depth reprojection error exceeds a threshold, ensuring that the optimization only enforces color consistency on surfaces visible in both the source and target views.

## How it works
The method operates on a multi-view setup where a target view is reconstructed using information from source views. It relies on the differentiability of the rendering operator to propagate gradients through the visibility mask.

**Inputs and Outputs:**
*   **Inputs:** Source view images, source view depth maps (or implicit field queries), target view image, target view depth map (or implicit field queries), camera poses for source and target views.
*   **Outputs:** A scalar loss value representing the photometric error on visible pixels.

**Data Structures:**
*   **Depth Maps:** 2D arrays of float values representing depth along the camera ray.
*   **Visibility Mask:** 2D boolean or float array of the same resolution as the image, where $1$ indicates a visible pixel and $0$ indicates an occluded or inconsistent pixel.

**Steps:**
1.  **Reprojection:** For each pixel $\mathbf{u}$ in the target view, compute the 3D point $\mathbf{P}_{target}$ using the target view's depth $d_{target}(\mathbf{u})$ and camera intrinsics/extrinsics. Transform $\mathbf{P}_{target}$ into the coordinate system of a source view. Project this point onto the source view image plane to obtain the reprojection coordinates $\mathbf{u}_{src}$.
2.  **Depth Sampling:** Sample the depth value $d_{src}(\mathbf{u}_{src})$ from the source view's depth map at $\mathbf{u}_{src}$. If using an implicit field, query the depth along the ray from the source camera to $\mathbf{P}_{target}$.
3.  **Depth Difference Calculation:** Compute the squared depth difference $\Delta d^2$:
    $$
    \Delta d^2 = (d_{target}(\mathbf{u}) - d_{src}(\mathbf{u}_{src}))^2
    $$
    Note: Depth values must be aligned to the same coordinate system (e.g., metric depth from camera center).
4.  **Visibility Masking:** Determine the visibility mask $V(\mathbf{u})$ for each pixel:
    $$
    V(\mathbf{u}) = \begin{cases} 
    1 & \text{if } \Delta d^2 < \tau \\
    0 & \text{otherwise}
    \end{cases}
    $$
    where $\tau$ is the visibility threshold.
5.  **Photometric Error Computation:** Compute the L1 photometric error between the target view color $\mathbf{C}_{target}(\mathbf{u})$ and the reprojected source view color $\mathbf{C}_{src}(\mathbf{u}_{src})$, weighted by the visibility mask:
    $$
    \mathcal{L}_{photo} = \frac{1}{N} \sum_{\mathbf{u}} V(\mathbf{u}) \left\| \mathbf{C}_{target}(\mathbf{u}) - \mathbf{C}_{src}(\mathbf{u}_{src}) \right\|_1
    $$
    where $N$ is the total number of pixels.

**Parameters:**
*   **Threshold ($\tau$):** The maximum allowed squared depth difference for a pixel to be considered visible. A typical value is $0.001$ [S1].

**Edge Cases and Pitfalls:**
*   **Occlusion Boundaries:** Pixels near occlusion boundaries may have large depth differences and be masked out, potentially leading to incomplete reconstruction of object boundaries if the threshold is too strict.
*   **Depth Precision:** The method is sensitive to depth map precision. Noisy depth estimates can lead to incorrect visibility masking.
*   **Coordinate Alignment:** Incorrect alignment of depth values (e.g., mixing metric depth with normalized device coordinates) will result in incorrect $\Delta d$ calculations.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def visibility_based_photometric_loss(target_color, target_depth, source_color, source_depth, 
                                    target_cam, source_cam, threshold=0.001):
    """
    Computes photometric loss only on pixels visible in both views.
    
    Args:
        target_color: Tensor of shape (H, W, 3), target view RGB.
        target_depth: Tensor of shape (H, W), target view depth.
        source_color: Tensor of shape (H, W, 3), source view RGB.
        source_depth: Tensor of shape (H, W), source view depth.
        target_cam: Camera object for target view (intrinsics, extrinsics).
        source_cam: Camera object for source view (intrinsics, extrinsics).
        threshold: Float, squared depth difference threshold for visibility.
        
    Returns:
        Float, the weighted L1 photometric loss.
    """
    # 1. Get 3D points from target depth
    # Assuming target_depth is metric depth from camera center
    target_points_3d = unproject_depth(target_depth, target_cam) # Shape (H, W, 3)
    
    # 2. Reproject to source view
    # Transform points to source camera coordinate system
    source_points_3d = transform_to_source(target_points_3d, target_cam, source_cam)
    
    # Project to source image plane
    source_uv, source_z = project_to_image(source_points_3d, source_cam) # Shape (H, W, 2), (H, W)
    
    # 3. Sample source depth at reprojection coordinates
    # Use bilinear interpolation for differentiability
    source_depth_sampled = bilinear_sample(source_depth, source_uv) # Shape (H, W)
    
    # 4. Compute depth difference
    # Ensure depths are in the same coordinate system (e.g., metric)
    depth_diff_sq = (target_depth - source_z) ** 2
    
    # 5. Create visibility mask
    visibility_mask = (depth_diff_sq < threshold).float() # Shape (H, W)
    
    # 6. Sample source color at reprojection coordinates
    source_color_sampled = bilinear_sample(source_color, source_uv) # Shape (H, W, 3)
    
    # 7. Compute L1 loss
    color_diff = torch.abs(target_color - source_color_sampled) # Shape (H, W, 3)
    loss_per_pixel = torch.sum(color_diff, dim=2) # Shape (H, W)
    
    # Weight by visibility mask
    weighted_loss = loss_per_pixel * visibility_mask
    
    # Normalize by total number of pixels (or visible pixels, depending on convention)
    # Using total pixels as per standard L1 normalization in [S1]
    loss = torch.sum(weighted_loss) / (target_depth.shape[0] * target_depth.shape[1])
    
    return loss
```

## Subtopics

### [Transient Object Handling via Visibility Map](3789-transient-object-handling-via-visibility-map.md)
This method mitigates reconstruction artifacts caused by transient objects (e.g., moving pedestrians or cars) in unconstrained image collections by using a learned 2D visibility map to weight the photometric loss. It sets itself apart from static scene reconstruction by explicitly segmenting dynamic regions to prevent the model from attempting to reconstruct non-static geometry, which typically leads to floating artifacts or depth map inconsistencies. It builds on [Visibility-Based Photometric Loss](2917-visibility-based-photometric-loss.md) by introducing a learnable mask rather than relying solely on geometric consistency or fixed thresholds.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels, as it involves a single pass over the image for reprojection and loss computation.
*   **Memory Complexity:** $O(N)$ for storing intermediate depth and color maps.
*   **Strengths:** Effectively reduces noise and artifacts from occlusions and non-Lambertian surfaces by ignoring inconsistent pixels.
*   **Limitations:** Requires accurate depth estimation or implicit field queries. May fail in textureless regions where depth estimation is unreliable.
*   **When to Use:** Multi-view reconstruction where geometric consistency is available and occlusions are present.
*   **When Not to Use:** Single-view reconstruction or scenarios where depth information is unavailable or highly noisy.

## Background
The method originates from work on differentiable rendering of implicit surfaces, specifically in the context of multi-view shape reconstruction. It was introduced in the DIST paper [S1] to improve the quality of reconstructed geometry by ensuring that photometric constraints are only applied where the geometry is consistent across views.

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
