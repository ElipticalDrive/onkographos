# GPS-Gaussian

Also called: Gaussian Parameter Maps for Gaussian Splatting

## Overview
GPS-Gaussian is a training-free inference method that accelerates 3D Gaussian Splatting (3DGS) reconstruction by regressing Gaussian parameters directly from sparse source views using a depth estimation module [S1]. It solves the problem of slow optimization-based training by bypassing the iterative densification and gradient descent steps, enabling high-resolution rendering on consumer hardware without fine-tuning [S1]. It differs from standard 3DGS Training [3761-3d-gaussian-splatting.md] by replacing the stochastic optimization loop with a deterministic regression pipeline, and from [Base 3DGS Optimization](7549-base-3dgs-optimization.md) by eliminating the need for metric evaluation and densification thresholds during the reconstruction phase.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of sparse source views (images) and their corresponding camera poses.
*   **Outputs:** A set of 3D Gaussians (position, covariance, opacity, spherical harmonics coefficients) and a depth map, ready for rasterization.

**Data Structures:**
The method utilizes a depth estimation module and a Gaussian parameter regression module. The depth module outputs a dense depth map for each input view. The regression module outputs Gaussian parameters (mean $\mu$, covariance $\Sigma$, opacity $\alpha$, and SH coefficients) associated with the depth map pixels.

**Steps:**
1.  **Depth Estimation:** For each sparse source view, a monocular depth estimation network predicts a dense depth map $D$. [G] The depth map is typically normalized to the scene scale or relative depth, depending on the specific depth estimator used (e.g., MiDaS, DPT).
2.  **Gaussian Parameter Regression:** A separate neural network (or the same network with multiple heads) takes the source view image and the estimated depth map as input. It regresses the 3D Gaussian parameters for each pixel. Specifically, for each pixel $i$ with depth $d_i$ and viewing direction $v_i$, the network outputs:
    *   Mean $\mu_i$: Derived from the camera projection of the depth $d_i$ into 3D space.
    *   Covariance $\Sigma_i$: Predicted or derived based on local geometry and uncertainty.
    *   Opacity $\alpha_i$: Predicted confidence or learned opacity.
    *   Spherical Harmonics (SH) coefficients: Predicted view-dependent color appearance.
3.  **Joint Optimization (Implicit):** The depth estimation and Gaussian parameter regression are performed jointly. The source states this is done "without any fine-tuning or optimization" [S1], implying the networks are pre-trained or trained in a supervised manner on a dataset, and inference is a single forward pass. [G] In practice, this means the weights are fixed during the reconstruction of a new scene.
4.  **Rendering:** The resulting 3D Gaussians are rendered using standard 3DGS rasterization [3761-3d-gaussian-splatting.md].

**Formulas:**
The core innovation is the regression of parameters. Let $I$ be the input image and $D$ be the depth map.
$$ \{\mu_i, \Sigma_i, \alpha_i, c_i\} = \text{RegNet}(I, D) $$
where $c_i$ are the SH coefficients. The mean $\mu_i$ for a pixel with camera intrinsic $K$ and pose $T$ is:
$$ \mu_i = T \cdot K^{-1} \cdot \begin{bmatrix} u_i \\ v_i \\ 1 \end{bmatrix} \cdot d_i $$
where $(u_i, v_i)$ is the pixel coordinate and $d_i$ is the depth value.

**Parameters:**
*   **Sparse View Count:** The number of source views. The method is designed for sparse inputs, but the exact number is not specified in the notes [S1].
*   **Network Architecture:** Not specified in the notes [S1].

**Edge Cases and Pitfalls:**
*   **Depth Accuracy:** Since the method relies on monocular depth estimation, errors in depth prediction directly translate to errors in Gaussian position and scale. [G] Poor depth estimates in textureless regions may lead to artifacts.
*   **No Refinement:** Because there is no optimization step, the method cannot correct for initial errors in depth or parameter prediction. [S1]

## Subtopics

### Conditional Slicing
A mechanism used in related dynamic Gaussian methods (like 7DGS) that projects high-dimensional (7D) Gaussians into 3D counterparts conditioned on specific time and view direction, preserving computational efficiency [S2]. It conditions the 6D Gaussian on the viewing direction $d$ (where $X_d = d$) to yield a conditional 3D Gaussian for the spatial component [S2]. The conditional mean is $\mu_{cond} = \mu_p + \Sigma_{pd} \Sigma_d^{-1} (d - \mu_d)$ and the conditional covariance is $\Sigma_{cond} = \Sigma_p - \Sigma_{pd} \Sigma_d^{-1} \Sigma_{pd}^\top$ [S2]. Opacity is modulated by $\alpha_{cond} = \alpha \cdot \exp(-\lambda (d - \mu_d)^\top \Sigma_d^{-1} (d - \mu_d))$, where $\lambda$ controls sensitivity to view direction alignment [S2]. This enhances view-dependent effects like specular highlights [S2].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels in the source views, dominated by the forward pass of the depth and regression networks. It is significantly faster than iterative optimization methods [S1].
*   **Memory Complexity:** $O(N)$ for storing the predicted Gaussians. The neural networks require additional VRAM during inference [G].
*   **Measured Performance:** Not given in the sources for GPS-Gaussian specifically. However, related methods like 7DGS (which uses conditional slicing) achieve real-time rendering speeds exceeding 401 FPS on challenging dynamic scenes [S2].
*   **Strengths:** High speed, no training required for new scenes, compatible with existing 3DGS pipelines [S1].
*   **Limitations:** Relies on the quality of the pre-trained depth and regression networks; cannot adapt to specific scene details via optimization; may struggle with complex view-dependent effects if the regression network is not sufficiently expressive [S1].
*   **When to use:** When fast reconstruction from sparse views is needed and high fidelity is secondary to speed, or as a warm-start for other methods [S1].
*   **When not to use:** When high-fidelity, optimized reconstruction is required, or when the scene contains complex dynamics that require temporal modeling (use [Dynamic and Temporal Modeling](9265-dynamic-and-temporal-modeling.md) instead) [S1].

## Background
GPS-Gaussian was introduced to address the computational cost of traditional 3DGS training. It leverages the dual-graph mechanism for high-fidelity rendering and aims to achieve higher rendering speeds with high resolution on consumer-level devices [S1]. It is part of a broader trend of using neural networks to regress 3DGS parameters directly, bypassing the need for per-scene optimization [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
