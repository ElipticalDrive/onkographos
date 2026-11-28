# Test-time Pose Optimization

Also called: Freezing 3DGS for Pose Optimization, Test View Pose Estimation, Test-time Pose Optimization for 3DGS, Test-time camera optimization, Test-time optimization for view synthesis

## Overview
Test-time Pose Optimization is a procedure to obtain accurate camera poses for testing views by freezing a pre-trained 3D Gaussian Splatting (3DGS) model and optimizing only the test view poses via photometric error minimization [S2]. It sets itself apart from sibling methods by decoupling the scene representation (frozen) from the camera parameters (optimized), specifically addressing pose errors that degrade novel view synthesis quality [S2]. It builds on [Test-time Optimization with Sim(3) Initialization](8565-test-time-optimization-with-sim-3-initialization.md) by applying optimization to the final test poses.

## How it works
**Inputs:**
- Pre-trained 3DGS model (Gaussian attributes and covariance fixed).
- Test view images.
- Initial test view poses (initialized via Procrustes alignment and nearest neighbor training poses).

**Outputs:**
- Optimized camera poses for the test views.

**Data Structures:**
- Camera Pose: A $4 \times 4$ transformation matrix $P \in \mathbb{R}^{4 \times 4}$ representing the world-to-camera transformation.
- Photometric Loss: Mean Squared Error (MSE) or L1 loss between rendered pixels and ground truth pixels.

**Steps:**
1. **Procrustes Alignment:** Register the cameras of the test images using the transformation matrix obtained via Procrustes analysis on the predicted and ground truth cameras of the training images [S1]. This provides a global alignment baseline.
2. **Initialization:** Initialize the camera pose of each test view using the closest camera position from the learned poses of all training frames [S2].
3. **Optimization:** Optimize the camera poses of the test images by minimizing the photometric loss between the synthesized images and the test view images [S1, S2]. This factors out the effect of pose error on view synthesis quality [S1].
4. **Interleaving (Optional):** In some pipelines (e.g., GNeRF), the test-time optimization is interleaved with the training process, where the total number of iterations depends on the number of training steps seen for the best checkpoint [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def test_time_pose_optimization(frozen_3dgs_model, test_images, initial_poses, num_iterations):
    """
    Optimizes test camera poses by minimizing photometric error.
    
    Args:
        frozen_3dgs_model: Pre-trained 3DGS model with fixed Gaussian attributes.
        test_images: List of ground truth test view images.
        initial_poses: List of initial camera poses (after Procrustes and NN initialization).
        num_iterations: Number of optimization steps.
        
    Returns:
        optimized_poses: List of optimized camera poses.
    """
    # Initialize trainable parameters with initial poses
    # Each pose is a 4x4 matrix, but typically only rotation (3x3) and translation (3x1) are optimized
    # or the full 4x4 matrix is treated as a flat vector of 16 elements for gradient descent.
    # For 3DGS, we assume the model renders given a pose.
    
    poses = [pose.clone().detach().requires_grad_(True) for pose in initial_poses]
    optimizer = torch.optim.Adam(poses, lr=0.001) # Learning rate not specified in sources [G]
    
    for i in range(num_iterations):
        optimizer.zero_grad()
        total_loss = 0.0
        
        for pose, gt_image in zip(poses, test_images):
            # Render image using frozen 3DGS model and current pose
            rendered_image = frozen_3dgs_model.render(pose)
            
            # Compute photometric loss (e.g., MSE or L1)
            # Note: Exact loss function (L1 vs L2) is not specified in sources [G]
            loss = torch.nn.functional.mse_loss(rendered_image, gt_image)
            total_loss += loss
            
        total_loss.backward()
        optimizer.step()
        
    return [pose.detach() for pose in poses]
```

**Parameters:**
- Number of test-time optimization iterations: Typically ~4230 [S1]. Can be increased (e.g., to 1000) for scenes with pose outliers to ensure convergence [S1]. Note: The source mentions increasing from 100 to 1000 improves metrics, but also states typical is 4230. This suggests 4230 is the standard for the GNeRF pipeline, while 1000 is a minimum for difficult cases [S1].

**Edge Cases and Pitfalls:**
- **Outlier Sensitivity:** Procrustes alignment is severely affected by outlier images, which can cause exaggerated misalignment and prevent test-time optimization from converging accurately [S1].
- **Convergence:** If Procrustes alignment is poor due to outliers, increasing the number of optimization iterations is necessary to converge to a more accurate viewpoint [S1].

## Operating characteristics
- **Time Complexity:** The optimization is iterative. The total time depends on the number of iterations and the cost of rendering each test view. For the GNeRF pipeline, it runs approximately 4230 iterations [S1].
- **Memory Complexity:** Low, as the 3DGS model is frozen and only camera poses are stored as trainable parameters.
- **Performance:** On a single RTX 3090 GPU, the method is significantly faster than methods like Nope-NeRF (~30 h) taking ~2 h [S2].
- **Strengths:** Factors out pose error effects on view synthesis quality [S1]. Simple to implement as it only optimizes camera poses.
- **Limitations:** Sensitive to initial alignment quality (Procrustes). May not converge accurately if initial poses are severely misaligned due to outliers [S1].
- **When to use:** When test view poses are unknown or inaccurate, and high-quality novel view synthesis is required.
- **When not to use:** When computational resources are extremely limited and approximate poses are acceptable, or when the scene has severe outliers that break Procrustes alignment.

## Background
The method is inspired by NeRFmm [49] and is used in the context of COLMAP-free 3D Gaussian Splatting [S2]. It is also employed in the GNeRF pipeline for evaluating novel view synthesis [S1]. The procedure follows established practices for evaluating novel view synthesis where test poses are not given [S1].

## Sources
[S1] LU-NeRF: Scene and Pose Estimation by Synchronizing Local Unposed NeRFs, https://doi.org/10.1109/iccv51070.2023.01679, fetched 2026-10-26
[S2] COLMAP-Free 3D Gaussian Splatting, https://doi.org/10.1109/cvpr52733.2024.01965, fetched 2026-10-26
