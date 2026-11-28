# GaussianCube

Also called: Differentiable 3D Gaussian Splatting, GaussianCube representation

## Overview
GaussianCube is a scene representation method that utilizes an unstructured, explicit set of 3D Gaussians to enable real-time, high-quality novel-view synthesis [S1, unofficial]. It solves the problem of efficient radiance field rendering by replacing continuous volumetric ray-marching with differentiable rasterization of anisotropic splats, allowing for fast visibility-aware rendering and adaptive density control [S1, unofficial]. Unlike NeRF methods that rely on costly random sampling of continuous representations, GaussianCube uses discrete points that support flexible creation, destruction, and displacement of geometry without volumetric overhead [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A set of images of a static scene and corresponding cameras calibrated via Structure-from-Motion (SfM), which produces a sparse point cloud [S1, unofficial].
*   **Output:** A set of 3D Gaussians defined by position, covariance, opacity, and spherical harmonics (SH) coefficients, optimized to render novel views [S1, unofficial].

**Data Structures**
Each 3D Gaussian $i$ in the set is defined by the following parameters:
*   **Position:** Mean vector $\mu_i \in \mathbb{R}^3$ [S1, unofficial].
*   **Covariance:** A full 3D covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$, parameterized for optimization using a scaling vector $s_i \in \mathbb{R}^3$ and a rotation quaternion $q_i \in \mathbb{R}^4$ [S1, unofficial].
*   **Opacity:** Scalar $\alpha_i \in (0, 1)$ [S1, unofficial].
*   **Color:** Spherical Harmonics (SH) coefficients $\mathbf{c}_i$ representing directional appearance [S1, unofficial].

**Formulas**
The 3D Gaussian function in world space is defined as [S1, unofficial]:
$$G(x) = e^{-\frac{1}{2}(x-\mu)^T\Sigma^{-1}(x-\mu)}$$

The covariance matrix $\Sigma$ is constructed from scale $S$ and rotation $R$ as [S1, unofficial]:
$$\Sigma = RSS^T R^T$$
where $R$ is the rotation matrix derived from quaternion $q$, and $S$ is the scaling matrix derived from vector $s$ [S1, unofficial].

To render, the 3D covariance is projected to 2D camera space using the Jacobian $J$ of the affine approximation of the projective transformation and the viewing transformation $W$ [S1, unofficial]:
$$\Sigma' = JW\Sigma W^T J^T$$
The 2D covariance is obtained by skipping the third row and column of $\Sigma'$ [S1, unofficial].

The image formation model uses alpha blending [S1, unofficial]:
$$C = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i$$
where $\alpha_i = (1 - \exp(-\sigma_i \delta_i))$ and $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ [S1, unofficial].

**Parameters**
*   **Scene Size:** Typically 1-5 million Gaussians [S1, unofficial].
*   **Loss Function:** $\mathcal{L} = (1 - \lambda)\mathcal{L}_1 + \lambda\mathcal{L}_{\text{D-SSIM}}$ with $\lambda = 0.2$ [S1, unofficial].
*   **Activations:** Sigmoid for opacity $\alpha$ to constrain it to $(0, 1)$; exponential for scale to ensure positivity [S1, unofficial].

**Steps**
1.  **Initialization:** Initialize the set of 3D Gaussians using the sparse point cloud produced by SfM [S1, unofficial]. Estimate initial covariance as an isotropic Gaussian with axes equal to the mean distance to the closest three points [S1, unofficial].
2.  **Optimization Loop:** Perform interleaved optimization of 3D Gaussian properties (position, covariance, opacity, SH coefficients) and adaptive density control steps (adding/removing/moving Gaussians) to handle ambiguities in 3D-to-2D projection [S1, unofficial].
3.  **Rendering:**
    *   Sort Gaussians by depth using fast GPU sorting algorithms inspired by tile-based rasterization [S1, unofficial].
    *   Project 3D Gaussians to 2D ellipses using the projected covariance [S1, unofficial].
    *   Rasterize anisotropic splats respecting visibility order via alpha blending [S1, unofficial].
    *   Compute gradients by tracking accumulated transmittance values for a fast backward pass [S1, unofficial].

**Pitfalls**
*   Directly optimizing the covariance matrix $\Sigma$ is invalid because gradient descent cannot easily constrain it to remain positive semi-definite, leading to invalid matrices [S1, unofficial]. This is solved by parameterizing $\Sigma$ via scale and rotation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering is real-time (≥ 30 fps) at 1080p resolution due to efficient tile-based rasterization and sorting [S1, unofficial]. Training is accelerated by the fast backward pass of the rasterizer [S1, unofficial].
*   **Memory Complexity:** Requires storage for 1-5 million Gaussians per scene [S1, unofficial].
*   **Strengths:** High visual quality, real-time rendering, handles anisotropic structures compactly, does not require Multi-View Stereo (MVS) data (only SfM points) [S1, unofficial].
*   **Limitations:** Unstructured representation may require significant memory for large scenes compared to grid-based methods; visibility ordering relies on sorting which can be costly if not optimized [S1, unofficial].
*   **When to use:** When real-time novel-view synthesis is required from sparse SfM data without MVS [S1, unofficial].
*   **When not to use:** When order-independent transparency is strictly required without sorting overhead, or when continuous volumetric properties are needed for physics simulation [S1, unofficial].

## Background
GaussianCube, also referred to as Differentiable 3D Gaussian Splatting in the context of the primary source, was introduced by Bernhard Kerbl, Georgios Kopanas, et al. [S1, unofficial]. The method combines the best of point-based rendering and neural radiance fields, using explicit 3D Gaussians for efficient rasterization while preserving the differentiable volumetric properties of NeRFs [S1, unofficial]. It was developed to address the computational expense of volumetric ray-marching in NeRFs and the limitations of point-based methods that require MVS data [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting for Real-Time Radiance Field Rendering, https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/3d_gaussian_splatting_low.pdf, fetched 2026-10-26 (unofficial)
[S2] Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
