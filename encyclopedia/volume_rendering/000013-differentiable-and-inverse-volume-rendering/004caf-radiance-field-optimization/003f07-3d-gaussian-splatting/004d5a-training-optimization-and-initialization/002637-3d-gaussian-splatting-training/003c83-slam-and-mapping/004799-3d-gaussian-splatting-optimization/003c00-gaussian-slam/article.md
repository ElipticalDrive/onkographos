# Gaussian-SLAM

Also called: Yugay et al. 2024 SLAM

## Overview
Gaussian-SLAM is a keyframe-based Simultaneous Localization and Mapping (SLAM) method that constructs a scene representation using 3D Gaussians and employs sub-map fusion to manage global consistency [S2, unofficial]. It distinguishes itself from sibling methods like SplaTAM, which relies on depth-assisted ICP for tracking, by utilizing a specific scale regularization loss to enforce isotropic Gaussians, thereby stabilizing the joint optimization of camera poses and map geometry [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of RGB images (and optionally depth, though the core definition emphasizes the Gaussian representation and regularization) captured by a moving camera.
*   **Outputs:** A set of 3D Gaussians $\mathcal{G}$ representing the scene, and a trajectory of camera poses $\mathbf{T}_t$.

**Data Structures:**
The scene is represented as a set of 3D Gaussians. Each Gaussian $i$ is defined by:
*   Mean position $\mu_i \in \mathbb{R}^3$.
*   Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$, parameterized by scale $\mathbf{s}_i \in \mathbb{R}^3$ and rotation $\mathbf{R}_i \in SO(3)$.
*   Opacity $\alpha_i \in [0, 1]$.
*   Color $c_i$ (typically represented via spherical harmonics coefficients).

**Optimization Objective:**
The method minimizes a loss function $L$ that couples photometric rendering with geometric regularization:
$$ L = L_{photo} + \lambda L_{depth} + L_{reg} + L_{scale} $$
Where:
*   $L_{photo} = || I_{rendered} - I_{captured} ||_1$ measures the color difference between the rendered image and the captured frame.
*   $L_{depth} = || D_{rendered} - D_{captured} ||_1$ measures the depth difference if depth data is available [S1].
*   $L_{reg}$ includes standard regularization terms.
*   $L_{scale}$ is an extra scale regularization loss applied to the scale of Gaussians to encourage isotropic Gaussians, which stabilizes localization and mapping [S1].

**Algorithm Steps:**
1.  **Initialization:** Initialize the set of 3D Gaussians $\mathcal{G}$ and camera poses $\mathbf{T}_t$ for the initial keyframes.
2.  **Keyframe Selection:** Select new keyframes from the input stream based on motion or coverage criteria.
3.  **Sub-map Fusion:** When a new keyframe is added, fuse its local Gaussian representation with the existing global map. This involves aligning the new Gaussians with the existing structure and resolving overlaps.
4.  **Joint Optimization:** Optimize the parameters of the Gaussians ($\mu_i, \mathbf{s}_i, \mathbf{R}_i, \alpha_i, c_i$) and the camera poses $\mathbf{T}_t$ simultaneously.
    *   Compute the rendered image $I_{rendered}$ from the current Gaussian set and camera poses.
    *   Compute the photometric loss $L_{photo}$.
    *   Compute the depth loss $L_{depth}$ if depth is available.
    *   Compute the scale regularization loss $L_{scale}$ to penalize anisotropic scales (e.g., large differences between scale components).
    *   Update parameters using gradient descent to minimize $L$.
5.  **Densification (Implicit):** While not explicitly detailed in the provided notes for Gaussian-SLAM specifically, the general framework implies that new Gaussians are added or existing ones are refined to cover areas with poor reconstruction, guided by the optimization gradients and the scale regularization [S1].
6.  **Loop Closure (Optional):** Detect revisited locations and correct drift by fusing sub-maps or adjusting poses globally.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_slam_optimization(keyframes, initial_gaussians):
    """
    Optimizes camera poses and 3D Gaussians for Gaussian-SLAM.
    """
    # Initialize global state
    global_gaussians = initial_gaussians
    camera_poses = {kf.id: kf.pose for kf in keyframes}
    
    for kf in keyframes:
        # 1. Add new keyframe to the map (Sub-map fusion)
        # This step aligns the new keyframe's local Gaussians with the global map
        local_gaussians = initialize_gaussians_from_keyframe(kf)
        global_gaussians = fuse_submaps(global_gaussians, local_gaussians)
        
        # 2. Joint Optimization
        # Optimize poses and Gaussian parameters for the current keyframe and its neighbors
        for iteration in range(num_iterations):
            # Render image for the current keyframe
            rendered_image = render_gaussians(global_gaussians, camera_poses[kf.id])
            
            # Compute losses
            loss_photo = l1_loss(rendered_image, kf.rgb_image)
            
            # Depth loss if available
            if kf.has_depth:
                rendered_depth = render_depth(global_gaussians, camera_poses[kf.id])
                loss_depth = l1_loss(rendered_depth, kf.depth_image)
            else:
                loss_depth = 0.0
                
            # Scale regularization to encourage isotropic Gaussians
            loss_scale = scale_regularization(global_gaussians)
            
            # Total loss
            total_loss = loss_photo + lambda_depth * loss_depth + lambda_scale * loss_scale
            
            # Backpropagate and update
            # Update camera poses
            camera_poses[kf.id] = optimize_pose(camera_poses[kf.id], global_gaussians, total_loss)
            
            # Update Gaussian parameters
            global_gaussians = optimize_gaussians(global_gaussians, total_loss)
            
            # Check for convergence or densification criteria
            if needs_densification(global_gaussians, kf):
                global_gaussians = densify_gaussians(global_gaussians, kf)
                
    return global_gaussians, camera_poses

def scale_regularization(gaussians):
    """
    Encourages isotropic Gaussians by penalizing variance in scale components.
    """
    loss = 0.0
    for g in gaussians:
        # Penalize the difference between scale components
        # e.g., variance of log(scales) or max/min ratio
        scales = g.scale
        loss += variance(scales)
    return loss
```

**Parameters:**
*   $\lambda_{depth}$: Weight for the depth loss. Not given in the sources.
*   $\lambda_{scale}$: Weight for the scale regularization loss. Not given in the sources.
*   $num\_iterations$: Number of optimization iterations per keyframe. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Gaussian Degeneracy:** Without $L_{scale}$, Gaussians may become overly anisotropic, leading to unstable gradients and poor localization [S1].
*   **Map Drift:** Accumulated pose errors can cause misalignment in sub-map fusion. Loop closure is necessary to correct this [S2, unofficial].
*   **Duplicate Densification:** Care must be taken to avoid adding redundant Gaussians in well-reconstructed areas, which increases computational cost [S1].

## Subtopics

### MonoGS
A monocular RGB-only SLAM method that uses online optimization with Gaussian Splatting [S2, unofficial]. It differs from Gaussian-SLAM by not relying on keyframe-based sub-map fusion explicitly in its primary definition, focusing instead on real-time monocular tracking and mapping.

### Vox-Fusion
A hybrid SLAM system combining a voxel-based neural implicit surface representation with an octree structure for dynamic scene expansion [S3]. It encodes and optimizes each scene voxel implicitly, supporting incremental mapping without prior environment knowledge and leveraging a multi-process framework for speed [S3].

### Neural SLAM Pipeline
An end-to-end simultaneous localization and mapping approach that produces dense, photorealistic maps while tracking camera pose, replacing sparse classical SLAM outputs [S4, unofficial]. Its pipeline involves: 1. Tracking the current frame against the existing map. 2. Updating the map with new views by inserting new Gaussians where coverage is poor and refining existing ones. 3. Optional loop closure to detect revisited locations and correct drift. 4. Continuously rendering the photorealistic map [S4, unofficial]. It is denser and more photorealistic than classical SLAM but requires heavier compute [S4, unofficial].

## Operating characteristics
*   **Performance:** On the Replica dataset, Gaussian-SLAM achieves a PSNR of 37.50, SSIM of 0.96, and LPIPS of 0.07 [S1].
*   **Complexity:** The method involves joint optimization of Gaussians and poses, which is computationally intensive. The sub-map fusion adds complexity to the mapping process.
*   **Strengths:** Provides a dense, photorealistic map that can be used for both localization and rendering. The scale regularization stabilizes the optimization [S1].
*   **Limitations:** Heavier compute requirements compared to classical SLAM. Less mature for real-time deployment on edge hardware [S4, unofficial].

## Background
Gaussian-SLAM was introduced by Yugay et al. in 2024 [S2, unofficial]. It is part of the wave of 2024 methods that apply 3D Gaussian Splatting to SLAM, leveraging the efficiency and quality of Gaussian representations for simultaneous localization and mapping.

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S4] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
