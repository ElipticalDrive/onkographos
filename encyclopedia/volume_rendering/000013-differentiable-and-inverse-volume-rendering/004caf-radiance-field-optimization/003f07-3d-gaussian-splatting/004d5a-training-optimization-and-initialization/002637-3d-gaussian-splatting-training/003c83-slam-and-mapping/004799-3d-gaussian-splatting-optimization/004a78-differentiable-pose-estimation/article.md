# Differentiable Pose Estimation

Also called: Pose Estimation via 3DGS Gradients

## Overview
Differentiable pose estimation optimizes camera poses by back-propagating photometric loss gradients through a 3D Gaussian Splatting (3DGS) renderer [S1]. It solves the problem of recovering accurate camera motion, particularly in scenarios involving motion blur, by treating pose as a differentiable variable within the rendering pipeline [S1]. It sets itself apart from sibling methods like SplaTAM and Gaussian-SLAM by explicitly decoupling the pose optimization from the scene geometry update in specific implementations (e.g., Deblur-GS), utilizing a simplified gradient derivation that ignores covariance and color terms to ensure stability during short optimization steps [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians $\mathcal{G}$ (initialized via COLMAP), a sequence of blurry input images $I_{captured}$, and an initial estimate of camera poses $\mathbf{T}$ [S1].
*   **Outputs:** Optimized camera poses $\mathbf{T}$ and potentially refined Gaussian parameters [S1].

**Data Structures:**
*   **Gaussians:** Each Gaussian $i$ is defined by mean position $\mu_i \in \mathbb{R}^3$, covariance $\Sigma_i \in \mathbb{R}^{3 \times 3}$, opacity $\alpha_i$, and color $c_i$ [S1].
*   **Camera Pose:** Represented as a transformation matrix $\mathbf{T} \in SE(3)$ [S1].
*   **Trajectory:** In Deblur-GS, camera motion is modeled using 7th-order Bézier interpolation with control points optimized during training [S1].

**Loss Function:**
The optimization minimizes a combined photometric loss [S1]:
$$ L = \lambda L_1 + (1 - \lambda) L_{D-SSIM} $$
where $\lambda = 0.2$ [S1]. $L_1$ is the L1 color difference, and $L_{D-SSIM}$ is the differentiable Structural Similarity index [S1].

**Gradient Derivation:**
The gradient of the loss $L$ with respect to the camera pose $\mathbf{T}$ is derived by expanding the chain rule through the rendered color $\hat{C}$ [S1]:
$$ \frac{\partial L}{\partial \mathbf{T}} = \sum_{i \in N} \frac{\partial L}{\partial \hat{C}} \left( \frac{\partial \hat{C}}{\partial c_i} \frac{\partial c_i}{\partial \mathbf{T}} + \frac{\partial \hat{C}}{\partial \alpha_i} \frac{\partial \alpha_i}{\partial \mathbf{T}} \right) $$
Expanding $\frac{\partial \alpha_i}{\partial \mathbf{T}}$ further yields three components [S1]:
1.  **Color Term:** $\frac{\partial \hat{C}}{\partial c_i} \frac{\partial c_i}{\partial \mathbf{T}}$. This involves spherical harmonics shading.
2.  **Covariance Term:** $\frac{\partial \hat{C}}{\partial \alpha_i} \frac{\partial \alpha_i}{\partial \Sigma'_i} \frac{\partial \Sigma'_i}{\partial \mathbf{T}}$. This involves the projection of the 3D covariance.
3.  **Position Term:** $\frac{\partial \hat{C}}{\partial \alpha_i} \frac{\partial \alpha_i}{\partial m'_i} \frac{\partial m'_i}{\partial \mathbf{T}}$. This involves the projection of the Gaussian mean.

**Simplifications and Assumptions:**
1.  **Eliminate Color Term:** Assuming short optimization steps, the color $c_i$ of Gaussians remains effectively unchanged. Thus, $\frac{\partial c_i}{\partial \mathbf{T}} \approx 0$ [S1].
2.  **Ignore Covariance Term:** If Gaussians form elongated ellipsoids, the covariance term becomes large and causes instability. The method ignores this term to enhance robustness [S1].
3.  **Position-Pose Equivalence:** Transforming the camera pose $\mathbf{T}$ by an external transformation $\hat{\mathbf{T}}$ is equivalent to transforming the Gaussian positions $\mu_i$ by $\hat{\mathbf{T}}$ [S1]:
    $$ \mathbf{K}(\mathbf{T}\hat{\mathbf{T}})\mu_i = \mathbf{K}\mathbf{T}(\hat{\mathbf{T}}\mu_i) $$
    where $\mathbf{K}$ is the intrinsic matrix [S1].

**Optimization Steps:**
1.  Initialize Gaussians and camera control points using COLMAP [S1].
2.  Define the camera trajectory using Bézier interpolation (e.g., 7th order, 31 sampled points) [S1].
3.  Render the image $I_{rendered}$ using the differentiable Gaussian rasterizer, which must support depth, pose, and cumulative opacity gradients [S1].
4.  Compute loss $L$ between $I_{rendered}$ and $I_{captured}$ [S1].
5.  Compute gradients $\frac{\partial L}{\partial \hat{\mathbf{T}}}$ using the simplified position term only [S1]:
    $$ \frac{\partial L}{\partial \hat{\mathbf{T}}} = \frac{\partial L}{\partial \hat{m}} \frac{\partial \hat{m}}{\partial \hat{\mathbf{T}}} $$
    where $\hat{m}_i = \hat{\mathbf{T}}\mu_i$ [S1].
6.  Update camera pose parameters (or trajectory control points) using Adam optimizer [S1].
7.  (Optional in Deblur-GS) Jointly optimize Gaussian scene parameters using a separate Adam optimizer [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_pose_gradient(gaussians, camera_pose, images, intrinsics):
    """
    Computes the gradient of the photometric loss with respect to the camera pose.
    Assumes simplified gradient derivation (Position term only).
    """
    # 1. Render image and get gradients w.r.t Gaussian properties
    # The rasterizer must return dL/d_alpha and dL/d_color
    rendered_image, dL_d_alpha, dL_d_color = differentiable_rasterize(
        gaussians, camera_pose, intrinsics
    )
    
    # 2. Calculate Loss
    loss = lambda_l1 * L1(rendered_image, images) + \
           (1 - lambda_l1) * D_SSIM(rendered_image, images)
           
    # 3. Simplify Gradient: Ignore Color and Covariance terms
    # We only need the gradient w.r.t Gaussian positions (m_i)
    # dL/d_m_i = dL/d_alpha * d_alpha/d_m_i
    
    # Note: d_alpha/d_m_i is provided by the rasterizer's backward pass
    # for the position component.
    dL_d_m = dL_d_alpha * rasterizer_gradient_alpha_wrt_position(gaussians, camera_pose)
    
    # 4. Map position gradient to Pose gradient
    # Using the equivalence: dL/d_T = dL/d_m * d_m/d_T
    # For a global transformation T_hat applied to positions m_hat = T_hat * m:
    # d(m_hat)/d(T_hat) is the Jacobian of the transformation.
    
    pose_gradient = compute_jacobian_gradient(dL_d_m, camera_pose)
    
    return pose_gradient, loss

def optimize_pose(gaussians, images, intrinsics, num_iterations=90000):
    """
    Optimizes camera poses using Deblur-GS strategy.
    """
    # Initialize optimizers
    pose_optimizer = Adam(params=gaussians.camera_params, lr=1e-3)
    # In Deblur-GS, scene params may also be optimized with a separate optimizer
    scene_optimizer = Adam(params=gaussians.scene_params, lr=1e-3) 
    
    for i in range(num_iterations):
        # Decay pose learning rate
        current_lr = 1e-3 * (1e-5 / 1e-3) ** (i / num_iterations)
        pose_optimizer.param_groups[0]['lr'] = current_lr
        
        # Forward pass
        pose_optimizer.zero_grad()
        scene_optimizer.zero_grad()
        
        rendered, loss = render_with_gaussians(gaussians, intrinsics)
        
        # Backward pass
        loss.backward()
        
        # Update
        pose_optimizer.step()
        scene_optimizer.step() # If joint optimization is enabled
        
    return gaussians.camera_params
```

**Parameters:**
*   $\lambda = 0.2$ [S1].
*   Pose Learning Rate: Exponentially decays from $1 \times 10^{-3}$ to $1 \times 10^{-5}$ [S1].
*   Trajectory Sampling: 31 points on the camera motion trajectory [S1].
*   Interpolation: 7th-order Bézier [S1].
*   Training Iterations: 90K [S1].

## Subtopics

### Deblur-GS Implementation
Deblur-GS is a specific implementation of differentiable pose estimation designed for camera motion blurred images [S1]. It extends the differentiable Gaussian splatting rasterizer to support depth, pose, and cumulative opacity for both forward and backward propagation [S1]. It uses two separate Adam optimizers for Gaussian scene parameters and camera pose parameters [S1]. The camera trajectory is modeled using 7th-order Bézier interpolation with 31 sampled points, and control points are initialized via COLMAP and optimized during training [S1]. It was trained for 90K iterations on a single NVIDIA RTX 3090 GPU [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians $N$ and the number of pixels. The rasterization is $O(N)$ per pixel, and the gradient computation is linear in the number of visible Gaussians [S1].
*   **Memory Complexity:** Stores gradients for all Gaussians and camera parameters. The extension to support pose gradients in the rasterizer increases memory overhead slightly compared to standard 3DGS [S1].
*   **Performance:** Evaluated on synthetic (ExBluRF) and real datasets [S1]. It achieves competitive deblurring and novel view synthesis performance compared to Deblur-NeRF and BAD-NeRF [S1].
*   **Strengths:** Robustness to elongated Gaussians by ignoring the covariance term; stability through simplified gradient derivation [S1].
*   **Limitations:** Relies on the assumption that color remains unchanged during short pose steps; may not converge if initial poses are poor or if the motion blur model is inaccurate [S1].
*   **When to use:** When dealing with motion-blurred images and needing to recover camera poses jointly with the scene representation [S1].

## Background
The method originates from the paper "Deblur-GS: 3D Gaussian Splatting from Camera Motion Blurred Images" by Chen and Liu et al. [S1]. It builds upon the differentiable rendering capabilities of 3D Gaussian Splatting [S1]. The approach addresses the limitations of NeRF-based methods (like Deblur-NeRF) by leveraging the explicit representation of 3D Gaussians for more efficient and stable optimization [S1].

## Sources
[S1] Deblur-GS: 3D Gaussian Splatting from Camera Motion Blurred Images, https://doi.org/10.1145/3651301, fetched 2026-11-08
