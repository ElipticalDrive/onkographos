# Test-time Pose Refinement

Also called: Camera Pose Delta Optimization

## Overview
Test-time pose refinement is a post-processing optimization step applied to a fixed 3D reconstruction to correct inaccuracies in the estimated camera poses of held-out views. It solves the problem where initial pose estimates (e.g., from SfM or learned regressors) are misaligned by a few pixels, causing rendering artifacts despite a correct underlying geometry. It differs from training-time pose optimization (e.g., [BARF](15954-barf.md)) by freezing the scene representation entirely, isolating the improvement in metrics to pose alignment rather than reconstruction quality [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A frozen 3D scene representation (e.g., NeRF, 3DGS) and a set of held-out target images with their initial coarse camera poses $P_i$.
*   **Output:** Refined camera poses $P'_i$ for each held-out view.

**Data Structures:**
*   **Scene Representation:** Frozen parameters. No gradients flow into this structure.
*   **Pose Delta:** A learnable parameter vector $\delta_i$ for each held-out camera $i$. This delta represents the adjustment to the initial pose.

**Algorithm:**
For each held-out camera $i$:
1.  Initialize the learnable pose delta $\delta_i$ to zero (or a small random value) [G].
2.  Define the refined pose $P'_i(\delta_i)$ by applying the delta to the initial pose $P_i$.
3.  Render the image $I_{render}$ from $P'_i(\delta_i)$ using the frozen scene.
4.  Compute a loss $L$ between $I_{render}$ and the target image $I_{target}$. Common losses include MSE or LPIPS [G].
5.  Optimize $\delta_i$ using gradient descent to minimize $L$, keeping the scene parameters constant.
6.  Repeat for a fixed number of steps.

**Parameters:**
*   **Optimization Steps:** 400 steps per camera [S1, unofficial].
*   **Learning Rate:** Not given in the sources.
*   **Optimizer:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def test_time_pose_refinement(scene, held_out_views, optimization_steps=400):
    """
    Refines camera poses for held-out views while keeping the scene frozen.
    
    Args:
        scene: A frozen 3D scene representation (e.g., NeRF, 3DGS).
        held_out_views: List of dictionaries containing 'target_image' and 'initial_pose'.
        optimization_steps: Number of optimization steps per view.
        
    Returns:
        refined_poses: List of refined camera poses.
    """
    refined_poses = []
    
    for view in held_out_views:
        target_image = view['target_image']
        initial_pose = view['initial_pose']
        
        # Initialize learnable pose delta. 
        # The delta is applied to the initial pose to get the refined pose.
        # Implementation detail: The delta can be a 6D vector (3 for translation, 3 for rotation)
        # or a 4x4 matrix update. Here we assume a generic differentiable pose update function.
        pose_delta = torch.zeros(6, requires_grad=True) # [G] Assuming 6D delta
        
        optimizer = torch.optim.Adam([pose_delta], lr=0.01) # [G] Standard default
        
        for step in range(optimization_steps):
            optimizer.zero_grad()
            
            # Compute refined pose from initial pose and delta
            refined_pose = apply_pose_delta(initial_pose, pose_delta)
            
            # Render image from the refined pose using the frozen scene
            rendered_image = render(scene, refined_pose)
            
            # Compute loss between rendered and target image
            # [G] Common losses: MSE, LPIPS, or SSIM. MSE is used here as a placeholder.
            loss = torch.nn.functional.mse_loss(rendered_image, target_image)
            
            # Backpropagate and update only the pose delta
            loss.backward()
            optimizer.step()
            
        # Store the final refined pose
        final_refined_pose = apply_pose_delta(initial_pose, pose_delta.detach())
        refined_poses.append(final_refined_pose)
        
    return refined_poses

def apply_pose_delta(initial_pose, delta):
    """
    Applies a 6D delta (tx, ty, tz, rx, ry, rz) to a pose.
    [G] This is a generic implementation detail. The exact method of applying
    the delta (e.g., matrix multiplication, exponential map for rotation)
    depends on the specific pose representation.
    """
    # Extract translation and rotation deltas
    tx, ty, tz, rx, ry, rz = delta
    
    # Update translation
    new_translation = initial_pose.translation + torch.tensor([tx, ty, tz])
    
    # Update rotation (simplified example using small angle approximation or axis-angle)
    # [G] For robust implementation, use proper rotation composition (e.g., quaternion multiplication)
    rotation_delta = axis_angle_to_rotation_matrix(torch.tensor([rx, ry, rz]))
    new_rotation = rotation_delta @ initial_pose.rotation
    
    return Pose(translation=new_translation, rotation=new_rotation)
```

**Edge Cases and Pitfalls:**
*   **Divergence on Weak Baselines:** Applying refinement to poor initial poses (e.g., from MVS-Texturing) can cause the optimizer to diverge away from the actual object, worsening the result [S1, unofficial]. Refinement is only reported for stable methods [S1, unofficial].
*   **Frozen Scene:** The scene must be strictly frozen to prevent overfitting to the held-out views, which would invalidate the metric comparison [S1, unofficial].

## Subtopics

### NeRF-
A specific pose refinement strategy used in S-NeRF that directly learns shifts to translation and rotation [S2, unofficial]. It is considered more straightforward than BARF or SCNeRF and achieves the best quality on self-driving datasets, likely because the shifts are easier to learn under depth supervision and with small camera FOV overlap [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear in the number of held-out views and optimization steps. Each view is optimized independently.
*   **Memory Complexity:** Low, as the scene is frozen and only per-camera pose deltas are stored.
*   **Performance:** Isolates the contribution of pose alignment to overall metrics. Metrics reported with refinement show the score attributable to pose alignment, while metrics without refinement reflect reconstruction quality [S1, unofficial].
*   **Strengths:** Simple to implement; effectively corrects small pose errors (a few pixels) [S1, unofficial].
*   **Limitations:** Can diverge if initial poses are too inaccurate [S1, unofficial]. Does not improve reconstruction quality, only alignment [S1, unofficial].

## Background
Test-time pose refinement was introduced in the context of evaluating 3D reconstruction quality, specifically in the CADSplat paper [S1, unofficial]. It was used to distinguish between errors in the reconstructed geometry and errors in the camera pose estimation. The method reframes the evaluation question to whether the object renders accurately from a camera near its nominal pose [S1, unofficial]. It is also used in S-NeRF for self-driving datasets, where variants like NeRF- are preferred for their simplicity and effectiveness [S2, unofficial].

## Sources
[S1] CADSplat: Sparse-View 3D Gaussian Splatting Aided by CAD Models for Robust, Photorealistic Digital-Twin Reconstruction, https://arxiv.org/abs/2609.18473, fetched 2026-10-30 (unofficial)
[S2] S-NeRF: Neural Radiance Fields for Street Views, https://doi.org/10.48550/arxiv.2303.00749, fetched 2026-10-29 (unofficial)
