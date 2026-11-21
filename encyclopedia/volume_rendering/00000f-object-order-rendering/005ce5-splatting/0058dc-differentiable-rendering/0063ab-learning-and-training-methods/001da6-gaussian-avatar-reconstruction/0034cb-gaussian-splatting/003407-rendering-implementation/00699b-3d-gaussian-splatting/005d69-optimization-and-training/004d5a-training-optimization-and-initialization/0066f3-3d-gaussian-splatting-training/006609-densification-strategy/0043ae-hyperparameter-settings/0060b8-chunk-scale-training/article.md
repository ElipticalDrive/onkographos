# Chunk-scale Training

Also called: Chunk-based training, Chunk optimization

## Overview
Chunk-scale Training addresses the optimization challenges of large, sparse, unbounded scenes by dividing the environment into independent spatial chunks, each trained separately to handle low capture density and non-object-centric data [S1]. It distinguishes itself from standard 3D Gaussian Splatting (3DGS) by introducing a skybox for environmental consistency, modifying densification policies to handle sparse gradients, and applying monocular depth regularization for unobserved surfaces [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A large scene dataset with sparse camera poses, Structure-from-Motion (SfM) points, and image observations.
*   **Output:** Optimized 3D Gaussian primitives for a specific spatial chunk, ready for hierarchical consolidation.

**Data Structures:**
*   **Chunk Bounds:** A spatial region defining the extent of the current training unit.
*   **Skybox:** A set of 3D Gaussian primitives placed on a sphere surrounding the scene.
*   **Coarse Environment:** A pre-optimized representation of the scene outside the current chunk, used to provide context and prevent sky inconsistencies.

**Parameters:**
*   **Skybox Size:** 100,000 3DGS primitives [S1].
*   **Skybox Radius:** A sphere with a diameter 10× the diameter of the scene extent [S1].
*   **Camera Selection Threshold:** Cameras are included if they are inside the chunk bounds or within 2× the chunk bounds, provided they have more than 50 SfM points within the chunk bounds [S1].
*   **Densification Metric:** Maximum of observed screen-space gradients (replacing the mean used in standard 3DGS) [S1].

**Steps:**
1.  **Define Skybox:** Place 100,000 3DGS primitives on a sphere with a diameter 10× the scene extent to capture sky effects and prevent inconsistent sky content across chunks [S1].
2.  **Select Cameras:** Identify cameras for the current chunk. Include cameras inside the chunk bounds or within 2× the chunk bounds that have >50 SfM points within the chunk bounds [S1].
3.  **Initialize Coarse Environment:** Load the coarse optimization of the scene outside the current chunk. This environment remains fixed in geometry but is temporarily optimized for opacity and SH coefficients during chunk training to ensure consistency [S1].
4.  **Train Chunk Content:** Perform 3DGS optimization for the content inside the chunk. Use correct anti-aliasing techniques [S1].
5.  **Temporary Environment Optimization:** Perform a small, temporary optimization of the coarse environment and skybox outside the chunk. Only optimize opacity and SH coefficients; do not modify geometry or position [S1].
6.  **Apply Modified Densification:** Use the maximum of the observed screen-space gradient for densification decisions, rather than the mean, to address sparse capture issues where mean gradients are unreliable [S1].
7.  **Depth Regularization:** Perform monocular depth prediction. Scale and shift the predicted depth based on available SfM points. Use this depth map to regularize the optimization, particularly for unobserved surfaces like streets [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chunk_scale_training(chunk_bounds, scene_sfm_points, images, cameras):
    # 1. Define Skybox
    # Sphere diameter = 10 * scene_diameter
    skybox_primitives = create_skybox_primitives(count=100000, radius=10 * scene_diameter)
    
    # 2. Select Cameras
    # Cameras inside chunk OR within 2x chunk bounds AND >50 SfM points in chunk
    selected_cameras = []
    for cam in cameras:
        if is_inside_bounds(cam, chunk_bounds):
            selected_cameras.append(cam)
        elif is_within_factor(cam, chunk_bounds, factor=2.0):
            sfm_points_in_chunk = count_sfm_points_in_bounds(cam.sfm_points, chunk_bounds)
            if sfm_points_in_chunk > 50:
                selected_cameras.append(cam)
    
    # 3. Load Coarse Environment
    # Geometry is fixed; Opacity and SH are optimized
    coarse_env = load_coarse_optimization_outside(chunk_bounds)
    
    # 4. Initialize Chunk Gaussians
    # Start from SfM points refined for this chunk
    chunk_gaussians = initialize_gaussians_from_sfm(chunk_bounds)
    
    # 5. Optimization Loop
    for iteration in range(num_iterations):
        # Render current view
        rendered_image = render(chunk_gaussians, coarse_env, skybox_primitives, cam)
        
        # Compute Loss
        loss = compute_loss(rendered_image, target_image)
        
        # Backpropagate
        # Note: Coarse env only updates Opacity and SH
        # Chunk Gaussians update all parameters
        
        # 6. Modified Densification
        # Use MAX gradient instead of MEAN
        max_gradients = compute_max_screen_space_gradients(chunk_gaussians)
        densify_gaussians(chunk_gaussians, max_gradients)
        
        # 7. Depth Regularization
        # Predict depth, scale/shift using SfM, add to loss
        pred_depth = monocular_depth_prediction(target_image)
        scaled_depth = scale_shift_depth(pred_depth, scene_sfm_points)
        depth_loss = compute_depth_loss(chunk_gaussians, scaled_depth)
        loss += depth_loss_weight * depth_loss
        
        # Update parameters
        update_parameters(chunk_gaussians, coarse_env, loss)
        
    return chunk_gaussians
```

**Edge Cases and Pitfalls:**
*   **Sparse Capture:** Standard mean-based densification rarely discourages densification in unbounded scenes and has low tendency in sparse captures, leading to poor quality in areas like streets [S1]. The maximum gradient policy addresses this [S1].
*   **Inconsistency:** Without a skybox and coarse environment, chunks may create inconsistent sky content [S1].
*   **Unobserved Surfaces:** Sparse camera captures lack information for good quality reconstruction of large unobserved areas (e.g., roads) [S1]. Depth regularization is required [S1].

## Operating characteristics
*   **Time Complexity:** Training is performed per chunk, allowing for parallelization across chunks. However, the need for temporary optimization of the coarse environment and skybox adds overhead per chunk [S1].
*   **Memory Complexity:** Memory usage is bounded by the size of a single chunk plus the skybox and coarse environment, making it feasible for very large datasets that would not fit in memory as a single 3DGS model [S1].
*   **Strengths:** Handles large, sparse, unbounded scenes; prevents sky inconsistencies; improves quality in unobserved areas via depth regularization [S1].
*   **Limitations:** Requires careful chunk boundary definition; relies on monocular depth prediction which may be inaccurate; temporary optimization of the coarse environment adds computational cost [S1].
*   **When to Use:** Large-scale urban or outdoor scenes with sparse camera captures and non-object-centric data [S1].
*   **When Not to Use:** Small, dense, object-centric scenes where standard 3DGS performs well [S1].

## Background
Chunk-scale Training was introduced in "A Hierarchical 3D Gaussian Representation for Real-Time Rendering of Very Large Datasets" by Kerbl et al. (2024) [S1]. It was developed to extend 3D Gaussian Splatting to very large datasets, addressing the limitations of the original method which assumed dense, object-centric captures [S1]. The method builds on the hierarchical structure and consolidation techniques described in the same work [S1].

## Sources
[S1] A Hierarchical 3D Gaussian Representation for Real-Time Rendering of Very Large Datasets, https://www.youtube.com/watch?v=p-mb2TzVJZk, fetched 2026-10-31
