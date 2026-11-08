# Jacobian Estimation via Central Differences

Also called: Central Difference Jacobian, Finite Difference Jacobian

## Overview
This method estimates the 3x3 Jacobian matrix of a vector-valued deformation field (or displacement field) by computing central finite differences of the field's components. It solves the problem of determining how a local coordinate transformation distorts space (stretching, shearing, rotating) at a specific point, which is essential for tasks like grid traversal in physical space or normal estimation in implicit surfaces. It differs from its parent, [Central Difference Stencils](9968-central-difference-stencils.md), by operating on a vector field $\mathbf{u}(\mathbf{x})$ to produce a matrix rather than a scalar field to produce a vector, and from [Jacobian Normalization for ODE Step](2946-jacobian-normalization-for-ode-step.md) by providing the raw matrix without subsequent inversion or normalization.

## How it works
**Inputs and Outputs:**
- Input: A vector-valued function $\mathbf{u}(\mathbf{x}) = (u_x, u_y, u_z)$ representing a deformation or displacement field, where $\mathbf{x} = (x, y, z)$ is the position in computational or physical space.
- Input: A step size $\epsilon$ (EPSILON).
- Output: A 3x3 Jacobian matrix $\mathbf{J}$, where each column $\mathbf{j}_i$ is the partial derivative of the field with respect to the $i$-th coordinate axis.

**Data Structures:**
- The Jacobian matrix $\mathbf{J}$ is a 3x3 array of floats.
- The field $\mathbf{u}$ is accessed via a function or grid lookup that returns a 3-component vector.

**Steps:**
1. Define the perturbation vectors for each axis:
   $$ \mathbf{d}_x = (\epsilon, 0, 0) $$
   $$ \mathbf{d}_y = (0, \epsilon, 0) $$
   $$ \mathbf{d}_z = (0, 0, \epsilon) $$
2. Compute the partial derivative with respect to $x$ by evaluating the field at $\mathbf{x} + \mathbf{d}_x$ and $\mathbf{x} - \mathbf{d}_x$:
   $$ \mathbf{j}_x = \frac{\mathbf{u}(\mathbf{x} + \mathbf{d}_x) - \mathbf{u}(\mathbf{x} - \mathbf{d}_x)}{2\epsilon} $$
3. Compute the partial derivative with respect to $y$:
   $$ \mathbf{j}_y = \frac{\mathbf{u}(\mathbf{x} + \mathbf{d}_y) - \mathbf{u}(\mathbf{x} - \mathbf{d}_y)}{2\epsilon} $$
4. Compute the partial derivative with respect to $z$:
   $$ \mathbf{j}_z = \frac{\mathbf{u}(\mathbf{x} + \mathbf{d}_z) - \mathbf{u}(\mathbf{x} - \mathbf{d}_z)}{2\epsilon} $$
5. Construct the Jacobian matrix $\mathbf{J}$ using these vectors as columns:
   $$ \mathbf{J} = \begin{bmatrix} | & | & | \\ \mathbf{j}_x & \mathbf{j}_y & \mathbf{j}_z \\ | & | & | \end{bmatrix} $$

**Parameters:**
- $\epsilon$ (EPSILON): The step size for finite differences. Typical values depend on the scale of the field and the precision of the arithmetic. In discrete grid contexts, this may correspond to the voxel spacing or a small fraction thereof [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_jacobian_central_difference(u_func, pos, epsilon):
    """
    Estimates the 3x3 Jacobian matrix of a vector field u at position pos.
    
    Args:
        u_func: A callable that takes a 3D position and returns a 3D vector.
        pos: A 3D vector (x, y, z) where the Jacobian is estimated.
        epsilon: The step size for central differences.
        
    Returns:
        A 3x3 matrix represented as a list of 3 lists, or a flat array of 9 elements.
    """
    # Define perturbation vectors
    dx = (epsilon, 0.0, 0.0)
    dy = (0.0, epsilon, 0.0)
    dz = (0.0, 0.0, epsilon)
    
    # Evaluate field at perturbed positions
    # Note: u_func returns a vector (ux, uy, uz)
    u_plus_x = u_func((pos[0] + dx[0], pos[1] + dx[1], pos[2] + dx[2]))
    u_minus_x = u_func((pos[0] - dx[0], pos[1] - dx[1], pos[2] - dx[2]))
    
    u_plus_y = u_func((pos[0] + dy[0], pos[1] + dy[1], pos[2] + dy[2]))
    u_minus_y = u_func((pos[0] - dy[0], pos[1] - dy[1], pos[2] - dy[2]))
    
    u_plus_z = u_func((pos[0] + dz[0], pos[1] + dz[1], pos[2] + dz[2]))
    u_minus_z = u_func((pos[0] - dz[0], pos[1] - dz[1], pos[2] - dz[2]))
    
    # Compute partial derivatives (columns of the Jacobian)
    # j_x = (u(pos + dx) - u(pos - dx)) / (2 * epsilon)
    j_x = [
        (u_plus_x[0] - u_minus_x[0]) / (2.0 * epsilon),
        (u_plus_x[1] - u_minus_x[1]) / (2.0 * epsilon),
        (u_plus_x[2] - u_minus_x[2]) / (2.0 * epsilon)
    ]
    
    # j_y = (u(pos + dy) - u(pos - dy)) / (2 * epsilon)
    j_y = [
        (u_plus_y[0] - u_minus_y[0]) / (2.0 * epsilon),
        (u_plus_y[1] - u_minus_y[1]) / (2.0 * epsilon),
        (u_plus_y[2] - u_minus_y[2]) / (2.0 * epsilon)
    ]
    
    # j_z = (u(pos + dz) - u(pos - dz)) / (2 * epsilon)
    j_z = [
        (u_plus_z[0] - u_minus_z[0]) / (2.0 * epsilon),
        (u_plus_z[1] - u_minus_z[1]) / (2.0 * epsilon),
        (u_plus_z[2] - u_minus_z[2]) / (2.0 * epsilon)
    ]
    
    # Construct Jacobian matrix (columns are j_x, j_y, j_z)
    # Matrix layout:
    # [ j_x[0] j_y[0] j_z[0] ]
    # [ j_x[1] j_y[1] j_z[1] ]
    # [ j_x[2] j_y[2] j_z[2] ]
    jacobian = [
        [j_x[0], j_y[0], j_z[0]],
        [j_x[1], j_y[1], j_z[1]],
        [j_x[2], j_y[2], j_z[2]]
    ]
    
    return jacobian
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** If $\mathbf{x} \pm \mathbf{d}_i$ falls outside the domain of the field $\mathbf{u}$, the value must be determined via clamping, zero-padding, or extrapolation [G]. This affects the accuracy of the Jacobian at the volume boundaries.
- **Step Size Selection:** If $\epsilon$ is too large, the approximation error (truncation error) increases. If $\epsilon$ is too small, numerical noise (round-off error) dominates. The optimal $\epsilon$ is typically around $\sqrt{\text{machine epsilon}} \times \text{scale}$ [G].
- **Singularity:** If the deformation field is not differentiable at $\mathbf{x}$, the Jacobian may not exist or may be ill-conditioned.

## Subtopics

### [Jacobian Normalization for ODE Step](2946-jacobian-normalization-for-ode-step.md)
This method computes the normalized direction vector for an Ordinary Differential Equation (ODE) solver by applying the inverse of an estimated Jacobian matrix to a base direction vector. It differs from raw Jacobian estimation by performing the matrix inversion and vector normalization required for stable stepping. It is chosen when integrating trajectories through a vector field where the local geometry needs to be accounted for to maintain step stability. See [Jacobian Normalization for ODE Step](2946-jacobian-normalization-for-ode-step.md).

### [Flow Jacobian Decomposition](10525-flow-jacobian-decomposition.md)
Flow Jacobian Decomposition splits the 3x3 Jacobian matrix of a displacement field into symmetric and antisymmetric components to isolate local volume/shape deformation from local rigid rotation. It builds on Jacobian Estimation via Central Differences by interpreting the estimated gradient $\nabla \mathbf{u}$ through continuum mechanics. It is chosen when analyzing the physical nature of a flow or deformation field. See [Flow Jacobian Decomposition](10525-flow-jacobian-decomposition.md).

## Operating characteristics
- **Time Complexity:** $O(1)$ per evaluation, requiring 6 evaluations of the field function $\mathbf{u}$ and a constant number of arithmetic operations.
- **Memory Complexity:** $O(1)$, storing only the current position, perturbed positions, and the resulting 3x3 matrix.
- **Accuracy:** Second-order accurate ($O(\epsilon^2)$) for smooth fields. Accuracy degrades near discontinuities or boundaries.
- **Strengths:** Simple to implement, robust for smooth fields, provides full directional derivative information.
- **Limitations:** Requires function evaluations (or grid lookups), sensitive to step size $\epsilon$, inaccurate for non-smooth fields.
- **When to use:** When a full local linear approximation of a vector field is needed, such as for grid traversal, normal estimation, or flow analysis.
- **When not to use:** When the field is not differentiable, or when higher-order accuracy is required (consider higher-order finite differences or analytical derivatives if available).

## Background
The method originates from the standard numerical analysis technique of central finite differences for approximating derivatives. In the context of volume rendering and graphics, it is used to estimate the Jacobian of deformation fields, such as those generated by procedural functions (e.g., `simpleBrush`) or displacement maps. The specific implementation using a step size `EPSILON` and constructing the matrix from column vectors is documented in graphics programming contexts [S1, unofficial]. The application to computational grids for resampling and traversal is described in interactive volume rendering literature [S2, unofficial].

## Sources
[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S2] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
