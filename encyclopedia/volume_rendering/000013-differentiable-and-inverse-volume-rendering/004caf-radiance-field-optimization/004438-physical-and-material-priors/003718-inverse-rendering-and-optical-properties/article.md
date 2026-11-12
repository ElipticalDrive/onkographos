# Inverse Rendering and Optical Properties

Also called: Inverse Volume Rendering, Differentiable Volume Rendering with Physical Priors

## Overview
Inverse Rendering and Optical Properties refers to methods that recover physical scene parameters—such as volume density, extinction coefficients, or material reflectance fields—from observed images by optimizing a differentiable forward rendering model [S2, S3, unofficial]. Unlike standard differentiable volume rendering which often relies on analytic inversion or simple reverse-mode AD on discrete steps, this family integrates physical priors (like path tracing or radiosity residuals) to solve for complex volumetric and surface properties [S2, S3, unofficial]. It is distinguished from its siblings by its specific focus on optimizing the underlying optical properties (density, BRDF, visibility) rather than just the radiance field or geometric SDF [S1, S2, S3, unofficial].

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
This method optimizes a 3D volume density (extinction/scattering) stored on a regular grid to match a set of synthetic reference images. It accounts for direct and indirect illumination by using path replay backpropagation to compute derivatives of delta tracking and volumetric multiple scattering. Unlike standard differentiable volume rendering which often relies on analytic inversion or simple reverse-mode AD on discrete steps, this approach integrates the adjoint method directly into the path tracing loop, allowing for the optimization of complex volumetric properties while maintaining differentiability through stochastic sampling events.

### [Neural Radiosity](6755-neural-radiosity.md)
Neural Radiosity is a method that represents the outgoing radiance field of a scene as a neural network and optimizes its parameters by minimizing the residual of the rendering equation. It solves the problem of computing global illumination by treating the rendering equation as a fixed-point problem where the network output is the solution. Unlike differentiable rendering methods that compute gradients via backpropagation through a rasterizer or path tracer, Neural Radiosity directly optimizes the radiance function itself, making it independent of specific camera viewpoints during training. It is distinguished from its sibling [Differentiable Neural Radiosity](7410-differentiable-neural-radiosity.md) by the latter's use of a separate network to cache differential radiance, whereas Neural Radiosity uses a single network for the radiance field itself.

### [NeRV (Neural Reflectance and Visibility Fields)](10246-nerv-neural-reflectance-and-visibility-fields.md)
NeRV is a differentiable inverse rendering method that reconstructs a 3D scene representation from images under unconstrained, known lighting, enabling view synthesis and relighting with indirect illumination. It extends the standard emission-absorption NeRF model by representing the scene as a volumetric field of oriented particles that absorb and reflect light, parameterized by neural networks. Unlike its sibling [Neural Radiance Fields](44-radiance-field-optimization.md) which only models direct emission, or [Neural Reflectance Fields](6755-neural-radiosity.md) which is limited to single-point-light direct illumination, NeRV supports arbitrary environment lighting and one-bounce global illumination. It achieves this tractability by introducing a learned "neural visibility field" that approximates visibility and surface termination depths, reducing the complexity of light transport simulation from quadratic/cubic to linear/double-spherical integral.

### Inverse Rendering with Adjoint Derivatives
- **Definition:** An optimization method that estimates scene optical properties (position, scale, rotation, cross-section, spherical harmonic coefficients) from images using gradient-based optimization with adjoint derivatives of the forward pass [S1, unofficial].
- **Step:** Start optimization with a small number of primitives and gradually generate new primitives to add detail [S1, unofficial].
- **Parameter:** Optimizer: Bound Adam (an improved Adam optimizer that respects parameter boundaries) [S1, unofficial]. Loss function: L1 error, D-SSIM, plus regularization terms for anisotropy and density of small primitives [S1, unofficial].

### Reflectance Fields
- **Definition:** A 3D representation for inverse rendering where the scene is represented as a field of volume density, surface normals, and Bidirectional Reflectance Distribution Functions (BRDFs) [S2].
- **Step:** Augment NeRF with an additional MLP to model lighting [S2]. Model BRDFs with spherical Gaussians [S2]. Incorporate SDF-based representation in the model [S2]. Introduce regularization to stabilize normals from NeRF's foggy geometry [S2].
- **Pitfall:** These methods suffer from extremely time-consuming optimization and slow rendering speed [S2].

## Background
The family builds on [Physical and Material Priors](11198-physical-and-material-priors.md). The use of neural fields for inverse rendering and reflectance fields was proposed in works such as [VHM∗ 22, LWL∗ 23, SDZ∗ 21, ZYL∗ 23, HWL24, ERB∗ 24, LLdC25] [S2]. Recent adaptations to 3DGS by [JTL∗ 24, LZF∗ 24, YHZ24] simplify the classical rendering equation for specular reflections [S2]. The specific volumetric path tracing approach using differentiable primitives is detailed in [S1, unofficial].

## Sources
[S1] Don’t Splat your Gaussians: Volumetric Ray-Traced Primitives for Modeling and Rendering Scattering and Emissive Media (2025), https://qiita.com/matumu20/items/67b33681dc97ccbcc849, fetched 2026-10-27 (unofficial)
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
