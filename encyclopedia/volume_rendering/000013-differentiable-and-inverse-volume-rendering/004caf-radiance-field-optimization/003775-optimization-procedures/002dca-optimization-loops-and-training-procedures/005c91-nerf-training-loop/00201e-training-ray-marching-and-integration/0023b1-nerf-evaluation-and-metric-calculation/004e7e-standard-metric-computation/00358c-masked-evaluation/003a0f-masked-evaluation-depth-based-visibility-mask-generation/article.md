# Masked Evaluation (Depth-based Visibility Mask Generation)

Also called: Depth reprojection masking, Invisible Mask Computation, Invisible Region Detection via Depth Warping

## Overview
This method generates a binary visibility mask to identify pixels in a reference view that are occluded (invisible) when observed from a new camera pose. It solves the problem of evaluating or inpainting regions that have no geometric correspondence in other views, which often leads to artifacts or incorrect depth predictions in radiance field optimization. It differs from sibling methods like [View-Consistency Metric](15720-view-consistency-metric-e-warp.md) by explicitly using depth warping errors to detect occlusion rather than relying solely on photometric color differences or optical flow consistency.

## How it works
**Inputs:**
- A reference view with RGB image $I_{rgb}$ and depth map $I_d$.
- A new (source) view with RGB image $I'_{rgb}$ and depth map $I'_d$.
- Camera intrinsics and extrinsics for both views.
- A trained NeRF or depth estimation model to provide initial depth values if not given directly.

**Outputs:**
- A final binary mask $m$ of the same spatial dimensions as the reference view, where $m_{ij}=1$ indicates an invisible (occluded) pixel and $m_{ij}=0$ indicates a visible pixel.
- Masked RGB image $\bar{I}_{rgb}$ and masked depth map $\bar{I}_d$ for subsequent processing (e.g., inpainting).

**Data Structures:**
- $I_d, I'_d$: 2D arrays of floating-point depth values.
- $m, m'$: 2D binary arrays (uint8 or bool).
- $v_{ij}$: Depth value at pixel $(i,j)$ in the reference view.
- $v'_{ij}$: Warped depth value at pixel $(i,j)$ from the new view.

**Steps:**
1. **Depth Warping:** For each pixel $(i,j)$ in the reference view, project the corresponding 3D point from the new view's depth map $I'_d$ into the reference view's coordinate system using the relative camera pose. Let the resulting depth value be $v'_{ij}$.
2. **Per-View Invisible Mask Generation:** Compute the intermediate mask $m'$ for the current new view using the indicator function:
   $$ m'_{ij} = \mathbb{I}(v'_{ij} - v_{ij} > \varepsilon) $$
   where $\mathbb{I}(\cdot)$ is the indicator function (1 if true, 0 otherwise). If the warped depth $v'_{ij}$ is significantly larger than the reference depth $v_{ij}$, the point is considered occluded in the reference view because the surface seen in the new view is "behind" the surface seen in the reference view.
3. **Final Mask Aggregation:** If multiple new views are used, compute $m'$ for each. The final mask $m$ is the logical AND of all intermediate masks:
   $$ m = \bigwedge_{k} m'_k $$
   A pixel is marked invisible only if it is occluded in all compared new views.
4. **Masking Inputs:** Apply the final mask to the reference RGB and depth images:
   $$ \bar{I}_{rgb} = I_{rgb} \otimes (1 - m) $$
   $$ \bar{I}_d = I_d \otimes (1 - m) $$
   where $\otimes$ denotes element-wise multiplication.

**Parameters:**
- $\varepsilon$: A small positive threshold value to allow for minor warping errors or depth estimation noise. Typical values are not specified in the source but should be small relative to the scene scale [S1].

**Edge Cases and Pitfalls:**
- **Precision:** Depth values must be consistent in units and coordinate systems. Misalignment in camera poses will cause false occlusions.
- **Degenerate Input:** If $I_d$ or $I'_d$ contains invalid values (e.g., infinity or zero), they must be handled or filtered out before warping.
- **Boundary Conditions:** Pixels at the image border may not have valid correspondences in the new view. These should be treated as invisible or masked out.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_visibility_mask(reference_depth, new_depth, ref_intrinsics, new_intrinsics, ref_pose, new_pose, epsilon):
    """
    Generates an invisible mask for a reference view based on depth warping from a new view.
    
    Args:
        reference_depth (np.ndarray): 2D array of depth values for the reference view.
        new_depth (np.ndarray): 2D array of depth values for the new view.
        ref_intrinsics (np.ndarray): 3x3 camera intrinsic matrix for the reference view.
        new_intrinsics (np.ndarray): 3x3 camera intrinsic matrix for the new view.
        ref_pose (np.ndarray): 4x4 camera pose matrix for the reference view.
        new_pose (np.ndarray): 4x4 camera pose matrix for the new view.
        epsilon (float): Threshold for depth difference.
        
    Returns:
        np.ndarray: Binary mask where 1 indicates invisible (occluded) pixels.
    """
    # Get image dimensions
    height, width = reference_depth.shape
    
    # Create grid of pixel coordinates
    y, x = np.meshgrid(np.arange(height), np.arange(width), indexing='ij')
    
    # Convert pixel coordinates to homogeneous coordinates in reference view
    ref_pixels = np.stack([x, y, np.ones_like(x)], axis=-1).astype(np.float32)
    
    # Unproject reference pixels to 3D points using reference depth and intrinsics
    # P_ref = K_ref^-1 * [x, y, 1]^T * d_ref
    ref_intrinsics_inv = np.linalg.inv(ref_intrinsics)
    ref_homogeneous = ref_pixels @ ref_intrinsics_inv.T  # Shape: (H, W, 3)
    ref_3d_points = ref_homogeneous * reference_depth[:, :, np.newaxis]  # Shape: (H, W, 3)
    
    # Transform 3D points to new view's coordinate system
    # P_new = R * P_ref + t, where [R|t] is the relative pose from ref to new
    # Relative pose: new_pose @ inv(ref_pose)
    rel_pose = new_pose @ np.linalg.inv(ref_pose)
    R = rel_pose[:3, :3]
    t = rel_pose[:3, 3]
    
    # Apply rotation and translation
    # Note: ref_3d_points is (H, W, 3), need to handle batched operation
    rotated_points = ref_3d_points @ R.T
    transformed_points = rotated_points + t[np.newaxis, np.newaxis, :]
    
    # Project transformed 3D points to new view's image plane
    # p_new = K_new * P_new
    new_homogeneous = transformed_points @ ref_intrinsics_inv.T # Wait, should use new_intrinsics
    # Correction: Project to new view
    new_homogeneous = transformed_points @ new_intrinsics.T
    new_u = new_homogeneous[:, :, 0] / new_homogeneous[:, :, 2]
    new_v = new_homogeneous[:, :, 1] / new_homogeneous[:, :, 2]
    new_z = new_homogeneous[:, :, 2]
    
    # Check if projected points are within image bounds
    valid_mask = (new_u >= 0) & (new_u < width) & (new_v >= 0) & (new_v < height) & (new_z > 0)
    
    # Interpolate depth from new_depth at (new_u, new_v)
    # Using bilinear interpolation
    warped_depth = interpolate_depth(new_depth, new_u, new_v)
    
    # Handle invalid projections
    warped_depth[~valid_mask] = np.inf
    
    # Compute depth difference
    depth_diff = warped_depth - reference_depth
    
    # Generate mask: 1 if warped depth is significantly larger (occluded)
    invisible_mask = (depth_diff > epsilon).astype(np.float32)
    
    return invisible_mask

def interpolate_depth(depth_map, u, v):
    """
    Bilinear interpolation of depth map at sub-pixel coordinates.
    """
    # Clamp coordinates to valid range
    u_clamped = np.clip(u, 0, depth_map.shape[1] - 1)
    v_clamped = np.clip(v, 0, depth_map.shape[0] - 1)
    
    # Get integer coordinates
    u0 = np.floor(u_clamped).astype(int)
    v0 = np.floor(v_clamped).astype(int)
    u1 = u0 + 1
    v1 = v0 + 1
    
    # Handle boundary conditions
    u1 = np.clip(u1, 0, depth_map.shape[1] - 1)
    v1 = np.clip(v1, 0, depth_map.shape[0] - 1)
    
    # Get depth values at corners
    d00 = depth_map[v0, u0]
    d01 = depth_map[v0, u1]
    d10 = depth_map[v1, u0]
    d11 = depth_map[v1, u1]
    
    # Interpolate
    du = u_clamped - u0
    dv = v_clamped - v0
    
    d0 = d00 * (1 - du) + d01 * du
    d1 = d10 * (1 - du) + d11 * du
    d = d0 * (1 - dv) + d1 * dv
    
    return d
```

## Subtopics

### [Occlusion-Aware Masking](8889-occlusion-aware-masking.md)
Generates a consistency mask $M_l$ to identify pixels with geometric inconsistencies during feature-level regularization. It distinguishes itself by explicitly comparing rendered depth maps between views to detect warping errors, rather than relying solely on photometric color differences or optical flow consistency.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of pixels in the reference view and the number of new views used for comparison. Depth warping and interpolation are the dominant operations.
- **Memory Complexity:** Linear with respect to the number of pixels, storing depth maps and masks.
- **Strengths:** Explicitly handles occlusions by leveraging geometric consistency. Effective for identifying regions that lack multi-view support.
- **Limitations:** Relies on accurate depth estimation or rendering. Sensitive to camera pose errors and depth noise. Does not account for textureless regions where depth estimation might be unreliable.
- **When to Use:** When inpainting or regularizing radiance fields in regions that are occluded in some views but visible in others.
- **When Not to Use:** When depth information is unavailable or highly unreliable. When computational resources are extremely limited and a simpler photometric consistency check suffices.

## Background
The method originates from the PERF (Panoramic Neural Radiance Field) paper, which addresses the challenge of completing large-size occlusions in complex 3D panoramic scenes using collaborative RGBD inpainting [S1]. It was developed to overcome the limitations of adversarial learning and high-level semantic constraints, which fail to fill large invisible regions effectively.

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
