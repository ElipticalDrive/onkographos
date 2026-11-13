# Spherical Harmonics Optimization

Also called: SH Optimization, SH Distillation, SH Vector Quantization, SH Degree Zero Optimization

## Overview
Spherical Harmonics (SH) Optimization refers to a family of techniques designed to reduce the computational cost and memory footprint of 3D Gaussian Splatting by modifying how directional radiance is represented and optimized. It addresses the high parameter count associated with high-degree SH coefficients (typically degree 3, requiring 45 coefficients per RGB channel) by employing distillation, quantization, or degree reduction. It sets itself apart from sibling methods like [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md) by targeting the appearance representation's dimensionality rather than the geometric or sorting pipeline.

## Shared principles
The methods in this family share the goal of compressing or simplifying the SH representation of 3D Gaussians. Standard 3D Gaussian Splatting uses SH coefficients to represent view-dependent color. The zeroth-order coefficient ($c_{0,0}$) represents the base color, while higher-order coefficients ($c_{l,m}$ for $l > 0$) represent directional variations. In the original implementation, higher-order coefficients are often static or change minimally during training, leading to redundancy. Optimization strategies exploit this by either:
1.  Reducing the maximum degree $L$ (e.g., from 3 to 0 or 1).
2.  Quantizing the high-dimensional SH vectors into a shared palette.
3.  Distilling high-degree coefficients from a teacher model into a lower-degree student model.

## Subtopics

### [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md)
First-order Spherical Harmonics Distillation reduces the parameter count by distilling high-fidelity third-order SH coefficients from a pretrained teacher model into a compact first-order representation for a student model. Unlike standard degree reduction which simply truncates coefficients, this method uses knowledge distillation to preserve rendering quality despite the aggressive reduction from 16 coefficients per color channel to 4. It sets itself apart from sibling methods like [RoofGS](9830-roofgs.md) or [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md) by specifically targeting the appearance representation's dimensionality rather than the geometric or sorting pipeline. See [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md) for details.

### [Spherical Harmonics Vector Quantization](14181-spherical-harmonics-vector-quantization.md)
Spherical Harmonics Vector Quantization (SH VQ) is a compression technique that replaces per-splat SH coefficients with indices pointing to a shared palette of distinct SH vectors, reducing memory footprint at the cost of minor lighting artifacts. It sets itself apart from quantization methods like Norm11 or Norm565 by leveraging the redundancy in directional lighting across the scene, allowing for significantly higher compression ratios (e.g., 14.9x vs 5.1x for Norm11) while retaining specular "shininess" effects that scalar quantization often blurs. See [Spherical Harmonics Vector Quantization](14181-spherical-harmonics-vector-quantization.md) for details.

### SH Degree Zero Optimization
**Definition**
Setting the maximum degree of Spherical Harmonics (SH) coefficients to 0 to remove superfluous operations and improve training speed, as higher-degree SH coefficients for color remain unchanged during training [S1, unofficial].

**Step**
Modify the model code to delete operations relative to the max degree in the model and set the max degree parameter to 0 [S1, unofficial].

**Pitfall**
In the original 3D Gaussian Splatting model, features represented by SH coefficients with degree > 0 are unchanged during all training, making them useless [S1, unofficial].

**Number**
Directly changing the max degree to 0 yields a 16.25% improvement compared to the default. With specific changing code, it yields a 16.57% improvement but increases CPU-GPU process time [S1, unofficial].

## Background
The use of Spherical Harmonics in 3D Gaussian Splatting was introduced in the original paper by Kerbl et al. (2023) [12]. Subsequent analysis, such as in I3DGS, identified that higher-order SH coefficients for color are often static during training, leading to optimization strategies that remove or compress these coefficients to improve efficiency [S1, unofficial].

## Sources
[S1] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
