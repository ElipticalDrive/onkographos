# EOGS (Earth-observation Gaussian Splatting)
Also called: Gaussian Splatting for Efficient Satellite Image Photogrammetry

## Overview
EOGS is a method for digital elevation modeling and appearance recovery from remote sensing images, adapting the standard 3DGS framework for satellite photogrammetry to achieve state-of-the-art performance with significantly faster training times (minutes vs. days) [S1]. It models remote sensing requirements such as shadow modeling and camera-specific corrections to achieve comparable accuracy to EO-NeRF while being over 300 times faster [S1]. Unlike its siblings which often focus on general scenes or specific artifacts, EOGS is specialized for earth observation, replacing complex RPC pipelines with affine approximations and introducing regularizations for sparsity, consistency, and opaqueness to handle semi-transparent shadows and low-visibility regions [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** $N$ non-orthorectified satellite images, their corresponding RPC camera model coefficients, and metadata including 3D satellite attitude and position [S1].
*   **Outputs:** A set of $K$ Gaussian primitives representing the 3D geometry and appearance of the scene, including elevation maps and orthorectified imagery [S1].

**Data Structures**
The method relies on the standard 3DGS primitive structure but modifies initialization and rendering parameters:
*   **Gaussian Primitives ($\gamma_k$):** Defined in world-space coordinates.
    *   **Center ($\mu_k$):** 3D position vector.
    *   **Covariance ($\Sigma_k$):** 3x3 symmetric positive-definite matrix representing shape and orientation.
    *   **Opacity ($\alpha_k$):** Scalar.
    *   **Color:** The method disables per-Gaussian view-direction color dependency (SH coefficients) [S1].
*   **Camera Parameters:**
    *   **Affine Color-Correction ($\phi_A$):** Camera-dependent affine transformation for radiometric correction [S1].
    *   **Ambient Color ($\psi_A$):** Camera-dependent ambient color term [S1].
*   **Coordinate System:** World-space is a uniformly rescaled and recentered version of the Universal Transverse Mercator (UTM) system. The scene center coincides with the origin, the scene is contained in a unit cube, and it is east-north-up aligned [S1].

**Numbered Steps**
1.  **Initialization:**
    *   Initialize Gaussian centers uniformly in the 3D scene such that the initial density is 0.13 Gaussians per cubic meter [S1].
    *   Initialize all Gaussians with white color and opacity set to the lowest possible value of 1% ($0.01$) [S1].
2.  **Camera Approximation:**
    *   Compute a per-scene affine approximation of the transformation from world-space to NDC-space, replacing the full RPC-based transformation pipeline [S1].
    *   For an affine camera model $A: \mathbf{x} \in \mathbb{R}^3 \rightarrow A\mathbf{x} + \mathbf{a} \in \mathbb{R}^2$, the projection of a Gaussian center $\mu_k$ and covariance $\Sigma_k$ simplifies to:
        $$ \mu_{A,k} = A\mu_k + \mathbf{a} $$
        $$ \Sigma'_{A,k} = A \Sigma_k A^T $$
    *   This eliminates the need for the local first-order approximation used in the original 3DGS method, as the approximation is moved to the camera models [S1]. The mean error of this approximation is negligible ($\approx 0.012$ pixels) [S1].
3.  **Rendering with Shadows:**
    *   Render the scene using the affine camera model $A_i$ and sun camera $S_i$ for each input image $i$ [S1].
    *   The synthesized view $\hat{I}_i$ is denoted as $I_{A_i, S_i}$, indicating dependence on both the satellite camera and the sun position [S1].
    *   Implement shadow mapping to render shadows in a physically accurate manner [S1].
4.  **Optimization:**
    *   Run optimization for 5000 iterations [S1].
    *   Use the same optimizer and scheduler as 3DGS for the primitive parameters (centers, covariances, opacities) [S1].
    *   Use a second Adam scheduler with a learning rate of $10^{-2}$ for learning camera-dependent parameters: the affine color-correction $\phi_A$ and the ambient color $\psi_A$ [S1].
    *   Enable shadow mapping and all three regularizations (sparsity, consistency, opaqueness) starting at iteration 1000 [S1].
5.  **Loss Calculation:**
    *   Minimize the final loss function defined as:
        $$ \min \sum_{i=1}^{N} \left( \ell(\hat{I}_i, I_i) + 0.1 L_o + 0.1 L_{cc} + 0.01 L_{ac} + 0.01 L_s \right) $$
        where $\ell$ is the photometric distance function (typically L1 + SSIM as in 3DGS) [S1].
    *   $L_o$: Opacity loss [S1].
    *   $L_{cc}$: Color correction loss [S1].
    *   $L_{ac}$: Ambient color loss [S1].
    *   $L_s$: Shadow entropy loss [S1].

**Formulas**
*   **Optimization Problem:**
    $$ \arg \min_{(\gamma_k)_{k=1}^K} \sum_{i=1}^{N} \ell(\hat{I}_i, I_i) $$
    where $I_i$ is the $i$-th input satellite observation, $\hat{I}_i$ is the corresponding synthesized view, and $\ell$ is the photometric distance function [S1].
*   **Shadow Entropy Loss ($L_s$):**
    Defined as an entropy-based penalty for incorrect use of shadows to handle semi-transparent objects casting semi-transparent shadows:
    $$ L_s = \sum_{u} H(s_{A,S}(u)) $$
    where $H$ is entropy and $s_{A,S}$ relates to the shadow map for camera $A$ and sun $S$ [S1].

**Parameters**
*   $N$: Number of non-orthorectified satellite images used as input [S1].
*   $K$: Number of Gaussian primitives used to represent the scene [S1].
*   Opacity initialization: 1% [S1].
*   Initial Gaussian density: 0.13 Gaussians per m³ [S1].
*   Total iterations: 5000 [S1].
*   Regularization enablement iteration: 1000 [S1].
*   Learning rate for camera-dependent parameters ($\phi_A, \psi_A$): $10^{-2}$ [S1].
*   Loss coefficients: $0.1$ for $L_o$ and $L_{cc}$, $0.01$ for $L_{ac}$ and $L_s$ [S1].

**Pitfalls**
*   **Semi-transparent Shadows:** Semi-transparent objects casting semi-transparent shadows cause geometry misuse where texture is embedded in geometric shadows. This is handled by the entropy-based penalty $L_s$ [S1].
*   **Low Visibility:** EOGS struggles in regions observed in only a few images (low visibility), performing worse than in high-coverage regions [S1].

## Operating characteristics
*   **Time Complexity:** Training time is approximately 3 minutes on tested datasets, compared to 15 hours for EO-NeRF and 25 minutes for SAT-NGP [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On the IEEE GRSS Data Fusion Contest (DFC2019) and IARPA Multi-View Stereo 3D Mapping Challenge (IARPA2016) datasets, EOGS achieves elevation MAE comparable to EO-NeRF (e.g., 1.45m vs 1.37m on DFC2019 no-mask, 0.89m vs 1.02m on DFC2019 foliage-mask) [S1].
*   **Scaling:** EOGS is Pareto-optimal with respect to elevation MAE and training time [S1].
*   **Strengths:** High efficiency (300x faster than EO-NeRF), state-of-the-art accuracy in minutes, handles radiometric correction and shadow modeling [S1].
*   **Limitations:** Struggles in regions with low image coverage (few visible cameras) [S1]. Performance is slightly worse than EO-NeRF on full AOIs without foliage masking, though equivalent when foliage is ignored [S1].
*   **When to use:** Large-scale earth observation datasets requiring fast digital elevation modeling and appearance recovery [S1].
*   **When not to use:** Scenes with very low visibility or sparse camera coverage [S1].

## Background
EOGS was introduced in the paper "Gaussian Splatting for Efficient Satellite Image Photogrammetry" [S1]. It is the first method for digital elevation modeling based on 3DGS [S1]. The method builds on the efficiency of 3DGS [23] and incorporates remote-sensing improvements from EO-NeRF [32], such as radiometric correction and shadow modeling, while introducing novel components including sparsity, view consistency, and opacity regularizations [S1]. It differs from SatNeRF and EO-NeRF by learning colors and not resampling input images that may contain transients or color shifts, and by not requiring pre-trained models for RGB renders or priors on elevation renders [S1].

## Sources
[S1] Gaussian Splatting for Efficient Satellite Image Photogrammetry, https://doi.org/10.1109/cvpr52734.2025.00559, fetched 2026-10-26
