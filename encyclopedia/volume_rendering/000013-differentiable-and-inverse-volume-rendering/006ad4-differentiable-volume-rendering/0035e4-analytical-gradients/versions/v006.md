# Analytical Gradients
Also called: Explicit Gradients, Closed-form Gradients, Analytic Inversion

## Overview
Analytical gradients compute the derivatives of the volume rendering equation with respect to input parameters (such as density, geometry, or camera pose) by explicitly expanding the chain rule, rather than relying on automatic differentiation (AD) or finite differences. This approach solves the problem of high memory consumption and computational overhead associated with reverse-mode AD, which requires storing $O(kn)$ intermediate states for $n$ pixels and $k$ samples [S5]. By deriving closed-form expressions for the gradients, this family reduces memory usage to $O(n)$ and enables efficient optimization of dynamic volumetric densities and geometric parameters [S1, S5].

## Shared principles
The core mechanism relies on the differentiability of the continuous components of the rendering pipeline (projection, shading, or emission-absorption integration) while acknowledging that discrete steps (such as triangle selection or voxel traversal order) are non-differentiable [S1, unofficial].

For volume rendering, the standard emission-absorption model computes color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [S1]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is density, $\delta_i$ is step size, $\mathbf{c}_i$ is color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is transmittance [S1].

Analytical gradients for this model are derived by differentiating the sum term-by-term. The gradient with respect to a specific density $\sigma_k$ involves two components: the direct effect on the emitted radiance at step $k$, and the indirect effect on the transmittance $T_j$ for all subsequent steps $j > k$ [S1]. This explicit expansion avoids the need to store the full computational graph, allowing gradients to be computed in a single backward pass or alongside the forward pass with constant memory overhead relative to the number of parameters [S5].

## Subtopics

### [Differentiable Volume Rendering via Box Splines](2449-differentiable-volume-rendering-via-box-splines.md)
Computes analytical gradients for ray-traced projections of volumes defined by box splines. It differs from standard DVR by explicitly calculating geometry gradients for ray anchors and directions using adjoint functions, supporting interpolation orders 0, 1, and 2. Best for geometry calibration in tomography.

### [Differentiable Volume Rendering with Analytical Gradients](5983-differentiable-volume-rendering-with-analytical-gradients.md)
Provides closed-form gradients for the discrete emission-absorption model with respect to volume density. It sets itself apart by relying on explicit chain-rule expansion of the DVR integral rather than basis function properties, making it suitable for dynamic fluid reconstruction.

### [Analytical Derivative of Rendering (Mesh)](6761-analytical-derivative-of-rendering-mesh.md)
Computes exact gradients for mesh-based rendering by differentiating continuous projection and shading stages. It differs from volumetric methods by operating on explicit triangle meshes and excludes the non-differentiable discrete triangle selection step. Best for geometry and material optimization where visibility is fixed.

## Background
The concept of analytical gradients in rendering originated from the need to optimize parameters in computer graphics without the prohibitive memory costs of reverse-mode automatic differentiation. Early work in differentiable rendering focused on surface-based primitives, where analytical derivatives of the rasterization and shading pipelines were derived [S1, unofficial]. In volume rendering, the application of analytical gradients was developed to handle the high-dimensional parameter spaces of volumetric data, such as density fields in fluid simulation or neural radiance fields. The method is primarily used in inverse rendering tasks, including tomography reconstruction, material optimization, and differentiable physics simulation.

## Sources
[S1] Differentiable Volume Rendering with Analytical Gradients, https://example.com/dvr-analytical, 2023-10-01
[S5] Automatic Differentiation (Reverse/Adjoint Mode), https://example.com/ad-reverse, 2023-10-01
