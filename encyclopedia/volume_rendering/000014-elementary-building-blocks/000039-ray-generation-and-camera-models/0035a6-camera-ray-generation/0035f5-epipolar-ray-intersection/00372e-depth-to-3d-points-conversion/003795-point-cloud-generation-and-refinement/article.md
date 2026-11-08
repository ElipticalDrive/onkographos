# Point Cloud Generation and Refinement

Also called: Depth-guided Point Cloud Refinement

## Overview
An offline process to generate and refine a 3D point cloud from a subset of training images to improve depth estimation accuracy for novel views in Neural Radiance Fields (NeRF) [S1]. It differs from standard Multi-View Stereo (MVS) by iteratively validating points against ground-truth depth from adjacent views, resulting in higher point density and better PSNR for similar computational time [S1].

## How it works
**Inputs:**
- A subset of training images with known camera poses and intrinsics.
- Ground-truth depth maps for the training images.
- Threshold parameter $\tau$.

**Outputs:**
- A refined 3D point cloud in absolute coordinates.

**Data Structures:**
- `Point`: A 3D vector $(x, y, z)$ representing a point in space.
- `PointCloud`: A list or array of `Point` objects.
- `DepthMap`: A 2D array where each element is a float representing depth $d_{GT}$ at pixel coordinates $(u, v)$.

**Steps:**
1. **Initialization:** Select a subset of training views with sufficient and uniform distance between adjacent viewpoints to ensure efficiency [S1]. Initialize an empty `PointCloud`.
2. **Iterative Generation:** For each viewpoint $V_i$ in the subset (sequentially following the trajectory):
    a. **Generate Initial Points:** Back-project the depth map of $V_i$ using its camera intrinsics to create an initial set of 3D points. Add these to `PointCloud`.
    b. **Projection:** For each 3D point $P$ in `PointCloud`, project it onto the image plane of the next viewpoint $V_{i+1}$ in the subset using multiple view geometry [S1].
    c. **Distance Calculation:** Calculate the distance $\tilde{d}$ between the projected 3D point $P$ and the camera location of $V_{i+1}$ [S1].
    d. **Validation:** Retrieve the ground-truth depth value $d_{GT}$ at the pixel location where $P$ was projected in $V_{i+1}$'s depth map. Check if the condition $|\tilde{d} - d_{GT}| \le \tau$ is satisfied [S1].
    e. **Refinement:** If the condition is **NOT** satisfied, generate a new 3D point by back-projecting the pixel with value $d_{GT}$ from $V_{i+1}$ into 3D space. Add this new point to `PointCloud`.
3. **Completion:** Repeat steps 2a-2e for all viewpoints in the subset. The resulting `PointCloud` is saved for inference [S1].

**Formulas:**
- Validation Condition:
  $$ |\tilde{d} - d_{GT}| \le \tau $$
  Where:
  - $\tilde{d}$: Calculated distance from the 3D point to the camera center of the next viewpoint.
  - $d_{GT}$: Ground-truth depth value of the projected pixel in the next viewpoint's depth map.
  - $\tau$: Specified threshold controlling point cloud density [S1].

**Parameters:**
- $\tau$: Threshold for comparing calculated distance and ground-truth depth.
  - Typical Value: Not given in the sources.
  - Effect: Too large $\rightarrow$ fewer points, faster rendering, more holes. Too small $\rightarrow$ overlapping points, slower rendering, fewer holes [S1].

**Edge Cases and Pitfalls:**
- **Precision:** The accuracy of $\tilde{d}$ depends on the precision of the multiple view geometry calculation [S1].
- **Boundaries:** The method relies on a subset of training views; using the entire training set is less efficient [S1].
- **Holes:** If $\tau$ is too large, estimated depth images may contain many holes [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_and_refine_point_cloud(training_views, ground_truth_depths, tau):
    """
    Generate and refine a 3D point cloud from a subset of training images.
    
    Args:
        training_views: List of view indices in sequential order.
        ground_truth_depths: Dict mapping view index to DepthMap.
        tau: Threshold for depth validation.
        
    Returns:
        point_cloud: List of 3D points.
    """
    point_cloud = []
    
    # Step 1: Initialize with first view
    first_view = training_views[0]
    initial_points = back_project_depth(ground_truth_depths[first_view], 
                                        get_camera_intrinsics(first_view))
    point_cloud.extend(initial_points)
    
    # Step 2: Iterate through subsequent views
    for i in range(len(training_views) - 1):
        current_view_idx = training_views[i]
        next_view_idx = training_views[i + 1]
        
        next_camera_intrinsics = get_camera_intrinsics(next_view_idx)
        next_camera_pose = get_camera_pose(next_view_idx)
        
        # Step 2b: Project existing points to next view
        projected_points = []
        for point in point_cloud:
            # Project 3D point to 2D pixel coordinates in next view
            pixel_u, pixel_v = project_point_to_image(point, next_camera_intrinsics, next_camera_pose)
            
            # Step 2c: Calculate distance from point to next camera center
            # Distance in 3D space between point and camera center
            camera_center = get_camera_center(next_camera_pose)
            distance = calculate_distance(point, camera_center)
            
            projected_points.append({
                'point': point,
                'pixel': (pixel_u, pixel_v),
                'distance': distance
            })
            
        # Step 2d & 2e: Validate and refine
        new_points = []
        for proj in projected_points:
            pixel_u, pixel_v = proj['pixel']
            distance = proj['distance']
            
            # Get ground truth depth at projected pixel
            depth_map = ground_truth_depths[next_view_idx]
            if is_valid_pixel(pixel_u, pixel_v, depth_map):
                d_gt = depth_map[pixel_v][pixel_u]
                
                # Check condition |d_tilde - d_GT| <= tau
                if abs(distance - d_gt) > tau:
                    # Generate new point by back-projecting d_GT
                    new_point = back_project_pixel(next_view_idx, pixel_u, pixel_v, d_gt)
                    new_points.append(new_point)
        
        point_cloud.extend(new_points)
        
    return point_cloud

def back_project_depth(depth_map, intrinsics):
    """Back-project a depth map to 3D points."""
    points = []
    for v in range(depth_map.height):
        for u in range(depth_map.width):
            d = depth_map[v][u]
            if d > 0: # Valid depth
                point = unproject_pixel(u, v, d, intrinsics)
                points.append(point)
    return points

def project_point_to_image(point, intrinsics, pose):
    """Project a 3D point to 2D image coordinates."""
    # Transform point to camera coordinates
    cam_point = transform_point(point, pose.inverse())
    # Project to pixel coordinates
    u, v = project_cam_to_pixel(cam_point, intrinsics)
    return u, v

def calculate_distance(point, camera_center):
    """Calculate Euclidean distance between two 3D points."""
    return sqrt(sum((p - c)**2 for p, c in zip(point, camera_center)))

def is_valid_pixel(u, v, depth_map):
    """Check if pixel coordinates are within bounds."""
    return 0 <= u < depth_map.width and 0 <= v < depth_map.height

def back_project_pixel(view_idx, u, v, d_gt):
    """Back-project a pixel with known depth to 3D."""
    intrinsics = get_camera_intrinsics(view_idx)
    pose = get_camera_pose(view_idx)
    cam_point = unproject_pixel(u, v, d_gt, intrinsics)
    # Transform back to world coordinates
    world_point = transform_point(cam_point, pose)
    return world_point
```

## Subtopics

### Projected Depth Refining
A method to obtain accurate depth images for novel views by refining a rough depth image projected from a 3D point cloud, specifically by removing unnecessary points and filling holes [S1]. It projects the saved 3D point cloud onto a novel view to generate a rough depth image, removes unnecessary 3D points to refine depth estimation, fills holes in the projected depth image, and uses the refined depth image for surface-based sampling during novel view rendering [S1].

## Operating characteristics
- **Time Complexity:** Processing time is comparable to standard MVS. On a GPU with 20 views, the proposed method takes 87.12 s, while standard MVS takes 86.78 s [S1].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** On the Pavillon dataset, the proposed method achieves 31.44 dB PSNR, compared to 30.27 dB for standard MVS (with $\alpha = 1/2, N = 8$) [S1].
- **Scaling:** The method uses a subset of training views for efficiency; using the entire training set is less efficient [S1].
- **Strengths:** Generates more points than standard MVS for similar computational time, leading to higher PSNR [S1].
- **Limitations:** Parameter $\tau$ requires careful tuning; too large causes holes, too small causes overlapping points and slower rendering [S1].

## Background
Originated in the paper "Improving Neural Radiance Fields Using Near-Surface Sampling with Point Cloud Generation" by H. B. Yoo et al. [S1]. The method was developed to improve NeRF by using an advanced sampling method around the surface of an object/scene using estimated depth images from a generated point cloud, outperforming original NeRF and other state-of-the-art NeRF methods [S1].

## Sources
[S1] Improving Neural Radiance Fields Using Near-Surface Sampling with Point Cloud Generation, https://link.springer.com/article/10.1007/s11063-024-11654-5, fetched 2026-10-30
