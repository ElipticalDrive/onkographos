# 3D Point Projection to 2D

Also called: Image Space Projection, Modelview-Projection Transformation, OpenGL Projection, Point Projection

## Overview
Projects 3D points into multiple camera views to compute 2D pixel locations and visibility masks [S1, unofficial]. It differs from [Perspective Projection of Points](4037-perspective-projection-of-points.md) by explicitly handling batched multi-view projection and visibility masking for rendering pipelines, and from [Depth Estimation from Point Cloud](8043-depth-estimation-from-point-cloud.md) by focusing on coordinate mapping rather than depth map generation or occlusion resolution [S1, unofficial].

## How it works
**Inputs:**
- `xyz`: Tensor of 3D points, shape $[N, 3]$.
- `train_cameras`: Tensor of camera parameters, shape $[V, 34]$, where $V$ is the number of views. The 34 values consist of 2 image size dimensions, 16 intrinsic parameters, and 16 extrinsic parameters [S1, unofficial].

**Outputs:**
- `pixel_locations`: Tensor of 2D pixel coordinates, shape $[V, N, 2]$.
- `visibility_mask`: Boolean tensor, shape $[V, N]$, indicating if points are in front of the camera [S1, unofficial].

**Data Structures:**
- `train_intrinsics`: $V \times 4 \times 4$ tensor extracted from indices 2 to 18 of `train_cameras` [S1, unofficial].
- `train_poses`: $V \times 4 \times 4$ tensor extracted from indices -16 to end of `train_cameras` [S1, unofficial].
- `xyz_h`: $N \times 4$ homogeneous point tensor with a trailing column of ones [S1, unofficial].
- `P`: $V \times 4 \times 4$ projection matrix computed as `train_intrinsics @ inverse(train_poses)` [S1, unofficial].
- `projections`: $V \times N \times 4$ tensor of transformed homogeneous coordinates [S1, unofficial].

**Steps:**
1. **Parse Camera Parameters:** Split `train_cameras` into intrinsics and poses. Reshape indices 2:18 into $V \times 4 \times 4$ matrices for intrinsics, and indices -16: into $V \times 4 \times 4$ matrices for poses [S1, unofficial].
2. **Homogenize Points:** Append a column of ones to `xyz` to form $x_h = [x, y, z, 1]^T$, resulting in shape $[N, 4]$ [S1, unofficial].
3. **Compute Projection Matrix:** Invert the pose matrices and multiply: $P = K_{4\times4} \cdot W^{-1}_{4\times4}$, yielding shape $[V, 4, 4]$ [S1, unofficial].
4. **Apply Projection:** Transpose $x_h$ to $[4, N]$, repeat it across the view dimension to $[V, 4, N]$, and compute $proj = P \cdot x_h$, resulting in $[V, 4, N]$. Transpose to $[V, N, 4]$ [S1, unofficial].
5. **Perspective Division:** Extract the first two coordinates ($x, y$) and the third coordinate ($z$). Clamp $z$ to a minimum of $10^{-8}$ to prevent division by zero, then compute $pixel\_x = x / z$ and $pixel\_y = y / z$ [S1, unofficial].
6. **Visibility Mask:** Mark points as visible if their depth $z > 0$ (in front of the camera plane) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def project_points_to_2d(xyz, train_cameras):
    """
    Projects 3D points to 2D pixel coordinates for multiple camera views.
    
    Args:
        xyz: Tensor of shape [N, 3] containing 3D points.
        train_cameras: Tensor of shape [V, 34] containing camera parameters.
                       Format: [img_w, img_h, 16 intrinsics, 16 extrinsics].
    
    Returns:
        pixel_locations: Tensor of shape [V, N, 2] with (x, y) coordinates.
        visibility_mask: Tensor of shape [V, N] with boolean visibility flags.
    """
    num_views = train_cameras.shape[0]
    
    # 1. Parse Camera Parameters
    train_intrinsics = train_cameras[:, 2:18].reshape(num_views, 4, 4)
    train_poses = train_cameras[:, -16:].reshape(num_views, 4, 4)
    
    # 2. Homogenize Points
    xyz_h = torch.cat([xyz, torch.ones_like(xyz[:, :1])], dim=-1)
    
    # 3. Compute Projection Matrix
    P = train_intrinsics @ torch.inverse(train_poses)
    
    # 4. Apply Projection
    xyz_h_batch = xyz_h.t().unsqueeze(0).repeat(num_views, 1, 1)
    projections = torch.bmm(P, xyz_h_batch).permute(0, 2, 1)
    
    # 5. Perspective Division
    z_clamped = torch.clamp(projections[..., 2:3], min=1e-8)
    pixel_locations = projections[..., :2] / z_clamped
    
    # 6. Visibility Mask
    visibility_mask = projections[..., 2] > 0
    
    return pixel_locations, visibility_mask
```

**Parameters:**
- `train_cameras`: Tensor of shape $[V, 34]$. The first two values are image width and height. The next 16 values encode the intrinsic matrix as a flattened $4 \times 4$ block. The last 16 values encode the extrinsic pose matrix as a flattened $4 \times 4$ block [S1, unofficial].

**Edge Cases:**
- **Division by Zero:** If $z=0$, the point lies on the camera plane. Clamping $z$ to $10^{-8}$ prevents NaNs during perspective division [S1, unofficial].
- **Negative Depth:** Points behind the camera ($z < 0$) are excluded by the visibility mask [S1, unofficial].

## Subtopics

### [Perspective Projection of Points](4037-perspective-projection-of-points.md)
Perspective projection maps 3D world points to 2D image coordinates using a pinhole camera model, where a line joining the 3D point to the center of projection intersects the image plane. It solves the problem of determining the 2D pixel location and visibility of 3D geometry for a specific camera view. It differs from orthographic projection by incorporating depth-dependent scaling (perspective division) and from stereo projection by generating a single view frustum rather than correlated pairs.

### [Depth Estimation from Point Cloud](8043-depth-estimation-from-point-cloud.md)
This method generates a depth image at a novel viewpoint by projecting a pre-existing 3D point cloud onto the camera's image plane. It solves the problem of converting geometric samples into a per-pixel depth map, handling occlusions by selecting the nearest surface point for each pixel. The method builds on [3D Point Projection to 2D](3148-3d-point-projection-to-2d.md) and is described in the context of improving Neural Radiance Fields using near-surface sampling with point cloud generation.

### Reprojection of 2D Features to 3D Space
For a query 3D point $p$, an observed 2D view $I_m$, camera pose $\xi_m$, and intrinsics $K_m$, the corresponding 2D pixel features $F^m_p$ are retrieved using the reprojection function $P$ [S3]. If the projected 3D point lies inside the image boundaries, the nearest pixel's features are selected and duplicated to the 3D point [S3]. If the point lies outside the image boundaries, a zero vector is assigned to the 3D point, indicating no observed information [S3]. Nearest neighbor interpolation is used instead of bilinear interpolation or kernel functions; nearest neighbor is simpler, though advanced techniques may address discretization issues [S3].

### Depth-Guided Projection
This variant calculates a 3D point cloud $p_i$ associated with view $i$ using its estimated depth map $D_i$, intrinsic parameters $K_i$, and extrinsic parameters $\xi_i$ via $p_i = P^{-1}(D_i, K_i, \xi_i)$ [S4]. The point cloud is projected to view $j$ to find corresponding pixels, where the color of pixels projected from $I_j$ onto $I_i$ is computed as $I_{i,j} = P(I_j, K_j, \xi_j, p_i)$ [S4]. Reprojection error is computed to filter out invalid pairs and mitigate occlusion effects [S4].

### PPC Colorization via Forward Projection
For a point $X$ in 3D space with world coordinates $X_w$, forward projection transforms it to camera coordinates $X_c$ using the 4x4 extrinsic matrix $d$ (containing translation $t$ and rotation $R$) as $\tilde{X}_c = d\tilde{X}_w$, where tildes denote homogeneous coordinates [S5]. 2D pixel coordinates $x_c$ are obtained by normalizing homogeneous image coordinates $\tilde{x}_c = K\tilde{X}_c$ by depth, using the known camera intrinsic matrix $K$ [S5]. The center of each occupied voxel is used as the starting position to reduce aliasing artifacts [S5]. Occlusion is identified by grouping points projecting to the same pixel $(u,v)$ into set $P_{uv}$, marking the point with the minimum $z$-value as non-occluded, and filtering the rest [S5]. Occluded voxels are filled with the average color of all non-occluded views for the same point, and an 'RGB main' triplet is created for renderers without plenoptic color support [S5].

## Operating characteristics
**Time Complexity:** $O(V \cdot N)$, dominated by batched matrix multiplication and element-wise division [G].
**Memory Complexity:** $O(V \cdot N)$ to store the output pixel locations and visibility masks, with intermediate tensors scaling linearly [G].
**Strengths:** Efficient batched computation suitable for GPU acceleration. Simple visibility masking handles front/back face culling implicitly [S1, unofficial].
**Limitations:** Does not resolve occlusion between points (multiple points may project to the same pixel) [S5]. Does not interpolate features; uses nearest neighbor which may cause aliasing [S3].
**When to use:** When projecting a static set of 3D points to multiple camera views for rendering or feature retrieval [S1, unofficial].
**When not to use:** When depth ordering or occlusion resolution is required (use [Depth Estimation from Point Cloud](8043-depth-estimation-from-point-cloud.md)). When smooth feature interpolation is needed (use bilinear interpolation variants) [S3].

## Background
Originated in computer graphics and multi-view geometry for rendering and feature extraction. The specific implementation details (tensor shapes, parameter layout) are derived from neural rendering pipelines like VF-NeRF [S1, unofficial]. The concept of reprojection for feature aggregation is discussed in General Radiance Field (GRF) literature [S3]. Depth-guided projection and reprojection error filtering are applied in ViCA-NeRF for view-consistency-aware 3D editing [S4]. Forward projection with occlusion masking for Plenoptic Point Clouds is detailed in comparative assessments of implicit and explicit scene representations [S5].

## Sources
[S1] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
[S2] Image-Space Tensor Field Visualization Using a LIC-like Method, https://doi.org/10.1007/978-3-642-21608-4_11, fetched 2026-10-27 (unofficial)
[S3] GRF: Learning a General Radiance Field for 3D Representation and Rendering, https://doi.org/10.1109/iccv48922.2021.01490, fetched 2026-10-26
[S4] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
[S5] A Comparative Assessment Of Implicit and Explicit Plenoptic Scene Representations, https://inria.hal.science/hal-04709909v1, fetched 2026-10-26
