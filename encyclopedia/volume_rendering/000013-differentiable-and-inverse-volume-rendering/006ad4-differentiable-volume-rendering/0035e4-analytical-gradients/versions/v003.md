# Analytical Gradients
Also called: Explicit Gradients, Closed-form Differentiable Rendering

## Overview
Analytical gradients compute exact derivatives of the rendering equation with respect to input parameters (such as volume density, geometry vertices, or camera rays) by differentiating the continuous mathematical operations of the rendering pipeline, rather than relying on automatic differentiation (AD) or finite differences. This approach solves the problem of high memory consumption and computational overhead associated with reverse-mode AD, which requires storing $O(kn)$ intermediate states for $n$ pixels and $k$ samples [S5]. By inverting the order of operations or deriving closed-form expressions, these methods reduce memory usage to $O(n)$ and enable efficient optimization of dynamic volumetric densities and geometry [S1, S5].

## Shared principles
The family shares the core mechanism of evaluating the volume rendering equation and propagating gradients through integration steps, as defined in [Differentiable volume rendering](43-differentiable-volume-rendering.md). The standard emission-absorption model computes color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [S1]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is density, $\delta_i$ is step size, $\mathbf{c}_i$ is color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is transmittance [S1].

Analytical gradients differ from siblings by explicitly deriving the partial derivatives $\frac{\partial \hat{C}}{\partial \theta}$ for parameters $\theta$ (e.g., $\sigma$, $\mathbf{t}$, $\mathbf{n}$, or vertex positions) using the chain rule on the continuous components of the rendering integral. This avoids the discrete, non-differentiable steps (like voxel traversal order or triangle selection) by either masking their contribution or using adjoint transforms that bypass the forward accumulation graph [S1, S5]. The gradients are computed in a backward pass, often using adjoint variables, ensuring that the computational complexity remains largely constant with respect to the number of parameters, which is critical for optimizing high-dimensional fields like transfer functions or dense voxel grids [S1].

## Subtopics

### [Differentiable Volume Rendering via Box Splines](2449-differentiable-volume-rendering-via-box-splines.md)
Computes analytical gradients for ray parameters and volume coefficients using box spline basis functions. It implements a differentiable X-ray transform operator for 2D/3D grids. Choose this for geometry calibration in tomography where basis function interpolation is required.

### [Differentiable Volume Rendering with Analytical Gradients](5983-differentiable-volume-rendering-with-analytical-gradients.md)
Derives explicit gradients for volume density parameters directly from the radiance transfer equation. Choose this for efficient optimization of dynamic volumetric densities (e.g., fluid reconstruction) where basis functions are not used.

### [Analytical Derivative of Rendering (Mesh)](6761-analytical-derivative-of-rendering-mesh.md)
Applies analytical differentiation to mesh rasterization, computing exact gradients for shape, camera, and material parameters. Choose this for differentiable mesh optimization. It yields zero gradients for occluded triangles, unlike methods that approximate visibility changes.

## Background
The concept of analytical gradients in rendering emerged as a solution to the memory bottlenecks of reverse-mode automatic differentiation in large-scale volume rendering. While standard AD stores all intermediate states for backpropagation, analytical methods leverage the mathematical structure of the rendering integral to compute gradients directly [S5]. The specific implementations for box splines and density fields were developed to support inverse rendering tasks such as tomography and fluid reconstruction, where the number of parameters is extremely large [S1, 2449-differentiable-volume-rendering-via-box-splines.md]. The mesh variant extends these principles to surface-based rendering, ensuring exact gradients for selected geometry while acknowledging the non-differentiability of discrete visibility decisions [6761-analytical-derivative-of-rendering-mesh.md].

## Sources
[S1] Differentiable Volume Rendering with Analytical Gradients, https://example.com/dvr-analytical, 2023-10-01
[S5] Differentiable Volume Rendering, https://example.com/dvr-overview, 2023-10-01
