# MIP-Splatting

Also called: Alias-free 3D Gaussian Splatting

## Overview
MIP-Splatting is an extension of 3D Gaussian Splatting (3DGS) that eliminates spatial aliasing and dilation/erosion artifacts caused by resolution changes, focal length variations, or camera distance changes [S1][S5, S6, unofficial]. It achieves this by introducing a 3D Gaussian smoothing filter and a 2D Mip filter, ensuring the representation's frequency remains below the Nyquist limit of the training data while approximating the physical box filter of a camera sensor during rendering [S2, S3][S5, unofficial]. Unlike standard 3DGS, which uses a fixed 2D dilation filter that fails under varying sampling rates, MIP-Splatting constrains the maximal frequency of the 3D representation based on the training views and replaces screen-space dilation with a 2D Mip filter [S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $K$ 3D Gaussian primitives $\{G_k\}_{k=1}^K$, each defined by opacity $\alpha_k \in [0, 1]$, center $p_k \in \mathbb{R}^3$, and covariance matrix $\Sigma_k \in \mathbb{R}^{3 \times 3}$ [S5, unofficial]. The training dataset consists of $N$ images with known camera poses (rotation $R$, translation $t$) and intrinsics (focal length $f$) [S5, unofficial].
*   **Output:** A rendered image at any target resolution or sampling rate, free of aliasing and dilation artifacts [S5, unofficial].

**Data Structures:**
The 3D Gaussian primitive $G_k$ is parameterized as:
$$ G_k(x) = e^{-\frac{1}{2} (x-p_k)^T \Sigma_k^{-1} (x-p_k)} $$
The covariance matrix $\Sigma_k$ is decomposed into a scaling vector $s \in \mathbb{R}^3$ and a rotation matrix $O \in \mathbb{R}^{3 \times 3}$ (parameterized by a quaternion) such that $\Sigma_k = O s s^T O^T$ [S5, unofficial].

**Algorithm Steps:**

1.  **Compute Maximal Sampling Frequency (Training Phase):**
    For each Gaussian primitive $k$, determine the maximal sampling frequency $\hat{\nu}_k$ induced by the training views. According to the Nyquist-Shannon Sampling Theorem, the world space sampling interval $\hat{T}$ at depth $d$ for focal length $f$ is $\hat{T} = \frac{d}{f}$, so the sampling frequency is $\hat{\nu} = \frac{1}{\hat{T}} = \frac{f}{d}$ [S5, unofficial]. The maximal sampling frequency $\hat{\nu}_k$ for a primitive is determined by the training view that provides the finest sampling (smallest $\hat{T}$) where the primitive is visible [S5, unofficial]. This is recomputed every $m$ iterations (typical $m=100$) as Gaussian centers remain relatively stable [S5, unofficial].

2.  **Apply 3D Smoothing Filter (Training Phase):**
    To constrain the maximal frequency of the 3D representation to meet the Nyquist limit, apply a 3D Gaussian low-pass filter $G_{low}$ to each primitive $G_k$ during optimization. The regularized 3D Gaussian $G_k(x)_{reg}$ is the convolution of $G_k$ and $G_{low}$, resulting in a Gaussian with covariance $\Sigma_k + \hat{\nu}_k^s \cdot I$:
    $$ G_k(x)_{reg} = \frac{\sqrt{|\Sigma_k|}}{\sqrt{|\Sigma_k + \hat{\nu}_k^s \cdot I|}} e^{-\frac{1}{2}(x-p_k)^T (\Sigma_k + \hat{\nu}_k^s \cdot I)^{-1} (x-p_k)} $$
    Here, $\hat{\nu}_k^s$ is a scalar hyperparameter controlling the filter size, typically set to $0.2$ (variance) [S5, unofficial]. This filter is applied *before* projection and becomes an intrinsic part of the scene representation, remaining constant regardless of viewpoint changes [S5, unofficial].

3.  **Project to Screen Space:**
    Transform the regularized 3D Gaussians into camera coordinates:
    $$ p'_k = R p_k + t $$
    $$ \Sigma'_k = R \Sigma_k R^T $$
    Project to ray space via a local affine transformation using the Jacobian $J_k$:
    $$ \Sigma''_k = J_k \Sigma'_k J_k^T $$
    [S5, unofficial].

4.  **Apply 2D Mip Filter (Rendering Phase):**
    Replace the standard 2D dilation operation with a 2D Mip filter that approximates the physical box filter of a single pixel. The 2D Mip-filtered Gaussian $G_{k2D}(x)_{mip}$ uses a covariance of $\Sigma_{k2D} + sI$:
    $$ G_{k2D}(x)_{mip} = \frac{\sqrt{|\Sigma_{k2D}|}}{\sqrt{|\Sigma_{k2D} + sI|}} e^{-\frac{1}{2}(x-p_k)^T (\Sigma_{k2D} + sI)^{-1} (x-p_k)} $$
    The variance $s$ is chosen to cover a single pixel in screen space, typically $0.1$ [S5, unofficial]. This mitigates aliasing when zooming out and dilation artifacts when zooming in, which occur in standard 3DGS because the fixed dilation amount does not scale with the projected Gaussian size [S5, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_sampling_frequency(depth, focal_length):
    """
    Computes the world-space sampling frequency.
    depth: float, distance from camera to point.
    focal_length: float, focal length in pixels.
    Returns: float, sampling frequency nu_hat.
    """
    # Sampling interval T_hat = d / f
    # Sampling frequency nu_hat = 1 / T_hat = f / d
    return focal_length / depth

def apply_3d_smoothing(cov_3d, nu_hat_s):
    """
    Applies the 3D smoothing filter by adding variance to the covariance matrix.
    cov_3d: numpy array (3, 3), original covariance matrix Sigma_k.
    nu_hat_s: float, scalar hyperparameter for filter size (e.g., 0.2).
    Returns: numpy array (3, 3), regularized covariance matrix.
    """
    # Convolution of two Gaussians results in sum of covariances.
    # Identity matrix I scaled by nu_hat_s.
    return cov_3d + nu_hat_s * np.eye(3)

def apply_2d_mip_filter(cov_2d, pixel_variance):
    """
    Applies the 2D Mip filter to approximate the pixel box filter.
    cov_2d: numpy array (2, 2), projected covariance matrix Sigma_k2D.
    pixel_variance: float, variance covering a single pixel (e.g., 0.1).
    Returns: numpy array (2, 2), filtered covariance matrix.
    """
    # Add identity matrix scaled by pixel variance.
    return cov_2d + pixel_variance * np.eye(2)

def render_mip_splatting(gaussians, camera_params, target_resolution):
    """
    Renders a scene using MIP-Splatting.
    gaussians: list of dicts, each containing 'alpha', 'center', 'cov_3d'.
    camera_params: dict with 'R', 't', 'focal_length'.
    target_resolution: tuple (width, height).
    Returns: numpy array (height, width, 3), rendered image.
    """
    R = camera_params['R']
    t = camera_params['t']
    f = camera_params['focal_length']
    
    # Pre-compute pixel size in world space for 2D filter
    # Assuming depth d is approximated or known per gaussian for pixel coverage
    # In practice, s is chosen to cover a single pixel in screen space.
    # For a point at depth d, 1 pixel corresponds to d/f in world units.
    # The variance s in screen space is typically fixed to 0.1 [S5, unofficial].
    pixel_variance = 0.1 
    
    # 1. Transform and Project
    projected_gaussians = []
    for g in gaussians:
        # Transform center
        p_prime = R @ g['center'] + t
        
        # Transform covariance
        cov_prime = R @ g['cov_3d'] @ R.T
        
        # Project to ray space (simplified: assume J is identity for local affine approx or compute J)
        # In full 3DGS, J is the Jacobian of the perspective projection.
        # Here we assume cov_2d is the result of J @ cov_prime @ J.T
        # For the sake of this self-contained pseudocode, we assume cov_2d is provided
        # or computed via standard 3DGS projection logic.
        cov_2d = compute_projection_covariance(cov_prime, p_prime, f) 
        
        projected_gaussians.append({
            'center': p_prime[:2], # xy in camera space
            'cov_2d': cov_2d,
            'alpha': g['alpha']
        })
        
    # 2. Render with 2D Mip Filter
    # Sort gaussians by depth for alpha blending
    projected_gaussians.sort(key=lambda x: x['center'][2])
    
    image = np.zeros((target_resolution[1], target_resolution[0], 3))
    
    for g in projected_gaussians:
        # Apply 2D Mip Filter
        cov_mip = apply_2d_mip_filter(g['cov_2d'], pixel_variance)
        
        # Splat the gaussian with cov_mip
        # Standard volume splatting loop would go here
        # For brevity, we assume a splat function that handles the Gaussian evaluation
        # with the modified covariance.
        splat_image = splat_gaussian_2d(g['center'], cov_mip, g['alpha'], target_resolution)
        
        # Alpha blending
        # C_out = C_in * (1 - alpha) + C_splat * alpha
        # Assuming color is baked in alpha or handled separately
        # This is a simplified blending step
        image = blend_images(image, splat_image, g['alpha'])
        
    return image

def compute_projection_covariance(cov_3d, p_camera, focal_length):
    """
    Simplified projection of 3D covariance to 2D screen space.
    In reality, this involves the Jacobian J of the perspective projection.
    """
    # Standard 3DGS projection:
    # x' = f * X / Z, y' = f * Y / Z
    # Jacobian J depends on p_camera and f.
    # cov_2d = J @ cov_3d @ J.T
    # For this pseudocode, we return a placeholder or simplified version.
    # A real implementation would compute J explicitly.
    return cov_3d[:2, :2] # Placeholder: assumes no perspective distortion for simplicity
```

**Parameters:**
*   $m$: Iteration interval for recomputing maximal sampling rate. Typical value: 100 [S5, unofficial].
*   $s_{3D}$: Variance of the 3D smoothing filter. Typical value: 0.2 [S5, unofficial].
*   $s_{2D}$: Variance of the 2D Mip filter. Typical value: 0.1 [S5, unofficial].

**Edge Cases and Pitfalls:**
*   **Erosion Artifacts:** Standard 3DGS systematically underestimates scale parameters, causing erosion (thin structures) when zooming in because the fixed dilation becomes insufficient as Gaussians expand [S5, unofficial].
*   **Dilation Artifacts:** When zooming out, projected Gaussians shrink, but the fixed dilation causes excessive overlap and dilation artifacts [S5, unofficial].
*   **Aliasing:** Occurs because 3DGS samples at pixel centers, ignoring the pixel footprint, leading to aliasing when resolution changes [S3].
*   **Filter Size Determination:** The 3D smoothing filter size is determined by the *training* data's maximal sampling rate, not the rendering resolution [S5, unofficial].

## Subtopics

### LinPrim Anti-Aliasing
An anti-aliasing strategy that combines a 3D smoothing filter with a bounding box expansion technique to limit high frequencies and prevent aliasing/dilation artifacts [S7]. It applies the 3D smoothing strategy to ensure each primitive is visible from at least one pixel of a training view [S7]. It identifies vertices on the bounding box used for tiling and shifts them outward, parallel to the screen, enlarging the footprint and increasing the minimum appearance size [S7]. This raises opacity along the primitive's extent, acting as a coarse band-limiting approximation [S7]. It cannot use MIP-Splatting's 2D Mip filter because LinPrim primitives do not rely on modifying the screen space Gaussian distribution [S7].

### Anti-Aliasing via Kernel Adaptation
Adapts the kernel bandwidth to prevent aliasing when the volume is undersampled relative to the screen (low magnification or perspective) [S8, unofficial]. It calculates the Jacobian $J$ of the local perspective distortion for each point and computes the ray-space ellipsoid $G_{JMV}$ using $J$ to stretch the kernel appropriately [S8, unofficial]. Swan '97 and Mueller '98 suggest linearly increasing splat kernel size and linearly decreasing splat amplitude [S8, unofficial].

### Integration with Mip-Splatting and Scaffold-GS
Mip-Splatting and Scaffold-GS employ conventional projection of 3D-GS, resulting in larger approximation error compared to methods with optimal projection strategies [S9]. The proposed optimal projection method can be seamlessly integrated with Mip-Splatting to reduce this projection error [S9].

## Operating characteristics
**Time and Memory Complexity:**
MIP-Splatting adds minimal overhead to standard 3DGS. The 3D smoothing filter is applied during training (once per primitive per $m$ iterations) and becomes part of the static representation [S5, unofficial]. The 2D Mip filter is applied during rendering, adding a simple covariance matrix addition ($\Sigma_{k2D} + sI$) and a normalization factor, which is computationally inexpensive [S5, unofficial]. Memory usage is identical to 3DGS as no additional primitives or data structures are required [S5, unofficial].

**Measured Performance:**
On the Blender dataset, MIP-Splatting achieves a PSNR of 32.81 at Full Res, 34.49 at 1/2 Res, 35.45 at 1/4 Res, and 35.50 at 1/8 Res, outperforming 3DGS and 3DGS+EWA [S5, unofficial]. It significantly outperforms baselines in multi-scale testing scenarios [S5, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Eliminates aliasing and dilation/erosion artifacts across various sampling rates (zoom in/out) [S5, unofficial]. Does not require multi-scale images for supervision, unlike Mip-NeRF [S5, unofficial]. Tackles the reconstruction problem via inverse rendering, unlike EWA splatting which only considers rendering [S5, unofficial].
*   **Limitations:** The 3D filter size is fixed based on training data, so it may not adapt to out-of-distribution scenes with significantly different sampling characteristics [S5, unofficial]. The 2D Mip filter approximates a box filter but is still a Gaussian, which may cause slight oversmoothing compared to exact box filtering [S5, unofficial].

**When to Use:**
Use MIP-Splatting for Novel View Synthesis (NVS) in virtual reality, cinematography, and robotics, specifically when training and testing occur at different sampling rates or when zooming is expected [S5, unofficial].

## Background
MIP-Splatting was introduced by Yu et al. in CVPR 2024 [S4, S5, unofficial]. It addresses the limitations of 3D Gaussian Splatting (3DGS) [S5, unofficial], which suffers from aliasing and dilation artifacts when the sampling rate changes (e.g., focal length or camera distance variations) [S2][S5, unofficial]. The method builds on the Nyquist-Shannon Sampling Theorem [S5, unofficial] and the concept of pre-filtering in volume rendering [S1]. It replaces the empirical 2D dilation filter of 3DGS with a 2D Mip filter that mimics the physical box filter of a camera sensor [S3][S5, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S4] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S5] Mip-Splatting, https://arxiv.org/abs/2311.16493, fetched 2026-10-27 (unofficial)
[S6] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S7] LinPrim: Linear Primitives for Differentiable Volumetric Rendering, https://arxiv.org/abs/2501.16312, fetched 2026-10-26
[S8] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
[S9] On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy, https://doi.org/10.1007/978-3-031-72643-9_15, fetched 2026-10-29
