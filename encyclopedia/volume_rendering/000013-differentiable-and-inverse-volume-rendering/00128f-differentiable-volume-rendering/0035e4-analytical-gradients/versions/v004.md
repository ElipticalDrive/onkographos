# Analytical Gradients
Also called: Explicit Gradients, Closed-Form Gradients

## Overview
Analytical gradients compute exact derivatives of the rendering equation with respect to input parameters by differentiating the continuous mathematical operations of the rendering pipeline, rather than relying on automatic differentiation (AD) or finite differences [S1, S5]. This approach reduces memory consumption to $O(n)$ by avoiding the storage of intermediate computational graph states required by reverse-mode AD, making it suitable for optimizing large parameter spaces such as dynamic volumetric densities or mesh geometry [S1, S5]. It sets itself apart from [Differentiable volume rendering](43-differentiable-volume-rendering.md) siblings that use [Automatic Differentiation (Reverse/Adjoint Mode)](7151-automatic-differentiation-reverse-adjoint-mode.md) by providing exact gradients for differentiable components while explicitly excluding discrete steps (like triangle selection) from differentiation [S1, S2].

## Shared principles
The family shares the principle of deriving closed-form expressions for the gradient $\frac{\partial \hat{C}}{\partial \theta}$ by applying the chain rule to the continuous parts of the rendering integral or rasterization pipeline. For volume rendering, this involves differentiating the emission-absorption model directly with respect to density $\sigma$ or ray parameters $\mathbf{t}, \mathbf{n}$ [S1]. For mesh rendering, it involves differentiating the projection, barycentric interpolation, and shading functions with respect to vertex positions and camera matrices [S1, unofficial]. The core mechanism avoids the $O(kn)$ memory overhead of standard backpropagation by computing gradients on-the-fly or via adjoint transforms specific to the basis functions (e.g., box splines) or geometric primitives [S5, 2449-differentiable-volume-rendering-via-box-splines.md].

## Subtopics

### [Differentiable Volume Rendering via Box Splines](2449-differentiable-volume-rendering-via-box-splines.md)
Computes analytical gradients for volume rendering using box spline basis functions, supporting 2D/3D uniform grids. It explicitly calculates ray-traced geometry gradients for ray origin/direction and volume coefficients using adjoint transforms, distinguishing it from standard DVR by handling continuous basis functions rather than discrete voxels. Choose for geometry calibration in tomography.

### [Differentiable Volume Rendering with Analytical Gradients](5983-differentiable-volume-rendering-with-analytical-gradients.md)
Derives an analytical form of the radiance transfer equation to compute gradients with respect to volume density directly. It avoids the overhead of propagating gradients through long computational graphs inherent in AD. Choose for efficient optimization of dynamic volumetric densities.

### [Analytical Derivative of Rendering (Mesh)](6761-analytical-derivative-of-rendering-mesh.md)
Computes exact gradients for mesh rendering by differentiating the continuous rasterization steps (projection, barycentric interpolation, shading) while excluding the discrete triangle selection step. It differs from volumetric methods by operating on explicit mesh geometry and from NMR by using exact analytical derivatives for differentiable components. Choose for differentiable mesh optimization.

## Background
The concept of analytical gradients in rendering originates from the need to overcome the memory and computational bottlenecks of automatic differentiation in large-scale inverse rendering problems. Early work in differentiable volume rendering highlighted the $O(kn)$ memory cost of reverse-mode AD [S5]. The specific analytical formulations for box splines [2449-differentiable-volume-rendering-via-box-splines.md] and radiance transfer [5983-differentiable-volume-rendering-with-analytical-gradients.md] were developed to enable efficient optimization of volumetric data. Similarly, analytical derivatives for mesh rendering were introduced to provide exact gradients for shape and appearance optimization, excluding the non-differentiable visibility step [S1, S2].

## Sources
[S1] Differentiable Volume Rendering with Analytical Gradients, https://example.com/5983, 2023-10-01
[S2] Analytical Derivative of Rendering (Mesh), https://example.com/6761, 2023-10-01
[S5] Differentiable Volume Rendering, https://example.com/43, 2023-10-01
