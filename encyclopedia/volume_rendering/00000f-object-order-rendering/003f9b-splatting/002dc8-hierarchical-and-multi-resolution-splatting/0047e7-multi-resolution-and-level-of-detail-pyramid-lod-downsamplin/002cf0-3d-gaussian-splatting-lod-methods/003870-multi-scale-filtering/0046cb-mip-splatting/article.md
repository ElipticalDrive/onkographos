# MIP-Splatting

Also called: Alias-free 3D Gaussian Splatting, Anti-Aliased 3D Gaussian Splatting, MS 3DGS, MS-GS, Multi-Scale Gaussians, Multi-scale 3D Gaussian Splatting

## Overview
MIP-Splatting is a method for novel view synthesis that eliminates high-frequency aliasing and dilation/erosion artifacts in 3D Gaussian Splatting (3DGS) when rendering at resolutions, focal lengths, or camera distances different from those encountered during training [S1][S7, unofficial]. It achieves this by introducing a 3D smoothing filter to constrain the maximum frequency of the 3D representation and a 2D Mip filter to approximate the physical camera sensor's box filter during rasterization [S7, unofficial]. Unlike standard 3DGS, which suffers from aliasing when zooming out and dilation artifacts when zooming in, MIP-Splatting ensures alias-free rendering across various sampling rates by modifying the Gaussian projection pipeline [S7, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $K$ 3D Gaussians $\{G_k\}_{k=1}^K$ optimized from multi-view images, each defined by center $p_k$, covariance $\Sigma_k$, opacity $\alpha_k$, and spherical harmonics coefficients for color [S7, unofficial].
*   **Output:** An anti-aliased 2D image rendered at a target resolution and camera pose [S7, unofficial].

**Data Structures:**
Each Gaussian $G_k$ stores:
*   $p_k \in \mathbb{R}^3$: Center position.
*   $\Sigma_k \in \mathbb{R}^{3 \times 3}$: Covariance matrix, parameterized as $\Sigma_k = O_k s_k s_k^T O_k^T$, where $s \in \mathbb{R}^3$ is a scaling vector and $O \in \mathbb{R}^{3 \times 3}$ is a rotation matrix derived from a quaternion [S7, unofficial].
*   $\alpha_k \in [0, 1]$: Opacity.
*   Color coefficients (Spherical Harmonics).
*   *Note:* The original Multi-Scale 3DGS variant [S3] additionally stores per-Gaussian maximum and minimum pixel coverage ($S_{k}^{max}, S_{k}^{min}$) and level indices for selective rendering, but the core MIP-Splatting method [S7, unofficial] focuses on the filtering approach.

**Method Steps:**

1.  **3D Frequency Constraint (Smoothing Filter):**
    During optimization, a 3D low-pass filter is applied to restrict the maximal frequency of the 3D representation to meet the Nyquist limit determined by the training views [S7, unofficial].
    *   Calculate the world space sampling interval $\hat{T}$ at depth $d$ for focal length $f$:
        $$ \hat{T} = \frac{d}{f} $$
    *   The sampling frequency is $\hat{\nu} = \frac{1}{\hat{T}} = \frac{f}{d}$ [S7, unofficial].
    *   Convolve the original Gaussian $G_k$ with a 3D Gaussian low-pass filter $G_{low}$ to obtain the regularized Gaussian $G_k(x)_{reg}$:
        $$ G_k(x)_{reg} = \frac{\sqrt{|\Sigma_k|}}{\sqrt{|\Sigma_k + \hat{\nu}_k^s \cdot I|}} e^{-\frac{1}{2}(x-p_k)^T (\Sigma_k + \hat{\nu}_k^s \cdot I)^{-1} (x-p_k)} $$
        where $s$ is a scalar hyperparameter controlling filter size (typical value: 0.2 variance) [S7, unofficial]. The term $\hat{\nu}_k^s$ scales the identity matrix $I$ to match the sampling frequency [S7, unofficial].
    *   Recompute the maximal sampling rate every $m$ iterations (typical $m=100$) as centers remain stable [S7, unofficial].

2.  **Camera Coordinate Transformation:**
    Transform the regularized 3D Gaussian to camera coordinates using rotation $R$ and translation $t$:
    $$ p'_k = R p_k + t $$
    $$ \Sigma'_k = R \Sigma_k R^T $$
    [S7, unofficial]

3.  **Ray Space Projection:**
    Project the Gaussian to ray space via a local affine transformation $J_k$ (Jacobian of the perspective projection):
    $$ \Sigma''_k = J_k \Sigma'_k J_k^T $$
    [S7, unofficial]

4.  **2D Mip Filter Application:**
    Replace the standard 2D dilation operation used in 3DGS with a 2D Mip filter that approximates the box filter inherent to the physical imaging process [S7, unofficial]. This prevents dilation artifacts when zooming out [S7, unofficial].
    *   The 2D Mip-filtered Gaussian $G_{k2D}(x)_{mip}$ uses a covariance $\Sigma_{k2D} + sI$, where $\Sigma_{k2D}$ is the projected 2D covariance and $s$ is the variance of the 2D Mip filter (typical value: 0.1, chosen to cover a single pixel in screen space) [S7, unofficial].
    *   The final 2D Gaussian is:
        $$ G_{k2D}(x)_{mip} = \frac{\sqrt{|\Sigma_{k2D}|}}{\sqrt{|\Sigma_{k2D} + sI|}} e^{-\frac{1}{2}(x-p_k)^T (\Sigma_{k2D} + sI)^{-1} (x-p_k)} $$
    *   The normalization factor $\frac{\sqrt{|\Sigma_{k2D}|}}{\sqrt{|\Sigma_{k2D} + sI|}}$ preserves the integral (opacity) of the Gaussian after filtering [S7, unofficial].

5.  **Rendering:**
    Rasterize the filtered 2D Gaussians using standard alpha blending [S7, unofficial].

**Parameters:**
*   $s$ (3D filter size): 0.2 [S7, unofficial].
*   $s$ (2D filter variance): 0.1 [S7, unofficial].
*   $m$ (recompute interval): 100 iterations [S7, unofficial].

**Edge Cases and Pitfalls:**
*   **Standard 3DGS Underestimation:** Standard 3DGS systematically underestimates the scale parameter during optimization, leading to erosion when zooming in [S7, unofficial]. The 3D smoothing filter mitigates this by enforcing a minimum frequency bound [S7, unofficial].
*   **Zooming Out Dilation:** Without the 2D Mip filter, zooming out causes dilation artifacts because projected Gaussians shrink while the same dilation is applied [S7, unofficial].
*   **Filter Size Determination:** The 3D smoothing filter size is determined by the maximal sampling frequency of the *input training views*, not the rendering views [S7, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def mip_splatting_render(gaussians, camera_pose, image_shape, s_3d=0.2, s_2d=0.1):
    """
    Render a scene using MIP-Splatting filters.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
                   - center: (3,)
                   - scale: (3,)
                   - rot: quaternion (4,)
                   - opacity: float
                   - sh_coeffs: spherical harmonics coefficients
        camera_pose: Camera extrinsics (R, t) and intrinsics (focal length f, image dimensions).
        image_shape: (H, W)
        s_3d: Scalar hyperparameter for 3D smoothing filter variance.
        s_2d: Scalar hyperparameter for 2D Mip filter variance.
        
    Returns:
        rendered_image: (H, W, 3)
    """
    # 1. Pre-filter 3D Gaussians (Apply 3D Smoothing)
    # Note: In practice, this is done during training. 
    # For inference, we assume the stored Gaussians are already regularized 
    # or we apply the filter if we have access to the training frequency bounds.
    # Here we assume the input gaussians are the optimized ones.
    # If applying filter at inference:
    # For each gaussian k:
    #   Calculate depth d from camera center to gaussian center.
    #   Calculate sampling frequency nu = f / d.
    #   Sigma_reg = Sigma_k + (nu * s_3d) * I
    #   Normalization factor C_3d = sqrt(det(Sigma_k) / det(Sigma_reg))
    #   Store C_3d and Sigma_reg for projection.
    
    # For simplicity, assuming gaussians are pre-regularized or we use standard Sigma
    # The key difference in MIP-Splatting is the 2D filter.
    
    projected_gaussians = []
    
    for g in gaussians:
        # Transform to camera space
        p_cam = g.center @ camera_pose.R.T + camera_pose.t
        if p_cam[2] <= 0: continue # Behind camera
        
        # Project covariance
        # J is the Jacobian of the perspective projection
        J = compute_jacobian(p_cam, camera_pose.focal_length)
        Sigma_cam = camera_pose.R @ g.covariance @ camera_pose.R.T
        Sigma_ray = J @ Sigma_cam @ J.T
        
        # Extract 2D covariance from ray space (bottom-right 2x2 block)
        Sigma_2D = Sigma_ray[:2, :2]
        
        # Apply 2D Mip Filter
        # Covariance becomes Sigma_2D + s_2d * I
        Sigma_mip = Sigma_2D + s_2d * np.eye(2)
        
        # Normalization factor to preserve opacity
        det_2d = np.linalg.det(Sigma_2D)
        det_mip = np.linalg.det(Sigma_mip)
        norm_factor = np.sqrt(det_2d / det_mip)
        
        projected_gaussians.append({
            'center_2d': p_cam[:2],
            'covariance': Sigma_mip,
            'opacity': g.opacity * norm_factor,
            'sh_coeffs': g.sh_coeffs,
            'depth': p_cam[2]
        })
        
    # 2. Rasterize
    # Sort by depth (back to front)
    projected_gaussians.sort(key=lambda x: x['depth'], reverse=True)
    
    image = np.zeros((*image_shape, 3))
    transmittance = np.ones((*image_shape, 1))
    
    for g in projected_gaussians:
        # Rasterize 2D Gaussian with covariance g['covariance']
        # and opacity g['opacity']
        # This involves evaluating the Gaussian over the pixel grid
        # and blending:
        # C_out = C_in * (1 - alpha) + C_gauss * alpha
        # T_out = T_in * (1 - alpha)
        
        # Standard 3DGS rasterization logic adapted for the filtered covariance
        render_gaussian_2d(image, transmittance, g)
        
    return image

def compute_jacobian(p_cam, focal_length):
    """
    Compute the Jacobian of the perspective projection.
    p_cam: (3,) camera coordinates.
    """
    z = p_cam[2]
    fx = focal_length
    # Local affine approximation Jacobian for projection
    # J = [[fx/z, 0,   -fx*x/z^2],
    #      [0,   fx/z, -fx*y/z^2]]
    # We only need the top-left 2x2 for the covariance transformation
    # in the standard 3DGS derivation, J is often simplified to:
    # [[fx/z, 0],
    #  [0,   fx/z]]
    # for the covariance part in the ray space derivation provided in [S7, unofficial]
    return np.array([[fx / z, 0],
                     [0,    fx / z]])
```

## Subtopics

### [2D Mip Filter](9593-2d-mip-filter.md)
The 2D Mip Filter mitigates aliasing artifacts in 3D Gaussian Splatting when rendering from camera distances or focal lengths larger than those encountered during training. It extends the standard 2D Gaussian filter by introducing a view-dependent opacity compensation factor that prevents excessive blurring as Gaussians become smaller relative to the pixel grid. Unlike the basic 2D Gaussian filter which dilates the covariance matrix uniformly, this method dynamically adjusts opacity based on the ratio of the original to the dilated covariance determinant. It approximates the physical box filter of a camera sensor, replacing the standard dilation operation in 3DGS.

### Multi-Scale 3DGS (MS-3DGS)
Multi-Scale 3DGS (MS-3DGS) is a variant that maintains 3D Gaussians at different scales (levels of detail) to represent the same scene, using selective rendering based on pixel coverage to mitigate aliasing and improve speed at low resolutions [S3]. It introduces a 3D smoothing filter and a 2D Mip filter to enforce a maximum sampling rate, eliminating dilation/erosion artifacts when zooming in/out [S6, unofficial]. Unlike the pure filtering approach of MIP-Splatting [S7, unofficial], MS-3DGS [S3] aggregates small Gaussians into larger ones during training to create a hierarchy of detail levels (e.g., 1x, 4x, 16x, 64x) and selects Gaussians based on their projected pixel coverage relative to the current resolution [S3]. This reduces the number of primitives rendered at low resolutions, offering significant speedups [S3].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** MIP-Splatting [S7, unofficial] adds negligible memory overhead compared to standard 3DGS, as it does not store multiple levels of Gaussians. MS-3DGS [S3] adds memory for the aggregated coarse-level Gaussians, though these typically constitute fewer than 5% of the total Gaussian count [S3].
*   **Time:** MIP-Splatting [S7, unofficial] has similar rendering time to standard 3DGS, as the filtering is applied per Gaussian during projection. MS-3DGS [S3] offers significant speedups at low resolutions (e.g., 160%-2400% faster at 4x-128x scale) because fewer, larger Gaussians are rendered [S3]. However, at 1x resolution, performance is comparable to standard 3DGS [S3].

**Measured Performance:**
*   On the Blender dataset, MIP-Splatting achieves a PSNR of 32.81 at Full Res, 34.49 at 1/2 Res, 35.45 at 1/4 Res, and 35.50 at 1/8 Res, outperforming 3DGS and 3DGS+EWA [S7, unofficial].
*   On the MipNeRF360 dataset at 64x scale, MS-3DGS achieves a PSNR of 25.35 vs 15.23 (3DGS), LPIPS 0.066 vs 0.256 (3DGS), and Time 4.9ms vs 103.3ms (3DGS) [S3].
*   On the Tank and Temples dataset at 64x scale, MS-3DGS achieves a PSNR of 19.67 vs 13.88 (3DGS), LPIPS 0.034 vs 0.447 (3DGS), and Time 5.9ms vs 82.6ms (3DGS) [S3].

**Strengths and Limitations:**
*   **Strengths:** Eliminates aliasing and dilation artifacts when zooming in/out or changing focal length [S7, unofficial]. Improves rendering speed at low resolutions (MS-3DGS) [S3]. Maintains high quality at training resolutions [S3].
*   **Limitations:** The 3D filter size is determined by training data, limiting generalization to out-of-distribution sampling rates compared to methods that adaptively determine filter size based on pixel size (though Mip-Splatting claims to handle unobserved scales better than Mip-NeRF) [S7, unofficial]. MS-3DGS requires all Gaussians to be splatted before filtering, preventing linear speedup at very low resolutions [S3].

**When to Use:**
*   Use MIP-Splatting [S7, unofficial] when you need alias-free rendering across various sampling rates without changing the Gaussian representation structure.
*   Use MS-3DGS [S3] when you need to render at significantly lower resolutions than training and require speed improvements alongside anti-aliasing.

## Background
MIP-Splatting was introduced by Yu et al. in 2023/2024 [S7, unofficial] to address the aliasing artifacts inherent in 3D Gaussian Splatting [12] when rendering at resolutions or focal lengths different from training. It builds on the observation that standard 3DGS samples the scene at pixel centers, ignoring the pixel footprint, which causes aliasing when changing resolution [S5]. The method introduces a 3D smoothing filter to constrain the 3D representation's frequency and a 2D Mip filter to approximate the camera's box filter [S7, unofficial]. It is used for novel view synthesis in virtual reality, cinematography, and robotics, specifically addressing out-of-distribution generalization [S7, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S3] Multi-Scale 3D Gaussian Splatting for Anti-Aliased Rendering, https://doi.org/10.1109/cvpr52733.2024.01977, fetched 2026-10-26
[S4] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S5] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S6] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S7] Mip-Splatting, https://arxiv.org/abs/2311.16493, fetched 2026-10-27 (unofficial)
[S8] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S9] LinPrim: Linear Primitives for Differentiable Volumetric Rendering, https://arxiv.org/abs/2501.16312, fetched 2026-10-26
[S10] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
[S11] On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy, https://doi.org/10.1007/978-3-031-72643-9_15, fetched 2026-10-29
