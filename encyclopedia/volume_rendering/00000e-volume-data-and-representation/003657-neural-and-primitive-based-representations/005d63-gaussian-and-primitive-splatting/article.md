# Gaussian and Primitive Splatting
## Overview
Gaussian and Primitive Splatting methods represent 3D scenes as a collection of parameterized volumetric primitives (typically Gaussians or neural-defined kernels) that are rasterized or splatted onto the image plane [G]. This approach solves the trade-off between the high expressivity of neural implicit fields and the high rendering speed of explicit primitives [G]. Unlike sibling methods like [MLP Neural Fields](3854-mlp-neural-fields.md) which rely on ray marching through a global network, this family achieves efficient rendering by approximating the volume rendering integral via analytic or semi-analytic integration of individual primitive contributions [G].
## Shared principles
## Subtopics

### [Splat the Net](3241-splat-the-net.md)
Splat the Net replaces Gaussian primitives with splattable neural density fields to represent 3D scenes. It uses MLP-based primitives that require roughly 10× fewer primitives than 3D Gaussian Splatting for the same scene representation while maintaining >100 FPS rendering speed. Unlike siblings using fixed Gaussian kernels, it uses an 8-neuron MLP to define each primitive's density profile, offering greater expressivity under identical memory budgets. Choose it when neural density expressivity is needed without sacrificing real-time rendering speed.

### [Neural Primitive Splatting](3300-neural-primitive-splatting.md)
Neural Primitive Splatting reconciles neural implicit expressivity with primitive-based splatting efficiency. It represents the volumetric kernel (density distribution) itself as a neural network bounded by an ellipsoid, enabling exact analytical integration along view rays for perspectively accurate splatting without costly ray marching. Unlike 3D Gaussian Splatting which uses an affine projection approximation, this method yields multi-view consistency by construction. Choose it when perspectively accurate splatting and multi-view consistency are required.

### 3D Gaussian Representation for CT Volumes
#### definition
- [S1, unofficial] The method represents Computed Tomography (CT) volumes using a 3D Gaussian representation, where the volume is modeled as a collection of 3D Gaussians rather than a voxel grid or implicit field.
#### step
- [S1, unofficial] The CT volume is parameterized by a set of 3D Gaussians, each defined by a mean position, a covariance matrix (or scale and rotation), and an opacity/density value, allowing for differentiable rendering of the volume from arbitrary viewpoints.

## Background
The family builds on the principles of 3D Gaussian Splatting for natural scene rendering, where light attenuation follows an exponential transmittance model $T(t) = \exp\left(-\int_{t_0}^t \sigma(\mathbf{r}(s)) ds\right)$ based on the Beer-Lambert law [S1, unofficial]. This model describes light propagation from one point $\mathbf{r}(t_0)$ to another $\mathbf{r}(t)$ along a ray with origin $\mathbf{o}$ and direction $\mathbf{d}$, where $\sigma(\mathbf{p})$ denotes density [S1, unofficial]. Applying this model to X-ray attenuation enables the reconstruction of CT volumes from sparse views using large Gaussian Reconstruction Models [S1, unofficial].
## Sources
[S1] X-GRM: Large Gaussian Reconstruction Model for Sparse-view X-rays to Computed Tomography, https://doi.org/10.48550/arXiv.2505.15235, fetched 2026-10-27 (unofficial)
