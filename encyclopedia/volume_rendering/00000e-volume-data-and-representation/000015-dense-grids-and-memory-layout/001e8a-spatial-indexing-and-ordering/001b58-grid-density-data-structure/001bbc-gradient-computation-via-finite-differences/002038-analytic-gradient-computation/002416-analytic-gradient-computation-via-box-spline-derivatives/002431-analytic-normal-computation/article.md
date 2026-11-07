# Analytic Normal Computation

Also called: Exact Gradient for Spline

## Overview
Analytic Normal Computation computes surface normal vectors for volume rendering by deriving the gradient directly from the underlying spline basis function, rather than approximating it via finite differences. It solves the problem of normal inaccuracy inherent in numerical differencing, where the choice of offset step size creates a trade-off between rounding errors (small offsets) and approximation errors (large offsets). It is distinguished from its parent method, Analytic Gradient Computation via Box-Spline Derivatives, by specifically focusing on the normal vector derivation for isosurface rendering, ensuring consistent accuracy regardless of zoom level or viewing angle.

## How it works
The method relies on the mathematical property that the derivative of a box-spline is another box-spline (or a linear combination thereof). By evaluating this analytic derivative at the ray intersection point, the gradient $\nabla f$ is obtained exactly. The normal vector $\mathbf{n}$ is then normalized:

$$ \mathbf{n} = \frac{\nabla f}{\|\nabla f\|} $$

**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Current intersection point $\mathbf{p}$ on the isosurface.
- The box-spline coefficients defining the volume data.
- The specific box-spline basis function definition (e.g., FCC box-spline).

**Outputs:**
- Normal vector $\mathbf{n}$ at $\mathbf{p}$.

**Data Structures:**
- The voxel grid stores scalar values. The analytic computation requires accessing the same set of control points (samples) as the scalar value interpolation but applies different weights corresponding to the derivative of the basis functions.

**Steps:**
1. Identify the voxel containing the intersection point $\mathbf{p}$.
2. Retrieve the relevant box-spline control points (coefficients) surrounding $\mathbf{p}$.
3. Evaluate the analytic derivative of the box-spline basis function at $\mathbf{p}$. This involves computing the directional derivatives of the spline basis functions.
4. Sum the contributions of the control points weighted by the analytic derivative values to form the gradient vector $\nabla f$.
5. Normalize $\nabla f$ to obtain the unit normal $\mathbf{n}$.

**Parameters:**
- None. The method is exact and does not rely on a step size $\delta$.

**Edge Cases and Pitfalls:**
- **Rounding-off error in finite differences:** If using finite differences, offsets $\delta$ that are too small lead to precision loss due to floating-point rounding.
- **Approximation error in finite differences:** Offsets $\delta$ that are too large lead to inaccurate normals because the linear approximation deviates from the true curve.
- **Zoom level:** While analytic normals remain accurate, the rendering quality may decrease when zoomed in if the step size for ray marching is fixed. This is a rendering artifact, not a normal computation error, and can be mitigated by using adaptive stepsizes based on viewing angle and distance from the camera [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_analytic_normal(p, coefficients, basis_function):
    """
    Computes the normal vector at point p using analytic derivatives.
    
    Args:
        p: Intersection point (vec3).
        coefficients: Array of box-spline control points.
        basis_function: The box-spline basis function object/method.
        
    Returns:
        n: Normalized normal vector (vec3).
    """
    # 1. Determine the local voxel and relative position
    # (Implementation details depend on the specific grid layout, e.g., FCC)
    local_coords = get_local_coords(p)
    
    # 2. Evaluate the analytic gradient
    # The gradient is the sum of coefficients multiplied by the derivative of the basis function
    # dN/dx, dN/dy, dN/dz are the partial derivatives of the box-spline basis function N
    grad_x = 0.0
    grad_y = 0.0
    grad_z = 0.0
    
    for i in range(num_control_points):
        coeff = coefficients[i]
        # Evaluate the derivative of the basis function at local_coords
        # This is the core analytic step, replacing finite difference approximation
        dN_dx = basis_function.derivative_x(local_coords, i)
        dN_dy = basis_function.derivative_y(local_coords, i)
        dN_dz = basis_function.derivative_z(local_coords, i)
        
        grad_x += coeff * dN_dx
        grad_y += coeff * dN_dy
        grad_z += coeff * dN_dz
        
    gradient = vec3(grad_x, grad_y, grad_z)
    
    # 3. Normalize
    length = sqrt(gradient.x**2 + gradient.y**2 + gradient.z**2)
    if length == 0:
        return vec3(0, 0, 1) # Default normal if gradient is zero
    return gradient / length
```

## Subtopics

### [Analytic Normal Computation for SDF Grids](9645-analytic-normal-computation-for-sdf-grids.md)
Computes the surface normal $\mathbf{n}$ for a differentiable implicit function $f$ defined by a cubic polynomial within a voxel, using analytic derivatives instead of numerical differencing to ensure $C^2$ continuity inside the voxel.

## Operating characteristics
- **Accuracy:** The analytic formula always results in accurate normal values, avoiding the pitfalls of offset selection inherent in finite difference methods [S1].
- **Performance:** The computation overhead is lower than finite differences because the data fetch overhead is reduced. The analytic method requires fetching the same samples as the scalar value but avoids the additional fetches required for finite difference offsets [S1].
- **Measured Performance:**
    - MRI-Head: 65.2 fps (analytic) vs 51.7 fps (finite difference), 1.3x speed-up.
    - VisMale: 68.9 fps (analytic) vs 55.1 fps (finite difference), 1.3x speed-up.
    - CT-Head: 54.6 fps (analytic) vs 45.1 fps (finite difference), 1.2x speed-up.
    - Carp: 22.3 fps (analytic) vs 22.2 fps (finite difference), 1.0x speed-up [S1].
- **Strengths:** Consistent accuracy across all zoom levels and viewing angles; lower data fetch overhead.
- **Limitations:** Requires the ability to evaluate analytic derivatives of the basis function; rendering quality may still degrade when zoomed in if ray marching step size is not adaptive [S1].

## Background
This method is described in the context of GPU isosurface raycasting of FCC (Face-Centered Cubic) datasets. It is part of a pipeline that includes quasi-interpolation prefiltering and min/max cell construction. The approach leverages the properties of box-splines to provide exact gradients, improving the visual quality of isosurface images compared to finite difference approximations.

## Sources
[S1] GPU Isosurface Raycasting of FCC Datasets, https://minho-kim-uos.github.io/papers/kim12gpu.pdf, fetched 2026-10-29
