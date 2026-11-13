# Finite Difference Stencils (Central/Forward/Backward)

Also called: Finite Difference Gradient Estimation, Spatial Derivative Approximation

## Overview
Finite difference stencils approximate the spatial gradient of a scalar volume field by computing weighted differences between voxel values at discrete offsets. This method solves the problem of estimating surface normals for lighting and opacity calculations in volume rendering when analytical derivatives are unavailable. It is distinguished from [Analytic Gradient Computation](2167-analytic-gradient-computation.md) by its reliance on discrete sampling rather than closed-form expressions, and from [Gradient Estimation from Samples](2180-gradient-estimation-from-samples.md) by its fixed, regular grid structure.

## Shared principles
All finite difference gradient methods in this family compute the gradient vector $\nabla V = (G_x, G_y, G_z)$ at a voxel center $\mathbf{p} = (x, y, z)$ by evaluating the scalar field $V$ at neighboring positions defined by offsets $\delta$ and weights $w$. The general form for a partial derivative along axis $i$ is:
$$ G_i = \frac{1}{h} \sum_{k} w_k V(\mathbf{p} + \delta_k \mathbf{e}_i) $$
where $h$ is the voxel spacing (assumed isotropic $h=1$ in integer indexing unless physical units are required) and $\mathbf{e}_i$ is the unit vector for axis $i$.

The core components shared by all children are:
1.  **Stencil Offsets:** A set of integer coordinate displacements relative to the center voxel.
2.  **Stencil Weights:** Coefficients applied to the neighbor values to approximate the derivative.
3.  **Normalization:** Division by the step size $h$ (or $2h$ for central differences) to convert discrete differences into derivative approximations.
4.  **Boundary Handling:** Strategies for out-of-bounds access, such as clamping, zero-padding, or extrapolation, which are critical because stencils often require neighbors outside the volume domain.

## Subtopics

### [Neumann Gradient](3342-neumann-gradient.md)
Uses a 3D regression hyperplane fitted to 26 neighbors to minimize error and produce isotropic gradients. Chosen for visual smoothness and normal accuracy over speed.

### [Forward Difference Gradient](3362-forward-difference-gradient.md)
First-order method using the current voxel and the next voxel in each direction. Chosen for minimal memory access and lower computational cost, despite directional bias.

### [4th-Order Finite Difference Gradient](4889-4th-order-finite-difference-gradient.md)
Uses a wider stencil with boundary extrapolation to maintain 4th-order accuracy for SDFs. Chosen when high-order smoothness is required for ray marching.

### [Gradient Selection](5889-gradient-selection.md)
Runtime strategy switching between Central Difference and Sobel based on configuration. Chosen to balance noise robustness and accuracy dynamically.

### [Gauss-Seidel with Successive Over-Relaxation (SOR)](6904-gauss-seidel-with-successive-over-relaxation-sor.md)
Iterative solver for the Laplace equation on the grid. Chosen for smoothing temperature/field data rather than direct gradient estimation.

### [Brick-local Gradient Estimation](8810-brick-local-gradient-estimation.md)
Central differences clamped to brick bounds with dynamic denominator adjustment. Chosen for sparse voxel grids to avoid cross-brick indirection.

### [Sobel-based Gradient Estimation](9967-sobel-based-gradient-estimation.md)
3x3x3 kernel with smoothing weights. Chosen for noise reduction and edge detection, at higher computational cost.

### [Central Difference Stencils](9968-central-difference-stencils.md)
Second-order accurate symmetric difference. Chosen as the standard baseline for accuracy when neighbors are available.

### WENO5 Interpolation
- **Definition**: Implementation of nominally fifth-order finite-difference WENO that returns the numerical flux for interpolation [S1].
- **Formula**: Given $v_1 = f(x-2dx)$, $v_2 = f(x-dx)$, $v_3 = f(x)$, $v_4 = f(x+dx)$ and $v_5 = f(x+2dx)$, return an interpolated value $f(x+dx/2)$ with the property that $( f(x+dx/2) - f(x-dx/2) ) / dx = df/dx (x) + error$, where the error is fifth-order in smooth regions: $O(dx) \le error \le O(dx^5)$ [S1].
- **Parameters**: `scale2` (scaling factor, default 1.0), `v1` through `v5` (input values) [S1].
- **History**: Introduced by Chi-Wang Shu, referenced in 'High Order Finite Difference and Finite Volume WENO Schemes and Discontinuous Galerkin Methods for CFD' (ICASE Report No 2001-11, page 6) and ICASE No 97-65 (Shu, 1997) [S1].

## Background
Finite difference methods are a cornerstone of numerical analysis for approximating derivatives of functions defined on discrete grids. In volume rendering, they provide the surface normals necessary for Phong shading, ambient occlusion, and opacity transfer functions. The family evolved from basic first-order approximations (forward/backward) to higher-order schemes (central, Sobel, WENO) to reduce truncation error and visual artifacts like "fringing" or "blockiness" in rendered volumes.

## Sources
[S1] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
