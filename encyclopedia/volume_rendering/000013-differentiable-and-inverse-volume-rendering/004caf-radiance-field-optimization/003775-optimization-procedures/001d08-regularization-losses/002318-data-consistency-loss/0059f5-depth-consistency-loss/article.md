# Depth Consistency Loss
## Overview
Depth Consistency Loss (LDC) is a regularization term in differentiable volume rendering that enforces geometric accuracy by aligning the zero level set of an implicit field with ground truth depth observations [S1]. Unlike volume rendering depth losses that constrain all samples integrated along a ray, LDC specifically targets surface points, ensuring their projected depth matches the observed depth map [S1].
## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $s$: A set of surface points $p'_j$ located on the zero level set of the implicit field [S1].
    *   $D^{GT}_k$: Ground truth depth map for view $k$ [S1].
    *   $K_k, T_k$: Intrinsic matrix and pose (extrinsic matrix) for view $k$ [S1].
    *   $w_j$: Binary mask weights for each point [S1].
*   **Outputs:**
    *   $L_{DC}$: Scalar loss value [S1].

**Data Structures**
*   Points $p'_j$ are 3D coordinates $(x, y, z)$ representing locations on the zero level set [S1].
*   Depth maps $D^{GT}_k$ are 2D arrays of float values representing depth per pixel [S1].

**Parameters**
*   $\epsilon_{\text{max}}$: Maximum allowable depth discrepancy threshold. The sources specify a value of $15$ mm [S1].
*   Interpolation method: Bilinear interpolation is used to sample depth values from the ground truth map [S1].

**Steps**
1.  For each surface point $p'_j \in s$:
    a. Project $p'_j$ into the camera coordinate system of view $k$ using the pose $T_k$ and intrinsic matrix $K_k$ [S1].
    b. Compute the projected depth $Z_k(p'_j)$, which corresponds to the depth along the viewing ray in the camera frame [G].
    c. Map the projected 2D coordinates to the pixel grid of the depth map $D^{GT}_k$ [S1].
    d. Retrieve the ground truth depth $D^{GT}_k(p'_j)$ by bilinearly interpolating the depth map at the projected 2D coordinates [S1].
    e. Determine the mask weight $w_j$: set $w_j = 0$ if the projection is outside the image boundaries or if $|D^{GT}_k(p'_j) - Z_k(p'_j)| > 15$ mm (indicating potential invisibility); otherwise set $w_j = 1$ [S1].
2.  Compute the squared error for valid points and sum them weighted by $w_j$:
    $$ L_{DC} = \sum_{p'_j \in s} w_j \times (D^{GT}_k(p'_j) - Z_k(p'_j))^2 $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_consistency_loss(surface_points, gt_depth_map, intrinsic_matrix, pose_matrix, max_depth_diff_mm=15.0):
    """
    Computes the Depth Consistency Loss (LDC).
    
    Args:
        surface_points: List of [x, y, z] in world coordinates.
        gt_depth_map: 2D list/array of shape (H, W) containing depth values.
        intrinsic_matrix: 3x3 list of lists K.
        pose_matrix: 4x4 list of lists T (world-to-camera).
        max_depth_diff_mm: Threshold in millimeters for masking invalid points.
        
    Returns:
        Scalar loss value.
    """
    loss = 0.0
    h, w = len(gt_depth_map), len(gt_depth_map[0])
    
    for p in surface_points:
        # Homogeneous coordinate
        p_hom = p + [1.0]
        
        # World to camera transformation
        p_cam = [0.0, 0.0, 0.0]
        for i in range(3):
            for j in range(4):
                p_cam[i] += pose_matrix[i][j] * p_hom[j]
        
        z_cam = p_cam[2]
        if z_cam <= 0.0:
            continue
            
        # Project to 2D pixel coordinates
        uv = [0.0, 0.0, 0.0]
        for i in range(3):
            for j in range(3):
                uv[i] += intrinsic_matrix[i][j] * p_cam[j]
        
        u = uv[0] / uv[2]
        v = uv[1] / uv[2]
        
        # Check bounds
        if u < 0 or u >= w - 1 or v < 0 or v >= h - 1:
            continue
            
        # Bilinear interpolation
        u0, v0 = int(u), int(v)
        u1, v1 = u0 + 1, v0 + 1
        du, dv = u - u0, v - v0
        
        d00 = gt_depth_map[v0][u0]
        d01 = gt_depth_map[v0][u1]
        d10 = gt_depth_map[v1][u0]
        d11 = gt_depth_map[v1][u1]
        
        d_gt = (1 - du) * (1 - dv) * d00 + du * (1 - dv) * d01 + \
               (1 - du) * dv * d10 + du * dv * d11
        
        # Masking: rule out if diff > 15mm
        diff = abs(d_gt - z_cam)
        if diff > max_depth_diff_mm:
            continue
            
        loss += (d_gt - z_cam) ** 2
        
    return loss
```

**Edge Cases and Pitfalls**
*   **Occlusion/Invisibility:** Points on the surface in one view may be occluded in another. The mask $w_j$ handles this by discarding points where the depth difference exceeds 15 mm, preventing the loss from pulling the surface towards incorrect depth values caused by occlusion [S1].
*   **Precision:** Bilinear interpolation is used for depth sampling. Ensure coordinate systems (world vs. camera) and units (meters vs. millimeters) are consistent between the SDF points and the depth map [G].
*   **Zero Level Set Dependency:** LDC only constrains points exactly on the zero level set [S1].

## Subtopics

### Vox-Surf Depth Loss
A variant of depth supervision that constrains Signed Distance Field (SDF) values along rays based on observed depth $t$, handling points outside, near, and inside the surface differently to improve accuracy [S2]. It defines a "near range" loss $L_{near}$ where points within $t \pm \delta t$ are constrained to have an SDF value of 0 [S2]. For points behind the observed depth ($t > \hat{t} + \delta t$), it assumes the point is inside the surface and applies an outside loss component [S2]. The near loss is calculated as $L_{near} = \sum_k F_\sigma(\Gamma(r(t)))[0]^2$, where $F_\sigma$ is the geometry extractor network, $\Gamma(r(t))$ is the voxel embedding, and $[0]$ denotes the SDF output channel [S2]. The parameter $\delta t$ controls the width of the near range and depends on the confidence of the depth observation [S2]. This method works well even if rays intersect multiple surfaces, provided sufficient observation data exists [S2].

## Operating characteristics
*   **Complexity:** Time complexity is linear with respect to the number of surface points $|s|$ and the number of views [S1]. Memory complexity is dominated by storing the surface points and the ground truth depth maps [G].
*   **Performance:** Effective for refining surface geometry in implicit volume rendering. It is computationally cheaper than volumetric depth losses because it only evaluates points on the surface rather than all samples along rays [S1].
*   **Strengths:** Directly enforces geometric consistency with sensor data; robust to occlusions via masking [S1].
*   **Limitations:** Relies on accurate identification of surface points (zero level set); sensitive to the quality of ground truth depth maps; the 15 mm threshold is a fixed hyperparameter that may not suit all scales [S1].
*   **When to use:** When high-fidelity surface reconstruction is required and ground truth depth maps are available [G].
*   **When not to use:** When depth maps are noisy or unavailable, or when the surface representation is not explicitly defined as a zero level set [G].

## Background
Depth Consistency Loss was introduced in the context of sensing surface patches in volume rendering to infer Signed Distance Functions (SDFs) [S1]. It addresses the limitation of standard volume rendering losses (like LDR) that integrate over all ray samples, by focusing specifically on the geometric accuracy of the surface boundary [S1]. The method leverages ground truth depth maps and camera poses to project surface points and enforce consistency, a technique commonly used in multi-view stereo and implicit surface reconstruction pipelines [S1].

## Sources
[S1] Sensing Surface Patches in Volume Rendering for Inferring Signed Distance Functions, https://doi.org/10.1609/aaai.v39i4.32426, fetched 2026-10-26
[S2] Vox-Surf: Voxel-Based Implicit Surface Representation, https://doi.org/10.1109/tvcg.2022.3225844, fetched 2026-10-26
