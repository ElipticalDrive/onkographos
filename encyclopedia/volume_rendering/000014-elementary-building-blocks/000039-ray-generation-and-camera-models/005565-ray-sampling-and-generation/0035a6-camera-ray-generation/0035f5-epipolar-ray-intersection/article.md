# Epipolar Ray Intersection

Also called: Epipolar 3D Lifting and Depth Computation, Get Depth, Lift 2D Points to 3D via Epipolar Lines, Lift to 3D, depth from 2D points, lift_to_3d / get_depth

## Overview
Epipolar Ray Intersection lifts 2D image points to 3D space by intersecting rays constructed from those points with a set of input world-space rays [S1, S2, S3, unofficial]. It solves the problem of finding the precise 3D positions and depths corresponding to specified 2D coordinates on epipolar lines defined by camera origins and directions [S2, S3, unofficial]. It differs from sibling methods like [Orthographic (Parallel) Camera Ray Generation](1319-orthographic-parallel-camera-ray-generation.md) by explicitly handling the intersection of two distinct ray sets (image-space and world-space) rather than generating rays from a single camera model, and from [Sample Streaming Payload Ray Computation](1509-sample-streaming-payload-ray-computation.md) by focusing on geometric reprojection via epipolar constraints rather than payload streaming [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   `origins`: Float tensor of shape `(*#batch, 3)` representing ray origins in world space [S1, unofficial].
*   `directions`: Float tensor of shape `(*#batch, 3)` representing ray directions in world space [S1, unofficial].
*   `xy`: Float tensor of shape `(*#batch, 2)` representing 2D image coordinates [S1, unofficial].
*   `extrinsics`: Float tensor of shape `(*#batch, 4, 4)` representing camera extrinsic matrices [S1, S2, unofficial].
*   `intrinsics`: Float tensor of shape `(*#batch, 3, 3)` representing camera intrinsic matrices [S1, S2, unofficial].
*   Output `xyz`: Float tensor of shape `(*#batch, 3)` representing the lifted 3D points [S1, S2, unofficial].
*   Output `depth`: Float tensor of shape `(*#batch)` representing the distance from the camera origin to the 3D point [S1, S3, unofficial].

**Data Structures:**
The method relies on standard 3D vector and matrix operations. The `get_world_rays` helper constructs ray origins and directions from 2D points using camera parameters. The `intersect_rays` helper computes the intersection between two rays.

**Steps:**
1.  **Construct World Rays from 2D Points:** Convert the 2D image coordinates `xy` and camera parameters (`extrinsics`, `intrinsics`) into world-space ray origins (`xy_origins`) and directions (`xy_directions`) using the `get_world_rays` function [S1, S2, S3, unofficial].
2.  **Intersect Rays:** Intersect the input world-space rays (`origins`, `directions`) with the newly constructed world rays (`xy_origins`, `xy_directions`) using the `intersect_rays` function to find the 3D point `xyz` [S1, S2, S3, unofficial].
3.  **Compute Depth:** Calculate the depth as the L2 norm (Euclidean distance) between the lifted 3D point `xyz` and the input ray `origins` [S1, S2, S3, unofficial].

**Formulas:**
The depth is computed as:
$$ \text{depth} = \| \text{xyz} - \text{origins} \|_2 $$
where $\| \cdot \|_2$ denotes the L2 norm [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
from typing import Tuple

def get_world_rays(
    xy: torch.Tensor, 
    extrinsics: torch.Tensor, 
    intrinsics: torch.Tensor
) -> Tuple[torch.Tensor, torch.Tensor]:
    """
    Converts 2D image coordinates to world-space ray origins and directions.
    
    Args:
        xy: Float tensor of shape (*batch, 2), 2D image coordinates.
        extrinsics: Float tensor of shape (*batch, 4, 4), camera extrinsics.
        intrinsics: Float tensor of shape (*batch, 3, 3), camera intrinsics.
        
    Returns:
        xy_origins: Float tensor of shape (*batch, 3), ray origins in world space.
        xy_directions: Float tensor of shape (*batch, 3), ray directions in world space.
    """
    # Expand dimensions for broadcasting if necessary
    # Assuming xy is (*batch, 2), extrinsics is (*batch, 4, 4), intrinsics is (*batch, 3, 3)
    
    # Homogeneous coordinates for xy (add 1s)
    xy_hom = torch.cat([xy, torch.ones_like(xy[..., :1])], dim=-1)  # (*batch, 3)
    
    # Invert intrinsics to go from pixel to camera coordinates
    # Note: Standard pinhole model: x_cam = K^-1 * x_pixel
    K_inv = torch.inverse(intrinsics)  # (*batch, 3, 3)
    
    # Camera space directions (assuming origin at camera center)
    directions_cam = torch.bmm(K_inv, xy_hom.unsqueeze(-1)).squeeze(-1)  # (*batch, 3)
    
    # Normalize directions
    directions_cam = directions_cam / torch.norm(directions_cam, dim=-1, keepdim=True)
    
    # Transform directions to world space using rotation part of extrinsics
    # Extrinsics: [R | t; 0 0 0 1]
    R = extrinsics[..., :3, :3]  # (*batch, 3, 3)
    t = extrinsics[..., :3, 3]   # (*batch, 3)
    
    directions_world = torch.bmm(R, directions_cam.unsqueeze(-1)).squeeze(-1)  # (*batch, 3)
    
    # Origins are the camera centers in world space
    origins_world = t  # (*batch, 3)
    
    return origins_world, directions_world

def intersect_rays(
    origins1: torch.Tensor, 
    directions1: torch.Tensor, 
    origins2: torch.Tensor, 
    directions2: torch.Tensor
) -> torch.Tensor:
    """
    Computes the intersection point of two rays in 3D space.
    Finds the point that minimizes the distance between the two rays.
    
    Args:
        origins1: Float tensor of shape (*batch, 3), first ray origins.
        directions1: Float tensor of shape (*batch, 3), first ray directions.
        origins2: Float tensor of shape (*batch, 3), second ray origins.
        directions2: Float tensor of shape (*batch, 3), second ray directions.
        
    Returns:
        intersection: Float tensor of shape (*batch, 3), the 3D intersection point.
    """
    # Vector between origins
    d1 = directions1 / torch.norm(directions1, dim=-1, keepdim=True)
    d2 = directions2 / torch.norm(directions2, dim=-1, keepdim=True)
    
    # Normalize directions
    d1 = directions1
    d2 = directions2
    
    # Cross product of directions
    n = torch.cross(d1, d2, dim=-1)  # (*batch, 3)
    n_norm_sq = torch.sum(n * n, dim=-1, keepdim=True)  # (*batch, 1)
    
    # Avoid division by zero for parallel rays
    n_norm_sq = torch.clamp(n_norm_sq, min=1e-8)
    
    # Vector between origins
    v = origins2 - origins1  # (*batch, 3)
    
    # Project v onto the normal vector n
    c1 = torch.sum(v * n, dim=-1, keepdim=True) / n_norm_sq  # (*batch, 1)
    
    # Intersection point on ray 1
    intersection = origins1 + d1 * c1  # (*batch, 3)
    
    return intersection.squeeze(-1)

def lift_to_3d(
    origins: torch.Tensor, 
    directions: torch.Tensor, 
    xy: torch.Tensor, 
    extrinsics: torch.Tensor, 
    intrinsics: torch.Tensor
) -> torch.Tensor:
    """
    Lifts 2D points to 3D space via epipolar ray intersection.
    
    Args:
        origins: Float tensor of shape (*batch, 3), input ray origins.
        directions: Float tensor of shape (*batch, 3), input ray directions.
        xy: Float tensor of shape (*batch, 2), 2D image coordinates.
        extrinsics: Float tensor of shape (*batch, 4, 4), camera extrinsics.
        intrinsics: Float tensor of shape (*batch, 3, 3), camera intrinsics.
        
    Returns:
        xyz: Float tensor of shape (*batch, 3), lifted 3D points.
    """
    # Step 1: Get world rays from 2D points
    xy_origins, xy_directions = get_world_rays(xy, extrinsics, intrinsics)
    
    # Step 2: Intersect input rays with world rays
    xyz = intersect_rays(origins, directions, xy_origins, xy_directions)
    
    return xyz

def get_depth(
    xyz: torch.Tensor, 
    origins: torch.Tensor
) -> torch.Tensor:
    """
    Computes depth as the L2 norm of the difference between the 3D point and the ray origin.
    
    Args:
        xyz: Float tensor of shape (*batch, 3), 3D points.
        origins: Float tensor of shape (*batch, 3), ray origins.
        
    Returns:
        depth: Float tensor of shape (*batch), depth values.
    """
    diff = xyz - origins
    depth = torch.norm(diff, dim=-1)
    return depth
```

**Edge Cases and Pitfalls:**
*   **Parallel Rays:** If the input ray and the constructed world ray are parallel, the cross product in `intersect_rays` will be zero, leading to division by zero. The pseudocode handles this by clamping the denominator to a small epsilon [G].
*   **Out-of-Bounds Coordinates:** The 2D coordinates `xy` should be within the valid image bounds. Some implementations check if `xy` is within `[-epsilon, 1+epsilon]` for normalized coordinates [S3, unofficial].
*   **Points Behind Camera:** The Z coordinate of the lifted point in the camera frame should be positive (in front of the camera). Checks like `Z > -epsilon` can be used to validate this [S3, unofficial].
*   **Negative Ray Parameter:** The intersection parameter `t` should be positive, indicating the intersection is in the direction of the ray [S3, unofficial].

## Subtopics

### Depth Labeling on Epipolar Lines
This variant involves rendering depth labels along the epipolar line. It interpolates epipolar line endpoints in pixel space using `pt = projection["xy_min"] * t + (1 - t) * projection["xy_max"]` for `t` in `[0, 1]` [S4, unofficial]. The depth is computed as the Euclidean distance from the camera center to the 3D point: `depth = (xyz - extrinsics[0, :3, 3]).norm(dim=-1)` [S4, unofficial]. Finally, it renders depth labels along the epipolar line with white backer rectangles and rotated text [S4, unofficial].

### Epipolar Line Projection
This variant focuses on validating the projection of points onto epipolar lines. It checks if XY coordinates are in bounds (normalized 0 to 1) by verifying `xy >= -epsilon` and `xy <= 1 + epsilon` for all dimensions [S3, unofficial]. It also checks if points are in front of the camera by verifying the Z coordinate is greater than `-epsilon` [S3, unofficial]. Additionally, it verifies if the ray parameter `t` is positive by checking `t > -epsilon` [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** The complexity is dominated by the matrix inversions and multiplications in `get_world_rays` and the cross product and division in `intersect_rays`. For `N` points, it is $O(N)$ assuming fixed-size matrix operations.
*   **Memory Complexity:** $O(N)$ for storing the input and output tensors.
*   **Strengths:** Provides a geometrically precise method for lifting 2D points to 3D using epipolar constraints, which is robust for multi-view stereo and novel view synthesis tasks [S1, S2, unofficial].
*   **Limitations:** Requires accurate camera intrinsics and extrinsics. Performance may degrade if rays are nearly parallel or if the intersection is numerically unstable [G].
*   **When to Use:** Use when you need to lift 2D points to 3D space using epipolar geometry, particularly in multi-view settings [S1, S2, unofficial].
*   **When Not to Use:** Avoid if camera parameters are inaccurate or if the rays are likely to be parallel [G].

## Background
Epipolar Ray Intersection is a fundamental technique in computer vision and graphics, particularly in multi-view stereo and structure from motion. It leverages the epipolar constraint to find corresponding points in different views. The method is widely used in modern neural rendering frameworks for initializing 3D points from 2D observations [S1, S2, S3, unofficial].

## Sources
[S1] cvg/NoPoSplat, https://github.com/cvg/NoPoSplat, fetched 2026-10-27 (unofficial)
[S2] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S3] pulangk97/VoLoDSplat, https://github.com/pulangk97/VoLoDSplat, fetched 2026-10-27 (unofficial)
[S4] dcharatan/pixelsplat, https://github.com/dcharatan/pixelsplat, fetched 2026-10-27 (unofficial)
