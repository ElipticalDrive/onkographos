# BAD-Gaussians

Also called: Bundle Adjusted Deblur Gaussian Splatting

## Overview
BAD-Gaussians recovers a sharp 3D scene representation by jointly learning camera motion trajectories and 3D Gaussian parameters from a sequence of motion-blurred images, inaccurate poses, and sparse point clouds. It solves the problem of reconstructing high-quality geometry and appearance from blurry inputs where standard 3D Gaussian Splatting fails due to unmodeled motion blur. It sets itself apart from sibling methods like [DeSplat](12925-desplat.md) and [360 Gaussian Splatting](12929-360-gaussian-splatting.md) by explicitly modeling the physical motion blur formation process via virtual camera poses and optimizing the camera trajectory (start and end poses) alongside the Gaussians, rather than assuming static sharp inputs or using different blur kernels.

## How it works
**Inputs:** A sequence of $N$ motion-blurred images $\{B_k\}_{k=0}^{N-1}$, their corresponding inaccurate initial poses and sparse point clouds estimated from COLMAP [S1].
**Outputs:** A set of 3D Gaussians (mean $\mu$, covariance $\Sigma$, opacity $o$, color $c$) and optimized camera start ($T_{start}$) and end ($T_{end}$) poses for each blurred image.

**Data Structures:**
*   **3D Gaussian:** Parameterized by $\mu \in \mathbb{R}^3$, $\Sigma \in \mathbb{R}^{3 \times 3}$ (or its Cholesky decomposition/rotation/scale representation as in 3D-GS), $o \in \mathbb{R}$, and $c \in \mathbb{R}^3$ [S1].
*   **Camera Pose:** $T_{start}$ and $T_{end}$ are parameterized by their corresponding Lie algebras of SE(3), represented by a 6D vector each [S1].
*   **Virtual Poses:** $n$ discrete virtual camera poses interpolated between $T_{start}$ and $T_{end}$ to approximate the exposure time [S1].

**Parameters:**
*   $n$: Number of discrete samples (virtual camera poses) used to approximate the motion blur integral. Typical value: 10 [S1].
*   Learning rate for Gaussians: Identical to original 3D-GS [S1].
*   Learning rate for camera poses ($T_{start}, T_{end}$): Exponentially decreased from $1 \times 10^{-3}$ to $1 \times 10^{-5}$ [S1].
*   Optimizer: Adam [S1].

**Method Steps:**

1.  **Initialization:** Initialize 3D Gaussians and camera poses using estimations obtained from COLMAP [S1].
2.  **Virtual Pose Generation:** For each blurred image $k$, define the start pose $T_{start}^{(k)}$ and end pose $T_{end}^{(k)}$. Generate $n$ virtual poses $T_i^{(k)}$ for $i=0 \dots n-1$ by interpolating between $T_{start}^{(k)}$ and $T_{end}^{(k)}$. The interpolation method can be linear or cubic B-spline [S1].
3.  **Virtual Image Synthesis:** For each virtual pose $T_i^{(k)}$, render a sharp virtual image $C_i^{(k)}(u)$ using standard 3D Gaussian Splatting rasterization. The pixel value at location $u$ is computed by sorting Gaussians front-to-back and compositing their contributions. The opacity contribution of a Gaussian $i$ is given by:
    $$ \alpha_i = o_i \cdot \exp(-\sigma_i) $$
    where $o_i$ is the learned opacity and $\sigma_i = \frac{1}{2} \Delta_i^\top \Sigma'^{-1} \Delta_i$, with $\Delta_i$ being the offset between the pixel center and the 2D Gaussian center projected onto the image plane [S1].
4.  **Blur Model Approximation:** Compute the synthesized blurred image $B_k(u)$ by averaging the virtual sharp images:
    $$ B_k(u) \approx \frac{1}{n} \sum_{i=0}^{n-1} C_i^{(k)}(u) \quad (7) $$
    [S1].
5.  **Loss Computation:** Calculate the photometric loss (e.g., L1 or SSIM) between the synthesized blurred image $B_k(u)$ and the input blurred image $B_k^{input}(u)$ [S1].
6.  **Gradient Backpropagation:** Compute gradients with respect to Gaussian parameters $\theta$ and camera poses $T$.
    *   Gradient w.r.t. Gaussian parameters $\theta$:
        $$ \frac{\partial L}{\partial \theta} = \sum_{k=0}^{K-1} \left( \frac{\partial L}{\partial B_k} \right) \cdot \frac{1}{n} \sum_{i=0}^{n-1} \left( \frac{\partial B_i}{\partial C_i} \right) \cdot \left( \frac{\partial C_i}{\partial \theta} \right) \quad (10) $$
        where $\frac{\partial B_i}{\partial C_i} = \frac{1}{n}$ [S1].
    *   Gradient w.r.t. Camera Poses $T$:
        $$ \frac{\partial L}{\partial T} = \sum_{k=0}^{K-1} \left( \frac{\partial L}{\partial B_k} \right) \cdot \frac{1}{n} \sum_{i=0}^{n-1} \left( \frac{\partial B_k}{\partial C_i} \right) \cdot \left( \frac{\partial C_i}{\partial \theta} \right) \cdot \left( \frac{\partial \theta}{\partial T} \right) \quad (11) $$
        Note: The source text contains a typo in the second term of the sum for Eq. 11, listing $\frac{\partial C_i}{\partial \theta}$ instead of $\frac{\partial C_i}{\partial T}$, but the formula provided is used as written [S1]. The term $\frac{\partial \theta}{\partial T}$ represents the Jacobian of the Gaussian parameters with respect to the camera pose [S1].
7.  **Optimization:** Update $\theta$ and $T$ using the Adam optimizer [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def render_blurred_image(gaussians, T_start, T_end, n_samples, image_width, image_height):
    """
    Renders a motion-blurred image by averaging n_samples virtual sharp images.
    
    Args:
        gaussians: List of 3D Gaussian parameters (mu, sigma, opacity, color).
        T_start: 6D vector representing start pose (Lie algebra of SE(3)).
        T_end: 6D vector representing end pose (Lie algebra of SE(3)).
        n_samples: Number of virtual poses to interpolate.
        image_width, image_height: Dimensions of the output image.
        
    Returns:
        blurred_image: Tensor of shape (H, W, 3).
    """
    # Interpolate virtual poses
    # Linear interpolation as a baseline; cubic B-spline can be used for complex trajectories
    virtual_poses = []
    for i in range(n_samples):
        t = i / (n_samples - 1)
        # Interpolate Lie algebra vectors
        T_virtual = (1 - t) * T_start + t * T_end
        virtual_poses.append(T_virtual)
        
    virtual_images = []
    for T_v in virtual_poses:
        # Render sharp image using standard 3D-GS rasterization
        # This function encapsulates the projection, sorting, and compositing of Gaussians
        sharp_image = render_sharp_3dgs(gaussians, T_v, image_width, image_height)
        virtual_images.append(sharp_image)
        
    # Average virtual images to approximate blur
    blurred_image = torch.stack(virtual_images, dim=0).mean(dim=0)
    return blurred_image

def train_bad_gaussians(blurred_images, initial_gaussians, initial_T_start, initial_T_end, n_samples=10):
    """
    Training loop for BAD-Gaussians.
    """
    # Initialize parameters
    gaussians = initial_gaussians.clone().requires_grad_(True)
    T_start = initial_T_start.clone().requires_grad_(True)
    T_end = initial_T_end.clone().requires_grad_(True)
    
    optimizer = torch.optim.Adam([{'params': gaussians}, {'params': [T_start, T_end]}], lr=1e-3)
    
    # Learning rate schedule for poses
    pose_lr = 1e-3
    
    for step in range(num_training_steps):
        # Exponential decay for pose learning rate
        current_pose_lr = 1e-3 * (1e-5 / 1e-3) ** (step / num_training_steps)
        
        for param_group in optimizer.param_groups:
            if param_group['params'] == [T_start, T_end]:
                param_group['lr'] = current_pose_lr
            else:
                param_group['lr'] = 1e-3 # Original 3D-GS LR for Gaussians
                
        optimizer.zero_grad()
        
        total_loss = 0
        for blurred_img in blurred_images:
            # Render blurred image
            rendered_blur = render_blurred_image(gaussians, T_start, T_end, n_samples, 
                                                 blurred_img.shape[1], blurred_img.shape[2])
            
            # Compute photometric loss
            loss = F.l1_loss(rendered_blur, blurred_img)
            total_loss += loss
            
        total_loss.backward()
        optimizer.step()
        
    return gaussians, T_start, T_end
```

**Edge Cases and Pitfalls:**
*   **Performance Saturation:** Increasing $n$ beyond 10 yields diminishing returns. Performance reaches a saturation point, so $n=10$ is recommended for the trade-off between performance and efficiency [S1].
*   **Trajectory Representation:** Cubic interpolation (B-spline) improves performance in scenes with complex camera trajectories (e.g., MBA-VO and Deblur-NeRF-R) compared to linear interpolation [S1].
*   **Pose Accuracy Sensitivity:** The method is sensitive to the accuracy of initial poses. Methods like Deblur-NeRF and DP-NeRF show a significant performance gap when trained with COLMAP poses versus ground-truth poses, highlighting the importance of good initialization [S1].

## Operating characteristics
*   **Time Complexity:** Training time is approximately 30 minutes on an NVIDIA RTX 4090 GPU [S1]. This is significantly faster than competing methods like Deblur-NeRF, DP-NeRF, and BAD-NeRF, which take more than 10 hours [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Rendering Efficiency:** Achieves over 200 FPS rendering efficiency [S1]. In contrast, Deblur-NeRF, DP-NeRF, and BAD-NeRF fall below 1 FPS [S1].
*   **Strengths:** High rendering speed, fast training time, ability to handle motion blur and inaccurate poses jointly.
*   **Limitations:** Sensitive to initial pose accuracy. Performance gains from increasing $n$ saturate quickly.
*   **When to use:** When reconstructing scenes from motion-blurred images with inaccurate camera poses and sparse initialization.
*   **When not to use:** When high-precision camera poses are already available and images are sharp (standard 3D-GS is sufficient and simpler).

## Background
BAD-Gaussians was introduced in the paper "BAD-Gaussians: Bundle Adjusted Deblur Gaussian Splatting" [S1]. It builds upon the 3D Gaussian Splatting framework [15] and addresses the limitations of NeRF-based deblurring methods (Deblur-NeRF [26], BAD-NeRF [46]) which are computationally expensive and slow to render. The method leverages the geometry within multi-view blurry images to reconstruct a high-quality 3D scene by minimizing the photometric error between input blurred images and synthesized blurred images based on a physical motion blur image formation model [S1].

## Sources
[S1] BAD-Gaussians: Bundle Adjusted Deblur Gaussian Splatting, https://doi.org/10.1007/978-3-031-72698-9_14, fetched 2026-10-26
