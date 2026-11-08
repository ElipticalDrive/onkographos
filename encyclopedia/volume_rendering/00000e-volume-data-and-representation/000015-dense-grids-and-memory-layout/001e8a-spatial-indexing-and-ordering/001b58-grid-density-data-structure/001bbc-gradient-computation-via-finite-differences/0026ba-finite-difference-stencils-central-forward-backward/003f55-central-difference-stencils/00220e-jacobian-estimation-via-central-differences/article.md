# Jacobian Estimation via Central Differences

Also called: Central Difference Jacobian, Finite Difference Jacobian

## Overview
This method estimates the 3x3 Jacobian matrix of a deformation field or coordinate transformation by computing central finite differences of the underlying function. It differs from scalar gradient computation by returning a full matrix representing local linear distortion rather than a single vector, and it differs from forward/backward differences by offering second-order accuracy with symmetric sampling [S1, S2, unofficial]. It is primarily used to map derivatives between computational and physical spaces in volume rendering and mesh deformation [S2, unofficial].

## How it works
**Inputs:**
- A position vector $\mathbf{p}$ (in the domain space, e.g., computational space).
- A function $f(\mathbf{x})$ that maps the domain to a codomain (e.g., physical space or a deformation vector field).
- A scalar step size $\epsilon$ (EPSILON).

**Outputs:**
- A $3 \times 3$ Jacobian matrix $\mathbf{J}$, where each column is the partial derivative of $f$ with respect to the corresponding domain axis.

**Data Structures:**
- The Jacobian is stored as a $3 \times 3$ matrix. In column-major layout (common in graphics APIs like OpenGL/GLM), the memory layout is:
  $$
  \mathbf{J} = \begin{bmatrix}
  \frac{\partial f_x}{\partial x} & \frac{\partial f_x}{\partial y} & \frac{\partial f_x}{\partial z} \\
  \frac{\partial f_y}{\partial x} & \frac{\partial f_y}{\partial y} & \frac{\partial f_y}{\partial z} \\
  \frac{\partial f_z}{\partial x} & \frac{\partial f_z}{\partial y} & \frac{\partial f_z}{\partial z}
  \end{bmatrix}
  $$
  where the columns are the gradient vectors $\mathbf{g}_x, \mathbf{g}_y, \mathbf{g}_z$.

**Steps:**
1. Define the perturbation vectors for each axis:
   $$ \mathbf{d}_x = (\epsilon, 0, 0), \quad \mathbf{d}_y = (0, \epsilon, 0), \quad \mathbf{d}_z = (0, 0, \epsilon) $$
2. Evaluate the function at the perturbed positions:
   $$ f_+^x = f(\mathbf{p} + \mathbf{d}_x), \quad f_-^x = f(\mathbf{p} - \mathbf{d}_x) $$
   $$ f_+^y = f(\mathbf{p} + \mathbf{d}_y), \quad f_-^y = f(\mathbf{p} - \mathbf{d}_y) $$
   $$ f_+^z = f(\mathbf{p} + \mathbf{d}_z), \quad f_-^z = f(\mathbf{p} - \mathbf{d}_z) $$
3. Compute the partial derivative vectors (columns of the Jacobian) using the central difference formula:
   $$ \mathbf{g}_x = \frac{f_+^x - f_-^x}{2\epsilon} $$
   $$ \mathbf{g}_y = \frac{f_+^y - f_-^y}{2\epsilon} $$
   $$ \mathbf{g}_z = \frac{f_+^z - f_-^z}{2\epsilon} $$
4. Construct the Jacobian matrix $\mathbf{J}$ using $\mathbf{g}_x, \mathbf{g}_y, \mathbf{g}_z$ as columns.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_jacobian_central_diff(f, p, epsilon):
    """
    Estimate the 3x3 Jacobian matrix of function f at point p using central differences.
    
    Args:
        f: Callable function taking a 3D vector and returning a 3D vector.
        p: 3D vector (x, y, z) at which to estimate the Jacobian.
        epsilon: Step size for finite differences.
        
    Returns:
        A 3x3 matrix (list of lists or similar structure) representing the Jacobian.
    """
    # Define perturbation vectors
    dx = (epsilon, 0.0, 0.0)
    dy = (0.0, epsilon, 0.0)
    dz = (0.0, 0.0, epsilon)
    
    # Evaluate function at perturbed points
    # Note: f must support vector addition and subtraction
    p_plus_x = add_vectors(p, dx)
    p_minus_x = subtract_vectors(p, dx)
    
    p_plus_y = add_vectors(p, dy)
    p_minus_y = subtract_vectors(p, dy)
    
    p_plus_z = add_vectors(p, dz)
    p_minus_z = subtract_vectors(p, dz)
    
    f_plus_x = f(p_plus_x)
    f_minus_x = f(p_minus_x)
    
    f_plus_y = f(p_plus_y)
    f_minus_y = f(p_minus_y)
    
    f_plus_z = f(p_plus_z)
    f_minus_z = f(p_minus_z)
    
    # Compute gradient vectors (columns of the Jacobian)
    # g_x = (f(p + dx) - f(p - dx)) / (2 * epsilon)
    diff_x = subtract_vectors(f_plus_x, f_minus_x)
    g_x = scale_vector(diff_x, 1.0 / (2.0 * epsilon))
    
    # g_y = (f(p + dy) - f(p - dy)) / (2 * epsilon)
    diff_y = subtract_vectors(f_plus_y, f_minus_y)
    g_y = scale_vector(diff_y, 1.0 / (2.0 * epsilon))
    
    # g_z = (f(p + dz) - f(p - dz)) / (2 * epsilon)
    diff_z = subtract_vectors(f_plus_z, f_minus_z)
    g_z = scale_vector(diff_z, 1.0 / (2.0 * epsilon))
    
    # Construct Jacobian matrix with g_x, g_y, g_z as columns
    # Matrix layout:
    # [ g_x[0] g_y[0] g_z[0] ]
    # [ g_x[1] g_y[1] g_z[1] ]
    # [ g_x[2] g_y[2] g_z[2] ]
    jacobian = [
        [g_x[0], g_y[0], g_z[0]],
        [g_x[1], g_y[1], g_z[1]],
        [g_x[2], g_y[2], g_z[2]]
    ]
    
    return jacobian

def add_vectors(v1, v2):
    return (v1[0] + v2[0], v1[1] + v2[1], v1[2] + v2[2])

def subtract_vectors(v1, v2):
    return (v1[0] - v2[0], v1[1] - v2[1], v1[2] - v2[2])

def scale_vector(v, s):
    return (v[0] * s, v[1] * s, v[2] * s)
```

**Parameters:**
- $\epsilon$ (EPSILON): The step size for finite differences. Typical values depend on the scale of the deformation field and floating-point precision. If $\epsilon$ is too large, truncation error increases; if too small, numerical noise dominates [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** If $\mathbf{p} \pm \mathbf{d}_i$ falls outside the valid domain of $f$, the function evaluation must handle this via clamping, zero-padding, or extrapolation [G].
- **Singularity:** If the deformation field is non-differentiable or discontinuous at $\mathbf{p}$, the central difference approximation may yield inaccurate or unstable results.
- **Precision:** For very small $\epsilon$, subtractive cancellation can occur in $f_+ - f_-$. For very large $\epsilon$, the linear approximation error grows.

## Subtopics

### [Jacobian Normalization for ODE Step](2946-jacobian-normalization-for-ode-step.md)
This method computes the normalized direction vector for an Ordinary Differential Equation (ODE) solver by applying the inverse of an estimated Jacobian matrix to a base direction vector. It differs from raw Jacobian estimation by performing the matrix inversion and vector normalization required for stable stepping [G].

### Jacobian Approximation via Central Differences
This variant approximates the Jacobian using central differences of adjacent vertices in computational and physical space [S2, unofficial]. It is specifically applied in the context of resampling scalar values between regular grids and curved meshes, where the Jacobian maps derivatives from computational space to physical space [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per evaluation, requiring 6 function evaluations and basic arithmetic operations.
- **Memory Complexity:** $O(1)$, storing only the perturbed points and the resulting $3 \times 3$ matrix.
- **Accuracy:** Second-order accurate ($O(\epsilon^2)$) for smooth functions, superior to forward/backward differences ($O(\epsilon)$) [S1, S2, unofficial].
- **Strengths:** Symmetric sampling reduces bias; simple to implement; no need for analytical derivatives.
- **Limitations:** Requires function evaluations; sensitive to $\epsilon$ choice; fails for non-differentiable functions; boundary handling is non-trivial.
- **When to Use:** When analytical derivatives are unavailable or too complex, and the function is smooth. Common in volume rendering for coordinate transformations and mesh deformation [S1, S2, unofficial].
- **When Not to Use:** For non-smooth functions, or when high performance is critical and analytical derivatives are available.

## Background
The method originates from the use of finite difference approximations in numerical analysis. In the context of volume rendering and graphics, it is used to estimate the Jacobian of deformation fields (e.g., in brush-based deformation) [S1, unofficial] and to map derivatives between computational and physical spaces in irregular grids [S2, unofficial]. The central difference scheme is a standard technique for approximating derivatives with higher accuracy than one-sided differences.

## Sources
[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S2] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
