# Differentiable Pose Estimation

Also called: Pose Estimation via 3DGS Gradients

## Overview
Differentiable Pose Estimation optimizes camera poses by back-propagating gradients from a photometric loss through a differentiable 3D Gaussian Splatting renderer. It solves the problem of estimating camera motion, particularly in motion-blurred scenarios, by treating pose estimation as a gradient descent problem on the Gaussian scene parameters. It sets itself apart from sibling methods like [SplaTAM](9193-splatam.md) and [Gaussian-SLAM](9195-gaussian-slam.md) by explicitly deriving and utilizing the analytical gradient of the loss with respect to the camera pose transformation, rather than relying solely on iterative closest point (ICP) or heuristic tracking loops.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians $\mathcal{G} = \{\mu_i, \Sigma_i, \alpha_i, c_i\}$, camera intrinsic matrix $K$, and a sequence of captured images $I_{captured}$.
*   **Outputs:** Optimized camera poses $\mathbf{T}$ (or global transformations $\hat{T}$).

**Data Structures:**
The scene is represented as a list of 3D Gaussians. Each Gaussian $i$ contains:
*   Mean position $\mu_i \in \mathbb{R}^3$.
*   Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$, derived from scale $\mathbf{s}_i$ and rotation $\mathbf{R}_i$.
*   Opacity $\alpha_i \in [0, 1]$.
*   Color $c_i$ (view-dependent via Spherical Harmonics or view-independent).

**Loss Function:**
The total loss $L$ combines $L_1$ color loss and Depth-SSIM loss [S1]:
$$ L = \lambda L_1 + (1 - \lambda) L_{D-SSIM} $$
Typical value: $\lambda = 0.2$ [S1].

**Gradient Derivation:**
The gradient of the loss with respect to the camera pose $\mathbf{T}$ is derived by expanding the chain rule through the rendered color $\hat{C}$ [S1]:
$$ \frac{\partial L}{\partial \mathbf{T}} = \sum_{i \in N} \frac{\partial L}{\partial \hat{C}} \left( \frac{\partial \hat{C}}{\partial c_i} \frac{\partial c_i}{\partial \mathbf{T}} + \frac{\partial \hat{C}}{\partial \alpha_i} \frac{\partial \alpha_i}{\partial \mathbf{T}} \right) $$
Expanding $\frac{\partial \alpha_i}{\partial \mathbf{T}}$ further yields three components [S1]:
1.  **Color Term:** $\frac{\partial \hat{C}}{\partial c_i} \frac{\partial c_i}{\partial \mathbf{T}}$. This involves spherical harmonics shading.
2.  **Covariance Term:** $\frac{\partial \hat{C}}{\partial \alpha_i} \frac{\partial \alpha_i}{\partial \Sigma'_i} \frac{\partial \Sigma'_i}{\partial \mathbf{T}}$. This involves the projection of the 3D covariance.
3.  **Position Term:** $\frac{\partial \hat{C}}{\partial \alpha_i} \frac{\partial \alpha_i}{\partial m'_i} \frac{\partial m'_i}{\partial \mathbf{T}}$. This involves the projection of the Gaussian mean.

**Simplifications and Pitfalls:**
1.  **Eliminate Color Term:** Assume color $c_i$ remains unchanged during short pose optimization steps. Thus, $\frac{\partial c_i}{\partial \mathbf{T}} \approx 0$ [S1].
2.  **Ignore Covariance Term:** If Gaussians form elongated ellipsoids, the covariance term becomes large and causes instability. In practice, ignore this term to enhance robustness [S1].
3.  **Position Term Equivalence:** Transforming the camera pose $\mathbf{T}$ by an external transformation $\hat{T}$ is equivalent to transforming the Gaussian positions $m_i$ by $\hat{T}$ [S1]:
    $$ K(\mathbf{T}\hat{T})m_i = K\mathbf{T}(\hat{T}m_i) $$
    Therefore, instead of optimizing $\mathbf{T}$ directly, optimize a global transformation $\hat{T}$ applied to all Gaussian positions $m_i$. Let $\hat{m}_i = \hat{T}m_i$. The gradient becomes [S1]:
    $$ \frac{\partial L}{\partial \hat{T}} = \frac{\partial L}{\partial \hat{m}} \frac{\partial \hat{m}}{\partial \hat{T}} $$
    The renderer computes $\frac{\partial L}{\partial \hat{m}}$, and automatic differentiation computes $\frac{\partial \hat{m}}{\partial \hat{T}}$ [S1].

**Algorithm Steps:**
1.  Initialize Gaussians $\mathcal{G}$ and camera poses $\mathbf{T}$ (e.g., using COLMAP) [S1].
2.  Render the current view using the differentiable Gaussian splatting rasterizer to obtain $\hat{C}$ and depth $D$ [S1].
3.  Compute loss $L$ against $I_{captured}$ and $D_{captured}$ [S1].
4.  Back-propagate through the renderer to get $\frac{\partial L}{\partial \hat{m}}$ [S1].
5.  Compute $\frac{\partial \hat{m}}{\partial \hat{T}}$ using automatic differentiation [S1].
6.  Update $\hat{T}$ (and thus $\mathbf{T}$) using an optimizer (e.g., Adam) [S1].
7.  Repeat until convergence.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def differentiable_pose_estimation(Gaussians, CameraPoses, Intrinsics, TargetImages, TargetDepths, Lambda=0.2):
    """
    Optimizes camera poses by back-propagating gradients through 3DGS.
    Gaussians: List of Gaussian objects with attributes .mu, .Sigma, .alpha, .c
    CameraPoses: List of 4x4 transformation matrices T_t
    Intrinsics: 3x3 camera matrix K
    TargetImages: List of captured images
    TargetDepths: List of captured depth maps (optional)
    """
    # Initialize optimizers
    # Note: In Deblur-GS, separate Adam optimizers are used for scene params and pose params [S1]
    pose_optimizer = Adam(params=CameraPoses, lr=1e-3)
    
    # Exponential decay for pose learning rate [S1]
    def get_pose_lr(step):
        return 1e-3 * (1e-5 / 1e-3) ** (step / 90000)

    for step in range(90000): # 90K iterations [S1]
        pose_optimizer.zero_grad()
        
        # 1. Apply transformation to Gaussian positions
        # Equivalent to optimizing global transformation T_hat [S1]
        # m_hat_i = T_hat * m_i
        # In implementation, this is often done by modifying the Gaussian positions 
        # or the view matrix before rasterization.
        transformed_gaussians = apply_pose_transformation(Gaussians, CameraPoses)
        
        # 2. Render
        # Extend rasterizer to support depth and cumulative opacity [S1]
        RenderedColor, RenderedDepth = differentiable_gaussian_rasterize(
            transformed_gaussians, Intrinsics, CameraPoses
        )
        
        # 3. Compute Loss
        # L = lambda * L1 + (1 - lambda) * D-SSIM [S1]
        L1_Loss = compute_l1_loss(RenderedColor, TargetImages)
        DSSIM_Loss = compute_dssim_loss(RenderedColor, TargetImages)
        Loss = Lambda * L1_Loss + (1 - Lambda) * DSSIM_Loss
        
        # 4. Back-propagation
        # The renderer computes dL/dm_hat automatically [S1]
        # We need dL/dT_hat. Since m_hat = T_hat * m, d(m_hat)/d(T_hat) is linear [S1]
        Loss.backward()
        
        # 5. Update
        # Gradient clipping or other standard optimizer steps [G]
        pose_optimizer.step()
        
        # Update learning rate [S1]
        for param_group in pose_optimizer.param_groups:
            param_group['lr'] = get_pose_lr(step)
            
    return CameraPoses
```

## Subtopics

### Deblur-GS Implementation
A specific implementation of Differentiable Pose Estimation designed for motion-blurred images [S1]. It extends the differentiable Gaussian splatting rasterizer to support depth, pose, and cumulative opacity for both forward and backward propagation [S1]. It uses two separate Adam optimizers for Gaussian scene parameters and camera pose parameters [S1]. The pose learning rate exponentially decays from $1 \times 10^{-3}$ to $1 \times 10^{-5}$ [S1]. It samples 31 points on the camera motion trajectory (using 7th order Bézier interpolation) and trains for 90K iterations on a single NVIDIA RTX 3090 GPU [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians $N$ and the rasterization cost. Each iteration requires a forward render and a backward pass. 90K iterations are typical for Deblur-GS [S1].
*   **Memory Complexity:** Stores gradients for all Gaussian parameters and pose parameters. The rasterizer must support depth and cumulative opacity back-propagation [S1].
*   **Performance:** Evaluated on synthetic (ExBluRF) and real datasets [S1]. Requires a single NVIDIA RTX 3090 GPU for 90K iterations [S1].
*   **Strengths:** Robust to elongated Gaussian distributions by ignoring the covariance term [S1]. Handles motion blur by optimizing pose trajectories [S1].
*   **Limitations:** Relies on the assumption that color remains unchanged during short pose steps [S1]. Instability can occur if the covariance term is not ignored for elongated ellipsoids [S1].
*   **When to use:** When camera poses are unknown or inaccurate, or when dealing with motion-blurred images where traditional SfM fails.
*   **When not to use:** When high-frequency color changes occur rapidly during pose optimization, violating the static color assumption [S1].

## Background
Originated from the work of Chen and Liu et al. in "Deblur-GS: 3D Gaussian Splatting from Camera Motion Blurred Images" [S1]. The method leverages the explicit representation of 3D Gaussians to derive analytical gradients for pose estimation, building on the differentiable rasterization capabilities of 3D Gaussian Splatting. It is used in scenarios involving camera motion blur, where traditional NeRF-based methods like Deblur-NeRF [Ma et al. 2022] or BAD-NeRF [Wang et al. 2023b] are compared [S1].

## Sources
[S1] Deblur-GS: 3D Gaussian Splatting from Camera Motion Blurred Images, https://doi.org/10.1145/3651301, fetched 2026-11-08
