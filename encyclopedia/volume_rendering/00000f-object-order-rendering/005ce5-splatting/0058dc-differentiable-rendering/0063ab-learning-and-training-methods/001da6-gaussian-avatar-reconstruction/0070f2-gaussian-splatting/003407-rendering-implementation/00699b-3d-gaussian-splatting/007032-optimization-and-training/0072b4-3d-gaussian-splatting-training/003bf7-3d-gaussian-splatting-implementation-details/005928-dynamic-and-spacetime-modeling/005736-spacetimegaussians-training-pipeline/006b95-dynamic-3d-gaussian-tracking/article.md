# Dynamic 3D Gaussian Tracking

Also called: Dense Correspondence via Gaussian Influence, Tracking with Dynamic 3D Gaussians

## Overview
Dynamic 3D Gaussian Tracking is a post-training technique that establishes dense 3D and 2D correspondence (tracking) for points in a scene represented by Dynamic 3D Gaussians across multiple timesteps [S1]. It solves the problem of mapping points between frames in a dynamic scene without requiring explicit deformation fields or neural networks for motion prediction [S2, unofficial]. Unlike sibling methods that predict per-frame deformations or use neural transformation caches, this method relies on the persistent influence of Gaussians initialized at the first timestamp, where only position and rotation are optimized while other parameters remain fixed [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained set of Dynamic 3D Gaussians $\mathcal{G} = \{g_i\}$ representing the scene at the initial timestamp $t_0$, and camera parameters for the source and target views/timesteps.
*   **Output:** A dense correspondence map linking 3D points or 2D pixels across different timesteps or camera poses.

**Data Structures:**
The scene is represented by a set of 3D Gaussians. Each Gaussian $g_i$ is defined by:
*   Mean position $\mu_i \in \mathbb{R}^3$
*   Rotation $R_i \in SO(3)$ (typically represented as a quaternion or rotation matrix)
*   Scale $s_i \in \mathbb{R}^3$ (fixed after initialization)
*   Opacity $\alpha_i \in [0, 1]$ (fixed after initialization)
*   Color $c_i$ (fixed after initialization)

**Method Steps:**

1.  **Influence Calculation:** For any 3D point $p$ in space, determine the Gaussian $g_k$ that has the most influence $f(p)$ over it. The influence function $f(p)$ is typically derived from the Gaussian's density contribution at that point.
2.  **Coordinate System Selection:**
    *   If the maximum influence $f(p) < \tau$ (where $\tau = 0.5$ [S1]), the point is considered part of the static background. The point's coordinates are mapped to the static background coordinate system.
    *   If $f(p) \ge \tau$, the point is mapped to the local coordinate system of the dominant Gaussian $g_k$. This establishes a well-defined, invertible one-to-one mapping for all points in space across all timesteps [S1].
3.  **2D Pixel Tracking:**
    *   **Unprojection:** To track a 2D pixel, first determine its corresponding 3D point. Render a depth map for the current view using the Max rendering equation for Gaussians [S1]. In this equation, replace the color component of each Gaussian with the depth of that Gaussian's center (distance from camera to $\mu_i$) [S1].
    *   **3D Location:** Unproject the 2D pixel using the rendered depth to find its 3D location $p$ [S1].
    *   **Tracking:** Determine the highest influence Gaussian for $p$ using the influence calculation (Step 1).
    *   **Reprojection:** Track the point through the selected Gaussian's coordinate transformation to the target timestep/view, and project it into the new camera frame [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_dominant_gaussian(point_3d, gaussians, threshold=0.5):
    """
    Determines the Gaussian with the highest influence over a 3D point.
    Returns the index of the dominant Gaussian or -1 if below threshold.
    """
    max_influence = -1.0
    dominant_idx = -1
    
    for i, g in enumerate(gaussians):
        # Calculate influence f(p) of point_3d on Gaussian g
        # This is typically the Gaussian density value at point_3d
        influence = gaussian_influence(point_3d, g)
        
        if influence > max_influence:
            max_influence = influence
            dominant_idx = i
            
    if max_influence < threshold:
        return -1, max_influence
    return dominant_idx, max_influence

def track_3d_point(point_3d, gaussians, threshold=0.5):
    """
    Maps a 3D point to a persistent coordinate system for tracking.
    """
    dom_idx, _ = get_dominant_gaussian(point_3d, gaussians, threshold)
    
    if dom_idx == -1:
        # Static background coordinate system
        return point_3d, "static"
    else:
        # Local coordinate system of the dominant Gaussian
        g = gaussians[dom_idx]
        # Transform point into Gaussian's local frame
        # Assuming g has rotation R and position mu
        local_point = g.R.inverse() @ (point_3d - g.mu)
        return local_point, dom_idx

def track_2d_pixel(pixel_2d, camera_params, gaussians, threshold=0.5):
    """
    Tracks a 2D pixel across timesteps/views.
    """
    # 1. Render depth map using Max rendering equation
    # Replace color with depth of Gaussian center
    depth_map = render_depth_max(gaussians, camera_params)
    
    # 2. Unproject pixel to 3D
    point_3d = unproject_pixel(pixel_2d, depth_map, camera_params)
    
    # 3. Determine dominant Gaussian
    local_point, dom_idx = track_3d_point(point_3d, gaussians, threshold)
    
    # 4. Track and reproject
    # Transform local_point back to world space using the Gaussian's current transform
    # at the target timestep, then project to target camera
    target_world_point = transform_to_target(local_point, dom_idx, target_timestep)
    target_pixel_2d = project_point(target_world_point, target_camera_params)
    
    return target_pixel_2d
```

**Parameters:**
*   **Influence Threshold ($\tau$):** 0.5 [S1]. Points with maximum Gaussian influence below this value are treated as static background.

**Edge Cases and Pitfalls:**
*   **Low Influence:** If a point is in a region with sparse Gaussian coverage (e.g., empty space or occluded regions), the influence may fall below 0.5, causing it to be mapped to the static background, potentially breaking correspondence for moving objects in those areas [S1].
*   **Static Background Assumption:** The method assumes a static background exists and is distinct from the dynamic foreground Gaussians [S1].

## Subtopics

### 4D Neural Voxel Splatting
A method that uses 4D neural voxels for dynamic scene representation. It differs by using voxel-based deformation rather than Gaussian influence mapping. Choose when voxel-based efficiency is preferred over Gaussian flexibility. [4D Neural Voxel Splatting](16404-4d-neural-voxel-splatting.md)

### Computational Deformation Field (CDF)
Uses a computational deformation field to model motion. It differs by explicitly learning a deformation field rather than relying on persistent Gaussian influence. Choose when explicit motion fields are required for physics-based consistency. [Computational Deformation Field (CDF)](19197-computational-deformation-field-cdf.md)

### MaGS Pipeline
A pipeline for multi-Gaussian scene representation. It differs in its overall architecture and training strategy. Choose when a comprehensive pipeline for multi-Gaussian scenes is needed. [MaGS Pipeline](20248-mags-pipeline.md)

### Dynamic Neural Motion Factorization (DynMF)
Factorizes motion into neural components. It differs by using neural factorization for motion rather than Gaussian persistence. Choose when motion factorization is beneficial for compact representation. [Dynamic Neural Motion Factorization (DynMF)](20633-dynamic-neural-motion-factorization-dynmf.md)

### Gated Second-Order Trajectory
Uses second-order trajectories with gating mechanisms. It differs by modeling motion with higher-order dynamics. Choose when smooth, high-order motion trajectories are critical. [Gated Second-Order Trajectory](20871-gated-second-order-trajectory.md)

### Neural Parametric Gaussians
Represents Gaussians with neural parametric functions. It differs by using neural networks to define Gaussian parameters. Choose when neural parameterization offers better generalization. [Neural Parametric Gaussians](21226-neural-parametric-gaussians.md)

## Operating characteristics
*   **Time Complexity:** Tracking is $O(N)$ per point, where $N$ is the number of Gaussians, as it requires calculating influence for each Gaussian to find the maximum [S1].
*   **Memory Complexity:** No additional memory is required for deformation fields or neural networks, as the Gaussians' parameters are fixed except for position and rotation [S2, unofficial].
*   **Strengths:** Simple implementation, no need for neural networks for motion prediction, dense correspondence [S1][S2, unofficial].
*   **Limitations:** Relies on the quality of the initial Gaussian reconstruction; may fail in regions with low Gaussian density or occlusion [S1].
*   **When to Use:** When dense tracking is needed for a scene already represented by Dynamic 3D Gaussians, and computational resources for neural motion fields are limited [S2, unofficial].
*   **When Not to Use:** When high accuracy in occluded regions is required, or when the scene has significant dynamic background elements that are not well-captured by the initial Gaussian set [S1].

## Background
Dynamic 3D Gaussian Tracking was introduced in "Dynamic 3D Gaussians: Tracking by Persistent Dynamic View Synthesis" [S1]. It builds on the Dynamic 3D Gaussian representation, which initializes Gaussians at the first timestamp and optimizes only position and rotation across frames [S2, unofficial]. This approach avoids the need for neural transformation caches or deformation fields used in methods like 3DGStream [S2, unofficial]. The method is particularly useful for applications requiring dense correspondence in dynamic scenes, such as sports analysis and human motion tracking [S1].

## Sources
[S1] Dynamic 3D Gaussians: Tracking by Persistent Dynamic View Synthesis, https://doi.org/10.1109/3dv62453.2024.00044, fetched 2026-10-26
[S2] Dynamic Scene Reconstruction: Recent Advance in Real-time Rendering and Streaming, https://doi.org/10.48550/arXiv.2503.08166, fetched 2026-10-29 (unofficial)
