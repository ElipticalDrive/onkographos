# Inverse Rendering and Optical Properties

Also called: Inverse Volume Rendering, Radiance Field Inversion

## Overview
Inverse rendering and optical properties methods reconstruct scene geometry, material properties, and lighting from observed images by optimizing differentiable rendering parameters [S2, 1523-volumetric-inverse-rendering-with-differentiable-path-tracin.md, 6755-neural-radiosity.md]. Unlike forward rendering which assumes known parameters, these methods treat volume density, BRDFs, or neural field weights as trainable variables to minimize the discrepancy between rendered and reference images [1523-volumetric-inverse-rendering-with-differentiable-path-tracin.md, 6755-neural-radiosity.md]. This family is distinguished from its siblings by its focus on recovering physical optical properties (such as extinction coefficients or reflectance fields) rather than just optimizing a static radiance field for novel view synthesis [1523-volumetric-inverse-rendering-with-differentiable-path-tracin.md, 6755-neural-radiosity.md].

## Shared principles
All methods in this family rely on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps [44-radiance-field-optimization.md]. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses) [44-radiance-field-optimization.md].

The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [44-radiance-field-optimization.md]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [44-radiance-field-optimization.md].

Differentiation strategies fall into three main categories [44-radiance-field-optimization.md]:
1. **Reverse-Mode Automatic Differentiation (AD):** Standard backpropagation through the computational graph. This requires storing $O(kn)$ intermediate states (where $n$ is pixels and $k$ is samples), leading to high memory usage [44-radiance-field-optimization.md].
2. **Adjoint Method:** Computes gradients by propagating adjoint variables backward through the rendering algorithm, evaluating the chain rule from output to input. This has computational complexity largely constant in the number of parameters, making it preferable for optimizing many parameters like the transfer function [44-radiance-field-optimization.md].
3. **Analytic Inversion / Explicit Gradients:** Uses a-priori knowledge of the DVR operations to invert the order of operations or derive closed-form gradients, reducing memory consumption to $O(n)$ by avoiding the storage of intermediate results [44-radiance-field-optimization.md].

## Subtopics

### [Volumetric Inverse Rendering with Differentiable Path Tracing](1523-volumetric-inverse-rendering-with-differentiable-path-tracin.md)
This method optimizes a 3D volume density (extinction/scattering) stored on a regular grid to match a set of synthetic reference images [1523-volumetric-inverse-rendering-with-differentiable-path-tracin.md]. It accounts for direct and indirect illumination by using path replay backpropagation to compute derivatives of delta tracking and volumetric multiple scattering [1523-volumetric-inverse-rendering-with-differentiable-path-tracin.md]. Unlike standard volume rendering which assumes known parameters, this approach treats the volume grid as trainable parameters, enabling the reconstruction of volumetric scenes from multi-view observations [1523-volumetric-inverse-rendering-with-differentiable-path-tracin.md].

### [Neural Radiosity](6755-neural-radiosity.md)
Neural Radiosity (NR) is an inverse rendering algorithm that represents the radiance function $L(\mathbf{x}, \omega_o)$ as a single neural network $L_\phi(\mathbf{x}, \omega_o)$ [6755-neural-radiosity.md]. It solves the rendering equation by directly optimizing the network parameters to minimize the $L_2$ norm of the residual of the rendering equation [6755-neural-radiosity.md]. Unlike methods that render images and compare them to ground truth, NR minimizes the physical inconsistency of the radiance field itself [6755-neural-radiosity.md]. It is distinguished from its siblings by its self-training approach on the residual rather than on pixel errors, and by its reliance on automatic differentiation frameworks (like Mitsuba2) to compute gradients of the residual with respect to scene parameters [6755-neural-radiosity.md].

### [NeRV (Neural Reflectance and Visibility Fields)](10246-nerv-neural-reflectance-and-visibility-fields.md)
A method that takes as input a set of images of a scene illuminated by unconstrained known lighting, and produces as output a 3D representation that can be rendered from novel viewpoints under arbitrary lighting conditions, including indirect illumination effects.

### Inverse Rendering with Adjoint Derivatives
- **Definition:** An optimization method that estimates scene optical properties (position, scale, rotation, cross-section, spherical harmonic coefficients) from images using gradient-based optimization with adjoint derivatives of the forward pass [S1, unofficial].
- **Step:** Start optimization with a small number of primitives and gradually generate new primitives to add detail [S1, unofficial].
- **Parameter:** Optimizer: Bound Adam (an improved Adam optimizer that respects parameter boundaries) [S1, unofficial]. Loss function: L1 error, D-SSIM, plus regularization terms for anisotropy and density of small primitives [S1, unofficial].

### Reflectance Fields
- **Definition:** A 3D representation for inverse rendering where the scene is represented as a field of volume density, surface normals, and Bidirectional Reflectance Distribution Functions (BRDFs) [S2].
- **Step:** Augment NeRF with an additional MLP to model lighting [S2]. Model BRDFs with spherical Gaussians [S2]. Incorporate SDF-based representation in the model [S2]. Introduce regularization to stabilize normals from NeRF's foggy geometry [S2].
- **Pitfall:** These methods suffer from extremely time-consuming optimization and slow rendering speed [S2].

## Background
Inverse rendering has evolved from traditional analytical approaches to differentiable neural representations. Early methods focused on recovering shape and lighting from images using analytical gradients [S1, unofficial]. The advent of Neural Radiance Fields (NeRF) enabled differentiable rendering at scale, leading to methods like Neural Volumes and Direct Voxel Grid Optimization that optimize volume densities [44-radiance-field-optimization.md]. More recently, methods like Neural Radiosity have shifted from image-space loss to physics-based residual minimization [6755-neural-radiosity.md]. Reflectance Fields extend NeRF to model complex lighting and material interactions [S2]. The field continues to integrate differentiable path tracing for accurate volumetric effects [1523-volumetric-inverse-rendering-with-differentiable-path-tracin.md] and adjoint methods for efficient optimization [S1, unofficial].

## Sources
[S1] Don’t Splat your Gaussians: Volumetric Ray-Traced Primitives for Modeling and Rendering Scattering and Emissive Media (2025), https://qiita.com/matumu20/items/67b33681dc97ccbcc849, fetched 2026-10-27 (unofficial)
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
