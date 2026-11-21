# Point Cloud Generation and Refinement

Also called: Depth-guided Point Cloud Refinement

## Overview
An offline process that generates and refines a 3D point cloud from a subset of training images to improve depth estimation accuracy for novel views in neural rendering [S1]. It differs from standard Multi-View Stereo (MVS) by iteratively validating points against ground-truth depth from adjacent views, allowing it to generate a denser, more accurate point cloud for similar computational time, which leads to higher rendering quality (PSNR) [S1].

## How it works
**Inputs:**
- A subset of training images with known camera poses and ground-truth depth maps.
- A threshold parameter $\tau$ controlling the tolerance for depth consistency [S1].

**Outputs:**
- A refined 3D point cloud used for surface-based sampling during novel view synthesis [S1].

**Data Structures:**
- **Point Cloud:** A list of 3D points $P = \{p_1, p_2, \dots, p_N\}$ in absolute coordinates.
- **Depth Map:** A 2D array of ground-truth depth values $d_{GT}$ for each pixel in the training views [S1].

**Steps:**
1. **Initialization:** Select a subset of training views with sufficient and uniform distance between adjacent viewpoints to ensure efficiency [S1].
2. **Iterative Generation:** For each viewpoint $V_i$ in the subset, sequentially follow the trajectory:
   a. **Generate:** Create an initial point cloud using the depth image from viewpoint $V_i$ [S1].
   b. **Project:** Project the 3D points of the generated point cloud onto the image plane of the next viewpoint $V_{i+1}$ [S1].
   c. **Calculate Distance:** For each projected point, calculate the distance $\tilde{d}$ between the 3D point and the camera location of $V_{i+1}$ using multiple view geometry [S1].
   d. **Validate:** Compare $\tilde{d}$ with the ground-truth depth value $d_{GT}$ of the corresponding pixel in $V_{i+1}$'s depth map. Check if the condition is satisfied:
      $$ |\tilde{d} - d_{GT}| \le \tau $$
      where $\tilde{d}$ is the calculated distance, $d_{GT}$ is the ground-truth depth, and $\tau$ is the specified threshold [S1].
   e. **Refine:** If the condition is **not** satisfied, discard the current point and generate a new 3D point by back-projecting the pixel with value $d_{GT}$ from $V_{i+1}$ [S1].
3. **Finalization:** Repeat the iteration for all viewpoints in the subset to produce the final refined point cloud [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_and_refine_point_cloud(training_views, tau):
    """
    Generate and refine a 3D point cloud from a subset of training views.
    
    Args:
        training_views: List of view objects containing camera pose, 
                        depth_map (2D array), and image dimensions.
        tau: Threshold for depth consistency check.
        
    Returns:
        point_cloud: List of 3D points.
    """
    point_cloud = []
    
    # Iterate through the subset of training views sequentially
    for i in range(len(training_views) - 1):
        current_view = training_views[i]
        next_view = training_views[i + 1]
        
        # Step 1: Generate initial point cloud from current view's depth
        # Back-project pixels using current_view's camera intrinsics/extrinsics
        current_points = back_project_depth(current_view.depth_map, current_view.camera)
        
        # Step 2: Project these points to the next view's image plane
        # and calculate distances to the next camera center
        for point in current_points:
            # Project 3D point to 2D pixel coordinates in next_view
            pixel_uv = project_to_image(point, next_view.camera)
            
            # Check if pixel is within image bounds
            if not is_valid_pixel(pixel_uv, next_view.width, next_view.height):
                continue
                
            # Get ground truth depth at projected pixel
            d_gt = next_view.depth_map[pixel_uv.y][pixel_uv.x]
            
            # Calculate distance from 3D point to next camera location
            # d_tilde is the Euclidean distance between point and next_view.camera.position
            d_tilde = calculate_distance(point, next_view.camera.position)
            
            # Step 3: Validate against threshold
            if abs(d_tilde - d_gt) <= tau:
                # Condition satisfied: keep the point
                point_cloud.append(point)
            else:
                # Condition not satisfied: generate new point from ground truth
                # Back-project the ground truth depth from next_view
                new_point = back_project_pixel_with_depth(
                    pixel_uv, d_gt, next_view.camera
                )
                point_cloud.append(new_point)
                
    return point_cloud
```

**Parameters:**
- $\tau$: Threshold for comparing calculated distance and ground-truth depth. Controls point cloud density and hole filling. Too large: fewer points, faster rendering, more holes. Too small: overlapping points, slower rendering, fewer holes [S1].

**Edge Cases and Pitfalls:**
- **Threshold Selection:** An inappropriate $\tau$ leads to either excessive holes (if too large) or overlapping points causing slow rendering (if too small) [S1].
- **Background Filling:** If the hole-filling kernel size $\kappa$ (used in subsequent refinement) is too small, holes in background areas may be filled, blurring the depth image [S1].
- **Subset Selection:** Using the entire training view set is less efficient; a subset with uniform spacing is recommended [S1].

## Subtopics

### [Depth-Guided Optimization](14749-depth-guided-optimization.md)
A strategy that uses sparse Structure-from-Motion (SfM) and dense PatchMatch Stereo point clouds to guide depth prediction in few-shot novel view synthesis, preventing overfitting to training views. It differs from standard depth-supervised methods by computing adaptive weights based on multiview color consistency to mitigate inaccuracies in estimated 3D points, and by restricting its application to early training stages.

### Projected Depth Refining
A method to obtain accurate depth images for novel views by refining a rough depth image projected from a 3D point cloud, specifically by removing unnecessary points and filling holes [S1].
- **Steps:**
  1. Project the saved 3D point cloud onto a novel view to generate a rough depth image [S1].
  2. Remove unnecessary 3D points from the point cloud to refine the depth estimation [S1].
  3. Fill the holes in the projected depth image [S1].
  4. Use the refined depth image for surface-based sampling during novel view rendering [S1].

## Operating characteristics
- **Time Complexity:** Similar to standard MVS. On a GPU with 20 views, processing time is approximately 87.12 s, compared to 86.78 s for standard MVS [S1].
- **Memory Complexity:** Not given in the sources.
- **Performance:** Generates more points than standard MVS for similar computational time, leading to higher PSNR (31.44 dB vs 30.27 dB on the Pavillon dataset) [S1].
- **Strengths:** Improves rendering quality by providing a denser, more accurate point cloud for surface-based sampling [S1].
- **Limitations:** Requires ground-truth depth maps for training views; performance is sensitive to the threshold $\tau$ [S1].
- **When to use:** When high-quality novel view synthesis is required and ground-truth depth data is available for training views.
- **When not to use:** When computational resources are extremely limited and slight quality degradation is acceptable, or when ground-truth depth is unavailable.

## Background
Originated in the work by H. B. Yoo et al., titled "Improving Neural Radiance Fields Using Near-Surface Sampling with Point Cloud Generation" [S1]. The method was developed to address the limitations of original NeRF's wide-range sampling by introducing surface-based sampling guided by a refined point cloud. It leverages multiple view geometry to iteratively correct point positions based on depth consistency across training views [S1].

## Sources
[S1] Improving Neural Radiance Fields Using Near-Surface Sampling with Point Cloud Generation, https://link.springer.com/article/10.1007/s11063-024-11654-5, fetched 2026-10-30
