# Spherical Harmonics Optimization

Also called: SH Optimization, View-Dependent Color Optimization

## Overview
Spherical Harmonics (SH) Optimization encompasses methods that define, store, and optimize the view-dependent color coefficients $\mathbf{c}_i(\omega)$ for 3D Gaussian primitives, moving beyond the fixed SH coefficients used in base 3D Gaussian Splatting (3DGS) [G]. It solves the problem of accurately modeling complex lighting and material properties (e.g., specularities, anisotropy) while managing the high memory and computational costs associated with high-degree SH representations. It sets itself apart from its siblings by offering diverse strategies: distillation to lower degrees [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md), vector quantization for compression [Spherical Harmonics Vector Quantization](14181-spherical-harmonics-vector-quantization.md), analytical Jacobian derivation for pose optimization [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md), multi-view training stability [Efficient Multi-View Training for 3D Gaussian Splatting](13982-efficient-multi-view-training-for-3d-gaussian-splatting.md), closed-form estimation for meshes [Closed-Form Scene Appearance Estimation](16750-closed-form-scene-appearance-estimation.md), adaptive regularization [Adaptive SH Regularization](16786-adaptive-sh-regularization.md), and geometry-guided neural substitution [Geometry-Guided Appearance Modeling](20876-geometry-guided-appearance-modeling.md).

## Shared principles
All methods in this family inherit the standard 3D Gaussian Splatting representation and rendering pipeline from [Scene Representation and Geometry Constraints](15421-scene-representation-and-geometry-constraints.md) [G]. The core variation across methods lies in the definition, storage, and optimization of $\mathbf{c}_i(\omega)$ and potentially $\alpha_i$, moving beyond the fixed SH coefficients used in base 3DGS [G]. Common strategies include post-processing transformation [DAVIGS](12021-davigs.md), factorized illumination [3iGS](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md), image-conditioned modulation [SWAG](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md), and neural field substitution [Compact View-dependent Color](12904-compact-view-dependent-color.md) [G].

## Subtopics

### [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md)
First-order Spherical Harmonics (SH) Distillation is a compression technique that reduces the SH degree of 3D Gaussian elements from the standard third-order (degree 3) to first-order (degree 1) to decrease parameter count and rendering computation cost. Unlike standard degree reduction which simply truncates coefficients, this method employs a distillation strategy where a "student" model with first-order SH approximates the directional radiance of a pretrained "teacher" model. It sets itself apart from siblings like [Spherical Harmonics Vector Quantization](14181-spherical-harmonics-vector-quantization.md) (which quantizes coefficients) and [Compact View-dependent Color](12904-compact-view-dependent-color.md) (which uses neural fields) by explicitly training a lower-degree SH representation to mimic the output of a higher-degree reference.

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
This method derives analytical Jacobians of the SE(3) camera pose with respect to 3D Gaussian Splatting (3DGS) primitives to enable efficient gradient-based camera pose optimization. It solves the problem of high computational overhead associated with automatic differentiation in differentiable rendering pipelines, particularly for SLAM systems where tracking must be performed in real-time. It sets itself apart from prior ray-intersection methods by providing the first analytical Jacobian specifically tailored for EWA splatting and 3DGS, ensuring the Jacobian dimensionality matches the 6 degrees of freedom of the Lie algebra without redundant computations.

### [Efficient Multi-View Training for 3D Gaussian Splatting](13982-efficient-multi-view-training-for-3d-gaussian-splatting.md)
This method enables stable optimization of 3D Gaussian Splatting (3DGS) by training on multiple views per iteration, addressing the high variance of gradients inherent in the standard single-view approach. It distinguishes itself from standard 3DGS by replacing the single-image mini-batch with a multi-image batch, while overcoming the rasterization overhead and adaptive density control incompatibilities that typically prevent such an implementation.

### [Spherical Harmonics Vector Quantization](14181-spherical-harmonics-vector-quantization.md)
Spherical Harmonics Vector Quantization (SH VQ) is a compression technique for 3D Gaussian Splatting that replaces per-splat SH coefficients with indices pointing to a shared palette of distinct SH vectors, significantly reducing memory footprint at the cost of minor lighting artifacts. Unlike methods that exploit spatial correlation between neighboring Gaussians, SH VQ treats each splat's SH data independently, clustering millions of high-dimensional vectors into a fixed-size palette. This approach sets it apart from spatially-aware compression methods like [Hybrid Compression Method for Trained 3D Gaussian Splatting Models Based on VQ and HEVC](16750-hybrid-compression-method-for-trained-3d-gaussian-splatting-models-based-on-vq-and-hevc.md) by avoiding inter-splat prediction, which simplifies implementation but can introduce localized color inconsistencies.

### [Closed-Form Scene Appearance Estimation](16750-closed-form-scene-appearance-estimation.md)
Closed-Form Scene Appearance Estimation solves the problem of estimating view-dependent spherical harmonic (SH) coefficients for mesh vertices by formulating a global weighted least-squares problem across all calibrated input views. It differs from iterative optimization methods by providing an analytical solution to the normal system, utilizing adaptive regularization to handle varying view support and a matrix-free Preconditioned Conjugate Gradient (PCG) solver to avoid explicit matrix inversion.

### [Adaptive SH Regularization](16786-adaptive-sh-regularization.md)
Adaptive SH Regularization is a technique for stabilizing the optimization of Spherical Harmonics (SH) coefficients in 3D Gaussian Splatting under sparse or uneven camera observations. It prevents overfitting by suppressing higher-order SH components for vertices that lack sufficient directional support, while preserving the zeroth-order (DC) component. Unlike methods that rely on iterative optimization with fixed SH complexity, this approach explicitly adapts the regularization strength to the available viewing directions, improving generalization to unseen viewpoints and conditioning of the linear system.

### [Geometry-Guided Appearance Modeling](20876-geometry-guided-appearance-modeling.md)
Geometry-Guided Appearance Modeling (GAM) is a method for predicting view-dependent colors in 3D Gaussian Splatting by incorporating geometric cues, specifically surface normals and learned geometric features derived from a Signed Distance Field (SDF). It addresses the failure of standard appearance modeling, which relies solely on spatial position and viewing direction, to capture fine-grained geometry and reflective surfaces. Unlike siblings that optimize fixed Spherical Harmonics coefficients or use post-processing transformations, GAM replaces the SH representation with a geometry-conditioned Multi-Layer Perceptron (MLP).

### SH Degree Zero Optimization
#### definition
- [S1, unofficial] Setting the maximum degree of Spherical Harmonics (SH) coefficients to 0 to remove superfluous operations and improve training speed, as higher-degree SH coefficients for color remain unchanged during training.
#### step
- [S1, unofficial] Modify the model code to delete operations relative to the max degree in the model and set the max degree parameter to 0.
#### pitfall
- [S1, unofficial] In the original 3D Gaussian Splatting model, features represented by SH coefficients with degree > 0 are unchanged during all training, making them useless.
#### number
- [S1, unofficial] Directly changing the max degree to 0 yields a 16.25% improvement compared to the default. With specific changing code, it yields a 16.57% improvement but increases CPU-GPU process time.

## Background
The standard 3D Gaussian Splatting representation uses third-order Spherical Harmonics (degree 3) to encode view-dependent color, resulting in 16 coefficients per color channel per Gaussian . This high dimensionality leads to significant memory overhead and computational cost during rendering and optimization. Early observations indicated that in standard training pipelines, higher-degree coefficients often fail to converge or provide negligible visual improvement, suggesting redundancy [S1, unofficial]. Subsequent research has focused on optimizing this representation through compression (distillation, quantization), efficient optimization (analytical Jacobians, multi-view training), and alternative modeling (closed-form estimation, neural fields) to balance quality, speed, and memory usage.

## Sources
[S1] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
