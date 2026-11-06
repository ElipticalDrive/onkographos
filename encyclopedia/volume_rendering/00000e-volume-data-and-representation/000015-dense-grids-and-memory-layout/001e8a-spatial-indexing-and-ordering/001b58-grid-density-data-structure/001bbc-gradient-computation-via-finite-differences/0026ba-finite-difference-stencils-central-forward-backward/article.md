# Finite Difference Stencils (Central/Forward/Backward)

Also called: Finite Difference Gradient, FD Gradient, Numerical Gradient

## Overview
Finite difference stencils compute the spatial gradient $\nabla V$ of a scalar volume field $V$ at each voxel by approximating partial derivatives using values from neighboring voxels [S1][S2, unofficial]. This method solves the problem of estimating surface orientation for lighting and shading in direct volume rendering when an analytic gradient is unavailable [S1]. It is distinguished from [Analytic Gradient Computation](2167-analytic-gradient-computation.md) by requiring additional sample evaluations (typically 6 for central differences) rather than deriving gradients from basis function coefficients, and from [Gradient Estimation from Samples](2180-gradient-estimation-from-samples.md) by operating on a structured grid rather than unstructured ray samples.

## Shared principles
All finite difference gradient methods approximate the partial derivative $\frac{\partial V}{\partial x_i}$ at a voxel center using a linear combination of scalar values $V$ at discrete grid points. The general form for a stencil is:
$$ \frac{\partial V}{\partial x_i} \approx \frac{1}{h} \sum_{k} c_k V(\mathbf{p} + \delta_k) $$
where $h$ is the voxel spacing (assumed isotropic unless specified), $c_k$ are integer weights, and $\delta_k$ are integer offsets from the current voxel index. The gradient vector $\mathbf{G} = (G_x, G_y, G_z)$ is constructed by computing these approximations for each axis independently. Boundary conditions must be handled for voxels where neighbors fall outside the volume bounds, typically via clamping, zero-padding, or extrapolation [G].

## Subtopics

### [Sobel Operator](1416-sobel-operator.md)
Uses separable smoothing kernels for noise reduction.

### [Jacobian Estimation via Central Differences](2650-jacobian-estimation-via-central-differences.md)
Extends central differences to vector-valued functions for 3x3 Jacobian matrices.

### [Forward Difference Gradient](3362-forward-difference-gradient.md)
First-order, biased method using only positive neighbors.

### [4th-Order Finite Difference Gradient](4889-4th-order-finite-difference-gradient.md)
High-accuracy method using larger stencils and boundary extrapolation.

### [Gauss-Seidel with Successive Over-Relaxation (SOR)](6904-gauss-seidel-with-successive-over-relaxation-sor.md)
Iterative solver for Laplace equation, not a direct gradient estimator.

### Clamped Central Differences
- **Definition**: A variant of central differencing that clamps offset positions to the current region to avoid additional ray traversals [S1].
- **Step**: Compute six additional basis reconstruction samples at positions offset from the current sample by a distance proportional to the region's sample rate. Clamp the offset positions to the current region before evaluating them [S1].
- **Pitfall**: Gradients are not necessarily continuous at level boundaries because the offset size can change [S1].
- **Comparison**: Accurate central differencing requires tracing 6 additional rays per sample via infinitesimal rays to find containing regions, which is expensive. Clamped version removes ray traversals but reduces quality [S1].

### Curl Grid Computation
- **Definition**: Computes the curl $\nabla \times \mathbf{F}$ at every active voxel of a vector grid using central differences [S2, unofficial].
- **Step**: Initialize a `GradStencil` with the grid tree. Iterate over all active voxels $(c, \_)$ in the grid tree. Move the stencil to the current coordinate $c$. Compute the x, y, and z components of the curl using the formulas and store the tuple $(c_x, c_y, c_z)$ in a dictionary keyed by coordinate. Build the result grid using the computed data, a background value of zero vector $(0,0,0)$, and the original grid's voxel size [S2, unofficial].
- **Formula**:
  $$ c_x = (v[4][3] - v[5][3] - v[6][2] + v[7][2]) \cdot 0.5 $$
  $$ c_y = (v[6][1] - v[7][1] - v[2][3] + v[3][3]) \cdot 0.5 $$
  $$ c_z = (v[2][2] - v[3][2] - v[4][1] + v[5][1]) \cdot 0.5 $$
- **Parameter**: `half`: The scalar multiplier 0.5, representing $1/(2 \cdot \text{voxel\_size})$ in index space [S2, unofficial].

### WENO5 Interpolation
- **Definition**: Implementation of nominally fifth-order finite-difference WENO that returns the numerical flux for interpolation [S3].
- **Formula**: Given $v_1 = f(x-2dx)$, $v_2 = f(x-dx)$, $v_3 = f(x)$, $v_4 = f(x+dx)$ and $v_5 = f(x+2dx)$, return an interpolated value $f(x+dx/2)$ with the special property that $(f(x+dx/2) - f(x-dx/2)) / dx = df/dx(x) + \text{error}$, where the error is fifth-order in smooth regions: $O(dx) \le \text{error} \le O(dx^5)$ [S3].
- **Parameter**: `scale2`: A scaling factor for the weights, default value is 1.0 [S3]. `v1` through `v5`: Five input values for the stencil [S3].
- **History**: Introduced by Chi-Wang Shu. Referenced in 'High Order Finite Difference and Finite Volume WENO Schemes and Discontinuous Galerkin Methods for CFD' (ICASE Report No 2001-11, page 6) and ICASE No 97-65 (Shu, 1997) [S3].

### Zucker-Hummel Operator
- **Definition**: Advanced gradient operators that use 54 voxels to produce softer and clearer gradients compared to the Central-Difference operator [S4, unofficial].
- **Step**: 1. Apply operator masks in X, Y, and Z planes at positions one voxel before, at, and one voxel after the sampling point. 2. Multiply voxel values by mask coefficients. 3. Sum the results for each plane to get gradient components [S4, unofficial].
- **Comparison**: Unlike the Central-Difference operator which uses 6 voxels, Zucker-Hummel and Sobel 3D use 54 voxels, providing a better description of local surface structure [S4, unofficial]. More complex calculation than Central-Difference, requiring 54 sampling points, but offers significantly improved image quality [S4, unofficial].

### Hessian-Vector Product via Central Differences
- **Definition**: An approximation of the Hessian-vector product $H(\theta)\mathbf{v}$ that avoids computing the full Hessian matrix [S5]. It estimates the directional gradient of the gradient, resulting in an n-dimensional vector [S5].
- **Step**: Evaluate the first-order gradient estimator at positions shifted from the current solution along the direction $\mathbf{v}$ by $\pm \epsilon$ [S5].
- **Formula**: Approximation using central differences: $H(\theta)\mathbf{v} \approx \lim_{\epsilon \to 0} \frac{\mathbf{g}(\theta + \epsilon \mathbf{v}) - \mathbf{g}(\theta - \epsilon \mathbf{v})}{2\epsilon}$ [S5]. The smooth HVP kernel is estimated using directional central differences of the gradient estimator: $D_{HVP} \kappa(\tau) = \frac{D_G \kappa(\tau - \epsilon \mathbf{v}) - D_G \kappa(\tau + \epsilon \mathbf{v})}{2\epsilon}$ [S5].
- **Complexity**: Time complexity: $O(n)$ function evaluations without IS, $O(n)$ with IS, $O(1)$ with Aggregate IS. Space complexity for Aggregate IS is $O(n)$ [S5].

## Background
Finite difference methods are a cornerstone of numerical analysis for approximating derivatives of functions defined on discrete grids. In the context of volume rendering, they were adopted to provide surface normals for Phong shading and other lighting models when the volume data was represented as a dense grid of scalar values without explicit surface extraction [S1]. The central difference scheme, being second-order accurate and unbiased, became the standard default due to its balance of accuracy and simplicity [S1]. Variants like forward/backward differences are used when boundary conditions or directional bias are acceptable, while higher-order schemes (like 4th-order or WENO) are employed for high-fidelity applications such as SDF rendering or fluid simulation [S3][S4, unofficial]. The method is widely used in scientific visualization, medical imaging, and physics-based rendering pipelines.

## Sources
[S1] PDF Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
[S2] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S3] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
[S4] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S5] Stochastic Gradient Estimation for Higher-Order Differentiable Rendering, https://doi.org/10.1109/iccv51701.2025.02618, fetched 2026-10-29
