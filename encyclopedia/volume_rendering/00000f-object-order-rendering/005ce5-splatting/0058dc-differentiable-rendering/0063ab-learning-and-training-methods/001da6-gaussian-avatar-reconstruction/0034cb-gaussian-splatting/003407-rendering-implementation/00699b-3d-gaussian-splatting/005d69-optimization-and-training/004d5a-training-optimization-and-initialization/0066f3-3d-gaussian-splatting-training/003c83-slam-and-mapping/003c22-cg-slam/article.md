# CG-SLAM


## Overview
CG-SLAM is a 3D Gaussian Splatting-based SLAM method that improves reconstruction quality by incorporating an uncertainty map derived from rendered depth into the training process [S1][S2, unofficial]. It leverages the explicit geometry of 3D Gaussians to enable flexible reprojection, addressing viewpoint misalignment more effectively than NeRF-based methods [S1][S2, unofficial].

## How it works
CG-SLAM operates within the iterative loop defined by [SLAM and Mapping](9268-slam-and-mapping.md), specifically modifying the loss computation and gradient propagation steps to account for depth uncertainty.

**Inputs and Outputs:**
*   **Inputs:** Camera poses $T$, 3D Gaussian parameters $\mathcal{G}$ (position $\mathbf{p}$, covariance $\Sigma$, opacity $\alpha$, color $\mathbf{c}$, spherical harmonics coefficients), and ground-truth depth $D$.
*   **Outputs:** Updated camera poses $T'$, updated Gaussian parameters $\mathcal{G}'$, and a refined 3D map.

**Data Structures:**
*   **Uncertainty Map ($U$):** A 2D image map aligned with the rendered depth, where each pixel value represents the estimated uncertainty of the corresponding depth measurement. The specific formula for generating $U$ from rendered depth is not detailed in the sources [G: Typically derived from the variance of depth estimates or consistency checks across frames].

**Steps:**
1.  **Differentiable Rendering:** Render the current scene state to produce a depth map $\hat{D}$ and an RGB image $\hat{I}$ using alpha-compositing of projected 3D Gaussians.
2.  **Uncertainty Estimation:** Generate an uncertainty map $U$ based on the rendered depth $\hat{D}$. This map quantifies the reliability of the depth predictions [S1][S2, unofficial].
3.  **Loss Computation:** Compute the total loss $\mathcal{L}$, which includes standard photometric and geometric terms, but is weighted or regularized by the uncertainty map $U$. The introduction of $U$ into the training process is the key differentiator that greatly improves reconstruction quality [S1][S2, unofficial].
4.  **Gradient Propagation:** Backpropagate gradients $\frac{\partial \mathcal{L}}{\partial T}$ and $\frac{\partial \mathcal{L}}{\partial \mathcal{G}}$ through the differentiable renderer, accounting for the uncertainty-weighted loss.
5.  **Optimization:** Update $T$ and $\mathcal{G}$ using gradient descent (e.g., Adam) or bundle adjustment.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cg_slam_step(camera_poses, gaussians, gt_depth, gt_rgb):
    # 1. Render scene
    rendered_depth, rendered_rgb = render_gaussians(camera_poses, gaussians)
    
    # 2. Generate uncertainty map from rendered depth
    # Note: Specific formula for uncertainty generation is not provided in sources [G]
    uncertainty_map = compute_uncertainty_from_depth(rendered_depth)
    
    # 3. Compute loss with uncertainty weighting
    # Standard losses (photometric, geometric) are modified by uncertainty
    loss = compute_uncertainty_aware_loss(rendered_rgb, rendered_depth, 
                                          gt_rgb, gt_depth, uncertainty_map)
    
    # 4. Backpropagate
    gradients = compute_gradients(loss, camera_poses, gaussians)
    
    # 5. Update parameters
    camera_poses = update_parameters(camera_poses, gradients['poses'])
    gaussians = update_parameters(gaussians, gradients['gaussians'])
    
    return camera_poses, gaussians
```

**Parameters:**
*   Uncertainty weighting scheme: Not given in the sources.
*   Optimizer: Not given in the sources (standard Adam or BA likely used).

**Edge Cases:**
*   Degenerate depth values (e.g., infinity) in rendered depth may require masking before uncertainty computation [G].

## Operating characteristics
*   **Time Complexity:** Real-time rendering is enabled by the 3DGS representation, making it suitable for SLAM applications [S1][S2, unofficial].
*   **Memory Complexity:** Explicit 3D Gaussian representation requires memory proportional to the number of Gaussians.
*   **Strengths:** Better reconstruction quality compared to NeRF-based methods due to explicit geometry and flexible reprojection [S1][S2, unofficial]. Real-time rendering capability [S1][S2, unofficial].
*   **Limitations:** Specific quantitative performance metrics are not provided in the sources.
*   **When to use:** When high-quality reconstruction and real-time performance are required for robotics tasks like relocalization, navigation, and manipulation [S1][S2, unofficial].

## Background
CG-SLAM was introduced in reference [189] (cited as [S1][S2, unofficial] in the provided text). It is part of the broader trend of using 3D Gaussian Splatting for SLAM, leveraging its explicit geometry for better reconstruction than implicit NeRF representations [S1][S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
