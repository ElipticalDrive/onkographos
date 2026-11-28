# LE3D

Also called: Lighting Every darkness with 3DGS

## Overview
LE3D is a method for High Dynamic Range (HDR) view synthesis from noisy RAW images that enables real-time rendering and fast training by replacing 3D Gaussian Splatting's (3DGS) spherical harmonics with a Color MLP, using Cone Scatter Initialization for Structure-from-Motion (SfM) enrichment, and applying depth distortion and near-far regularizations [S1]. It differs from its parent [3iGS](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md) by focusing specifically on RAW input handling and nighttime scene reconstruction rather than factorized tensorial illumination modeling [S1].

## How it works
**Inputs:** Noisy RAW images, camera poses (initially estimated via COLMAP).
**Outputs:** A set of 3D Gaussian primitives optimized for real-time rasterization, supporting HDR rendering, refocusing, and tone-mapping.

**Data Structures:**
Each Gaussian primitive $i$ is defined by the following attributes [S1]:
*   Center position: $\mu_i \in \mathbb{R}^3$
*   Scaling factor: $s_i \in \mathbb{R}^3$
*   Rotation: quaternion $q_i \in \mathbb{R}^4$
*   Opacity: $o_i \in \mathbb{R}$
*   Color features: $f_i$ (input to the Color MLP)
*   Primitive-aware bias: $b_i$ (input to the Color MLP)

**Numbered Steps:**
1.  **Initialization:** Use COLMAP to obtain the initial point cloud and camera poses [S1].
2.  **Cone Scatter Initialization:** To address poor SfM estimation in distant nighttime views, perform random point sampling within a cone using the estimated camera poses to enrich the point clouds [S1].
3.  **Training Setup:** Perform standard 3DGS training. Replace the original Spherical Harmonics (SH) with a tiny Color MLP to represent the RAW linear color space. The MLP takes $f_i$ and $b_i$ as inputs to produce the final rendered color $c_i$ [S1].
4.  **Loss Calculation:** Apply RawNeRF's weighted L2 loss as image-level supervision. Apply proposed depth distortion regularization ($R_{dist}$) and near-far regularization ($R_{nf}$) as scene structure regularizations [S1].

**Formulas:**
The Gaussian basis function is defined as [S1]:
$$G(x) = \exp\left(-\frac{1}{2} (x - \mu)^T \Sigma^{-1} (x - \mu)\right)$$
where $\mu$ is the center in 3D space, and $\Sigma$ is the covariance matrix derived from scaling and rotation parameters.

The pixel color $C$ is computed by blending $N$ ordered Gaussians overlapping the pixel [S1]:
$$C = \sum_{i \in N} c_i \cdot \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)$$
where $\alpha_i = G^2_{2i} o_i$, $c_i$ is the decoded color from the Color MLP, $G^2_{2i}$ is the projected 2D Gaussian, and $o_i$ is the opacity.

The weighted L2 loss used for supervision is [S1]:
$$L_\psi(\hat{y}, y) = \sum_i \left(\frac{\hat{y}_i - y_i}{sg(\hat{y}_i) + \epsilon}\right)^2$$
where $\hat{y}_i$ is the rendered color, $y_i$ is the noisy reference color, $sg(\cdot)$ is the stop-gradient function, and $\epsilon = 10^{-3}$.

**Parameters:**
*   $\epsilon$: A small constant used in the weighted L2 loss denominator to prevent division by zero. Typical value: $10^{-3}$ [S1].

**Pitfalls:**
*   **Nighttime SfM Failure:** In nighttime scenes, extremely low signal-to-noise ratio (SNR) leads to poor SfM estimation in distant views, causing blurred distant views or floaters. LE3D handles this by introducing Cone Scatter Initialization [S1].
*   **Color Representation:** Spherical Harmonics (SH) have limited representation capacity and are unsuitable for RAW linear color space. LE3D handles this by replacing SH with a tiny Color MLP [S1].
*   **Scene Structure:** Inaccurate scene structure hampers downstream tasks like refocusing. LE3D handles this by introducing depth distortion and near-far regularizations [S1].

## Operating characteristics
*   **Training Time:** Approximately 1.5 GPU hours [S1]. This is 1% of the training time required by RawNeRF [S1].
*   **Rendering Speed:** Approximately 100 FPS for 2K resolution images [S1]. This is approximately 4,000 times faster than RawNeRF [S1].
*   **Strengths:** Real-time rendering, fast training, support for HDR rendering, refocusing, and tone-mapping changes [S1].
*   **Limitations:** Relies on COLMAP for initial pose estimation, which may be insufficient for very dark scenes without the Cone Scatter Initialization [S1].

## Background
LE3D (Lighting Every Darkness with 3DGS) was proposed to address the challenges of implementing RAW image-based view synthesis directly using 3D Gaussian Splatting. Previous methods like RawNeRF suffered from slow rendering and long training times due to dense sampling requirements [S1]. LE3D leverages the differentiable rasterization of 3DGS to achieve real-time performance while maintaining the HDR capabilities of RAW-based methods [S1].

## Sources
[S1] Lighting Every Darkness with 3DGS: Fast Training and Real-Time Rendering for HDR View Synthesis, https://doi.org/10.52202/079017-2549, fetched 2026-10-26
