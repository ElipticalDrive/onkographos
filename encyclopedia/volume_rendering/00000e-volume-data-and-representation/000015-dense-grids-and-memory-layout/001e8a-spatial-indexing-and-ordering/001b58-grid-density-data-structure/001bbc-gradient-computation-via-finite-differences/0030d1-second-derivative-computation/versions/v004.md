# Second Derivative Computation

## Overview
This family computes second-order spatial derivatives (pure, mixed, Laplacian, divergence, and curvature) from discrete scalar or vector field grids using finite difference stencils [G]. It solves the problem of extracting local geometric properties (curvature, boundary strength, compressibility) from volumetric or image data where analytic derivatives are unavailable [G]. Unlike [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md) which yields first-order vectors, this family yields scalar fields (or flattened arrays of scalar values) that highlight boundaries, suppress homogeneous regions, or enforce smoothness constraints in optimization [G].

## Shared principles
All methods in this family operate on a discrete grid with active voxels/pixels and a defined topology [G]. They rely on a `Stencil` mechanism to access neighbor values at integer offsets $(dx, dy, dz)$ relative to a center coordinate $c$ [G].
1. **Stencil Access**: For a center $c=(x,y,z)$, values are retrieved via `value_at(c + offset)`. Offsets are typically $\pm 1$ along axes for pure derivatives, or combinations like $(\pm 1, \pm 1, 0)$ for mixed derivatives [G].
2. **Central Differences**: The primary approximation for second derivatives uses the 3-point central difference formula. For unit spacing $\Delta x = 1$:
   - Pure second derivative: $f_{xx} \approx f(x+1) - 2f(x) + f(x-1)$ [G].
   - Mixed second derivative: $f_{xy} \approx \frac{1}{4}[f(x+1,y+1) - f(x-1,y+1) - f(x+1,y-1) + f(x-1,y-1)]$ [G].
3. **Boundary Handling**: Neighbors outside the active domain or grid boundaries require specific handling. Common strategies include:
   - **Clamping**: Repeating the boundary value [G].
   - **Zero-padding**: Assuming zero values outside [G].
   - **Obstacle Substitution**: In fluid contexts, replacing inactive neighbor velocities with obstacle velocities, as detailed in [Divergence Grid Computation](3151-divergence-grid-computation.md).
   - **Exclusion**: Skipping boundary pixels in image regularization, as detailed in [Depth and Image Gradient Regularization](6439-depth-and-image-gradient-regularization.md).
4. **Normalization**: If physical voxel spacing $\Delta x, \Delta y, \Delta z$ is not unity, the computed differences must be divided by the appropriate squared spacing terms (e.g., $\Delta x^2$) to maintain dimensional correctness [G].
5. **Singularity Avoidance**: Methods involving division by gradient magnitude (e.g., mean curvature) must check for near-zero gradients to prevent division by zero, typically using machine epsilon `eps(T)`, as detailed in [Mean Curvature Grid Computation](3126-mean-curvature-grid-computation.md).

## Subtopics

### [Mean Curvature Grid Computation](3126-mean-curvature-grid-computation.md)
Computes the mean curvature $\kappa$ of a scalar field $f$ by evaluating the divergence of the normalized gradient. It calculates all required first and second derivatives in a single pass, avoiding intermediate grid storage. Chosen when surface geometry analysis is required.

### [Divergence Grid Computation](3151-divergence-grid-computation.md)
Computes the divergence $\nabla \cdot \mathbf{V}$ of a vector field $\mathbf{V}$ using central differences on face neighbors. It explicitly handles solid boundaries by substituting obstacle velocities. Chosen for fluid simulation compressibility checks or vector field analysis.

### [Laplacian Grid Computation](3160-laplacian-grid-computation.md)
Computes the Laplacian $\nabla^2 f$ (trace of the Hessian) using a 6-point stencil on scalar fields. It highlights boundaries and suppresses homogeneous interiors. Chosen for edge detection, noise reduction, or as a regularizer in inverse problems.

### [Depth and Image Gradient Regularization](6439-depth-and-image-gradient-regularization.md)
Applies a spatial regularization loss to depth maps by penalizing second-order derivatives weighted by local image gradient magnitudes. It enforces smoothness in textureless regions while preserving discontinuities at object boundaries. Chosen for depth map optimization in computer vision.

### Laplacian Grid Computation via Finite Differences
A matrix-based implementation of the Laplacian using forward and backward difference operators in 2D. It constructs the Laplacian matrix by applying derivative operators and their transposes. It supports optional gamma weighting for anisotropic smoothing.
- **Forward Derivative (Rows)**: `Dx` is `np.diff(u, axis=0)` with a zero row appended at the bottom.
- **Forward Derivative (Cols)**: `Dy` is `np.diff(u, axis=1)` with a zero column appended at the right.
- **Transpose Derivative (Rows)**: `DxT` is constructed by prepending `-u[0:1, :]`, inserting `-np.diff(u[:-1, :], axis=0)`, and appending `u[-2:-1, :]`.
- **Transpose Derivative (Cols)**: `DyT` is constructed by prepending `-u[:, 0:1]`, inserting `-np.diff(u[:, :-1], axis=1)`, and appending `u[:, -2:-1]`.
- **Unweighted Laplacian**: `dxx = DxT @ Dx(f)`, `dyy = DyT @ Dy(f)`, result is `dxx + dyy`.
- **Weighted Laplacian**: If `gamma` is provided, `dxx = sqrt(gamma) @ DxT @ sqrt(gamma) @ Dx(f)`, similarly for `dyy`. If `gamma` is scalar, it becomes an identity matrix.
[S1, unofficial]

## Background
The use of finite differences for second-order derivatives originates from numerical analysis methods for solving partial differential equations (PDEs) such as the Poisson and Laplace equations. In computer graphics and volume rendering, these operators are adapted for discrete voxel grids to extract geometric features like curvature for isosurface rendering or to regularize depth maps in photometric stereo and structure-from-motion. The specific matrix formulations for 2D Laplacians are often found in inverse problem solvers for computed tomography (CT) and image reconstruction [S1, unofficial].

## Sources
[S1] imanfs/inverse-problems-CT-scans, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
