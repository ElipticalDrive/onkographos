# Second Derivative Computation
Also called: Hessian computation, second-order finite differences, curvature and Laplacian grids

## Overview
Second Derivative Computation computes second-order spatial derivatives (the Hessian matrix or its scalar contractions) of a scalar field defined on a grid. It solves the problem of quantifying local curvature, boundary sharpness, and structural homogeneity, which are essential for regularization, surface extraction, and physical simulation. Unlike [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md), which yields the first-order vector field, this family computes the rate of change of that gradient. It differs from [Analytic Gradient Computation](2167-analytic-gradient-computation.md) by relying on discrete stencil approximations rather than closed-form derivatives of basis functions, and from [Gradient Estimation from Samples](2180-gradient-estimation-from-samples.md) by operating on structured grid data rather than unstructured point clouds.

## Shared principles
All methods in this family operate on a scalar field $f$ defined on a grid with coordinates $(x, y, z)$. The core mechanism is the application of finite difference stencils to approximate partial derivatives.

**Stencils:**
The second pure partial derivatives are typically computed using the central difference stencil with unit spacing ($\Delta x = \Delta y = \Delta z = 1$):
$$ f_{xx} = f(x+1, y, z) - 2f(x, y, z) + f(x-1, y, z) $$
$$ f_{yy} = f(x, y+1, z) - 2f(x, y, z) + f(x, y-1, z) $$
$$ f_{zz} = f(x, y, z+1) - 2f(x, y, z) + f(x, y, z-1) $$

Mixed partial derivatives (e.g., $f_{xy}$) use a 4-point stencil:
$$ f_{xy} = \frac{f(x+1, y+1, z) - f(x-1, y+1, z) - f(x+1, y-1, z) + f(x-1, y-1, z)}{4} $$

**Scaling:**
If the grid has non-unit voxel spacing $\Delta x, \Delta y, \Delta z$, the second derivatives must be scaled by the inverse square of the spacing:
$$ \frac{\partial^2 f}{\partial x^2} \approx \frac{f_{xx}}{\Delta x^2} $$

**Boundary Handling:**
Stencils require neighbors outside the active domain. Strategies include zero-padding (implicit background value), clamping, or substituting obstacle velocities for fluid boundaries [S2, G]. Inactive voxels are typically set to zero in the output.

**Hessian Matrix:**
The full Hessian $H$ is a symmetric $3 \times 3$ matrix:
$$ H = \begin{bmatrix} f_{xx} & f_{xy} & f_{xz} \\ f_{xy} & f_{yy} & f_{yz} \\ f_{xz} & f_{yz} & f_{zz} \end{bmatrix} $$

## Subtopics

### [Mean Curvature Grid Computation](3126-mean-curvature-grid-computation.md)
Computes the mean curvature $\kappa$ by evaluating the divergence of the normalized gradient. It calculates all required first and second derivatives in a single pass, avoiding intermediate grids. Choose this for surface smoothness analysis.

### [Divergence Grid Computation](3151-divergence-grid-computation.md)
Computes the divergence $\nabla \cdot \mathbf{F}$ of a vector field. It handles solid boundaries by substituting obstacle velocities. Choose this for fluid simulation and vector field analysis.

### [Laplacian Grid Computation](3160-laplacian-grid-computation.md)
Computes the Laplacian $\nabla^2 f$ as the trace of the Hessian. It highlights boundaries and suppresses homogeneous regions. Choose this for edge detection and diffusion-based smoothing.

### [Depth and Image Gradient Regularization](6439-depth-and-image-gradient-regularization.md)
Computes a regularization loss for 2D depth maps by weighting second-order depth derivatives with local image gradient magnitudes. Choose this for depth estimation tasks requiring edge-aware smoothness.

### [Eikonal Omega Term Computation](13240-eikonal-omega-term-computation.md)
Computes the gradient of the Eikonal constraint term with respect to hash grid features in neural implicit surfaces. It uses double backpropagation or second-order kernels to enforce unit-norm gradients. Choose this for training neural signed distance fields.

### [Second-order Grid Sampler](13559-second-order-grid-sampler.md)
Computes the Hessian of an SDF with respect to query coordinates and feature grid parameters analytically for tri-linear interpolation. It enables differentiable regularization without finite differences. Choose this for differentiable rendering and reconstruction pipelines.

### Laplacian Grid Computation via Finite Differences
This variant computes the Laplacian on 2D images using matrix operators. It applies forward differences along axes and appends zero rows/columns to maintain dimensions. The transpose of the derivative operator is applied to complete the second derivative. For weighted Laplacians, a gamma matrix is used to scale the derivatives before and after application. The operation is defined as $L = D_x^T D_x + D_y^T D_y$ [S1, unofficial].

## Background
The computation of second derivatives via finite differences is a foundational technique in numerical analysis and scientific computing. In the context of volume rendering and computer graphics, it is used for surface reconstruction, curvature-based shading, and regularization in inverse problems. The specific implementation details for mean curvature and Laplacian grids are found in sparse grid libraries and inverse problem solvers [S1, unofficial]. The use of second derivatives in neural implicit surfaces (Eikonal terms) is a more recent development in differentiable rendering [S1, unofficial].

## Sources
[S1] imanfs/inverse-problems-CT-scans - GitHub, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
