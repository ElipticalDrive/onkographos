# Epipolar Ray Intersection

Also called: Lift 2D Points to 3D via Epipolar Lines, depth from 2D points, lift_to_3d / get_depth

## Overview
Epipolar Ray Intersection calculates the 3D positions and depths corresponding to specified 2D points on epipolar lines defined by camera origins and directions, using image extrinsics and intrinsics [S1, S2, unofficial]. It constructs rays from the 2D points in their respective camera frames and intersects them with given world-space rays [S1, unofficial]. This method serves as a geometric primitive for lifting 2D image coordinates into 3D space constrained by epipolar geometry, distinct from general ray generation methods like [Camera Matrix Ray Generation](2172-camera-matrix-ray-generation.md) which produce rays for all pixels, or [Orthographic (Parallel) Camera Ray Generation](1319-orthographic-parallel-camera-ray-generation.md) which assumes parallel projection.

## How it works
**Inputs:**
- `xy`: Tensor of 2D points with shape `(N, 2)` representing pixel coordinates $(u, v)$ [S1, unofficial].
- `extrinsics`: Camera extrinsic matrices with shape `(N, 4, 4)` or `(N, 3, 4)`, defining the transformation from world to camera space [S1, unofficial].
- `intrinsics`: Camera intrinsic matrices with shape `(N, 3, 3)` [S1, unofficial].
- `origins`: World-space ray origins with shape `(M, 3)` [S1, unofficial].
- `directions`: World-space ray directions with shape `(M, 3)` [S1, unofficial].

**Outputs:**
- `xyz`: 3D intersection points with shape `(N, 3)` [S1, unofficial].
- `depth`: Scalar depth values with shape `(N,)` [S1, unofficial].

**Data Structures:**
- Ray representation: A ray is defined by an origin point $\mathbf{o}$ and a direction vector $\mathbf{d}$.
- Camera matrices: Standard homogeneous transformation matrices.

**Steps:**
1. **Construct Epipolar Rays:** Convert the 2D points `xy` and camera parameters (`extrinsics`, `intrinsics`) into world-space ray origins and directions. This is performed using a helper function `get_world_rays` [S1, unofficial].
   - Let $\mathbf{r}_{xy\_origins}$ and $\mathbf{r}_{xy\_directions}$ be the resulting origins and directions for the rays passing through the 2D points [S1, unofficial].
2. **Intersect Rays:** Intersect the input world-space rays (`origins`, `directions`) with the newly created epipolar rays (`xy_origins`, `xy_directions`) [S1, unofficial].
   - The intersection is computed using a helper function `intersect_rays` [S1, unofficial].
   - The result is the 3D point $\mathbf{xyz}$ [S1, unofficial].
3. **Compute Depth:** Calculate the depth as the L2 norm of the difference between the 3D point and the ray origin along the last dimension [S1, unofficial].
   - Formula: $\text{depth} = \| \mathbf{xyz} - \mathbf{o} \|_2$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.linalg

def get_world_rays(xy, extrinsics, intrinsics):
    """
    Converts 2D pixel coordinates to world-space rays.
    xy: (N, 2) pixel coordinates
    extrinsics: (N, 4, 4) or (N, 3, 4) world-to-camera transform
    intrinsics: (N, 3, 3) camera intrinsic matrix
    Returns:
        origins: (N, 3)
        directions: (N, 3)
    """
    # Extract camera center from extrinsics (inverse of extrinsics gives camera-to-world)
    # Assuming extrinsics is world-to-camera, we invert it to get camera-to-world
    # For standard 4x4 extrinsics:
    if extrinsics.shape[-1] == 4 and extrinsics.shape[-2] == 4:
        cam_to_world = torch.linalg.inv(extrinsics)
    else:
        # Handle 3x4 case if necessary, typically extrinsics are 4x4 in these contexts
        cam_to_world = torch.eye(4).unsqueeze(0).repeat(extrinsics.shape[0], 1, 1)
        cam_to_world[:, :3, :3] = extrinsics[:, :3, :3]
        cam_to_world[:, :3, 3] = extrinsics[:, :3, 3]
    
    camera_centers = cam_to_world[:, :3, 3]
    
    # Project 2D points to camera space direction
    # xy is in pixel coordinates. Convert to normalized camera coordinates.
    # u, v = xy[:, 0], xy[:, 1]
    # x = (u - cx) / fx
    # y = (v - cy) / fy
    # z = 1
    
    fx = intrinsics[:, 0, 0]
    fy = intrinsics[:, 1, 1]
    cx = intrinsics[:, 0, 2]
    cy = intrinsics[:, 1, 2]
    
    u = xy[:, 0]
    v = xy[:, 1]
    
    x_cam = (u - cx) / fx
    y_cam = (v - cy) / fy
    z_cam = torch.ones_like(u)
    
    directions_cam = torch.stack([x_cam, y_cam, z_cam], dim=-1)
    
    # Transform directions to world space
    # Rotation part of cam_to_world
    R = cam_to_world[:, :3, :3]
    directions_world = torch.bmm(R, directions_cam.unsqueeze(-1)).squeeze(-1)
    
    return camera_centers, directions_world

def intersect_rays(origins1, directions1, origins2, directions2):
    """
    Computes the closest point between two skew lines (rays).
    origins1, directions1: (N, 3)
    origins2, directions2: (N, 3)
    Returns:
        xyz: (N, 3) midpoint between the two closest points on the rays
    """
    # Vector between origins
    d1 = directions1 / torch.norm(directions1, dim=-1, keepdim=True)
    d2 = directions2 / torch.norm(directions2, dim=-1, keepdim=True)
    
    # Cross product of directions
    n = torch.cross(d1, d2, dim=-1)
    n_norm_sq = torch.sum(n * n, dim=-1, keepdim=True)
    
    # Avoid division by zero for parallel rays
    n_norm_sq = torch.clamp(n_norm_sq, min=1e-8)
    
    # Project vector between origins onto normal
    # p1 = origins1, p2 = origins2
    # t1 = dot(p2 - p1, n x d2) / |n|^2
    # t2 = dot(p2 - p1, n x d1) / |n|^2
    
    p2_p1 = origins2 - origins1
    
    n_cross_d2 = torch.cross(n, d2, dim=-1)
    n_cross_d1 = torch.cross(n, d1, dim=-1)
    
    t1 = torch.sum(p2_p1.unsqueeze(-1) * n_cross_d2.unsqueeze(-1), dim=-1) / n_norm_sq.squeeze(-1)
    t2 = torch.sum(p2_p1.unsqueeze(-1) * n_cross_d1.unsqueeze(-1), dim=-1) / n_norm_sq.squeeze(-1)
    
    # Closest points
    closest1 = origins1 + d1 * t1.unsqueeze(-1)
    closest2 = origins2 + d2 * t2.unsqueeze(-1)
    
    # Midpoint
    xyz = (closest1 + closest2) / 2.0
    
    return xyz

def epipolar_ray_intersection(xy, extrinsics, intrinsics, origins, directions):
    """
    Main function for Epipolar Ray Intersection.
    """
    # Step 1: Get world rays from 2D points
    xy_origins, xy_directions = get_world_rays(xy, extrinsics, intrinsics)
    
    # Step 2: Intersect rays
    xyz = intersect_rays(origins, directions, xy_origins, xy_directions)
    
    # Step 3: Compute depth
    depth = torch.linalg.norm(xyz - origins, dim=-1)
    
    return xyz, depth
```

**Edge Cases and Pitfalls:**
- **Parallel Rays:** If the input ray and the epipolar ray are parallel, the cross product of their directions is zero, leading to division by zero in the intersection calculation. The pseudocode handles this by clamping the norm squared to a small positive value [G].
- **Precision:** Floating-point precision issues may arise when rays are nearly parallel or when coordinates are very large. Using double precision for intermediate calculations may be necessary for high-accuracy applications [G].
- **Boundary Conditions:** The method assumes valid 2D points within the image bounds. Points outside the valid range may produce rays that do not intersect the scene or produce invalid 3D points [G].

## Subtopics

### Depth Labeling on Epipolar Lines
This variant focuses on rendering depth labels along epipolar lines. It involves interpolating epipolar line endpoints in pixel space using `pt = projection["xy_min"] * t + (1 - t) * projection["xy_max"]` for `t` in `[0, 1]` [S2, unofficial]. The depth is computed as the Euclidean distance from the camera center to the 3D point: `depth = (xyz - extrinsics[0, :3, 3]).norm(dim=-1)` [S2, unofficial]. The depth labels are then rendered along the epipolar line with white backer rectangles and rotated text [S2, unofficial]. This is useful for visualization and debugging of depth estimation models [S2, unofficial].

## Operating characteristics
- **Time Complexity:** The complexity is dominated by the matrix inversions and multiplications in `get_world_rays` and the vector operations in `intersect_rays`. For $N$ points, the complexity is approximately $O(N)$ assuming fixed-size matrices.
- **Memory Complexity:** $O(N)$ for storing the input and output tensors.
- **Performance:** Performance depends on the hardware and the implementation of the underlying PyTorch operations. GPU acceleration is typically used for large-scale point clouds [S1, S2, unofficial].
- **Strengths:** Provides a direct geometric method for lifting 2D points to 3D using epipolar constraints. It is efficient and straightforward to implement.
- **Limitations:** Requires accurate camera parameters (extrinsics and intrinsics). The intersection method assumes linear rays and may not account for lens distortion or other non-linear effects [G].
- **When to Use:** Use when you need to lift specific 2D points to 3D space using known camera geometry, such as in multi-view stereo or depth estimation tasks [S1, S2, unofficial].
- **When Not to Use:** Avoid if the camera parameters are unknown or inaccurate, or if the scene contains significant non-linear distortions [G].

## Background
Epipolar geometry is a fundamental concept in computer vision, describing the geometric relationship between two views of a scene. The method described here is implemented in the VolSplat [S1, unofficial] and PixelSplat [S2, unofficial] repositories, which are used for 3D Gaussian Splatting and depth estimation from multi-view images. VolSplat uses this method to lift 2D points to 3D for splatting operations, while PixelSplat uses it for depth labeling and visualization [S1, S2, unofficial].

## Sources
[S1] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S2] dcharatan/pixelsplat, https://github.com/dcharatan/pixelsplat, fetched 2026-10-27 (unofficial)
