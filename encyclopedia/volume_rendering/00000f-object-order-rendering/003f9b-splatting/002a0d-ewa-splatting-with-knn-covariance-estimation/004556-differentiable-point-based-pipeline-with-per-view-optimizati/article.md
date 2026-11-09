# Differentiable Point-Based Pipeline with Per-View Optimization

Also called: Differentiable Point-Splatting Framework, Point-Based Neural Rendering

## Overview
This method is a neural rendering pipeline initialized with Multi-View Stereo (MVS) geometry that allows for the optimization of scene properties, specifically depth and reprojected features, within the space of input views to improve novel-view synthesis [S1]. It distinguishes itself from fixed MVS approaches by recovering from reconstruction errors through per-view optimization, and from volumetric neural radiance fields (NeRFs) by using explicit point-based geometry which is more stable and faster [S1]. It builds on [EWA Splatting with kNN Covariance Estimation](4690-ewa-splatting-with-knn-covariance-estimation.md) by extending forward EWA splatting to a bi-directional differentiable pipeline [S1].

## How it works
The pipeline initializes with standard 3D reconstruction (MVS geometry) and optimizes per-input-view attributes (reprojected latent features and depth) via backpropagation through a differentiable renderer [S1].

**Inputs:**
*   MVS-reconstructed point cloud (positions, initial colors/features).
*   Input views: Camera poses, intrinsic parameters, and source images.
*   Target novel view: Camera pose and intrinsic parameters.

**Outputs:**
*   Rendered image of the novel view.
*   Gradients for optimizing per-view depth and features.

**Data Structures:**
*   **Point Cloud:** A list of points $P_i$. Each point $i$ stores:
    *   3D position $\mathbf{p}_i \in \mathbb{R}^3$.
    *   Per-view reprojection attributes: A feature vector $\mathbf{f}_{i,v}$ and a depth offset $\delta_{i,v}$ for each input view $v$ that contributes to the point. [G] The feature vector typically includes color and potentially other latent attributes. The depth offset allows the point's effective depth to vary per view to correct MVS errors.
    *   3D Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$, estimated from k-nearest neighbors as per the parent method [4690-ewa-splatting-with-knn-covariance-estimation.md].

**Steps:**

1.  **Camera Selection:**
    Select the subset of input views $V_{sel}$ that provide the best coverage for the novel view. The method uses the K-Maximum coverage algorithm [HP98] to efficiently select cameras that maximize the projected area of the point cloud onto the novel view's image plane [S1].

2.  **Bi-Directional Reprojection:**
    For each selected input view $v \in V_{sel}$ and each point $i$:
    *   Project the 3D point $\mathbf{p}_i$ into the 2D image plane of view $v$.
    *   Sample the source image $I_v$ at the projected 2D coordinates to obtain the initial feature/color $\mathbf{c}_{i,v}$.
    *   Apply **bi-directional EWA filtering**: The point is splatted from the 3D space into the 2D image of view $v$ using the 3D covariance $\Sigma_i$ transformed by the view matrix. This creates a soft, alpha-blended representation of the point in the 2D view space [S1]. This step allows gradients to flow from the 2D image space back to the 3D point attributes.

3.  **Per-View Optimization (Training Phase):**
    *   The renderer computes the novel view image by compositing the reprojected points from all $v \in V_{sel}$.
    *   A **probabilistic depth test** is applied to resolve depth conflicts. Since points are soft splats, the method uses the distribution of depths from each reprojected input view to determine the visibility and contribution of each point, rather than a hard z-buffer [S1].
    *   The rendered novel view is compared to a ground-truth novel view (if available) or used in a self-supervised loss.
    *   Backpropagation updates the per-view parameters: the reprojected features $\mathbf{f}_{i,v}$ and the depth offsets $\delta_{i,v}$ for each point $i$ and view $v$. This allows the system to correct inaccurate MVS geometry and appearance inconsistencies per view [S1].

4.  **Rendering (Inference Phase):**
    *   Once optimized, the renderer uses the fixed optimized parameters to generate novel views.
    *   Soft, alpha-blended splatting of the reprojected points is performed.
    *   The probabilistic depth test resolves occlusions using the optimized depth distributions [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def differentiable_point_pipeline(points, input_views, novel_view, num_epochs):
    """
    points: List of Point objects with 3D pos, covariance, and per-view attributes.
    input_views: List of View objects (camera, image).
    novel_view: Target View object (camera).
    """
    # 1. Camera Selection
    selected_views = k_maximum_coverage(points, novel_view, input_views)
    
    # 2. Initialize Optimization
    for view in selected_views:
        for point in points:
            # Initialize per-view depth offset and feature gradient buffers
            point.per_view_depth_offset[view.id] = 0.0
            point.per_view_feature_grad[view.id] = zeros_like(point.color)

    for epoch in range(num_epochs):
        total_loss = 0.0
        
        # 3. Forward Pass: Reprojection and Splatting
        rendered_image = zeros_like(novel_view.image)
        alpha_accum = zeros_like(rendered_image[:,:,0])
        
        # Sort points by depth relative to novel view for correct blending
        sorted_points = sort_points_by_depth(points, novel_view)
        
        for point in sorted_points:
            # Bi-directional EWA Reprojection for each selected input view
            contributions = []
            for view in selected_views:
                # Project 3D point to 2D view space, accounting for depth offset
                pos_3d = point.pos + view.normal * point.per_view_depth_offset[view.id]
                pos_2d = project_to_image(pos_3d, view.camera)
                
                # Sample feature from input view image using EWA filtering
                # This step is differentiable w.r.t pos_2d and thus pos_3d
                feature = ewa_sample(view.image, pos_2d, point.covariance_3d_to_2d(view.camera))
                
                # Compute weight based on EWA kernel and depth probability
                weight = ewa_kernel(pos_2d, point.covariance_3d_to_2d(view.camera))
                depth_prob = probabilistic_depth_test(point, view, novel_view)
                
                contributions.append({
                    'feature': feature,
                    'weight': weight * depth_prob,
                    'alpha': ewa_alpha(pos_2d, point.covariance_3d_to_2d(view.camera))
                })
            
            # Composite contributions into novel view space
            # Soft alpha-blending
            for contrib in contributions:
                rendered_image += contrib['feature'] * contrib['weight']
                alpha_accum += contrib['alpha'] * contrib['weight']
        
        # Normalize by accumulated alpha
        rendered_image = safe_div(rendered_image, alpha_accum)
        
        # 4. Loss Calculation
        # Assuming ground truth novel view is available for supervised loss
        loss = mse_loss(rendered_image, novel_view.image)
        
        # 5. Backpropagation
        # Gradients flow back through the differentiable splatting and reprojection
        # to update point.per_view_depth_offset and point.per_view_feature_grad
        gradients = compute_gradients(loss, rendered_image, novel_view.image)
        
        for view in selected_views:
            for point in sorted_points:
                # Update per-view attributes
                point.per_view_depth_offset[view.id] -= lr * gradients.depth_offset[view.id]
                point.per_view_feature_grad[view.id] -= lr * gradients.feature[view.id]
                
        total_loss += loss
        
    return rendered_image
```

**Parameters:**
*   **K-Maximum Coverage:** The number of cameras to select. Not given in the sources.
*   **Learning Rate:** Not given in the sources.
*   **EWA Kernel Width:** Determined by the projected covariance matrix. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Inaccurate Geometry:** When reprojecting from input views with inaccurate MVS geometry, it is hard to determine which view contains the correct information. The probabilistic depth test and per-view optimization help mitigate this, but severe errors may lead to artifacts [S1].
*   **Wide-Baseline Artifacts:** In wide-baseline, free-viewpoint scenarios, methods that do not optimize per-view parameters (like standard NeRFs) often present artifacts. This method avoids this by optimizing per-view features and depth [S1].
*   **Holes in Point Cloud:** If holes occur during point-cloud generation, the reconstruction result shows vacancy in the rendered result [S2] (Note: This specific pitfall is cited for ADOP, a variant, but applies to point-based methods generally).

## Subtopics

### [Bi-Directional Differentiable Point Cloud Rasterization](11503-bi-directional-differentiable-point-cloud-rasterization.md)
A bi-directional point splatting approach from input views to a common novel view space and back, using EWA filtering with Jacobians to allow per-view attribute optimization. It enables the optimization of per-view features in the 2D space of the input images, extending forward EWA splatting [11503-bi-directional-differentiable-point-cloud-rasterization.md].

### ADOP (Approximate Differentiable One-Pixel Point Rendering)
A point-cloud-based rendering model for novel view synthesis that uses dense reconstruction results (e.g., from COLMAP) [S2]. It trains for 100 epochs and applies gamma correction ($\gamma = 1/2.2$) by design, not a learned CRF function [S2]. It uses a mask covering the right half side of the images for dense reconstruction and model optimization [S2]. Pitfalls include instability in local optima leading to imperfect color, and vacancies in rendered results if holes occur in the point cloud [S2]. It shows biased results in estimating satisfying color and shadows compared to HDR-Plenoxels, with low performance in inferring overall white balance and changing colors [S2].

## Operating characteristics
*   **Time Complexity:** The camera selection uses the K-Maximum coverage algorithm for fast selection [S1]. The rendering involves bi-directional EWA splatting for each point and selected view. Complexity is proportional to $N \times K$, where $N$ is the number of points and $K$ is the number of selected views.
*   **Memory Complexity:** Stores per-point attributes for each selected view (depth offset, features). This is higher than fixed MVS methods but lower than volumetric NeRFs which store dense voxel grids or large MLPs.
*   **Measured Performance:** The method outperforms all previous methods in quality and speed in almost all tested scenes [S1].
*   **Strengths:** Recovers from MVS reconstruction errors; high-quality, temporally coherent rendering; suitable for free-viewpoint IBR, multi-view harmonization, and stylization [S1].
*   **Limitations:** Requires initial MVS reconstruction; performance depends on the quality of the initial point cloud; bi-directional reprojection adds computational overhead compared to forward-only splatting.
*   **When to use:** High-quality image-based rendering, multi-view stylization, and multi-view harmonization where per-view optimization is beneficial [S1].
*   **When not to use:** Scenes with significant transient elements (unless handled by extensions like NeRF-A) or where real-time performance is critical and MVS initialization is too slow [S2].

## Background
The method was introduced by G. Kopanas, J. Philip, T. Leimkühler, and G. Drettakis [S1]. It builds on the field of neural rendering, which has seen rapid expansion [S1]. It addresses limitations of MVS-based methods (fixed geometry) and volumetric neural representations (expensive training/inference, artifacts in wide-baseline scenarios) [S1]. The approach extends forward EWA point splatting [YSW*19] to be bi-directional [S1]. It is used for high-quality image-based rendering, multi-view stylization, and multi-view harmonization [S1].

## Sources
[S1] Point-Based Neural Rendering with Per-View Optimization, https://www.youtube.com/watch?v=TupOewenSAk, fetched 2026-10-31
[S2] HDR-Plenoxels: Self-Calibrating High Dynamic Range Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_23, fetched 2026-10-26
