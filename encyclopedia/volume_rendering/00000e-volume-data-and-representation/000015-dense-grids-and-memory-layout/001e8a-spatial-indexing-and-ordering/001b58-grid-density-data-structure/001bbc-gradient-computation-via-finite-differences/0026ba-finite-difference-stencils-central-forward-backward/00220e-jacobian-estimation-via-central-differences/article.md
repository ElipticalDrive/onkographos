# Jacobian Estimation via Central Differences

Also called: Central Difference Jacobian, Finite Difference Jacobian

## Overview
This method estimates the $3 \times 3$ Jacobian matrix of a vector-valued deformation field by computing central differences of the field's component functions. It differs from sibling methods like the [Sobel Operator](1416-sobel-operator.md) or [Forward Difference Gradient](3362-forward-difference-gradient.md) by providing a full matrix of partial derivatives rather than just a gradient vector, and by using symmetric sampling which yields second-order accuracy compared to the first-order accuracy of forward/backward differences. It builds on [Finite Difference Stencils (Central/Forward/Backward)](3858-finite-difference-stencils-central-forward-backward.md).

## How it works
**Inputs and Outputs:**
- Input: A position vector $\mathbf{p} = (x, y, z)$ and a deformation field function $f(\mathbf{p})$ returning a 3D vector.
- Output: A $3 \times 3$ Jacobian matrix $\mathbf{J}$.

**Data Structures:**
- The Jacobian $\mathbf{J}$ is a $3 \times 3$ matrix where the $i$-th column is the gradient of the $i$-th component of the deformation field with respect to the spatial coordinates.
- $\mathbf{J} = [\mathbf{g}_x, \mathbf{g}_y, \mathbf{g}_z]$, where $\mathbf{g}_x = \frac{\partial f}{\partial x}$, etc.

**Steps:**
1. Define a small step size $\epsilon$ (EPSILON).
2. Compute the perturbation vectors:
   - $\mathbf{d}_x = (\epsilon, 0, 0)$
   - $\mathbf{d}_y = (0, \epsilon, 0)$
   - $\mathbf{d}_z = (0, 0, \epsilon)$
3. Evaluate the deformation field at perturbed positions:
   - $\mathbf{v}_{x+} = f(\mathbf{p} + \mathbf{d}_x)$
   - $\mathbf{v}_{x-} = f(\mathbf{p} - \mathbf{d}_x)$
   - $\mathbf{v}_{y+} = f(\mathbf{p} + \mathbf{d}_y)$
   - $\mathbf{v}_{y-} = f(\mathbf{p} - \mathbf{d}_y)$
   - $\mathbf{v}_{z+} = f(\mathbf{p} + \mathbf{d}_z)$
   - $\mathbf{v}_{z-} = f(\mathbf{p} - \mathbf{d}_z)$
4. Compute the gradient columns using central differences:
   - $\mathbf{g}_x = \frac{\mathbf{v}_{x+} - \mathbf{v}_{x-}}{2\epsilon}$
   - $\mathbf{g}_y = \frac{\mathbf{v}_{y+} - \mathbf{v}_{y-}}{2\epsilon}$
   - $\mathbf{g}_z = \frac{\mathbf{v}_{z+} - \mathbf{v}_{z-}}{2\epsilon}$
5. Construct the matrix $\mathbf{J}$ using $\mathbf{g}_x, \mathbf{g}_y, \mathbf{g}_z$ as columns.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_jacobian_central(f, p, epsilon):
    """
    Estimate the 3x3 Jacobian matrix of vector field f at position p.
    
    Args:
        f: Function taking a 3D vector and returning a 3D vector.
        p: 3D position vector (x, y, z).
        epsilon: Step size for finite differences.
        
    Returns:
        J: 3x3 Jacobian matrix (list of lists or 2D array).
    """
    # Define perturbation vectors
    dx = [epsilon, 0.0, 0.0]
    dy = [0.0, epsilon, 0.0]
    dz = [0.0, 0.0, epsilon]
    
    # Evaluate field at perturbed positions
    p_plus_x = [p[0] + dx[0], p[1] + dx[1], p[2] + dx[2]]
    p_minus_x = [p[0] - dx[0], p[1] - dx[1], p[2] - dx[2]]
    p_plus_y = [p[0] + dy[0], p[1] + dy[1], p[2] + dy[2]]
    p_minus_y = [p[0] - dy[0], p[1] - dy[1], p[2] - dy[2]]
    p_plus_z = [p[0] + dz[0], p[1] + dz[1], p[2] + dz[2]]
    p_minus_z = [p[0] - dz[0], p[1] - dz[1], p[2] - dz[2]]
    
    v_plus_x = f(p_plus_x)
    v_minus_x = f(p_minus_x)
    v_plus_y = f(p_plus_y)
    v_minus_y = f(p_minus_y)
    v_plus_z = f(p_plus_z)
    v_minus_z = f(p_minus_z)
    
    # Compute gradient columns
    # g_x = (f(p + dx) - f(p - dx)) / (2 * epsilon)
    g_x = [(v_plus_x[i] - v_minus_x[i]) / (2.0 * epsilon) for i in range(3)]
    
    # g_y = (f(p + dy) - f(p - dy)) / (2 * epsilon)
    g_y = [(v_plus_y[i] - v_minus_y[i]) / (2.0 * epsilon) for i in range(3)]
    
    # g_z = (f(p + dz) - f(p - dz)) / (2 * epsilon)
    g_z = [(v_plus_z[i] - v_minus_z[i]) / (2.0 * epsilon) for i in range(3)]
    
    # Construct Jacobian matrix with columns g_x, g_y, g_z
    J = [g_x, g_y, g_z]
    
    return J
```

**Parameters:**
- $\epsilon$ (EPSILON): The step size for finite differences. Typical values depend on the scale of the deformation field and the precision of the floating-point representation. [S1, unofficial]

**Edge Cases and Pitfalls:**
- If $\epsilon$ is too small, numerical cancellation errors may dominate.
- If $\epsilon$ is too large, the approximation error (truncation error) increases.
- Boundary conditions: If $\mathbf{p} \pm \mathbf{d}_i$ falls outside the domain of $f$, the behavior is undefined unless $f$ handles out-of-bounds queries (e.g., via clamping or extrapolation) [G].

## Subtopics

### [Jacobian Normalization for ODE Step](2946-jacobian-normalization-for-ode-step.md)
This method computes the normalized direction vector for an Ordinary Differential Equation (ODE) solver by applying the inverse of an estimated Jacobian matrix to a base direction vector. It differs from raw Jacobian estimation by performing the matrix inversion and vector normalization required for stable stepping [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ per evaluation, requiring 6 evaluations of the deformation field $f$.
- **Memory Complexity:** $O(1)$, storing only a few vectors and the resulting matrix.
- **Accuracy:** Second-order accurate ($O(\epsilon^2)$) for smooth fields.
- **Strengths:** Simple to implement, symmetric sampling reduces bias.
- **Limitations:** Requires the deformation field to be differentiable. Sensitive to the choice of $\epsilon$. Not suitable for non-differentiable or noisy fields without smoothing.
- **When to use:** When the full Jacobian matrix is needed for tasks like inverse deformation, ray marching step size adaptation, or normal estimation from implicit surfaces.
- **When not to use:** When only the gradient vector is needed (use [Forward Difference Gradient](3362-forward-difference-gradient.md) or [Sobel Operator](1416-sobel-operator.md) for efficiency), or when the field is not differentiable.

## Background
This method is a standard application of central finite differences to vector fields. It is commonly used in computer graphics for estimating normals from implicit surfaces, for deformation field analysis, and in physics-based simulations. The specific implementation cited here is from the `ntsl-graphics` project, which uses it for deformation field estimation in the context of marching cubes and ray marching [S1, unofficial].

## Sources
[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
