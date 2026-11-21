# Spherical Harmonics Optimization

Also called: SH Optimization, SH Coefficient Optimization, View-Dependent Color Optimization

## Overview
Spherical Harmonics (SH) Optimization refers to the family of methods that determine, compress, or regularize the view-dependent color coefficients $\mathbf{c}_i(\omega)$ stored per 3D Gaussian primitive in 3D Gaussian Splatting (3DGS). It solves the problem of efficiently representing complex lighting and specular effects while managing memory bandwidth, training stability, and computational cost. It differs from its siblings by focusing specifically on the SH coefficient representation itself (storage, degree, or estimation) rather than the camera pose optimization [10077-explicit-camera-jacobian-derivation.md] or the multi-view training loop structure [13982-efficient-multi-view-training-for-3d-gaussian-splatting.md].

## Shared principles
All methods in this family operate on the standard 3D Gaussian Splatting representation, where the scene is defined by a set of $N$ 3D Gaussians $\{G_i\}_{i=1}^N$. Each Gaussian $G_i$ is defined by its mean $\boldsymbol{\mu}_i$, covariance matrix $\boldsymbol{\Sigma}_i$ (derived from rotation $\mathbf{R}_i$ and scale $\mathbf{S}_i$), opacity $\alpha_i$, and view-dependent color $\mathbf{c}_i(\omega)$.

The rendering process evaluates the Gaussian density along a ray $\mathbf{r}(t) = \mathbf{o} + t\omega$ and integrates contributions. The Gaussian density function is:
$$ G(\mathbf{x}) = \exp\left(-\frac{1}{2}(\mathbf{x} - \boldsymbol{\mu})^\top \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu})\right) $$

The core variation across methods lies in the definition, storage, and optimization of $\mathbf{c}_i(\omega)$ and potentially $\alpha_i$, moving beyond the fixed SH coefficients used in base 3DGS. Common strategies include:
1.  **Post-Processing Transformation:** Applying a learned affine transformation to the rendered image based on 3D position and view embedding [14740-color-and-appearance-modeling.md].
2.  **Factorized Illumination:** Separating scene lighting (via a tensor grid) from material properties (via BRDF features) to model specular effects [14740-color-and-appearance-modeling.md].
3.  **Image-Conditioned Modulation:** Using an MLP and per-image embeddings to modulate Gaussian color and opacity, enabling transient object removal and appearance synthesis [14740-color-and-appearance-modeling.md].
4.  **Neural Field Substitution:** Replacing per-Gaussian SH with a shared grid-based neural field to reduce parameters and improve generalization [14740-color-and-appearance-modeling.md].
5.  **Volumetric Attenuation:** Modifying the alpha-blending equation to incorporate medium properties, such as direct attenuation $B_d$ and backscatter parameters $B_\infty, B_b$, to model light transport in non-vacuum media [15148-physics-and-optical-modeling.md].

## Subtopics

### [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md)
First-order Spherical Harmonics Distillation reduces the spherical harmonics (SH) degree of 3D Gaussian elements from third-order to first-order to decrease parameter count and computation cost, enabling real-time rendering on mobile devices. It differs from standard SH degree reduction by employing a distillation strategy where a student model with first-order SH approximates the directional radiance of a pretrained third-order teacher model, rather than simply truncating coefficients. This approach minimizes the quality loss associated with aggressive compression, achieving a 61% reduction in parameters and 52% reduction in computation with only a 0.48 dB PSNR drop. See [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md).

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
This method derives analytical Jacobians of the SE(3) camera pose with respect to 3D Gaussian Splatting (3DGS) primitives to enable efficient gradient-based camera tracking. It solves the problem of high computational overhead and redundant parameterization inherent in automatic differentiation when optimizing camera poses in SLAM systems. Unlike prior ray-intersection methods that assume fewer Gaussians, this approach provides the first analytical Jacobians for EWA splatting and 3DGS, ensuring the Jacobian dimensionality matches the six degrees of freedom of the Lie algebra. See [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md).

### [Efficient Multi-View Training for 3D Gaussian Splatting](13982-efficient-multi-view-training-for-3d-gaussian-splatting.md)
This method enables stable optimization of 3D Gaussian Splatting (3DGS) by training on multiple images per iteration, addressing the high variance of gradients inherent in the standard single-view approach. While standard 3DGS renders one image per iteration due to rasterization overhead, this method uses thread-efficient partial rendering to gather pixels from multiple views without idle thread waste, reducing mini-batch gradient variance and improving convergence stability compared to [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md) and [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md). See [Efficient Multi-View Training for 3D Gaussian Splatting](13982-efficient-multi-view-training-for-3d-gaussian-splatting.md).

### [Spherical Harmonics Vector Quantization](14181-spherical-harmonics-vector-quantization.md)
Spherical Harmonics Vector Quantization (SH VQ) is a compression technique for 3D Gaussian Splatting that replaces per-splat SH coefficients with indices pointing to a shared palette of distinct SH vectors, reducing memory footprint at the cost of minor lighting artifacts. It differs from [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md) by retaining full SH basis functions (rather than reducing to first-order) and from [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md) by focusing on storage compression rather than gradient computation. It is chosen when memory bandwidth or storage size is the primary bottleneck and view-dependent specular effects must be preserved better than simple color quantization allows. See [Spherical Harmonics Vector Quantization](14181-spherical-harmonics-vector-quantization.md).

### [Closed-Form Scene Appearance Estimation](16750-closed-form-scene-appearance-estimation.md)
This method estimates spherical harmonic (SH) coefficients for mesh vertices by solving a global weighted least-squares problem that jointly fits RGB observations from all calibrated views. It distinguishes itself from iterative optimization approaches by providing an analytical closed-form solution, utilizing adaptive regularization to constrain SH bands based on the directional support available at each vertex. See [Closed-Form Scene Appearance Estimation](16750-closed-form-scene-appearance-estimation.md).

### [Adaptive SH Regularization](16786-adaptive-sh-regularization.md)
Adaptive SH Regularization is a technique for stabilizing spherical harmonic (SH) coefficient estimation in sparse or unevenly observed scenes by suppressing higher-order SH components that lack sufficient directional support. It prevents overfitting to input views and improves generalization to unseen viewpoints, distinguishing itself from methods that rely on iterative optimization or fixed SH complexity. See [Adaptive SH Regularization](16786-adaptive-sh-regularization.md).

#### SH Degree Zero Optimization
- **Definition:** Setting the maximum degree of Spherical Harmonics (SH) coefficients to 0 to remove superfluous operations and improve training speed, as higher-degree SH coefficients for color remain unchanged during training [S1, unofficial].
- **Step:** Modify the model code to delete operations relative to the max degree in the model and set the max degree parameter to 0 [S1, unofficial].
- **Pitfall:** In the original 3D Gaussian Splatting model, features represented by SH coefficients with degree > 0 are unchanged during all training, making them useless [S1, unofficial].
- **Number:** Directly changing the max degree to 0 yields a 16.25% improvement compared to the default. With specific changing code, it yields a 16.57% improvement but increases CPU-GPU process time [S1, unofficial].

## Background
The use of Spherical Harmonics for view-dependent appearance in 3D Gaussian Splatting was introduced in the original 3DGS paper [12] (cited in [S1, unofficial]). Subsequent work has focused on optimizing the efficiency and stability of these coefficients, including distillation for mobile deployment , vector quantization for memory reduction [S1, unofficial], and regularization for sparse view scenarios [S1, unofficial].

## Sources
[S1] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
