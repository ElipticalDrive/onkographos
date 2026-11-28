# DARBF-Splatting

Also called: DARB-Splatting, Decaying Anisotropic Radial Basis Functions Splatting

## Overview
DARBF-Splatting replaces the standard Gaussian kernel in 3D Gaussian Splatting with a class of non-exponential, decaying anisotropic radial basis functions (DARBFs) [S1]. It solves the limitation of the exponential family by allowing kernels like half-cosine or inverse multiquadric, which span larger regions with higher opacity for the same variance, thereby reducing the number of primitives needed and improving training efficiency [S1]. Unlike standard 3DGS, it introduces a scalar correction factor to approximate the closed-form integration shortcut of Gaussians, preserving computational efficiency while enabling differentiable rendering with non-Gaussian kernels [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D primitives (positions, covariances, colors, opacities) initialized from an SfM pipeline (e.g., COLMAP) [S1].
*   **Output:** A rendered 2D image and optimized primitive attributes via backpropagation [S1].

**Data Structures:**
The primitive $i$ retains the standard 3DGS structure but uses a DARBF kernel instead of a Gaussian [S1]:
*   **Position:** $\mathbf{p}_i \in \mathbb{R}^3$
*   **Covariance:** $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$ (symmetric positive definite). Parameterized by rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$ as in [Parent].
*   **Color:** $\mathbf{c}_i$ (e.g., spherical harmonics coefficients) [S1].

**Method Steps:**

1.  **3D to 2D Projection:** Transform the 3D primitive into the camera coordinate system [S1]. Project the 3D covariance matrix $\boldsymbol{\Sigma}_i$ to a 2D covariance matrix $\boldsymbol{\Sigma}'_i$ by skipping the third row and column of the transformed 3x3 matrix [S1]. This bypasses costly integration, a property specific to the Gaussian family that DARBF-Splatting approximates [S1].

2.  **Compute Mahalanobis Distance:** For a pixel center $x'$ in the 2D image plane, calculate the Mahalanobis distance $d_M$ from the projected primitive center $\boldsymbol{\mu}'$ using the 2D covariance $\boldsymbol{\Sigma}'_i$ [S1]:
    $$ d_M(x'; \boldsymbol{\mu}', \boldsymbol{\Sigma}'_i) = \sqrt{(x' - \boldsymbol{\mu}')^T (\boldsymbol{\Sigma}'_i)^{-1} (x' - \boldsymbol{\mu}')} $$
    where $\boldsymbol{\mu}'$ is the 2D projection of $\mathbf{p}_i$ [S1].

3.  **Evaluate DARBF Kernel:** Compute the kernel value $K(d_M)$ using a chosen DARBF function. The function must be non-negative, decaying, and anisotropic (via $d_M$) [S1].
    *   **Half-Cosine Kernel:**
        $$ K_{hc}(d_M) = \begin{cases} \cos\left(\frac{d_M^2}{\xi}\right) & \text{if } d_M^2 < \frac{\xi \pi}{2} \\ 0 & \text{otherwise} \end{cases} $$
        where $\xi > 0$ is a shape parameter [S1].
    *   **Inverse Multiquadric (Inverse MQ):**
        $$ K_{imq}(d_M) = \frac{1}{\sqrt{1 + d_M^2}} $$
        (Note: Specific parameterization for Inverse MQ in the source is implied by the formula structure, but exact scaling factors are not explicitly detailed beyond the general form [S1]).
    *   **Other DARBFs:** Modified raised cosine, modular sinc [S1].

4.  **Apply Correction Factor:** Since non-Gaussian functions do not naturally integrate to a closed-form constant like Gaussians do during projection, apply a scalar correction factor $\psi$ to the kernel value to approximate the Gaussian's integration shortcut [S1]:
    $$ \sigma_i(x') = \alpha_i \cdot \psi \cdot K(d_M) $$
    The value of $\psi$ is precomputed from closed-form regression experiments to ensure comparable accumulated opacity to Gaussians [S1].

5.  **Rasterization and Blending:** Sort primitives by depth. For each pixel, accumulate color using alpha blending [Parent]:
    $$ C(x') = \sum_{i \in N} \mathbf{c}_i \sigma_i(x') \prod_{j=1}^{i-1} (1 - \sigma_j(x')) $$
    where $\sigma_i(x')$ is the effective opacity contribution of primitive $i$ at pixel $x'$ [S1, Parent].

6.  **Optimization:** Backpropagate the pixel error between views and rasterized splats to optimize the primitives [S1].

**Parameters:**
*   **Kernel Type:** Half-cosine, Inverse MQ, etc. [S1].
*   **Shape Parameter ($\xi$):** Controls the width of the half-cosine kernel [S1].
*   **Correction Factor ($\psi$):** Scalar precomputed from regression [S1].

**Edge Cases and Pitfalls:**
*   **Unbounded Functions:** Growing functions cause instability and are omitted; only decaying functions are used [S1].
*   **Integration Cost:** Using functions outside the Gaussian family typically incurs costly integration [S1].
*   **Precision:** The half-cosine kernel has finite support ($d_M^2 < \frac{\xi \pi}{2}$), which must be checked to avoid evaluating cosine outside the valid domain [S1].

`# Pseudocode not provided in the sources.`

## Subtopics

### Negative Gaussian Splatting
Allows negative opacity values to cancel out artifacts. [Negative Gaussian Splatting](9152-negative-gaussian-splatting.md)

### Scaffold-GS
Wraps primitives around surfaces using a scaffold structure. [Scaffold-GS](9186-scaffold-gs.md)

### GeoGaussian
Enforces geometric constraints on primitives. [GeoGaussian](9187-geogaussian.md)

### Grid-based Blur Kernel
Injects differentiable blur kernels into rasterization. [Grid-based Blur Kernel](14244-grid-based-blur-kernel.md)

## Operating characteristics
*   **Time and memory complexity:** Training convergence is 34% faster than standard 3DGS with the half-cosine kernel [S1]. Memory usage is reduced by 15% compared to standard 3DGS with the half-cosine kernel [S1].
*   **Measured performance:** Achieves comparable PSNR, SSIM, and LPIPS results to 3DGS on Mip-NeRF 360 scenes [S1].
*   **Scaling:** Wider-spread kernels (half-cosine, Inverse MQ) reduce the need for multiple splats to achieve the same accumulated value [S1].
*   **Strengths and limitations:** More efficient training and lower memory footprint [S1]. Requires precomputed correction factors for non-Gaussian kernels; not all DARBFs perform equally (e.g., radial cosine had lower SSIM than half-cosine in Table 6) [S1].
*   **When to use:** When computational savings (training time, memory) are critical and comparable visual quality to 3DGS is acceptable [S1].
*   **When not to use:** Not given in the sources.

## Background
DARBF-Splatting was introduced by Hashiru Pramuditha, Vinasirajan Viruthshaan, Vishagar Arunan, Saeedha Nazar, Sameera Ramasinghe, Simon Lucey, and Ranga Rodrigo from the University of Moratuwa and University of Adelaide [S1]. The work generalizes splatting techniques beyond the exponential family, addressing the underexplored area of non-Gaussian reconstruction kernels in 3D Gaussian Splatting [S1].

## Sources
[S1] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
