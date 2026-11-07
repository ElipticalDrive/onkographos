# Differentiable volume rendering

Also called: Differentiable DVR, DiffDVR, Differentiable Ray Marching

## Overview
Differentiable volume rendering computes the gradient of a rendered image (or derived metric like depth) with respect to scene parameters (density, geometry, transfer functions, camera poses) by making the rendering integral differentiable [S5]. It solves the inverse problem of optimizing 3D scene representations from 2D observations without requiring 3D ground truth supervision [S2, unofficial][S3, unofficial]. It sets itself apart from [Radiance-field optimization](44-radiance-field-optimization.md) by focusing on the general differentiability of the volume rendering integral itself, rather than specific neural field architectures, and from [Tomographic reconstruction as inverse rendering](45-tomographic-reconstruction-as-inverse-rendering.md) by encompassing both tomographic and non-tomographic (e.g., RGB-only) optimization loops [S5].

## Shared principles
The core mechanism is the evaluation of the volume rendering equation and the propagation of gradients through the integration steps [G]. The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [S1]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [S1].

Differentiation strategies fall into three main categories:
1.  **Reverse-Mode Automatic Differentiation (AD):** Standard backpropagation through the computational graph [G]. This requires storing $O(kn)$ intermediate states (where $n$ is pixels and $k$ is samples), leading to high memory usage [S5].
2.  **Adjoint Method:** Computes gradients by propagating adjoint variables backward through the rendering algorithm, evaluating the chain rule from output to input [G]. This has computational complexity largely constant in the number of parameters, making it preferable for optimizing many parameters like the transfer function [S1].
3.  **Analytic Inversion / Explicit Gradients:** Uses a-priori knowledge of the DVR operations to invert the order of operations or derive closed-form gradients, reducing memory consumption to $O(n)$ by avoiding the storage of intermediate results [S5].

## Subtopics

### [Plenoxel Data Structure](930-plenoxel-data-structure.md)
Plenoxel is an explicit, sparse 3D grid representation for photorealistic view synthesis that stores opacity and spherical harmonic (SH) coefficients directly, optimized via gradient descent without neural networks. It differs from its sibling [Differentiable Volumetric Rendering (DVR)](775-differentiable-volumetric-rendering-dvr.md) by using a discrete, sparse voxel structure with trilinear interpolation rather than a continuous implicit field, and from [PlenOctrees](https://arxiv.org/abs/2112.05131) by maintaining continuity in the plenoptic function through interpolation, which is critical for successful optimization. Choose this for explicit, memory-efficient voxel-based optimization of view synthesis.

### [Differentiable Depth Rendering](1332-differentiable-depth-rendering.md)
Differentiable Depth Rendering computes exact gradients of rendered depth values with respect to implicit function parameters by applying the Implicit Function Theorem (IFT). It solves the problem of differentiating through robust, non-differentiable forward tracing algorithms (such as bisection or sphere tracing) by replacing the backward pass with a surrogate gradient expression. Unlike [Differentiable Volumetric Rendering (DVR)](775-differentiable-volumetric-rendering-dvr.md) which integrates density along rays, this method targets implicit surfaces defined by zero-level sets. Choose this for differentiable surface reconstruction from depth or normal targets.

### [Implicit Inverse Opacity Gradient](1372-implicit-inverse-opacity-gradient.md)
Implicit Inverse Opacity Gradient is a method for computing the gradient of the inverse opacity function $t(\theta) = F_r^{-1}(y, \theta)$ with respect to model parameters $\theta$, without back-propagating through the iterative binary search used to find the depth $t$. It solves the problem of differentiating through a root-finding procedure by applying the implicit function theorem to the forward opacity equation $y = F_r(t, \theta)$. Unlike [Differentiable Volumetric Rendering (DVR)](775-differentiable-volumetric-rendering-dvr.md) which differentiates the forward integral directly, this method isolates the depth estimation step, allowing the binary search to remain non-differentiable while providing an analytical gradient for the inverse mapping. Choose this when optimizing depth targets derived from opacity constraints.

### [Differential Diffusion Theory for Participating Media](3028-differential-diffusion-theory-for-participating-media.md)
Differential Diffusion Theory is a differentiable rendering framework for participating media that computes arbitrary scene parameter derivatives (density, absorption/scattering coefficients, anisotropic parameters) by solving the differential form of the diffusion equation. It replaces expensive Monte Carlo path tracing for multiply-scattered light with a partial differential equation (PDE) solver, enabling real-time optimization of millions of variables. Unlike standard ray marching which treats scattering as a local process, it decouples the global scattering solution (fluence) from local ray integration, allowing the use of fast PDE solvers instead of heavy precomputation or path tracing. Choose this for differentiable rendering of dense participating media where standard ray marching is too noisy or expensive.

### [Dual-Pass Differentiable Volume Rendering with Sparse Gradient Recording](3558-dual-pass-differentiable-volume-rendering-with-sparse-gradie.md)
This method is a differentiable rendering technique that executes forward and backward passes concurrently to compute gradients for density field optimization. It solves the high memory cost of standard reverse-mode automatic differentiation by storing gradients only along active sampled ray paths in a sparse structure, rather than the entire volume grid. It sets itself apart from naive differentiable rendering by avoiding a separate backward pass or recomputation of the forward ray march; instead, it relies on gradient records stored during the forward pass to scatter gradients directly to neighboring voxels. Choose this when memory constraints prevent storing full Jacobians but ray paths are sparse.

### [Differentiable Rendering and Sampling Methods](4973-differentiable-rendering-and-sampling-methods.md)
Differentiable Rendering and Sampling Methods constitute the core algorithmic framework for optimizing 3D scene parameters (such as density, color, geometry, or material properties) by propagating gradients from a 2D image loss back through the rendering integral. Unlike traditional rendering which produces images, this method produces a differentiable computational graph where the output is the rendered pixel color $\hat{C}(\mathbf{r})$ and the inputs are the scene representation parameters. It sets itself apart from differentiable rasterization by operating on volumetric fields (density, SDF) via numerical integration rather than explicit geometry, and from standard Monte Carlo ray tracing by ensuring the sampling and integration steps are fully differentiable to enable gradient-based optimization. Choose this as the foundational framework for gradient-based optimization of volumetric scenes.

### [Differential path integral estimator with MIS and NEE](6332-differential-path-integral-estimator-with-mis-and-nee.md)
This method computes the gradient of the rendered image with respect to scene parameters $\theta$ by integrating differential measurement contributions over path space. It addresses the limitation of unidirectional estimators, which produce noisy gradients and inaccurate reconstructions in scenes with hard-to-sample lights and specular materials. The method distinguishes itself by combining Multiple Importance Sampling (MIS) with Next Event Estimation (NEE) to allow the differential vertex to be located at any intermediate vertex or at the emitter connection, rather than being restricted to the final vertex as in standard NEE. Choose this for differentiable path tracing in complex scenes with multiple bounces and emissive objects.

### [Reparameterization Estimators](6499-reparameterization-estimators.md)
Reparameterization Estimators compute gradients for differentiable volume rendering by transforming stochastic sampling operations into deterministic, differentiable functions of the underlying parameters. Unlike standard Monte Carlo estimators where gradients vanish through discrete sampling decisions, this family allows back-propagation through the sample locations themselves. It distinguishes itself from [Adjoint Method](789-adjoint-method.md) by maintaining a forward-mode computational graph for the sampling process, and from [Differentiable Volumetric Rendering (DVR)](775-differentiable-volumetric-rendering-dvr.md) by explicitly handling the non-differentiable nature of discrete ray marching via continuous density approximations. Choose this when differentiating through stochastic sampling or stratified sampling steps.

### [Reynolds Transport Theorem for Differentiable Rendering](6839-reynolds-transport-theorem-for-differentiable-rendering.md)
The Reynolds Transport Theorem (RTT) for differentiable rendering computes the gradient of a rendering integral with respect to scene parameters by decomposing it into an interior term (derivative of the continuous integrand) and a boundary term (integral over discontinuities). Unlike standard rasterization-based differentiable renderers that approximate boundaries via finite differences or edge rasterization, this method provides a principled mathematical model that supports transparent surfaces and complex occlusions without assuming a single triangle per pixel. It distinguishes itself from [Differentiable Volumetric Rendering (DVR)](775-differentiable-volumetric-rendering-dvr.md) by explicitly handling the geometric discontinuities inherent in path integrals through a material-form parameterization, as detailed in [Path-Space Differentiable Rendering (PSDR)](775-differentiable-volumetric-rendering-dvr.md). Choose this for differentiable rendering of scenes with complex visibility boundaries and transparent surfaces.

### [Automatic Differentiation (Reverse/Adjoint Mode)](7151-automatic-differentiation-reverse-adjoint-mode.md)
Not given in the sources.

### [Gradient Estimation and Smoothing](7152-gradient-estimation-and-smoothing.md)
Not given in the sources.

### [Analytical Gradients](7613-analytical-gradients.md)
Not given in the sources.

### [GAN-based Volume Rendering](8825-gan-based-volume-rendering.md)
A method that treats volume rendering as a computational process mapping viewpoint and transfer functions (TFs) directly to an image, bypassing the explicit volume data, using a Generative Adversarial Network (GAN). Choose this when learning implicit mappings from views to images without explicit volumetric representations.

### [Plateau-Reduced Differentiable Path Tracing](8826-plateau-reduced-differentiable-path-tracing.md)
A technique to estimate gradients of a rendering operator with respect to parameters by convolving the rendering function with a blur kernel, allowing differentiation through plateaus and discontinuities in non-differentiable renderers. Choose this when dealing with non-differentiable renderers or visibility discontinuities that cause vanishing gradients.

### [Volumetric Sensor for Differentiable Rendering](8829-volumetric-sensor-for-differentiable-rendering.md)
A sensor implementation for differentiable rendering that accumulates radiance contributions into a volumetric film using delta tracking or similar path tracing logic, supporting both forward and reverse-mode automatic differentiation. Choose this for differentiable path tracing where radiance accumulation needs to be tracked in a volumetric film.

### [Material-Form Differential Path Integral](8831-material-form-differential-path-integral.md)
A formulation for estimating the partial derivative of radiometric measurements with respect to scene parameters by differentiating the material-form path integral, resulting in an integral over the interior path space and a boundary path space. Choose this for differentiable path tracing formulations that require explicit handling of material property derivatives.

### [Hybrid Differentiable Rendering Method](8832-hybrid-differentiable-rendering-method.md)
An analysis-by-synthesis inverse rendering technique that reconstructs 3D geometry and physically-based reflectance (PBR) materials from multi-view 2D images captured by conventional hand-held cameras under uncontrolled lighting. Choose this for inverse rendering of complex scenes with PBR materials from unconstrained multi-view imagery.

### [Differentiable Direct Volume Rendering (DVR)](8871-differentiable-direct-volume-rendering-dvr.md)
Differentiable Direct Volume Rendering (DVR) computes pixel colors by integrating light along rays through a volume, enabling gradient-based optimization of rendering parameters. It distinguishes itself from sibling methods by offering specific implementations for implicit neural representations ([Differentiable Volumetric Rendering (DVR)](775-differentiable-volumetric-rendering-dvr.md)) and analytic inversion for transfer function optimization ([Differentiable Direct Volume Rendering (DiffDVR)](1355-differentiable-direct-volume-rendering-diffdvr.md)), avoiding the high memory costs of standard reverse-mode automatic differentiation. Choose this for optimizing transfer functions or voxel densities in large volumes where memory is a constraint.

## Background
Differentiable volume rendering emerged as a key technique for inverse graphics and 3D reconstruction from 2D images [S5]. Early work focused on differentiable ray marching for neural implicit representations, such as Differentiable Volumetric Rendering (DVR) [S1][S2, unofficial][S3, unofficial]. The challenge of memory consumption in reverse-mode automatic differentiation for large volumes was addressed by DiffDVR, which introduced analytic inversion of blending functions to reduce memory complexity from $O(kn)$ to $O(n)$ [S5]. Subsequent developments included the Adjoint Method for efficient gradient computation with respect to many parameters [S1], and explicit gradient methods for tomography using box splines [S1]. The field has expanded to include differentiable depth rendering via the Implicit Function Theorem [S2, unofficial], sparse gradient recording [3558], and hybrid grey-box differentials [S7, unofficial]. It is widely used in neural radiance fields (NeRF), tomographic reconstruction, and transfer function optimization [S5][S6, unofficial].

## Sources
[S1] Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video, https://doi.org/10.3390/e25091348, fetched 2026-10-26
[S2] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S3] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26 (unofficial)
[S4] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S5] (PDF) Differentiable Direct Volume Rendering - ResearchGate, https://www.researchgate.net/publication/353510271_Differentiable_Direct_Volume_Rendering, fetched 2026-10-30
[S6] PDF TensoRF: Tensorial Radiance Fields - ECVA, https://www.ecva.net/papers/eccv_2022/papers_ECCV/papers/136920332.pdf, fetched 2026-10-29 (unofficial)
[S7] Stochastic Gradient Estimation for Higher-Order Differentiable Rendering, https://doi.org/10.1109/iccv51701.2025.02618, fetched 2026-10-29 (unofficial)
