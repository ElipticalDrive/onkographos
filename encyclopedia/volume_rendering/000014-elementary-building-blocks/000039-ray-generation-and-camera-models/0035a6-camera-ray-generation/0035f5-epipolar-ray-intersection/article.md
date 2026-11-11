# Epipolar Ray Intersection

Also called: Lift 2D Points to 3D via Epipolar Lines, depth from 2D points, lift_to_3d / get_depth

## Overview
Epipolar Ray Intersection calculates the 3D positions and depths corresponding to specified 2D points on epipolar lines defined by camera origins and directions, using image extrinsics and intrinsics [S1, unofficial][S2, unofficial]. It constructs rays from 2D points in their respective camera frames and intersects them with given world-space rays to resolve the depth ambiguity inherent in single-view reconstruction [S1, unofficial]. Unlike [Camera Ray Generation](7456-camera-ray-generation.md) which generates rays for volume traversal, this method specifically solves for the geometric intersection between a known world-space ray and a ray projected from a 2D image point, serving as the geometric foundation for initializing point clouds in neural rendering pipelines [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- `xy`: Tensor of 2D points with shape `(N, 2)` representing pixel coordinates $(u, v)$ in the image plane.
- `extrinsics`: Camera extrinsic matrix (world-to-camera or camera-to-world depending on convention, typically $4 \times 4$) defining the camera pose.
- `intrinsics`: Camera intrinsic matrix ($3 \times 3$) defining focal length and principal point.
- `world_origins`: Tensor of ray origins in world space with shape `(M, 3)`.
- `world_directions`: Tensor of ray directions in world space with shape `(M, 3)`.

**Outputs:**
- `xyz`: Tensor of 3D intersection points with shape `(N, 3)` (or `(N, M)` if intersecting with multiple rays).
- `depth`: Optional scalar or tensor representing the Euclidean distance from the camera center to the 3D point.

**Data Structures:**
- Ray representation: A ray is defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$ (normalized).
- Camera matrices: Standard pinhole camera model parameters.

**Steps:**
1. **Convert 2D Points to World-Space Rays:**
   For each 2D point $\mathbf{xy}_i = (u_i, v_i)$, compute the corresponding ray origin $\mathbf{o}_{xy}$ and direction $\mathbf{d}_{xy}$ in world space using the camera's extrinsic and intrinsic parameters. This is typically done by unprojecting the pixel coordinates through the inverse intrinsic matrix and applying the extrinsic transformation.
   $$ \mathbf{o}_{xy} = \mathbf{C} $$
   $$ \mathbf{d}_{xy} = \text{normalize}(\mathbf{K}^{-1} \begin{bmatrix} u_i \\ v_i \\ 1 \end{bmatrix}) \text{ transformed to world space} $$
   Where $\mathbf{C}$ is the camera center and $\mathbf{K}$ is the intrinsic matrix [S1, unofficial].

2. **Intersect Rays:**
   Intersect the newly created rays $(\mathbf{o}_{xy}, \mathbf{d}_{xy})$ with the input world-space rays $(\mathbf{o}_{w}, \mathbf{d}_{w})$. Since these rays are generally skew (non-intersecting) in 3D space, the intersection is computed as the midpoint of the shortest line segment connecting the two rays.
   Let $\mathbf{r}_1(t) = \mathbf{o}_{xy} + t \mathbf{d}_{xy}$ and $\mathbf{r}_2(s) = \mathbf{o}_{w} + s \mathbf{d}_{w}$.
   The intersection point $\mathbf{xyz}$ is the midpoint of the segment connecting the closest points on both rays:
   $$ t = \frac{(\mathbf{o}_{w} - \mathbf{o}_{xy}) \cdot (\mathbf{d}_{w} \times \mathbf{d}_{xy}) \times \mathbf{d}_{w}}{\|\mathbf{d}_{w} \times \mathbf{d}_{xy}\|^2} $$
   $$ s = \frac{(\mathbf{o}_{w} - \mathbf{o}_{xy}) \cdot (\mathbf{d}_{w} \times \mathbf{d}_{xy}) \times \mathbf{d}_{xy}}{\|\mathbf{d}_{w} \times \mathbf{d}_{xy}\|^2} $$
   $$ \mathbf{xyz} = \frac{1}{2} (\mathbf{r}_1(t) + \mathbf{r}_2(s)) $$
   [S1, unofficial]. Note: If the rays are parallel, the cross product is zero, requiring a fallback (e.g., projecting one origin onto the other ray).

3. **Compute Depth (Optional):**
   If depth is required, compute the Euclidean distance from the camera center $\mathbf{C}$ to the computed 3D point $\mathbf{xyz}$:
   $$ \text{depth} = \|\mathbf{xyz} - \mathbf{C}\|_2 $$
   [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.linalg

def get_world_rays(xy, extrinsics, intrinsics):
    """
    Converts 2D pixel coordinates to world-space rays.
    xy: (N, 2) tensor of (u, v) coordinates.
    extrinsics: (4, 4) camera-to-world matrix.
    intrinsics: (3, 3) camera intrinsic matrix.
    Returns: origins (N, 3), directions (N, 3)
    """
    # Unproject to camera space
    # xy_homogeneous: (N, 3)
    xy_homogeneous = torch.cat([xy, torch.ones_like(xy[:, :1])], dim=1)
    
    # Inverse intrinsics
    K_inv = torch.linalg.inv(intrinsics)
    
    # Ray directions in camera space
    directions_cam = torch.matmul(K_inv, xy_homogeneous.t()).t() # (N, 3)
    
    # Normalize directions
    directions_cam = torch.nn.functional.normalize(directions_cam, dim=1)
    
    # Transform to world space
    # Extrinsics: (4, 4) matrix [R|t; 0 0 0 1]
    R = extrinsics[:3, :3]
    t = extrinsics[:3, 3]
    
    origins_world = t.unsqueeze(0).expand(xy.shape[0], -1) # Camera center in world
    directions_world = torch.matmul(directions_cam, R.t()) # (N, 3)
    
    return origins_world, directions_world

def intersect_rays(origins1, directions1, origins2, directions2):
    """
    Computes the midpoint of the shortest segment between two skew rays.
    origins1, directions1: (N, 3)
    origins2, directions2: (N, 3)
    Returns: intersection points (N, 3)
    """
    # Vector between origins
    diff = origins2 - origins1 # (N, 3)
    
    # Cross product of directions
    cross = torch.cross(directions1, directions2, dim=1) # (N, 3)
    cross_norm_sq = torch.sum(cross ** 2, dim=1, keepdim=True) # (N, 1)
    
    # Avoid division by zero for parallel rays
    # [G] In practice, add a small epsilon to cross_norm_sq to prevent NaNs
    epsilon = 1e-8
    cross_norm_sq = torch.clamp(cross_norm_sq, min=epsilon)
    
    # Compute parameters t and s for the closest points
    # t = dot(diff, cross x d2) / |cross|^2
    # s = dot(diff, cross x d1) / |cross|^2
    
    cross_x_d2 = torch.cross(cross, directions2, dim=1)
    cross_x_d1 = torch.cross(cross, directions1, dim=1)
    
    t = torch.sum(diff * cross_x_d2, dim=1, keepdim=True) / cross_norm_sq
    s = torch.sum(diff * cross_x_d1, dim=1, keepdim=True) / cross_norm_sq
    
    # Closest points on each ray
    point1 = origins1 + directions1 * t
    point2 = origins2 + directions2 * s
    
    # Midpoint
    intersection = (point1 + point2) / 2.0
    
    return intersection

def epipolar_ray_intersection(xy, extrinsics, intrinsics, world_origins, world_directions):
    """
    Main function to perform epipolar ray intersection.
    """
    # Step 1: Get rays from 2D points
    xy_origins, xy_directions = get_world_rays(xy, extrinsics, intrinsics)
    
    # Step 2: Intersect with world rays
    xyz = intersect_rays(xy_origins, xy_directions, world_origins, world_directions)
    
    # Step 3: Compute depth (optional)
    # [G] Assuming extrinsics includes the camera center at extrinsics[:3, 3]
    camera_center = extrinsics[:3, 3].unsqueeze(0).expand(xyz.shape[0], -1)
    depth = torch.linalg.norm(xyz - camera_center, dim=1)
    
    return xyz, depth
```

**Parameters:**
- `xy`: Pixel coordinates. Typical values depend on image resolution (e.g., $[0, W-1], [0, H-1]$).
- `extrinsics`: $4 \times 4$ matrix.
- `intrinsics`: $3 \times 3$ matrix.

**Edge Cases:**
- **Parallel Rays:** If the 2D ray and world ray are parallel, the cross product norm is zero. The intersection is undefined or requires a specific fallback (e.g., projecting one origin onto the other). [G] Add epsilon to denominator.
- **Degenerate Intrinsics:** If the intrinsic matrix is singular (e.g., zero focal length), unprojection fails.

## Subtopics

### Depth Labeling on Epipolar Lines
This variant focuses on rendering depth labels along epipolar lines for visualization or annotation purposes [S2, unofficial]. It interpolates epipolar line endpoints in pixel space and computes depth as the Euclidean distance from the camera center to the 3D point. The depth labels are rendered with white backer rectangles and rotated text [S2, unofficial].
- **Step:** Interpolate epipolar line endpoints: $\mathbf{pt} = \mathbf{xy}_{min} \cdot t + (1 - t) \cdot \mathbf{xy}_{max}$ for $t \in [0, 1]$.
- **Step:** Compute depth: $\text{depth} = \|\mathbf{xyz} - \mathbf{extrinsics}[0, :3, 3]\|_2$.
- **Step:** Render depth labels along the epipolar line with white backer rectangles and rotated text [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of 2D points, assuming constant-time ray intersection.
- **Memory Complexity:** $O(N)$ to store the 3D points and depths.
- **Strengths:** Provides exact geometric backprojection, which is crucial for initializing point clouds in neural rendering pipelines [S1, unofficial][S2, unofficial]. It is computationally efficient compared to iterative depth estimation methods.
- **Limitations:** Relies on accurate camera parameters (intrinsics and extrinsics). Errors in calibration propagate directly to the 3D positions. It does not handle occlusions or multi-view consistency inherently; it simply finds the geometric intersection.
- **When to use:** When initializing 3D points from 2D correspondences in multi-view stereo or neural rendering pipelines where geometric accuracy is prioritized over learned priors.
- **When not to use:** When camera parameters are unknown or inaccurate, or when dealing with scenes where occlusions make simple ray intersection ambiguous.

## Background
Epipolar geometry is a fundamental concept in computer vision, describing the geometric relationship between two views of the same scene [S1, unofficial][S2, unofficial]. The method of lifting 2D points to 3D via epipolar lines is a direct application of this principle, used in various neural rendering and multi-view stereo pipelines. The specific implementation details described here are derived from open-source repositories such as VolSplat and PixelSplat, which utilize this technique for point cloud initialization [S1, unofficial][S2, unofficial].

## Sources
[S1] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S2] dcharatan/pixelsplat, https://github.com/dcharatan/pixelsplat, fetched 2026-10-27 (unofficial)
