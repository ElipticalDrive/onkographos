# SNeRF

Also called: Neural Radiance Fields for Street Views, S-NeRF

## Overview
SNeRF is a neural radiance field method designed for novel view synthesis in large-scale, unbounded street-view scenes collected by self-driving cars, jointly reconstructing static backgrounds and foreground moving vehicles [S2, unofficial]. Unlike sibling methods such as [Block-NeRF](52-block-nerf.md) or [Urban-NeRF](52-urban-nerf.md) which require heavy camera overlap or dense LiDAR depth, SNeRF operates effectively with sparse, noisy LiDAR points and minimal camera overlap by employing implicit pose refinement and a robust confidence metric [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Sparse and noisy LiDAR point clouds, multi-view RGB images from self-driving car cameras, initial camera poses derived from sensor-fusion SLAM and IMU, and viewing directions.
*   **Outputs:** Synthesized novel-view RGB images and depth maps for both static background and dynamic foreground objects.

**Data Structures:**
*   **NeRF MLPs:** Two successive Multi-Layer Perceptrons (MLPs) implement the mapping function $f$.
    *   Input: 3D position $\mathbf{x} \in \mathbb{R}^3$ and viewing direction $\boldsymbol{\theta} \in \mathbb{R}^3$.
    *   Output: Color $\mathbf{c} \in \mathbb{R}^3$ and differential density $\sigma \in \mathbb{R}$.
*   **Pose Refinement Offsets:** A learnable pair $\Delta P = (\Delta R, \Delta T)$ where $\Delta R \in \mathbb{R}^{3 \times 3}$ (rotation offset) and $\Delta T \in \mathbb{R}^3$ (translation offset) are added to initial poses $P = [R, T]$.
*   **Confidence Map:** A scalar value per depth sample derived from geometry and reprojection consistency, used to weight depth losses.

**Numbered Steps:**

1.  **Initial Pose Estimation:** Obtain initial camera poses $P$ using sensor-fusion SLAM and IMU data from the self-driving vehicle [S2, unofficial].
2.  **Pose Refinement (Static Background):** For static background scenes, a learning-based network implicitly learns refinement offsets $\Delta P$ to correct SLAM errors and reduce inconsistency between multi-camera views [S2, unofficial]. The refined pose is $P' = P + \Delta P$.
3.  **Virtual Camera Transformation (Moving Vehicles):** For foreground moving vehicles, compute the relative position $\hat{P}$ between the camera and the target object center. Apply a virtual camera transformation to handle the complex motion of both the ego-vehicle and the target object, enabling reconstruction of dynamic entities [S2, unofficial].
4.  **Depth Completion and Confidence Estimation:**
    *   Propagate sparse LiDAR depths using a depth completion network (e.g., Park et al., 2020) [S2, unofficial].
    *   Compute a robust confidence metric based on geometry consistency and reprojection consistency to identify and handle outliers in the sparse LiDAR data [S2, unofficial].
5.  **Volume Rendering:** For each ray $r(t) = o + td$, sample $N$ points along the ray between near and far bounds. Render the color $\hat{I}(r)$ using the volume rendering equation [S2, unofficial]:
    $$ \hat{I}(r) = \sum_{i=1}^{N} T_i (1 - e^{-\sigma_i \delta_i}) c_i $$
    where $T_i = \exp \left( -\sum_{j=1}^{i-1} \sigma_j \delta_j \right)$, $\delta_j = t_{j+1} - t_j$, and $c_i, \sigma_i$ are the color and density at sample $t_i$ [S2, unofficial].
6.  **Loss Computation:** Calculate RGB loss between rendered and ground truth images, and depth loss between rendered depth and LiDAR-derived depth, weighted by the confidence metric [S2, unofficial].
7.  **Optimization:** Train the NeRF MLPs, pose refinement network, and confidence estimator using the combined losses [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def snerf_render(ray_origin, ray_direction, nerf_mlp, pose_refinement_net, 
                 initial_pose, lidar_depths, confidence_net):
    """
    Renders a single ray using SNeRF components.
    """
    # 1. Refine Camera Pose
    # Learn offsets delta_R and delta_T to correct SLAM errors
    delta_R, delta_T = pose_refinement_net(initial_pose)
    refined_rotation = initial_pose.R @ delta_R
    refined_translation = initial_pose.T + delta_T
    
    # Transform ray origin and direction to the refined camera frame
    # [G] Standard rigid body transformation: 
    # r_cam = R_refined^T * (r_world - T_refined)
    ray_origin_cam = transform_to_camera_frame(ray_origin, refined_rotation, refined_translation)
    ray_direction_cam = transform_direction(ray_direction, refined_rotation)

    # 2. Sample Points along Ray
    # Stratified sampling between near and far bounds
    t_vals = stratified_sample(near_bound, far_bound, num_samples=N)
    points_3d = ray_origin_cam + t_vals[:, None] * ray_direction_cam
    
    # 3. Query NeRF MLP
    # Input: 3D position and viewing direction
    # Output: Color c and density sigma
    colors, densities = nerf_mlp(points_3d, ray_direction_cam)
    
    # 4. Volume Rendering
    # Compute transmittance T_i
    deltas = t_vals[1:] - t_vals[:-1]
    # [G] Handle first element: T_1 = 1.0
    transmittance = torch.cumprod(torch.exp(-densities[:-1] * deltas), dim=0)
    transmittance = torch.cat([torch.ones(1), transmittance], dim=0)
    
    # Compute alpha: alpha_i = 1 - exp(-sigma_i * delta_i)
    alphas = 1.0 - torch.exp(-densities * deltas)
    
    # Weighted sum of colors
    weights = alphas * transmittance
    rendered_color = torch.sum(weights[:, None] * colors, dim=0)
    
    # 5. Depth and Confidence (for training supervision)
    # Use depth completion network to propagate sparse LiDAR
    completed_depth = depth_completion_network(lidar_depths)
    
    # Compute confidence based on geometry and reprojection consistency
    confidence = confidence_net(completed_depth, rendered_depth)
    
    return rendered_color, confidence

def train_snerf(dataset, nerf_mlp, pose_net, confidence_net, optimizer):
    """
    Alternating training loop for SNeRF.
    """
    for epoch in range(num_epochs):
        # Step A: Render images from different views
        rendered_images = []
        for view in dataset.views:
            ray = generate_ray(view.camera, view.pixel_coords)
            color, conf = snerf_render(ray, nerf_mlp, pose_net, 
                                       view.initial_pose, view.lidar, confidence_net)
            rendered_images.append(color)
            
        # Step B: Stylize rendered images (if stylization module is active)
        # [S1] SNeRF proposes alternating training to manage memory.
        # Stylize the rendered images using an external image stylization module.
        stylized_images = image_stylization_module(rendered_images)
        
        # Step C: Train NeRF to match stylized images and depth
        # Compute RGB loss between stylized and ground truth (or original if not stylizing)
        rgb_loss = compute_rgb_loss(stylized_images, dataset.gt_images)
        
        # Compute depth loss weighted by confidence
        depth_loss = compute_depth_loss(rendered_depths, dataset.lidar_depths, confidence)
        
        # Backpropagate and update parameters
        total_loss = rgb_loss + depth_loss
        optimizer.zero_grad()
        total_loss.backward()
        optimizer.step()
```

**Parameters:**
*   **Number of samples ($N$):** Not given in the sources.
*   **Learning Rate:** Not given in the sources.
*   **Near/Far Bounds:** Determined by the scene scale and camera frustum; specific values not given.

**Edge Cases and Pitfalls:**
*   **Sparse LiDAR Outliers:** Standard depth supervision fails with noisy/sparse LiDAR. SNeRF addresses this by learning a confidence metric based on reprojection and geometry consistency [S2, unofficial].
*   **Moving Objects:** Conventional NeRFs fail on moving vehicles. SNeRF uses virtual camera transformation to handle dynamic foregrounds [S2, unofficial].
*   **Memory Limitations:** The alternating training approach (stylize then train) allows hardware capacity to be allocated separately, addressing memory constraints [S1].

## Subtopics

### Pose Refinement
Implicitly learns rotation and translation offsets to correct SLAM errors. Choose when initial poses are noisy. Link: [Pose Refinement](52-pose-refinement.md).

### Virtual Camera Transformation
Handles dynamic foregrounds by computing relative positions. Choose for scenes with moving objects. Link: [Virtual Camera Transform](52-virtual-camera-transform.md).

### Confidence-Based Depth Loss
Uses reprojection and geometry consistency to weight depth errors. Choose when LiDAR is sparse/noisy. Link: [Confidence Metric](52-confidence-metric.md).

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On large-scale driving datasets (nuScenes and Waymo), SNeRF reduces mean-squared error by 7–40% compared to state-of-the-art rivals [S2, unofficial]. It achieves a 45% PSNR gain for moving vehicles rendering compared to conventional NeRFs or mesh-based methods [S2, unofficial].
*   **Strengths:** Works with panoramic settings where rays barely intersect and overlaps are small; handles noisy/sparse LiDAR; reconstructs moving vehicles.
*   **Limitations:** Requires SLAM/IMU data for initial poses; relies on external depth completion network.
*   **When to use:** Large-scale street-view synthesis with sparse LiDAR and low camera overlap.
*   **When not to use:** Object-centric scenes with high overlap (use standard NeRF); scenes with dense, accurate LiDAR (Urban-NeRF may be simpler).

## Background
SNeRF was published as a conference paper at ICLR 2023 by Ziyang Xie, Junge Zhang, Wenye Li, Feihu Zhang, and Li Zhang from Fudan University and the University of Oxford [S2, unofficial]. It addresses the limitations of original NeRF, which is designed for object-centric scenes with heavy camera overlap, and fails on unbounded outdoor street-view data collected by self-driving cars [S2, unofficial]. The method builds on the NeRF++ framework [Parent] to handle large-scale backgrounds and foregrounds jointly.

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S2] S-NeRF: Neural Radiance Fields for Street Views, https://doi.org/10.48550/arxiv.2303.00749, fetched 2026-10-29 (unofficial)
