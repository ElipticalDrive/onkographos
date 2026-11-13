# Projective TSDF Fusion

Also called: Depth-based TSDF Integration, TSDF Fusion from Depth Images

## Overview
Projective TSDF Fusion computes signed distance values for voxels in a global volume by projecting voxel centers into the camera frame of calibrated depth images and comparing the projected depth with the observed depth [S1]. It solves the problem of integrating multi-view depth measurements into a coherent volumetric representation, specifically handling the coordinate transformations required to map 3D geometry to 2D sensor data [S1]. Unlike methods that rely on explicit surface meshes or point clouds, this approach operates directly on raw depth buffers, making it suitable for real-time systems like KinectFusion [S1].

## How it works
**Inputs:**
- A global voxel grid with centers $\mathbf{p}$ in world coordinates.
- A set of calibrated depth images $D(u, v)$ with pinhole intrinsics $\mathbf{f} = \{f_x, f_y, c_x, c_y\}$ and camera poses $(R, t)$, where $R$ is the rotation matrix and $t$ is the translation vector.
- A truncation limit $\tau$ (not explicitly defined in [S1] but implied by validity checks).

**Outputs:**
- A global TSDF volume where each voxel stores a fused signed distance value.

**Data Structures:**
- **Voxel Center:** $\mathbf{p} \in \mathbb{R}^3$, world coordinates.
- **Camera Frame Coordinates:** $\mathbf{p}_c = (x, y, z)^\top \in \mathbb{R}^3$.
- **Image Coordinates:** $(u, v) \in \mathbb{R}^2$.
- **Depth Value:** $\hat{z} \in \mathbb{R}$.

**Steps:**

1. **Transform to Camera Frame:** For each voxel center $\mathbf{p}$ in the global frame, compute its coordinates in the camera frame $\mathbf{p}_c$ using the inverse rigid body transformation [S1]:
   $$
   \mathbf{p}_c = R^\top (\mathbf{p} - \mathbf{t}) = (x, y, z)^\top
   $$
   Here, $R$ is the rotation matrix and $t$ is the translation vector of the camera relative to the world frame [S1].

2. **Project to Image Plane:** Project the camera-frame coordinates $(x, y, z)$ to 2D image coordinates $(u, v)$ using the pinhole camera model [S1]:
   $$
   u = f_x \frac{x}{z} + c_x, \quad v = f_y \frac{y}{z} + c_y
   $$
   Here, $f_x, f_y$ are the focal lengths and $c_x, c_y$ are the principal points [S1].

3. **Validate Projection:** Check if the projection is valid. The measurement is valid only if:
   - $z > 0$ (voxel is in front of the camera).
   - $(u, v)$ lies inside the image boundaries.
   - The depth image $D(u, v)$ contains a valid depth value [S1].

4. **Compute Signed Distance:** If valid, read the observed depth $\hat{z} = D(u, v)$ and compute the projective signed distance $s(\mathbf{p})$ [S1]:
   $$
   s(\mathbf{p}) = \hat{z} - z
   $$
   Here, $\hat{z}$ is the observed depth at pixel $(u, v)$ and $z$ is the depth of the voxel center in the camera frame [S1].

5. **Truncation Check:** The measurement is considered valid for fusion only if the absolute signed distance satisfies $|s(\mathbf{p})| \le \tau$. The specific value of $\tau$ is not given in [S1] but is standard in TSDF fusion [G].

6. **Fusion:** With multiple calibrated views, each view produces a per-voxel TSDF measurement. Valid measurements are fused into the global TSDF volume by weighted averaging [S1, S2]. The weights are typically proportional to the confidence or inverse variance of the depth measurement, though [S1] specifies "weighted averaging" without defining the specific weight function [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def projective_tsdf_fusion(voxel_centers, depth_images, poses, intrinsics, truncation_limit):
    """
    voxel_centers: List of 3D points p in world coordinates.
    depth_images: List of depth maps D(u, v).
    poses: List of (R, t) tuples representing camera poses.
    intrinsics: Tuple (fx, fy, cx, cy).
    truncation_limit: Scalar tau.
    Returns: Dictionary mapping voxel index to fused TSDF value.
    """
    fx, fy, cx, cy = intrinsics
    fused_tsdf = {} # Map from voxel_index to (sum_weighted_dist, sum_weights)

    for i, (R, t) in enumerate(poses):
        D = depth_images[i]
        height, width = D.shape
        
        # Inverse rotation and translation for camera frame
        R_inv = R.T
        t_inv = -R_inv @ t

        for idx, p in enumerate(voxel_centers):
            # Step 1: Transform to camera frame
            p_c = R_inv @ p + t_inv
            x, y, z = p_c

            # Step 2: Project to image plane
            if z <= 0:
                continue
            
            u = fx * (x / z) + cx
            v = fy * (y / z) + cy

            # Step 3: Validate projection
            if not (0 <= u < width and 0 <= v < height):
                continue
            
            # Check for valid depth (e.g., non-zero, non-NaN)
            if not is_valid_depth(D[v, u]):
                continue

            # Step 4: Compute signed distance
            z_hat = D[v, u]
            s = z_hat - z

            # Step 5: Truncation check
            if abs(s) > truncation_limit:
                continue

            # Step 6: Fusion (Weighted Average)
            # Weight is often 1.0 or based on depth confidence. 
            # [S1] specifies weighted averaging but does not define the weight.
            # Using uniform weight 1.0 as a standard default [G].
            weight = 1.0
            
            if idx not in fused_tsdf:
                fused_tsdf[idx] = (0.0, 0.0)
            
            current_sum, current_weight = fused_tsdf[idx]
            fused_tsdf[idx] = (current_sum + weight * s, current_weight + weight)

    # Normalize by total weight
    final_tsdf = {}
    for idx, (sum_dist, sum_w) in fused_tsdf.items():
        if sum_w > 0:
            final_tsdf[idx] = sum_dist / sum_w
        else:
            final_tsdf[idx] = 0.0 # Or undefined
            
    return final_tsdf
```

**Edge Cases and Pitfalls:**
- **Invalid Depth:** Pixels with no depth measurement (e.g., occlusions, reflective surfaces) must be skipped [S1].
- **Z-Buffer Clipping:** Voxels behind the camera ($z \le 0$) are invalid [S1].
- **Truncation:** The choice of $\tau$ affects the smoothness and completeness of the surface. Too small a value leads to noisy surfaces; too large a value leads to thick, inaccurate geometry [G].
- **Weighting:** The specific weighting scheme for fusion is not detailed in [S1]. Common practices include weighting by depth confidence or inverse variance [G].

## Subtopics

### [Single Layer TSDF Update](13744-single-layer-tsdf-update.md)
The process of computing a signed distance field (SDF) sample for a voxel and integrating it into the global map using a weighted average, handling dynamic scale allocation. This variant focuses on the incremental update mechanism rather than the full multi-view fusion pipeline. It is chosen when memory constraints require dynamic allocation or when processing streams of depth data sequentially. See [Single Layer TSDF Update](13744-single-layer-tsdf-update.md).

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels and the number of depth images. Each voxel is transformed and projected once per view.
- **Memory Complexity:** Linear with respect to the number of voxels in the global volume.
- **Strengths:** Directly uses depth sensor data; robust to noise through averaging; suitable for real-time applications (e.g., KinectFusion) [S1].
- **Limitations:** Requires accurate camera calibration and pose estimation; sensitive to depth errors; truncation limits the range of valid distances [S1].
- **When to Use:** When integrating depth images from calibrated cameras into a volumetric representation.
- **When Not to Use:** When input is explicit geometry (meshes/point clouds) without depth buffers; when high precision is required beyond the truncation limit.

## Background
Projective TSDF fusion was introduced in volumetric fusion systems such as KinectFusion [Newcombe et al. 2011] and voxel hashing systems [Nießner et al. 2013] [S1]. It builds on the foundational work of Curless and Levoy (1996) for volumetric integration [S1]. The method is widely used in real-time 3D reconstruction and robotics applications where depth sensors are the primary input.

## Sources
[S1] Implicit Surface Compression -- with Good Old Discrete Cosine Transform and Motion Compensation, https://doi.org/10.1145/3811298, fetched 2026-10-30
[S2] Compressed Voxel-Based Mapping Using Unsupervised Learning, https://doi.org/10.3390/robotics6030015, fetched 2026-10-26
