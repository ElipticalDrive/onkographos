# Test-time Optimization with Sim(3) Initialization

Also called: BARF test optimization, Sim(3) + opt

## Overview
A testing procedure involving Sim(3) alignment initialization followed by additional test-time optimization to adjust test poses for view synthesis [S1, unofficial]. It sets itself apart from sibling methods like [Rigid Parametric Alignment](8609-rigid-parametric-alignment.md) by explicitly decoupling the coarse alignment (via Procrustes analysis on locations) from the fine-grained pose refinement (via gradient-based optimization) [S1, unofficial].

## How it works
**Inputs:**
*   Estimated camera poses for the test set (rotation matrices $R_i$ and translation vectors $t_i$).
*   Ground truth camera poses for the training set (used for alignment reference).
*   Neural Radiance Field (NeRF) or similar implicit representation.
*   Optimization hyperparameters (learning rate, iteration count).

**Outputs:**
*   Refined camera poses for the test set.
*   Rendered images from the refined poses.

**Data Structures:**
*   Camera Pose: A 4x4 homogeneous transformation matrix $P_i$ or separate $R_i \in SO(3)$ and $t_i \in \mathbb{R}^3$.
*   NeRF Model: An MLP mapping 5D coordinates $(x, y, z, \theta, \phi)$ to color and density.

**Steps:**
1.  **Sim(3) Initialization:** Compute a 3D similarity transformation (rotation $R_{sim}$, scale $s$, translation $t_{sim}$) that aligns the estimated test poses to the ground truth training poses using Procrustes analysis on the camera location coordinates [S1, unofficial]. Apply this transformation to the initial test poses.
    *   *Note:* This step assumes the estimated poses can be aligned precisely to the reference poses. Incorrect pose estimations can cause this initialization to fail [S1, unofficial].
2.  **Test-Time Optimization:** Perform gradient-based optimization to further adjust the test poses. The optimization minimizes the rendering loss (e.g., MSE between predicted and ground truth pixel colors) with respect to the pose parameters ($R_i, t_i$) and potentially NeRF weights, though the primary focus is pose adjustment [S1, unofficial].
3.  **Iteration:** Repeat the optimization for a fixed number of iterations.

**Parameters:**
*   **Test-optimization iteration count:** 100 iterations for methods other than NoPe-NeRF [S1, unofficial].
*   **Learning Rate:** Not given in the sources.
*   **Optimizer:** Not given in the sources (BARF typically uses Adam).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def test_time_optimization_with_sim3_initialization(
    test_poses_estimated, 
    train_poses_gt, 
    nerf_model, 
    num_iterations=100
):
    """
    Refines test poses using Sim(3) initialization followed by optimization.
    
    Args:
        test_poses_estimated: List of estimated poses for test images.
        train_poses_gt: Ground truth poses for training images (reference).
        nerf_model: The NeRF model to optimize.
        num_iterations: Number of optimization steps.
        
    Returns:
        refined_test_poses: List of optimized test poses.
    """
    # Step 1: Sim(3) Alignment Initialization
    # Compute Sim(3) transform from estimated test poses to ground truth train poses
    # This involves Procrustes analysis on camera locations
    sim_transform = compute_sim3_alignment(test_poses_estimated, train_poses_gt)
    
    # Apply Sim(3) transform to initialize test poses
    test_poses_initial = apply_sim3_transform(test_poses_estimated, sim_transform)
    
    # Step 2: Test-Time Optimization
    # Initialize optimizer with test_poses_initial and nerf_model parameters
    optimizer = torch.optim.Adam(
        list(nerf_model.parameters()) + [test_poses_initial], 
        lr=0.001  # [G] Typical learning rate for BARF-style optimization
    )
    
    for i in range(num_iterations):
        optimizer.zero_grad()
        
        # Render images using current test poses and NeRF model
        # For each test image, sample rays based on current pose
        rendered_images = []
        for pose in test_poses_initial:
            rays = generate_rays_from_pose(pose)
            colors, densities = nerf_model.forward(rays)
            rendered_images.append(colors)
        
        # Compute loss against ground truth images
        # Note: Ground truth images for test set are required for this loss
        loss = compute_rendering_loss(rendered_images, ground_truth_test_images)
        
        # Backpropagate and update
        loss.backward()
        optimizer.step()
        
    # Extract refined poses from the optimized parameters
    refined_test_poses = extract_poses_from_optimizer_state(optimizer)
    
    return refined_test_poses

def compute_sim3_alignment(poses_est, poses_gt):
    """
    Computes Sim(3) alignment using Procrustes analysis on camera locations.
    Returns rotation, scale, and translation.
    """
    # Extract camera centers from poses
    centers_est = [pose.translation for pose in poses_est]
    centers_gt = [pose.translation for pose in poses_gt]
    
    # Align centers using Procrustes analysis
    # This finds R, s, t that minimize || s*R*centers_est + t - centers_gt ||^2
    # [G] Standard Procrustes algorithm: center data, compute SVD of covariance matrix
    R, s, t = procrustes_analysis(centers_est, centers_gt)
    
    return {'R': R, 's': s, 't': t}

def apply_sim3_transform(poses, sim_transform):
    """
    Applies Sim(3) transform to a list of poses.
    """
    R_sim = sim_transform['R']
    s_sim = sim_transform['s']
    t_sim = sim_transform['t']
    
    transformed_poses = []
    for pose in poses:
        # Apply rotation, scale, and translation
        # P_new = [s*R, t; 0, 1] * P_old
        new_R = s_sim * pose.rotation @ R_sim.T # [G] Order of multiplication depends on convention
        new_t = s_sim * (pose.rotation @ t_sim) + pose.translation # [G] Simplified assumption
        # Note: Exact application depends on whether Sim(3) is applied pre- or post-multiplication
        # and whether the pose represents world-to-camera or camera-to-world.
        # For simplicity, assuming standard application to camera centers and orientations.
        new_pose = Pose(new_R, new_t)
        transformed_poses.append(new_pose)
        
    return transformed_poses
```

## Subtopics

### SAMURAI Initialization
A variant of the initialization scheme used in the SAMURAI method [S2]. It offers two specific initialization strategies:
1.  **Fixed Initialization:** All initial view directions are set to `[Center, Above, Center]` (North pole) [S2]. When optimizing poses in $SO(3)$, the predicted camera locations are overwritten using the formula $t_i = -R_i \times T_i$ (Equation 19 in the source) [S2].
2.  **Manual Initialization:** Each view direction is specified with a triplet `[Left/Center/Right, Above/Center/Below, Front/Center/Back]` [S2]. The combination `[Center, Center, Center]` is disallowed, resulting in 26 possible initial directions [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of optimization iterations and the number of test images. Each iteration requires a forward pass through the NeRF and a backward pass for pose gradients.
*   **Memory Complexity:** Depends on the NeRF model size and the number of test images being optimized simultaneously.
*   **Strengths:** Provides a robust initialization for test poses when the estimated poses are roughly aligned with the ground truth. It can refine poses to improve view synthesis quality [S1, unofficial].
*   **Limitations:** Performance is highly dependent on the quality of the initial pose estimates. Incorrect pose estimations can lead to failure in the Sim(3) alignment step [S1, unofficial]. It may struggle with complex trajectories and reduced frame overlap compared to methods like Neighbor initialization in high-overlap scenarios [S1, unofficial].
*   **When to use:** When estimated poses are reasonably accurate and can be aligned via a global similarity transform. Suitable for scenarios where test-time compute is available.
*   **When not to use:** When estimated poses are significantly incorrect or when the trajectory has large variations and low overlap that prevent a global Sim(3) alignment from being meaningful.

## Background
The method is derived from the BARF (Blending Aliased Reconstructions for Conditional NeRFs) approach [S1, unofficial]. It was introduced in the context of NeRF pose optimization, where test-time optimization is used to refine camera poses for novel view synthesis. The Sim(3) alignment step leverages Procrustes analysis to provide a good starting point for the optimization, addressing the issue of pose ambiguity in NeRF training [S1, unofficial].

## Sources
[S1] CT-NeRF: Incremental Optimizing Neural Radiance Field and Poses with Complex Trajectory, https://doi.org/10.48550/arxiv.2404.13896, fetched 2026-10-26 (unofficial)
[S2] MELON: NeRF with Unposed Images in SO(3), https://doi.org/10.1109/3dv62453.2024.00084, fetched 2026-10-30
