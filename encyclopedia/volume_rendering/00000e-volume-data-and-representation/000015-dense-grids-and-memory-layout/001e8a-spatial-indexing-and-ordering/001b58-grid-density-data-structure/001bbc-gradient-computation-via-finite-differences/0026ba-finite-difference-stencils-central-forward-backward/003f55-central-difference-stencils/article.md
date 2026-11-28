# Central Difference Stencils


## Overview
Central difference stencils approximate the partial derivative of a scalar field at a voxel center by computing the difference between field values at symmetric neighbors, divided by twice the step size [G]. This method is second-order accurate, offering higher precision than forward or backward differences for smooth fields, at the cost of requiring samples from both sides of the evaluation point [G]. In volume rendering, it is employed to estimate surface normals or density gradients from discrete voxel data when analytic derivatives are not available [S1].

## Shared principles
See [Finite Difference Stencils (Central/Forward/Backward)](3858-finite-difference-stencils-central-forward-backward.md) for the shared mechanism, terms, and math [G]. All methods in this family approximate the partial derivative $\frac{\partial V}{\partial x_i}$ at a voxel center $\mathbf{p}$ using a linear combination of scalar field values $V(\mathbf{p}_k)$ at neighboring positions $\mathbf{p}_k$ [G]. The general form is $G_i = \sum_{k} w_k V(\mathbf{p}_k)$, where for central differences the weights are $w_k = \pm 1/(2h)$ and $h$ is the voxel spacing [G]. Boundary handling is required when stencil neighbors fall outside the volume domain; common strategies include clamping, zero-padding, or extrapolation [G]. The core gradient computation for an isotropic grid is implemented as follows:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_central_gradient(grid: Grid, p: Vector3, h: float) -> Vector3:
    """
    Computes the gradient of a scalar field at point p using central differences.
    Assumes isotropic grid with spacing h. Boundary handling uses clamping [G].
    """
    # Offsets for central difference
    dx = Vector3(h, 0.0, 0.0)
    dy = Vector3(0.0, h, 0.0)
    dz = Vector3(0.0, 0.0, h)
    
    # Evaluate field at neighbors
    v_plus_x = grid.sample(p + dx)
    v_minus_x = grid.sample(p - dx)
    v_plus_y = grid.sample(p + dy)
    v_minus_y = grid.sample(p - dy)
    v_plus_z = grid.sample(p + dz)
    v_minus_z = grid.sample(p - dz)
    
    # Compute partial derivatives
    gx = (v_plus_x - v_minus_x) / (2.0 * h)
    gy = (v_plus_y - v_minus_y) / (2.0 * h)
    gz = (v_plus_z - v_minus_z) / (2.0 * h)
    
    return Vector3(gx, gy, gz)
```

## Subtopics

### [Jacobian Estimation via Central Differences](2650-jacobian-estimation-via-central-differences.md)
This method estimates the 3x3 Jacobian matrix of a vector field or deformation map by computing central finite differences of the field's components. It builds on [Central Difference Stencils](9968-central-difference-stencils.md) to provide the local linear transformation required for coordinate mapping, ray marching, or flow analysis. Unlike forward or backward differences, the central difference approach is second-order accurate, offering higher precision for smooth fields at the cost of requiring samples from both sides of the evaluation point. See [Jacobian Estimation via Central Differences](2650-jacobian-estimation-via-central-differences.md).

### Clamped Central Differences
**Definition**
A variant of central differencing for gradient computation that clamps offset positions to the current region to avoid additional ray traversals [S1].

**Steps**
1. Compute six additional basis reconstruction samples at positions offset from the current sample by a distance proportional to the region's sample rate [S1].
2. Clamp the offset positions to the current region before evaluating them [S1].

**Pitfalls**
Gradients are not necessarily continuous at level boundaries because the offset size can change [S1].

**Comparison**
Accurate central differencing requires tracing 6 additional rays per sample via infinitesimal rays to find containing regions, which is expensive [S1]. The clamped version removes ray traversals but reduces quality [S1].

### Curl Grid Computation
**Definition**
Computes the curl $\nabla \times \mathbf{F}$ at every active voxel of a vector grid using central differences [S2, unofficial].

**Steps**
1. Initialize a `GradStencil` with the grid tree [S2, unofficial].
2. Iterate over all active voxels $(c, \_)$ in the grid tree [S2, unofficial].
3. Move the stencil to the current coordinate $c$ [S2, unofficial].
4. Compute the x, y, and z components of the curl using the formulas and store the tuple $(c_x, c_y, c_z)$ in a dictionary keyed by coordinate [S2, unofficial].
5. Build the result grid using the computed data, a background value of zero vector $(0,0,0)$, and the original grid's voxel size [S2, unofficial].

**Formulas**
$$ c_x = (v[4][3] - v[5][3] - v[6][2] + v[7][2]) \times 0.5 $$
$$ c_y = (v[6][1] - v[7][1] - v[2][3] + v[3][3]) \times 0.5 $$
$$ c_z = (v[2][2] - v[3][2] - v[4][1] + v[5][1]) \times 0.5 $$
where $v$ represents the stencil values and $0.5$ is the scalar multiplier representing $1/(2 \times \text{voxel\_size})$ in index space [S2, unofficial].

### Hessian-Vector Product via Central Differences
**Definition**
An approximation of the Hessian-vector product $H(\theta)\mathbf{v}$ that avoids computing the full Hessian matrix [S3]. It estimates the directional gradient of the gradient, resulting in an $n$-dimensional vector [S3].

**Steps**
Evaluate the first-order gradient estimator at positions shifted from the current solution along the direction $\mathbf{v}$ by $\pm \epsilon$ [S3].

**Formulas**
$$ H(\theta)\mathbf{v} \approx \lim_{\epsilon \to 0} \frac{g(\theta + \epsilon \mathbf{v}) - g(\theta - \epsilon \mathbf{v})}{2\epsilon} $$
The smooth HVP kernel is estimated using directional central differences of the gradient estimator:
$$ D_{HVP} \kappa(\tau) = \frac{D_G \kappa(\tau - \epsilon \mathbf{v}) - D_G \kappa(\tau + \epsilon \mathbf{v})}{2\epsilon} $$
[S3]

**Complexity**
Time complexity is $O(n)$ function evaluations without importance sampling (IS), $O(n)$ with IS, and $O(1)$ with Aggregate IS [S3]. Space complexity for Aggregate IS is $O(n)$ [S3].

## Background
Central difference stencils are a fundamental numerical analysis technique widely used in scientific computing and computer graphics [G]. In the context of volume rendering, they are employed to estimate surface normals or density gradients from discrete voxel data when analytic derivatives are not available [S1]. The method is compared against analytic gradient approaches for basis reconstruction in structured AMR data ray tracing [S1]. It is also used in differentiable rendering for estimating higher-order derivatives like the Hessian-vector product, where it serves as a baseline for stochastic gradient estimation [S3]. The technique is implemented in research codebases for tasks such as curl grid computation [S2, unofficial].

## Sources
[S1] PDF Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
[S2] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S3] Stochastic Gradient Estimation for Higher-Order Differentiable Rendering, https://doi.org/10.1109/iccv51701.2025.02618, fetched 2026-10-29
