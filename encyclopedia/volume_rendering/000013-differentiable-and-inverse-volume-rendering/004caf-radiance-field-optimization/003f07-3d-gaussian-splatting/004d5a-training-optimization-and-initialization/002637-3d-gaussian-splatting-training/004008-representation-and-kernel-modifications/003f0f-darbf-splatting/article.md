# DARBF-Splatting

Also called: DARB-Splatting, Decaying Anisotropic Radial Basis Functions Splatting

## Overview
DARBF-Splatting is a splatting-based 3D reconstruction method that replaces the standard Gaussian kernels of 3D Gaussian Splatting (3DGS) with Decaying Anisotropic Radial Basis Functions (DARBFs) [S1]. It solves the limitation of existing splatting methods being restricted to the exponential family by allowing non-exponential, non-negative functions of the Mahalanobis distance as reconstruction kernels [S1]. Unlike standard 3DGS, which relies on the closed-form integration shortcut of Gaussians, DARBF-Splatting introduces a correction factor to approximate this efficiency for other kernels, enabling faster training convergence and lower memory usage while maintaining comparable visual quality [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Sparse 3D point cloud from Structure-from-Motion (SfM) (e.g., COLMAP), camera poses, and input images [S1].
*   **Outputs:** Optimized 3D primitives (DARBFs) and a differentiable rasterization pipeline producing novel view images [S1].

**Data Structures:**
The primitive representation retains the standard 3DGS attribute set but modifies the kernel function [S1]:
*   **Mean:** $\boldsymbol{\mu} \in \mathbb{R}^3$ (position) [S1].
*   **Covariance:** $\boldsymbol{\Sigma} \in \mathbb{R}^{3 \times 3}$ (shape and orientation) [S1]. The matrix is symmetric positive definite.
*   **Opacity:** $\alpha \in [0, 1]$ [S1].
*   **Color:** $\mathbf{c}$ (represented by spherical harmonics coefficients) [S1].

**The Mahalanobis Distance:**
The core of the DARBF kernel is the Mahalanobis distance $d_M$ from a point $\mathbf{x}$ to the center $\boldsymbol{\mu}$ with covariance $\boldsymbol{\Sigma}$ [S1]:
$$
d_M(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma}) = [(\mathbf{x} - \boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu})]^{\frac{1}{2}}
$$
where $\boldsymbol{\Sigma}$ defines the anisotropic shape [S1].

**The Correction Factor ($\psi$):**
Standard Gaussians benefit from a closed-form integration shortcut when projecting from 3D to 2D, where the 2D covariance is obtained by skipping the third row and column of the 3D covariance matrix [S1]. Non-Gaussian DARBFs do not possess this property, requiring costly integration [S1]. DARBF-Splatting introduces a scalar correction factor $\psi$ to approximate this closed-form integration shortcut, preserving computational efficiency [S1]. The scalar $\psi$ is precomputed via closed-form regression experiments [S1].

**Kernel Functions:**
DARBFs are non-negative, decaying functions of $d_M$ [S1]. Examples include:
*   **Half-Cosine:** Defined as $\cos \left (\frac {d\_M^2}{\xi }\right )$ for $d\_M^2 < \frac {\xi \pi }{2}$, where $\xi > 0$ [S1]. It has finite support and a blunt peak [S1].
*   **Inverse Multiquadric (MQ):** Spans a larger region with higher opacity for the same variance [S1].
*   **Modified Raised Cosine / Modular Sinc:** Other variants mentioned as part of the DARBF spectrum [S1].

**Rendering Pipeline Steps:**
1.  **Initialization:** Place DARBF primitives at 3D points from the SfM pipeline [S1].
2.  **Transformation:** Transform 3D DARBF primitives (ellipsoids) into the camera frame [S1].
3.  **Projection:** Project the 3D DARBF primitive onto the 2D image plane. Unlike Gaussians, this does not yield a simple 2D DARBF via covariance truncation. Instead, the method uses the correction factor $\psi$ to approximate the footprint function (splat function) [S1].
4.  **Rasterization:** Render the splats using differentiable rasterization, accumulating opacity and color [S1].
5.  **Optimization:** Optimize primitive attributes ($\boldsymbol{\mu}, \boldsymbol{\Sigma}, \alpha, \mathbf{c}$) via gradient descent to minimize rendering loss against ground truth images [S1].

**Pseudocode:**
The following pseudocode illustrates the rendering of a single DARBF primitive, incorporating the correction factor $\psi$. Note that standard 3DGS projection logic is modified to include $\psi$.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def mahalanobis_distance(x, mu, sigma_inv):
    """
    Calculate the Mahalanobis distance squared.
    x: (3,) point in 3D space
    mu: (3,) mean of the primitive
    sigma_inv: (3,3) inverse covariance matrix
    """
    diff = x - mu
    return np.dot(diff, np.dot(sigma_inv, diff))

def half_cosine_kernel(d_m_sq, xi):
    """
    Half-cosine DARBF kernel.
    d_m_sq: Mahalanobis distance squared
    xi: width parameter
    """
    # Kernel is defined for d_m_sq < xi * pi / 2
    # Outside this range, value is 0 (finite support)
    if d_m_sq >= xi * np.pi / 2:
        return 0.0
    return np.cos(d_m_sq / xi)

def inverse_mq_kernel(d_m_sq, beta):
    """
    Inverse Multiquadric DARBF kernel.
    d_m_sq: Mahalanobis distance squared
    beta: shape parameter
    """
    return 1.0 / np.sqrt(1.0 + beta * d_m_sq)

def project_covariance_3d_to_2d(sigma_3d, view_matrix):
    """
    Approximate 2D covariance for DARBF using the Gaussian shortcut
    modified by the correction factor psi.
    sigma_3d: (3,3) 3D covariance matrix
    view_matrix: (3,4) view transformation matrix (rotation + translation)
    psi: scalar correction factor precomputed for the specific kernel
    """
    # Standard 3DGS shortcut: J is the Jacobian of the projection
    # For simplicity, assume we have the 2x2 lower block of the transformed covariance
    # In 3DGS, this is just skipping the 3rd row/col of the transformed covariance.
    # Here, we apply the correction factor to the resulting 2D covariance or opacity.
    # The paper states psi is a scalar precomputed from regression.
    # Implementation detail: The exact application of psi to the 2D covariance
    # is not fully detailed in the text, but it approximates the integration.
    # We assume psi scales the effective variance or opacity contribution.
    
    # Extract rotation part of view_matrix (3x3)
    R = view_matrix[:3, :3]
    
    # Transform 3D covariance to camera space
    sigma_cam = R @ sigma_3d @ R.T
    
    # Standard Gaussian shortcut: take top-left 2x2
    sigma_2d_base = sigma_cam[:2, :2]
    
    # Apply correction factor psi. 
    # The paper implies psi helps approximate the closed-form integration.
    # A common interpretation for such factors in splatting is scaling the 
    # resulting 2D covariance or the opacity. 
    # Given the text "approximating the Gaussian's closed-form integration shortcut",
    # and psi being a scalar, it likely scales the 2D covariance determinant 
    # or the resulting opacity weight. 
    # For the purpose of this implementation guide, we scale the 2D covariance 
    # to match the effective spread of the DARBF.
    psi = 1.0 # Placeholder for the precomputed scalar psi for the chosen kernel
    
    sigma_2d = sigma_2d_base * psi
    
    return sigma_2d

def render_darbf_splat(x_screen, y_screen, mu_2d, sigma_2d, alpha, color, psi):
    """
    Render a single DARBF splat at a pixel.
    x_screen, y_screen: pixel coordinates
    mu_2d: (2,) 2D mean
    sigma_2d: (2,2) 2D covariance (already corrected)
    alpha: opacity
    color: color vector
    psi: correction factor (used in projection, passed here for completeness if needed in kernel eval)
    """
    # Calculate distance in 2D screen space
    diff = np.array([x_screen - mu_2d[0], y_screen - mu_2d[1]])
    sigma_2d_inv = np.linalg.inv(sigma_2d)
    d_m_sq_2d = np.dot(diff, np.dot(sigma_2d_inv, diff))
    
    # Evaluate 2D kernel function (e.g., Half-Cosine)
    # Note: The kernel is defined in 3D Mahalanobis space, but projected to 2D.
    # The correction factor psi approximates the integration loss.
    # We use the 2D Mahalanobis distance for the kernel evaluation.
    # For Half-Cosine:
    xi = 1.0 # Example parameter, typically related to variance
    kernel_val = half_cosine_kernel(d_m_sq_2d, xi)
    
    # Apply opacity
    contribution = alpha * kernel_val
    
    return contribution, color

def optimize_darbf_primitives(points_3d, images, camera_poses, kernel_type='half_cosine'):
    """
    Main optimization loop.
    """
    # Initialize primitives
    primitives = []
    for p in points_3d:
        primitives.append({
            'mu': p,
            'sigma': np.eye(3), # Identity or SfM-derived
            'alpha': 0.5,
            'color': np.array([0.5, 0.5, 0.5]),
            'kernel_type': kernel_type
        })
    
    # Precompute psi for the chosen kernel type
    # This is done via closed-form regression experiments as per [S1]
    psi = precompute_psi(kernel_type) 
    
    for epoch in range(num_epochs):
        for image, camera_pose in zip(images, camera_poses):
            loss = 0
            for prim in primitives:
                # Project 3D primitive to 2D
                sigma_2d = project_covariance_3d_to_2d(prim['sigma'], camera_pose)
                mu_2d = project_point_to_screen(prim['mu'], camera_pose)
                
                # Render splat to image plane (simplified)
                # In reality, this is done via rasterization over all pixels
                # For each pixel covered by the splat:
                for x, y in get_splat_pixels(mu_2d, sigma_2d):
                    contrib, color = render_darbf_splat(x, y, mu_2d, sigma_2d, prim['alpha'], prim['color'], psi)
                    # Accumulate into image buffer
                    # Calculate loss against ground truth
                    loss += calculate_loss(image[x, y], color * contrib)
            
            # Backpropagate and update primitives
            update_primitives(primitives, loss)
```

**Edge Cases and Pitfalls:**
*   **Unbounded Functions:** Functions that grow unbounded cause instability and are omitted in favor of decaying functions (DARBFs) [S1].
*   **Integration Cost:** Using functions outside the Gaussian family typically incurs costly integration processes, making them computationally inefficient if the correction factor $\psi$ is not used [S1].
*   **Parameter Selection:** The width parameter $\xi$ for half-cosine and $\beta$ for inverse MQ must be chosen appropriately to match the variance of the Gaussian counterparts for fair comparison [S1].

## Operating characteristics
*   **Training Speed:** Half-cosine kernels achieve 34% faster training convergence compared to standard Gaussian kernels [S1].
*   **Memory Usage:** Half-cosine kernels reduce memory usage by 15% compared to standard Gaussian kernels [S1].
*   **Visual Quality:** Achieves comparable PSNR, SSIM, and LPIPS results to 3DGS [S1].
*   **Kernel Spread:** Half-cosines and Inverse MQ functions span a larger region with higher opacity for the same variance, reducing the need for multiple splats to achieve the same accumulated value [S1]. They have a steeper rolloff than Gaussians [S1].
*   **Hardware:** Modified CUDA codes are available for each DARBF, implying similar hardware requirements to 3DGS (GPU-accelerated rasterization) [S1].

## Background
*   **Origin:** DARBF-Splatting was introduced by Hashiru Pramuditha, Vinasirajan Viruthshaan, Vishagar Arunan, Saeedha Nazar, Sameera Ramasinghe, Simon Lucey, and Ranga Rodrigo from the University of Moratuwa and University of Adelaide [S1].
*   **Publication:** The method was published in the IEEE Winter Conference on Applications of Computer Vision (WACV) in 2026 [S1].
*   **Context:** It addresses the limitation of 3D Gaussian Splatting (3DGS) [11] being restricted to the exponential family of functions, leaving generalized reconstruction kernels underexplored due to integration difficulties [S1].

## Sources
[S1] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
