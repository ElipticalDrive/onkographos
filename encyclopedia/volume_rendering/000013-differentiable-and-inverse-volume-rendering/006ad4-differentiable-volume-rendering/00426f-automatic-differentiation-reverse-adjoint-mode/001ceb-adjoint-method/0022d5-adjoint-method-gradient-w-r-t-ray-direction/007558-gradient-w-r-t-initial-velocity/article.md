# Gradient w.r.t. Initial Velocity

Also called: Derivative of ray position w.r.t. initial velocity, Derivative of ray position with respect to initial velocity, Velocity Gradient in Symplectic Integration

## Overview
This method computes the gradient of the ray position $\mathbf{x}$ with respect to the initial velocity $\mathbf{v}_0$ in differentiable volume rendering using symplectic integration. It is required to propagate gradients through the ray tracing step when the refractive index or medium properties depend on parameters that influence the initial velocity. It builds on [Adjoint Method > Gradient w.r.t Ray Direction](2891-adjoint-method-gradient-w-r-t-ray-direction.md) by handling the velocity parameter space. The method distinguishes between Type-1 connections (where the ray hits an implicit surface) and Type-2 connections (where the ray connects two points inside the medium).

## How it works
The method computes the total derivative $\frac{d\mathbf{x}}{d\mathbf{v}_0}$ by combining the partial derivatives obtained from the symplectic integrator with correction terms that account for constraints on the ray path length or endpoint.

### Inputs and Outputs
*   **Inputs**:
    *   $\mathbf{v}_0$: Initial velocity vector (3D).
    *   $s^*$: The path length parameter at the point of interest (e.g., surface intersection or target projection).
    *   $\mathbf{x}_{s^*}$: Ray position at $s^*$.
    *   $\mathbf{v}_{s^*}$: Ray velocity at $s^*$.
    *   $\frac{\partial \mathbf{x}_{s^*}}{\partial \mathbf{v}_0}$: Partial derivative of position w.r.t. initial velocity from the integrator.
    *   $\frac{\partial \mathbf{v}_{s^*}}{\partial \mathbf{v}_0}$: Partial derivative of velocity w.r.t. initial velocity from the integrator.
    *   $\frac{\partial \mathbf{x}_{s^*}}{\partial s^*}$: Partial derivative of position w.r.t. path length.
    *   $\frac{\partial \mathbf{v}_{s^*}}{\partial s^*}$: Partial derivative of velocity w.r.t. path length.
    *   $g(\mathbf{x})$: Implicit surface function (for Type-1).
    *   $\mathbf{y}$: Target point (for Type-2).
*   **Outputs**:
    *   $\frac{d\mathbf{x}_{s^*}}{d\mathbf{v}_0}$: Total derivative of ray position w.r.t. initial velocity.

### Data Structures
The derivatives are $3 \times 3$ Jacobian matrices (or $3 \times 1$ vectors if $\mathbf{v}_0$ is treated as a scalar direction, but generally $3 \times 3$ for full velocity vector).
*   $\mathbf{J}_{xv} = \frac{\partial \mathbf{x}_{s^*}}{\partial \mathbf{v}_0}$: Layout $3 \times 3$.
*   $\mathbf{J}_{vv} = \frac{\partial \mathbf{v}_{s^*}}{\partial \mathbf{v}_0}$: Layout $3 \times 3$.
*   $\mathbf{J}_{xs} = \frac{\partial \mathbf{x}_{s^*}}{\partial s^*}$: Layout $3 \times 1$ (vector).
*   $\mathbf{J}_{vs} = \frac{\partial \mathbf{v}_{s^*}}{\partial s^*}$: Layout $3 \times 1$ (vector).

### Steps and Formulas

#### 1. Compute Partial Derivatives via Symplectic Integration
The partial derivatives $\frac{\partial \mathbf{x}_{s^*}}{\partial \mathbf{v}_0}$ and $\frac{\partial \mathbf{v}_{s^*}}{\partial \mathbf{v}_0}$ are computed in parallel with the ray tracing using the symplectic integration procedure for Hamilton’s equations [S1]. These are propagated through each integration step.

The derivatives with respect to path length $s$ are given by:
$$
\frac{\partial \mathbf{x}_s}{\partial s} = \frac{\mathbf{v}_s}{n(\mathbf{x}_s)}, \quad \frac{\partial \mathbf{v}_s}{\partial s} = \nabla n(\mathbf{x}_s)
$$
where $n(\mathbf{x}_s)$ is the refractive index and $\nabla n$ is its gradient [S1]. Note: The source text for Eq. (19) shows $\frac{\partial \mathbf{v}_s}{\partial s} = n(\mathbf{x}_s)$, but physically this should be the gradient of the refractive index or related force term. We follow the source notation strictly: $\frac{\partial \mathbf{v}_s}{\partial s} = n(\mathbf{x}_s)$ [S1]. [G: In standard Hamiltonian ray tracing, $\frac{d\mathbf{v}}{ds} = \nabla n$. The source likely implies a specific normalization or simplified model. We use the source's explicit formula.]

#### 2. Compute Total Derivative for Type-1 Connections
For Type-1 connections, the ray hits an implicit surface defined by $g(\mathbf{x}_{s^*}) = 0$. The path length $s^*$ is determined by this constraint. The total derivative is:

$$
\frac{d\mathbf{x}_{s^*}}{d\mathbf{v}_0} = \left( I_{3 \times 3} - \frac{\frac{\partial \mathbf{x}_{s^*}}{\partial s^*} \frac{dg(\mathbf{x}_{s^*})}{d\mathbf{x}_{s^*}}}{\frac{dg(\mathbf{x}_{s^*})}{d\mathbf{x}_{s^*}} \frac{\partial \mathbf{x}_{s^*}}{\partial s^*}} \right) \frac{\partial \mathbf{x}_{s^*}}{\partial \mathbf{v}_0}
$$

Where:
*   $I_{3 \times 3}$ is the $3 \times 3$ identity matrix.
*   $\frac{dg(\mathbf{x}_{s^*})}{d\mathbf{x}_{s^*}}$ is the gradient of the implicit surface function $g$ at $\mathbf{x}_{s^*}$ (a row vector $1 \times 3$).
*   The term $\frac{\partial \mathbf{x}_{s^*}}{\partial s^*} \frac{dg(\mathbf{x}_{s^*})}{d\mathbf{x}_{s^*}}$ is an outer product ($3 \times 1$ times $1 \times 3$), resulting in a $3 \times 3$ matrix.
*   The denominator $\frac{dg(\mathbf{x}_{s^*})}{d\mathbf{x}_{s^*}} \frac{\partial \mathbf{x}_{s^*}}{\partial s^*}$ is a scalar (dot product).

#### 3. Compute Total Derivative for Type-2 Connections
For Type-2 connections, both vertices are inside the medium. The path length $s^*$ is defined as the shortest geometric length to a target point $\mathbf{y}$, i.e., $s^* := \arg \min_s \|\mathbf{x}_s - \mathbf{y}\|^2$ [S1]. The derivative is:

$$
\frac{d\mathbf{x}_s}{d\mathbf{v}_0} = \frac{\partial \mathbf{x}_s}{\partial \mathbf{v}_0} - \frac{\partial \mathbf{x}_s}{\partial s^*} \frac{\mathbf{v}_s^T \frac{\partial \mathbf{x}_s}{\partial \mathbf{v}_0} + (\mathbf{x}_s - \mathbf{y})^T \frac{\partial \mathbf{v}_s}{\partial \mathbf{v}_0}}{\mathbf{v}_s^T \frac{\partial \mathbf{x}_s}{\partial s^*} + (\mathbf{x}_s - \mathbf{y})^T \frac{\partial \mathbf{v}_s}{\partial s^*}}
$$

Where:
*   $\mathbf{y}$ is the target point.
*   $\mathbf{v}_s^T \frac{\partial \mathbf{x}_s}{\partial \mathbf{v}_0}$ is a dot product resulting in a row vector ($1 \times 3$).
*   $(\mathbf{x}_s - \mathbf{y})^T \frac{\partial \mathbf{v}_s}{\partial \mathbf{v}_0}$ is a dot product resulting in a row vector ($1 \times 3$).
*   The numerator is the sum of these row vectors.
*   The denominator is a scalar sum of dot products.
*   The fraction scales the vector $\frac{\partial \mathbf{x}_s}{\partial s^*}$ (which is $3 \times 1$) by the row vector result, effectively creating a correction term. Note: The source formula (20) implies the numerator is treated as a scalar multiplier for the vector $\frac{\partial \mathbf{x}_s}{\partial s^*}$? No, looking at dimensions: $\frac{\partial \mathbf{x}_s}{\partial s^*}$ is $3 \times 1$. The term subtracted must be $3 \times 3$ if applied to $\mathbf{v}_0$? No, the result $\frac{d\mathbf{x}_s}{d\mathbf{v}_0}$ is $3 \times 3$. $\frac{\partial \mathbf{x}_s}{\partial \mathbf{v}_0}$ is $3 \times 3$. The second term must be $3 \times 3$.
    *   $\frac{\partial \mathbf{x}_s}{\partial s^*}$ is $3 \times 1$.
    *   The fraction $\frac{\dots}{\dots}$ must be $1 \times 3$ to make the outer product $3 \times 3$.
    *   Let's check the source formula (20) again: $\frac{\mathbf{v}_s^T \frac{\partial \mathbf{x}_s}{\partial \mathbf{v}_0} + (\mathbf{x}_s - \mathbf{y})^T \frac{\partial \mathbf{v}_s}{\partial \mathbf{v}_0}}{\dots}$.
    *   $\mathbf{v}_s^T$ is $1 \times 3$. $\frac{\partial \mathbf{x}_s}{\partial \mathbf{v}_0}$ is $3 \times 3$. Result is $1 \times 3$.
    *   $(\mathbf{x}_s - \mathbf{y})^T$ is $1 \times 3$. $\frac{\partial \mathbf{v}_s}{\partial \mathbf{v}_0}$ is $3 \times 3$. Result is $1 \times 3$.
    *   Sum is $1 \times 3$.
    *   Denominator is scalar.
    *   So the fraction is $1 \times 3$.
    *   $\frac{\partial \mathbf{x}_s}{\partial s^*}$ is $3 \times 1$.
    *   Outer product $(3 \times 1) \times (1 \times 3) = 3 \times 3$. This matches.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_type1(x_s, v_s, v0, dx_dv0, dv_dv0, dx_ds, dg_dx, g_func):
    """
    Computes d(x_s)/d(v0) for Type-1 connections (implicit surface constraint).
    
    Args:
        x_s: Ray position at s* (3,)
        v_s: Ray velocity at s* (3,)
        v0: Initial velocity (3,)
        dx_dv0: Partial derivative dx/dv0 from integrator (3,3)
        dv_dv0: Partial derivative dv/dv0 from integrator (3,3)
        dx_ds: Partial derivative dx/ds (3,)
        dg_dx: Gradient of implicit surface g at x_s (3,)
        g_func: Function to evaluate g(x)
        
    Returns:
        d_x_d_v0: Total derivative (3,3)
    """
    # Identity matrix
    I = np.eye(3)
    
    # Gradient of g is a row vector for outer product
    dg_dx_row = dg_dx.reshape(1, 3)
    
    # Outer product: dx_ds (3,1) * dg_dx (1,3) -> (3,3)
    outer_product = np.outer(dx_ds, dg_dx)
    
    # Denominator: dg_dx (1,3) * dx_ds (3,1) -> scalar
    denominator = np.dot(dg_dx, dx_ds)
    
    # Correction matrix
    correction = outer_product / denominator
    
    # Final formula: (I - correction) * dx_dv0
    d_x_d_v0 = (I - correction) @ dx_dv0
    
    return d_x_d_v0

def compute_gradient_type2(x_s, v_s, v0, y, dx_dv0, dv_dv0, dx_ds, dv_ds):
    """
    Computes d(x_s)/d(v0) for Type-2 connections (target point constraint).
    
    Args:
        x_s: Ray position at s (3,)
        v_s: Ray velocity at s (3,)
        v0: Initial velocity (3,)
        y: Target point (3,)
        dx_dv0: Partial derivative dx/dv0 from integrator (3,3)
        dv_dv0: Partial derivative dv/dv0 from integrator (3,3)
        dx_ds: Partial derivative dx/ds (3,)
        dv_ds: Partial derivative dv/ds (3,)
        
    Returns:
        d_x_d_v0: Total derivative (3,3)
    """
    # Vector from x_s to y
    diff = x_s - y
    
    # Numerator terms (row vectors 1x3)
    # v_s^T @ dx_dv0
    term1 = v_s @ dx_dv0
    # (x_s - y)^T @ dv_dv0
    term2 = diff @ dv_dv0
    
    # Sum of row vectors
    numerator_row = term1 + term2
    
    # Denominator terms (scalars)
    # v_s^T @ dx_ds
    denom1 = np.dot(v_s, dx_ds)
    # (x_s - y)^T @ dv_ds
    denom2 = np.dot(diff, dv_ds)
    
    denominator = denom1 + denom2
    
    # Avoid division by zero
    if np.isclose(denominator, 0):
        return dx_dv0
        
    # Fraction is a row vector (1x3)
    fraction_row = numerator_row / denominator
    
    # Correction term: dx_ds (3,1) * fraction_row (1,3) -> (3,3)
    correction_matrix = np.outer(dx_ds, fraction_row)
    
    # Final formula: dx_dv0 - correction_matrix
    d_x_d_v0 = dx_dv0 - correction_matrix
    
    return d_x_d_v0
```

### Edge Cases and Pitfalls
*   **Division by Zero**: In Type-1, if $\frac{dg(\mathbf{x}_{s^*})}{d\mathbf{x}_{s^*}} \frac{\partial \mathbf{x}_{s^*}}{\partial s^*} = 0$, the ray is tangent to the surface or the surface gradient is zero. In Type-2, if the denominator is zero, the constraint is degenerate.
*   **Precision**: The symplectic integration must maintain high precision for the partial derivatives $\frac{\partial \mathbf{x}}{\partial \mathbf{v}_0}$ and $\frac{\partial \mathbf{v}}{\partial \mathbf{v}_0}$ to ensure accurate gradients.
*   **Type-2 Definition**: The definition of $s^*$ as $\arg \min_s \|\mathbf{x}_s - \mathbf{y}\|^2$ implies that $s^*$ is not a fixed parameter but depends on the ray path. This requires solving for $s^*$ implicitly or iteratively.

## Subtopics

### Type-2 Connection Derivative
A formula to compute the total derivative of the ray position with respect to the initial velocity for type-2 connections, which corrects for the constraint that the ray must hit a target point $\mathbf{y}$. It is used when both vertices of the connection are inside the medium, and the path length is determined by the shortest distance to the target point rather than an implicit surface intersection [S1].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per ray step for the derivative computation, assuming the partial derivatives are already computed by the symplectic integrator. The symplectic integration itself is $O(N)$ where $N$ is the number of steps.
*   **Memory Complexity**: $O(1)$ additional memory for the Jacobian matrices ($3 \times 3$) per ray.
*   **Strengths**: Provides exact gradients for refractive ray tracing with implicit surfaces and target constraints, enabling differentiable rendering in complex media.
*   **Limitations**: Requires the computation of partial derivatives during the forward pass, increasing memory and compute overhead. Degenerate cases (tangency, zero denominators) require special handling.

## Background
This method originates from the work on path tracing estimators for refractive radiative transfer [S1]. It extends the adjoint method for differentiable volume rendering to handle velocity-dependent parameters in symplectic integration schemes. The distinction between Type-1 (implicit surface) and Type-2 (target point) connections allows for flexible handling of different boundary conditions in participating media.

## Sources
[S1] Path tracing estimators for refractive radiative transfer, https://doi.org/10.1145/3414685.3417793, fetched 2026-10-29
