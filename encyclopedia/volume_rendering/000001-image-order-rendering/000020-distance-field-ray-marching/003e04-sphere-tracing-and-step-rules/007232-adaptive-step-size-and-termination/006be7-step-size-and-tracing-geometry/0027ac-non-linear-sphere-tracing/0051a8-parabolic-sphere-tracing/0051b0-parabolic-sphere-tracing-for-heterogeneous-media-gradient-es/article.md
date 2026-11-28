# Parabolic Sphere Tracing for Heterogeneous Media Gradient Estimation

Also called: Paraxial ray tracing for gradient, Transmitted caustic estimation

## Overview
This method estimates transmitted caustics by finding stationary optical paths through a heterogeneous refractive medium (such as a glass panel) using a gradient-based optimizer [S1]. It computes the gradient of the Optical Path Length (OPL) using paraxial ray theory to handle non-linear ray trajectories in varying index of refraction (RIF) fields [S1]. It sets itself apart from [Paraxial Ray Tracing](14547-paraxial-ray-tracing.md) by explicitly deriving the gradient corrections required for heterogeneous media where the RIF varies spatially, enabling the use of quasi-Newton solvers for caustic estimation [S1].

## How it works
The method formulates the estimation of transmitted caustics as an optimization problem: finding the stationary points of the Optical Path Length (OPL) function with respect to the intersection points on the medium's boundaries [S1].

### Inputs and Outputs
*   **Inputs**:
    *   Fixed source point $\mathbf{x}_0$ and receiver point $\mathbf{x}_3$ [S1].
    *   Parametric surfaces defining the front and back faces of the refractive panel, given by differentiable elevation functions $h_1(u_1, v_1)$ and $h_2(u_2, v_2)$ [S1].
    *   Heterogeneous RIF field $\eta(\mathbf{x})$ defined within the panel volume [S1].
    *   Homogeneous RIF $\eta_0$ outside the panel [S1].
*   **Outputs**:
    *   Stationary path points $\mathbf{x}_1$ (on front face) and $\mathbf{x}_2$ (on back face) satisfying $\nabla L = 0$ [S1].
    *   Gradient of the OPL with respect to surface parameters $(u_1, v_1, u_2, v_2)$ [S1].

### Data Structures
*   **Ray State**: For each tetrahedral cell $k$ traversed by the parabolic ray, store the parabolic coefficients derived from the RIF gradient $\boldsymbol{\gamma}$ and initial direction $\mathbf{p}_0$ [S1].
*   **Propagator Matrices**: $P_2$ and $Q_2$, sub-matrices of the ray propagator matrix defined in the parent article [Parabolic Sphere Tracing](14525-parabolic-sphere-tracing.md), used for paraxial corrections [S1].
*   **Optimizer State**: BFGS approximation of the Hessian inverse and current parameters $(u_1, v_1, u_2, v_2)$ [S1].

### Formulas

**1. Optical Path Length (OPL)**
The total OPL for a path $\Gamma(\mathbf{x}_0, \mathbf{x}_1, \mathbf{x}_2, \mathbf{x}_3)$ composed of two homogeneous sections and one curved heterogeneous section is:
$$
L(\mathbf{x}_0, \mathbf{x}_1, \mathbf{x}_2, \mathbf{x}_3) = \eta_0 \|\mathbf{x}_1 - \mathbf{x}_0\| + \int_{\mathbf{x}_1}^{\mathbf{x}_2} \eta(\Gamma_1(\tau)) d\tau + \eta_0 \|\mathbf{x}_3 - \mathbf{x}_2\|
$$
where $\Gamma_1$ is the curved ray section inside the panel [S1].

**2. Parabolic Ray OPL per Cell**
Inside a tetrahedral cell $k$, the OPL $l_k(\tau)$ of the ray section parameterized by $\tau$ is:
$$
l_k(\tau) = \frac{1}{12}[\boldsymbol{\gamma} \cdot \boldsymbol{\gamma}]\tau^3 + \frac{1}{2}\boldsymbol{\gamma} \cdot \mathbf{p}_0\tau^2 + [\mathbf{p}_0 \cdot \mathbf{p}_0]\tau
$$
where $\boldsymbol{\gamma}$ is the gradient of the squared RIF and $\mathbf{p}_0$ is the initial direction vector [S1].

**3. Paraxial OPL Correction**
The traced parabolic ray starting at $\mathbf{x}_1$ towards $\mathbf{x}_2$ typically intersects the back face at $\hat{\mathbf{x}}_2$ rather than the target $\mathbf{x}_2$ [S1]. The OPL $L(\mathbf{x}_1, \mathbf{x}_2)$ is corrected from $L(\mathbf{x}_1, \hat{\mathbf{x}}_2)$ using the paraxial approximation:
$$
L(\mathbf{x}_1, \mathbf{x}_2) = L(\mathbf{x}_1, \hat{\mathbf{x}}_2) + \hat{\mathbf{p}}_2 \cdot \mathbf{d}\mathbf{x}_2 - \frac{1}{2} L^{-1}(\mathbf{x}_1, \hat{\mathbf{x}}_2)(\hat{\mathbf{p}}_2 \cdot \mathbf{d}\mathbf{x}_2)^2 + \frac{1}{2} \mathbf{d}\mathbf{x}_2 \cdot ((P_2 Q_2^{-1}) \mathbf{d}\mathbf{x}_2)
$$
where $\mathbf{d}\mathbf{x}_2 = \mathbf{x}_2 - \hat{\mathbf{x}}_2$, $\hat{\mathbf{p}}_2$ is the final direction of the traced ray, and $L^{-1}$ denotes the reciprocal $1/L$ [S1].

**4. Paraxial Direction Corrections for Gradient**
The effective initial and final directions $\mathbf{p}_1$ and $\mathbf{p}_2$ for the gradient calculation are:
$$
\mathbf{p}_1 = \hat{\mathbf{p}}_1 - L^{-1}(\mathbf{x}_1, \hat{\mathbf{x}}_2)(\hat{\mathbf{p}}_2 \cdot \mathbf{d}\mathbf{x}_2)\hat{\mathbf{p}}_1 + Q_2^{-1} \mathbf{d}\mathbf{x}_2
$$
$$
\mathbf{p}_2 = \hat{\mathbf{p}}_2 - L^{-1}(\mathbf{x}_1, \hat{\mathbf{x}}_2)(\hat{\mathbf{p}}_2 \cdot \mathbf{d}\mathbf{x}_2)\hat{\mathbf{p}}_2 + (P_2 Q_2^{-1}) \mathbf{d}\mathbf{x}_2
$$
where $\hat{\mathbf{p}}_1$ is the initial direction of the traced ray [S1].

**5. Gradient Expression**
The differential of the OPL $dL$ with respect to spatial perturbations $d\mathbf{x}_1$ and $d\mathbf{x}_2$ is:
$$
dL = \eta_0 \frac{\mathbf{x}_1 - \mathbf{x}_0}{\|\mathbf{x}_1 - \mathbf{x}_0\|} \cdot d\mathbf{x}_1 + \eta_0 \frac{\mathbf{x}_3 - \mathbf{x}_2}{\|\mathbf{x}_3 - \mathbf{x}_2\|} \cdot d\mathbf{x}_2 - \mathbf{p}_1 \cdot d\mathbf{x}_1 + \mathbf{p}_2 \cdot d\mathbf{x}_2
$$
[S1]

**6. Surface Parameter Mapping**
Since $\mathbf{x}_1$ and $\mathbf{x}_2$ are constrained to surfaces $h_1$ and $h_2$, the spatial differentials are mapped to surface parameter differentials:
$$
d\mathbf{x}_j = \begin{pmatrix} 1 \\ 0 \\ \frac{\partial h_j}{\partial u_j} \end{pmatrix} du_j + \begin{pmatrix} 0 \\ \frac{\partial h_j}{\partial v_j} \\ 0 \end{pmatrix} dv_j
$$
for $j \in \{1, 2\}$ [S1].

### Steps

1.  **Initialization**: Choose random initial parameters $(u_1, v_1, u_2, v_2)$ to define $\mathbf{x}_1$ and $\mathbf{x}_2$ on the panel surfaces [S1].
2.  **Homogeneous Approximation (Optional Two-Stage)**:
    *   Assume constant RIF $\eta_0$ inside the panel.
    *   Solve for stationary path using a simpler solver (ignoring RIF variations) to get an approximate solution [S1].
3.  **Heterogeneous Refinement**:
    *   **Trace Ray**: From $\mathbf{x}_1$, trace a parabolic ray through the heterogeneous medium towards the current estimate of $\mathbf{x}_2$ [S1].
    *   **Compute Traced OPL**: Sum $l_k(\tau)$ for all cells $k$ visited by the ray to get $L(\mathbf{x}_1, \hat{\mathbf{x}}_2)$ [S1].
    *   **Compute Correction**: Calculate $\mathbf{d}\mathbf{x}_2 = \mathbf{x}_2 - \hat{\mathbf{x}}_2$. Use the paraxial correction formula (16) to compute the corrected OPL $L(\mathbf{x}_1, \mathbf{x}_2)$ [S1].
    *   **Compute Directions**: Calculate $\mathbf{p}_1$ and $\mathbf{p}_2$ using the paraxial direction correction formulas [S1].
    *   **Compute Gradient**: Evaluate $dL$ using formula (17) and map to surface parameters using the Jacobian of the surface elevation functions [S1].
4.  **Optimization**:
    *   Use a BFGS solver with an approximate line search (Armijo criteria) to minimize $L$ (find $\nabla L = 0$) [S1].
    *   Repeat steps 2-3 until convergence or max iterations [S1].
5.  **Multiple Paths**: Repeat the optimization with different random initializations to find all possible stationary paths [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_and_opl(x0, x3, eta0, h1, dh1_du, dh1_dv, h2, dh2_du, dh2_dv, u1, v1, u2, v2, rif_field, propagator_cache):
    """
    Computes the OPL and gradient for a given set of surface parameters [S1].
    """
    # 1. Map surface parameters to 3D points
    x1 = point_on_surface(h1, u1, v1)
    x2 = point_on_surface(h2, u2, v2)
    
    # 2. Trace parabolic ray from x1 towards x2 through the heterogeneous medium [S1].
    # The ray intersects the back face at x_hat2 instead of x2 [S1].
    # We march through tetrahedral cells, accumulating OPL and tracking directions [S1].
    # Note: The exact cell traversal and parabolic integration are standard steps [G].
    x_hat2, L_traced, p_hat1, p_hat2, P2, Q2 = trace_parabolic_ray_through_cells(x1, x2, eta0, rif_field)
    
    # 3. Compute spatial differential dx2
    dx2 = x2 - x_hat2
    
    # 4. Compute paraxial OPL correction
    # L_inv denotes the reciprocal 1/L [S1]
    L_inv = 1.0 / L_traced
    term1 = dot(p_hat2, dx2)
    term2 = 0.5 * L_inv * (term1 ** 2)
    Q2_inv = inverse(Q2)
    P_Q_inv_dx2 = dot(P2, Q2_inv)
    term3 = 0.5 * dot(dx2, dot(P_Q_inv_dx2, dx2))
    L_corrected = L_traced + term1 - term2 + term3
    
    # 5. Compute paraxial direction corrections p1 and p2 [S1]
    correction_factor = L_inv * term1
    p1 = p_hat1 - correction_factor * p_hat1 + dot(Q2_inv, dx2)
    p2 = p_hat2 - correction_factor * p_hat2 + dot(P_Q_inv_dx2, dx2)
    
    # 6. Compute gradient dL with respect to spatial perturbations dx1, dx2 [S1]
    grad_x1 = eta0 * normalize(x1 - x0) - p1
    grad_x2 = eta0 * normalize(x3 - x2) + p2
    
    # 7. Map spatial gradients to surface parameter gradients using surface Jacobians [S1]
    basis_u1 = vector(1, 0, dh1_du(u1, v1))
    basis_v1 = vector(0, dh1_dv(u1, v1), 0)
    grad_u1 = dot(grad_x1, basis_u1)
    grad_v1 = dot(grad_x1, basis_v1)
    
    basis_u2 = vector(1, 0, dh2_du(u2, v2))
    basis_v2 = vector(0, dh2_dv(u2, v2), 0)
    grad_u2 = dot(grad_x2, basis_u2)
    grad_v2 = dot(grad_x2, basis_v2)
    
    return L_corrected, (grad_u1, grad_v1, grad_u2, grad_v2)
```

## Operating characteristics
The method relies on a BFGS quasi-Newton solver with an Armijo line search, requiring multiple evaluations of the OPL and its gradient per iteration [S1]. It uses a two-stage optimization strategy: a fast homogeneous solver provides an initial guess, which is then refined by the costly heterogeneous solver to limit non-linear tracing steps [S1]. The initialization strategy does not guarantee finding all stationary paths in regions with strong RIF variations, potentially causing local underestimation of transmitted lighting [S1].

## Background
The method was introduced to estimate transmitted caustics through transparent glass panels with heterogeneous refractive index distributions [S1]. It extends paraxial ray theory to handle spatially varying RIF fields by deriving explicit gradient corrections for the optical path length [S1]. The approach builds on parabolic sphere tracing for distance fields and adapts it for gradient-based optimization in heterogeneous media [S1].

## Sources
[S1] Parabolic Sphere Tracing of Signed Distance Fields for Old Glass Modelling and Rendering, https://doi.org/10.1109/tvcg.2025.3613853, fetched 2026-11-05
