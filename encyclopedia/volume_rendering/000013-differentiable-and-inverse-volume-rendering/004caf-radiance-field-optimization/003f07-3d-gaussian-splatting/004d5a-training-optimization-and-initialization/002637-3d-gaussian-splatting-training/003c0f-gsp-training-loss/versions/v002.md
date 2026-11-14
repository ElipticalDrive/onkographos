# GSP Training Loss

Also called: GSP Loss

## Overview
The GSP Training Loss is the objective function used to optimize the parameters of 3D Gaussian kernels, including position, opacity, covariance, and material properties, within the Gaussian Splashing method [S1]. It extends the standard color reconstruction loss with normal consistency and anisotropy regularization to prevent artifacts caused by excessive kernel elongation during solid object simulation [S1]. Unlike standard 3D Gaussian Splatting which focuses on static geometry, this loss is designed to maintain geometric stability and physical plausibility in physically-based rendering contexts [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Rendered image, ground truth training image, Gaussian kernel parameters (position $\mathbf{x}_p$, opacity $\sigma_p$, covariance $\mathbf{A}_p$, material properties $\{d_p, s_p, \rho_p, \mathbf{n}_p\}$), and camera parameters [S1].
*   **Outputs:** Scalar loss value used for back-propagation to update kernel parameters [S1].

**Data Structures:**
*   **Gaussian Kernel $p$:** Defined by position $\mathbf{x}_p$, opacity $\sigma_p$, covariance matrix $\mathbf{A}_p$, and material properties [S1].
*   **Scaling Vector $\mathbf{S}_p$:** Derived from the covariance matrix, containing scalings $S_{p1}, S_{p2}, S_{p3}$ such that $S_{p1} \ge S_{p2} \ge S_{p3}$ [S1].

**Formula:**
The total loss $\mathcal{L}$ is defined as [S1]:

$$ \mathcal{L} = \mathcal{L}_{color} + \lambda_n \mathcal{L}_{normal} + \lambda_a \mathcal{L}_{aniso} $$

Where:
*   $\mathcal{L}_{color}$ is the color loss between the rendered image and the training image [S1].
*   $\mathcal{L}_{normal}$ is the normal consistency regularization, adopted from GaussianShader [S1].
*   $\mathcal{L}_{aniso}$ is the anisotropy loss [S1].
*   $\lambda_n$ and $\lambda_a$ are weighting coefficients [S1].

The anisotropy loss $\mathcal{L}_{aniso}$ is calculated as [S1]:

$$ \mathcal{L}_{aniso} = \frac{1}{|\mathcal{P}|} \sum_{p \in \mathcal{P}} \max\left( \frac{S_{p1}}{S_{p2}} - a, 0 \right) $$

Where:
*   $\mathcal{P}$ is the set of all Gaussian kernels [S1].
*   $S_{p1}$ is the largest scaling of kernel $p$ [S1].
*   $S_{p2}$ is the second largest scaling of kernel $p$ [S1].
*   $a$ is a ratio threshold [S1].

**Parameters:**
*   $\lambda_n = 0.2$ [S1].
*   $\lambda_a = 10$ [S1].
*   $a = 1.1$ [S1].

**Edge Cases and Pitfalls:**
*   **Normal Ambiguity:** The minimal axis scaling $S_{p3}$ is explicitly excluded from the anisotropy loss constraint [S1]. Constraining $S_{p3}$ would cause normal ambiguity for spherical Gaussian kernels, as the shading normal is based on the minimal axis [S1].
*   **Over-stretching:** The anisotropy loss specifically mitigates rendering artifacts near solid objects by preventing Gaussian kernels from becoming excessively elongated or compressed under large deformations [S1].

## Subtopics

### Velocity Distortion Loss
A loss term used in adaptive spatio-temporal 3D Gaussian Splatting to ensure that Gaussians splatted to the same pixels have consistent velocities [S2]. It is inspired by depth distortion loss [S2]. For a single pixel $p$, the loss is defined as:
$$ \sum_{g} \sum_{g'} w_{g,p} w_{g',p} \| \mathbf{v}_g - \mathbf{v}_{g'} \|^2 $$
where $w_{g,p}$ are the splatting weights of Gaussian $g$ to pixel $p$, and $\mathbf{v}_g$ is the velocity of Gaussian $g$ [S2]. This term is applied in addition to the geometric and color losses [S2].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** The method enables high-frequency detail reconstruction and handles solid simulation dynamics effectively [S1].
*   **Strengths:** Prevents artifacts from over-stretched kernels in solid object rendering [S1]. Ensures temporal consistency in dynamic scenes via velocity distortion [S2].
*   **Limitations:** Requires careful tuning of $\lambda_n$ and $\lambda_a$ [S1]. The velocity distortion loss involves pairwise Gaussian comparisons per pixel [S2].

## Background
The GSP Training Loss originates from the Gaussian Splashing method, which unifies particles for versatile motion synthesis and rendering [S1]. It builds upon the 3D Gaussian Splatting method by incorporating physically-based rendering concepts and a position-based tension model [S1]. The normal consistency term is adapted from GaussianShader [S1]. The velocity distortion loss is introduced in the context of adaptive spatio-temporal 3D Gaussian Splatting for scenes with oscillatory motion [S2].

## Sources
[S1] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
[S2] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
