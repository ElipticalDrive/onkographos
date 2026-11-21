# Analytical Gradients
Also called: Explicit Gradients, Closed-Form Gradients

## Overview
Analytical gradients compute exact derivatives of the volume rendering integral with respect to input parameters (such as density, geometry, or camera pose) by differentiating the rendering equation directly, rather than relying on automatic differentiation (AD) or finite differences [S1, S5]. This approach solves the problem of high memory consumption and computational overhead associated with reverse-mode AD, which requires storing $O(kn)$ intermediate states for $n$ pixels and $k$ samples [S5]. By deriving closed-form expressions for the gradients, the method reduces memory usage to $O(n)$ and enables efficient optimization for dynamic volumetric densities and geometry [S1, S5].

## Shared principles
The family shares the mechanism of inverting the order of operations or deriving explicit chain-rule expansions for the volume rendering equation to avoid storing the full computational graph. The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [S1]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [S1].

Analytical differentiation of this sum requires computing $\frac{\partial \hat{C}}{\partial \theta}$ for a parameter $\theta$ (e.g., $\sigma_i$). Since $\sigma_i$ affects both the emission term at step $i$ and the transmittance $T_j$ for all subsequent steps $j > i$, the gradient is:

$$
\frac{\partial \hat{C}}{\partial \sigma_i} = T_i (1 - e^{-\sigma_i \delta_i})' \mathbf{c}_i + \sum_{j=i+1}^N \frac{\partial T_j}{\partial \sigma_i} (1 - e^{-\sigma_j \delta_j}) \mathbf{c}_j
$$

The term $\frac{\partial T_j}{\partial \sigma_i}$ is derived from the chain rule on the exponential sum, yielding a factor proportional to $-T_j \delta_i$. This explicit propagation allows the gradient to be computed in a single pass or with minimal auxiliary storage, contrasting with the $O(kn)$ memory requirement of standard backpropagation [S5].

## Subtopics

### [Differentiable Volume Rendering via Box Splines](2449-differentiable-volume-rendering-via-box-splines.md)
Computes analytical gradients for ray-traced projections of box spline volumes. It explicitly derives gradients for ray origin/direction and volume coefficients using adjoint transforms, supporting 2D/3D grids with interpolation orders 0, 1, and 2. Choose this for geometry calibration in tomography where exact ray-geometry interaction gradients are needed.

### [Differentiable Volume Rendering with Analytical Gradients](5983-differentiable-volume-rendering-with-analytical-gradients.md)
Derives closed-form radiance derivatives with respect to volume density directly. It avoids the overhead of propagating gradients through long computational graphs inherent in AD. Choose this for efficient optimization of dynamic volumetric densities, such as in fluid reconstruction.

### [Analytical Derivative of Rendering (Mesh)](6761-analytical-derivative-of-rendering-mesh.md)
Computes exact gradients for mesh rendering by differentiating continuous steps (projection, interpolation, shading) while treating triangle selection as a discrete, non-differentiable gate. It relies on barycentric interpolation of vertex attributes. Choose this for differentiable mesh optimization, noting that it yields zero gradients for occluded geometry.

## Background
The concept of analytical gradients in rendering originates from the need to optimize parameters in inverse rendering problems where the number of parameters is large [S1]. The term "Analytical Inversion / Explicit Gradients" was categorized as a distinct strategy from Reverse-Mode AD and the Adjoint Method in the context of differentiable volume rendering [S5]. The specific application to box splines for tomography and to mesh rendering for geometry optimization represents specialized developments within this family [S1, S5].

## Sources
[S1] Differentiable Volume Rendering with Analytical Gradients, https://example.com/5983, 2023-10-01
[S5] Differentiable volume rendering overview, https://example.com/43, 2023-10-01
