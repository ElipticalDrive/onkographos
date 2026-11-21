# GS-SLAM
## Overview
GS-SLAM is a Simultaneous Localization and Mapping (SLAM) method that simultaneously estimates camera poses and reconstructs 3D scenes using 3D Gaussians [S1][S2, unofficial]. It distinguishes itself from other radiance-field SLAM approaches by employing an adaptive 3D Gaussian expanding strategy to dynamically add new Gaussians and delete unreliable ones based on captured depth and rendered opacity [S1][S2, unofficial].
## How it works
**Inputs and Outputs:**
- Inputs: RGB-D image frames (color and depth), initial camera poses, and an initial set of 3D Gaussians.
- Outputs: Optimized camera poses, updated 3D Gaussian parameters (position, covariance, opacity, spherical harmonics coefficients), and a dense 3D scene representation.

**Data Structures:**
- **3D Gaussians:** Each Gaussian $i$ is defined by a mean position $\mu_i$, a covariance matrix $\Sigma_i$ (parameterized by rotation $R_i$ and scale $S_i$), an opacity $\alpha_i$, and spherical harmonics coefficients $c_{i,lm}$ for view-dependent color [S1][S2, unofficial].
- **Depth Buffer:** Captured depth values from the RGB-D sensor for each pixel.
- **Opacity Buffer:** Rendered opacity values for each pixel.

**Algorithm Steps:**
1. **Initialization:** Initialize the scene with a sparse set of 3D Gaussians, typically derived from initial pose estimates or feature points [S1][S2, unofficial].
2. **Rendering:** For each incoming RGB-D frame, render the current set of 3D Gaussians to produce predicted color and opacity images using differentiable volume rendering [S1][S2, unofficial].
3. **Loss Computation:** Compute the photometric loss between the rendered color and the observed RGB image. Compute a depth loss between the rendered depth (or expected depth from Gaussians) and the captured depth [S1][S2, unofficial].
4. **Adaptive Expanding (Adding Gaussians):**
   - Identify regions where the rendered opacity is low or the depth error is high, indicating missing geometry [S1][S2, unofficial].
   - Add new 3D Gaussians in these regions. The position of new Gaussians can be sampled from the depth map or existing Gaussian distributions [S1][S2, unofficial].
5. **Pruning (Deleting Gaussians):**
   - Evaluate the reliability of existing Gaussians.
   - Delete Gaussians that are deemed unreliable. Criteria include low opacity and/or high discrepancy with captured depth [S1][S2, unofficial].
6. **Optimization:** Update the parameters of all active Gaussians (position, scale, rotation, opacity, color) and the camera pose using stochastic gradient descent to minimize the combined loss [S1][S2, unofficial].
7. **Iteration:** Repeat steps 2-6 for subsequent frames.

**Formulas:**
- Photometric Loss: $L_{rgb} = ||I_{rendered} - I_{observed}||_1$ [S1][S2, unofficial].
- Depth Loss: $L_{depth} = ||D_{rendered} - D_{observed}||_1$ [S1][S2, unofficial].
- Total Loss: $L = L_{rgb} + \lambda L_{depth}$, where $\lambda$ is a weighting factor [S1][S2, unofficial].

**Parameters:**
- $\lambda$: Weight for depth loss. Not given in the sources.
- Opacity threshold for pruning: Not given in the sources.
- Depth error threshold for adding Gaussians: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Duplicate Densification:** Without careful management, new Gaussians may be added in regions already well-represented. GS-SLAM avoids this by using captured depths and rendered opacity to guide expansion [S1][S2, unofficial].
- **Unreliable Gaussians:** Gaussians with low opacity or high depth error should be removed to maintain efficiency and accuracy [S1][S2, unofficial].
- **Initialization:** Poor initial poses or sparse initial Gaussians can lead to convergence issues [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gs_slam_loop(frames, initial_gaussians, initial_poses):
    gaussians = initial_gaussians
    poses = initial_poses
    
    for frame in frames:
        rgb = frame.rgb
        depth = frame.depth
        
        # Render current scene
        rendered_rgb, rendered_opacity, rendered_depth = render(gaussians, poses[-1])
        
        # Compute losses
        loss_rgb = l1_loss(rendered_rgb, rgb)
        loss_depth = l1_loss(rendered_depth, depth)
        loss = loss_rgb + lambda_depth * loss_depth
        
        # Backpropagate to update Gaussians and Pose
        update_gaussians_and_pose(gaussians, poses, loss)
        
        # Adaptive Expanding
        # Identify regions with low opacity or high depth error
        low_opacity_mask = rendered_opacity < opacity_threshold
        high_depth_error_mask = abs(rendered_depth - depth) > depth_error_threshold
        expand_mask = low_opacity_mask | high_depth_error_mask
        
        # Add new Gaussians in expanded regions
        new_gaussians = add_gaussians(gaussians, poses[-1], depth, expand_mask)
        gaussians = gaussians + new_gaussians
        
        # Prune unreliable Gaussians
        # Remove Gaussians with low opacity or high depth error
        gaussians = prune_gaussians(gaussians, rendered_opacity, depth)
        
        poses.append(poses[-1]) # Update pose estimate
        
    return gaussians, poses
```
[G] The pseudocode assumes standard differentiable rendering and optimization routines. Specific thresholds and addition/pruning heuristics are not detailed in the sources and would require implementation-specific tuning.

## Subtopics

### [SplaTAM](9193-splatam.md)
Uses view-independent colors and a densification mask to avoid duplicate densification.

### [Gaussian-SLAM](9195-gaussian-slam.md)
Uses an extra scale regularization loss to encourage isotropic Gaussians.

### [Gaussian Splatting SLAM](10078-gaussian-splatting-slam.md)
Uses an extra scale regularization loss to encourage isotropic Gaussians.

### [Differentiable Pose Estimation](12756-differentiable-pose-estimation.md)
Focuses on the pose estimation component within the SLAM framework.

## Operating characteristics
- **Performance:** On the Replica dataset, GS-SLAM achieves a PSNR of 31.56, SSIM of 0.97, and LPIPS of 0.094 [S1][S2, unofficial].
- **Complexity:** Not given in the sources.
- **Strengths:** Adaptive strategy allows for dynamic scene reconstruction and handling of unseen areas [S1][S2, unofficial].
- **Limitations:** Performance is lower than some other 3DGS SLAM methods like GaussianSplattingSLAM (PSNR 38.90) on the Replica dataset [S1][S2, unofficial].

## Background
GS-SLAM was proposed in the context of recent advances in 3D Gaussian Splatting for SLAM applications [S1][S2, unofficial]. It addresses the limitations of implicit representation-based methods by using explicit 3D Gaussians, which can be more efficiently optimized and rendered [S1][S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
