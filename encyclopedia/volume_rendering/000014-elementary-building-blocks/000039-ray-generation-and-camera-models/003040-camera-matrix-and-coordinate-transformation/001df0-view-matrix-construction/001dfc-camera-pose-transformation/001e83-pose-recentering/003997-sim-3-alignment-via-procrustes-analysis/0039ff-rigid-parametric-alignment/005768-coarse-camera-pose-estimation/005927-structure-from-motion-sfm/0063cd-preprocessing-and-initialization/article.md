# Preprocessing and Initialization
## Overview
This method establishes the geometric foundation for neural volume rendering by estimating camera poses and initializing the spatial data structures (binoctree) from raw input video or images. It solves the problem of aligning unstructured input data into a coherent global coordinate frame and initializing the scene representation with a sparse but accurate 3D point cloud. It differs from sibling methods like [Canonical Frame Estimation via Facial Landmarks](16399-canonical-frame-estimation-via-facial-landmarks.md) by relying on general Structure-from-Motion (SfM) pipelines rather than semantic landmarks, making it applicable to general 3D scenes rather than just faces.
## How it works
### Inputs and Outputs
**Inputs:**
- Raw video frames or image sequence.
- (Optional) Pre-trained disparity estimation network weights.

**Outputs:**
- Camera poses (extrinsics) and intrinsics for each frame.
- A global 3D point cloud in metric units.
- An initialized spherical binoctree structure.
- Scaling coefficient for unit sphere normalization.

### Data Structures
1. **Camera Pose:** $4 \times 4$ matrix $P$ representing world-to-camera transformation.
2. **Intrinsics:** $3 \times 3$ matrix $K$ containing focal lengths and principal points.
3. **Point Cloud:** Set of 3D points $C = \{c_i\}$ where $c_i \in \mathbb{R}^3$.
4. **Binoctree:** Hierarchical structure where each node (sphoxel) stores SDF and color information. Initialized via subdivision.

### Steps
1. **Camera Pose Estimation:**
   - Use a Structure-from-Motion (SfM) algorithm (e.g., OpenVSLAM [S1] or COLMAP [S2]) to estimate per-frame camera poses from the input video/images.
   - If using COLMAP [S2], configure a "simple radial" camera model with a single radial distortion coefficient and shared intrinsic parameters across all images.
   - Enable "sift feature guided matching" in the exhaustive matcher step.
   - Refine principal points of the intrinsic parameters during bundle adjustment.

2. **Depth and Point Cloud Extraction:**
   - For each frame, extract a dense 3D point cloud using spherical rectification [S1] and a fine-tuned disparity estimation network [S1].
   - Combine per-frame poses and calibrated depth maps to form a global coordinate frame in metric units.

3. **Coordinate Frame Construction:**
   - Place the origin at the center of all camera positions.
   - Set the near sphere bound to contain all camera positions.
   - Set the far sphere bound per scene to fit the recovered point cloud inside the sphere.
   - Place a final sphere at the far plane with view-independent color decoded by a background MLP to represent scene elements at infinity (e.g., sky) [S1].

4. **Normalization:**
   - Scale both the binoctree coordinates and the point cloud coordinates into a unit sphere.
   - Store the scaling coefficient for later metric re-scaling during rendering or evaluation.

5. **Binoctree Initialization:**
   - Use the scaled initial 3D point cloud to subdivide the binoctree to its finest level.
   - Continue subdivision until all sphoxels reach a minimum solid angle $\alpha_{min}$.
   - Derive $\alpha_{min}$ per scene from baseline and image sizes, typically setting $\alpha_{min}$ between $5 \times 10^{-4}$ and $1 \times 10^{-3}$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_and_initialize(video_frames, disparity_model=None):
    # Step 1: Estimate Camera Poses
    # Use OpenVSLAM [S1] or COLMAP [S2]
    if use_openvslam:
        poses = OpenVSLAM.estimate_poses(video_frames)
    elif use_colmap:
        # Configure COLMAP with simple radial model
        colmap_config = {
            "camera_model": "simple_radial",
            "radial_distortion_coeff": 1,
            "shared_intrinsic": True,
            "matcher": "sift_feature_guided",
            "refine_principal_points": True
        }
        poses, intrinsics = COLMAP.run(video_frames, colmap_config)
    
    # Step 2: Extract Point Cloud
    point_clouds = []
    for frame, pose in zip(video_frames, poses):
        # Rectify and estimate disparity
        rectified_frame = spherical_rectification(frame)
        if disparity_model:
            disparity_map = disparity_model.predict(rectified_frame)
        else:
            disparity_map = estimate_disparity(rectified_frame)
        
        # Convert to 3D points using pose and intrinsics
        points = depth_to_3d(disparity_map, pose, intrinsics)
        point_clouds.append(points)
    
    # Step 3: Form Global Coordinate Frame
    # Combine all point clouds
    global_point_cloud = concatenate(point_clouds)
    
    # Calculate center of cameras
    camera_centers = [pose.translation for pose in poses]
    origin = mean(camera_centers)
    
    # Set bounds
    near_bound = max_distance_from_origin(camera_centers)
    far_bound = max_distance_from_origin(global_point_cloud)
    
    # Step 4: Normalize to Unit Sphere
    scale_coefficient = 1.0 / far_bound
    scaled_point_cloud = global_point_cloud * scale_coefficient
    scaled_binoctree_bounds = [-1, 1] # Unit sphere
    
    # Step 5: Initialize Binoctree
    alpha_min = 1e-3 # Typical value from [S1]
    binoctree = Binoctree()
    binoctree.initialize_from_points(scaled_point_cloud, alpha_min)
    
    return poses, intrinsics, scaled_point_cloud, binoctree, scale_coefficient
```

### Parameters
- $\alpha_{min}$: Minimum solid angle for sphoxels. Typical range: $5 \times 10^{-4}$ to $1 \times 10^{-3}$ [S1].
- Camera Model: "simple_radial" with 1 distortion coefficient [S2].

### Edge Cases and Pitfalls
- **Inaccurate Poses:** Errors in camera poses and intrinsic parameters lead to poor synthesis results [S2].
- **Boundary Conditions:** The near sphere must contain all camera positions; the far sphere must contain the point cloud [S1].
- **Infinity Representation:** Scene elements at infinity (e.g., sky) require a separate background MLP at the far plane [S1].

## Subtopics

### COLMAP Camera Calibration & SfM Pipeline
A specific implementation of the pose estimation step using the COLMAP software package. It differs from other SfM implementations by its specific configuration options for neural rendering pipelines. It is chosen when high-accuracy intrinsic refinement is required. Key steps include using a "simple radial" camera model, "sift feature guided matching," and refining principal points during bundle adjustment [S2]. Pitfall: Inaccurate poses/intrinsics lead to poor results [S2].

## Operating characteristics
- **Time Complexity:** Dominated by SfM feature matching and depth estimation. COLMAP's exhaustive matching is $O(N^2)$ in worst case but optimized with SIFT [S2].
- **Memory Complexity:** Proportional to the number of frames and the density of the point cloud. The binoctree initialization scales with the number of points and the target resolution ($\alpha_{min}$) [S1].
- **Strengths:** Provides a metric global coordinate frame; handles omnidirectional video [S1]; initializes structure accurately from sparse data.
- **Limitations:** Requires accurate SfM; fails if feature matching is poor (e.g., textureless regions); depth estimation network may fail on untrained domains [S1].
- **When to Use:** When starting from raw video/images and a global metric frame is needed.
- **When Not to Use:** When camera poses are already known and accurate, or when semantic landmarks are more appropriate (e.g., faces) [S1, S2].

## Background
Originating from the need to initialize neural radiance fields (NeRF) and related methods from unstructured video, this method combines traditional SfM with modern depth estimation. OpenVSLAM [S1] is used for omnidirectional video, while COLMAP [S2] is used for standard image sets. The binoctree initialization strategy was introduced to efficiently represent sparse 3D data in a hierarchical structure suitable for neural rendering [S1].

## Sources
[S1] OmniSDF: Scene Reconstruction Using Omnidirectional Signed Distance Functions and Adaptive Binoctrees, https://doi.org/10.1109/cvpr52733.2024.01912, fetched 2026-10-26
[S2] NeX: Real-time View Synthesis with Neural Basis Expansion, https://doi.org/10.1109/cvpr46437.2021.00843, fetched 2026-10-26
