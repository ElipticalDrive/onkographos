# Differentiable Radiance Derivative Computation
Also called: Analytical radiance derivatives, Direct radiance derivative computation

## Overview
A method to compute radiance derivatives with respect to volume density analytically, avoiding the gradient tracking and redundant operations inherent in automatic differentiation (AD) [S1, unofficial]. It derives an analytical form of the radiance transfer equation to enable direct and efficient computation of gradients for optimizing dynamic volumetric densities [S1, unofficial]. Unlike AD, which propagates gradients through long computational graphs carrying inherent overhead, this approach calculates derivatives directly [S1, unofficial].

## How it works
**Inputs:**
- Volume density field $\kappa_t(x)$ (extinction coefficient).
- Viewpoint position $x$.
- Integration path endpoints $x_0, x_1$ inside the participating medium.
- Distances $D_0, D_1$ from viewpoint $x$ to $x_0, x_1$.
- Incident radiance $L_r(v_p, \omega')$ at voxel position $v_p$.
- Phase function $p(\omega, \omega')$.

**Outputs:**
- Transmittance $\tau(x_0, x_1)$.
- Single-scattering radiance $L_{ss}$.
- Analytical derivatives $\frac{\partial L}{\partial \kappa_t}$.

**Data Structures:**
- Discrete voxel grid representing $\kappa_t$.
- Intermediate forward rendering outputs (transmittance and radiance values) stored for backward pass reuse [S1, unofficial].

**Steps:**
1. **Discretize Transmittance:** Convert the continuous transmittance integral into a discrete form suitable for implementation. The continuous form is:
   $$ \tau(x_0, x_1) = e^{-\int_{D_0}^{D_1} \kappa_t(x_u) du} $$
   where $x_u$ is a position along the ray parameterized by distance $u$ [S1, unofficial].
2. **Discretize Single-Scattering Radiance:** Convert the continuous single-scattering term $J_{ss}$ into a discrete form. The continuous form for reduced incident radiance is:
   $$ J_{ss}(v_p, \omega) = \frac{\Omega}{4\pi} \int_{S^2} L_r(v_p, \omega') p(\omega, \omega') d\omega' $$
   where $S^2$ is the unit sphere solid angle [S1, unofficial]. Monte Carlo integration is used to estimate $J_{ss}$ in the discrete domain [S1, unofficial].
3. **Compute Forward Pass:** Calculate $\tau$ and $L_{ss}$ using the discrete forms. Store intermediate outputs (e.g., accumulated optical depth, transmittance values) [S1, unofficial].
4. **Compute Analytical Derivatives:** Derive the analytical gradient of the radiance transfer equation with respect to $\kappa_t$. This avoids constructing a computational graph for AD. The derivative is computed directly using the stored intermediate outputs from the forward pass [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_analytical_derivative(kappa_t, x0, x1, D0, D1, L_r, p, omega):
    """
    Computes transmittance, single-scattering radiance, and their analytical derivatives
    with respect to extinction coefficient kappa_t.
    
    Args:
        kappa_t: Array of extinction coefficients along the ray.
        x0, x1: Endpoints of the integration segment.
        D0, D1: Distances from viewpoint to x0, x1.
        L_r: Incident radiance at scattering points.
        p: Phase function.
        omega: Solid angle factor.
        
    Returns:
        tau: Transmittance.
        L_ss: Single-scattering radiance.
        dL_dkappa: Derivative of radiance w.r.t. kappa_t.
    """
    # Step 1: Discretize and compute optical depth
    # Integral of kappa_t from D0 to D1
    # Assuming uniform sampling or trapezoidal rule for discretization
    optical_depth = 0.0
    for i in range(len(kappa_t) - 1):
        # Placeholder for specific discretization scheme (e.g., midpoint rule)
        # delta_u = (D1 - D0) / len(kappa_t)
        # u_mid = D0 + (i + 0.5) * delta_u
        # optical_depth += kappa_t[i] * delta_u
        pass # Implementation detail not fully specified in source
    
    # Step 2: Compute Transmittance
    tau = exp(-optical_depth)
    
    # Step 3: Compute Single-Scattering Radiance (J_ss)
    # Monte Carlo integration over S^2
    # J_ss = (Omega / 4pi) * Integral(L_r * p) domega'
    # Discrete MC estimate
    num_samples = 100 # Not given in sources, typical value
    j_ss_sum = 0.0
    for _ in range(num_samples):
        omega_prime = sample_sphere() # Standard math operation
        j_ss_sum += L_r * p(omega, omega_prime)
    j_ss = (omega / (4 * pi)) * (j_ss_sum / num_samples)
    
    # Step 4: Compute Analytical Derivative
    # The derivative of L w.r.t. kappa_t is derived analytically from the 
    # radiance transfer equation. This avoids AD graph propagation.
    # dL/dkappa_t = -L * (integral of tau from current point to end) + ...
    # Exact formula depends on the specific radiance transfer equation form used.
    # Source indicates it is derived from the continuous form and discretized.
    dL_dkappa = compute_analytical_gradient(tau, j_ss, kappa_t) # [G] Placeholder for specific derivative formula
    
    return tau, j_ss, dL_dkappa
```

**Edge Cases and Pitfalls:**
- Precision: Exponential decay in transmittance can lead to underflow for large optical depths [S1, unofficial].
- Discretization: The choice of discretization for the integral affects accuracy. The source mentions deriving discrete forms but does not specify the exact numerical integration scheme (e.g., trapezoidal, Simpson's) [S1, unofficial].
- Boundary Conditions: The method assumes arbitrary positions $x_0, x_1$ inside the medium. Handling boundaries of the volume grid requires care [S1, unofficial].

## Subtopics

### [Many-Worlds Derivative Transport](23178-many-worlds-derivative-transport.md)
A derivative propagation method that treats potential surfaces along a ray as mutually exclusive events, ensuring they do not interact via shadowing or scattering, unlike exponential volume rendering which treats them as independent Poisson processes.

## Operating characteristics
- **Time Complexity:** Not given in the sources. However, it is stated to be more efficient than Automatic Differentiation (AD) by avoiding gradient tracking overhead [S1, unofficial].
- **Memory Complexity:** Not given in the sources. Requires storing intermediate forward rendering outputs for the backward pass [S1, unofficial].
- **Performance:** Achieves excellent performance for differentiating rendered images with respect to volume densities, particularly for optimizing dynamic volumetric densities such as in fluid reconstruction tasks [S1, unofficial].
- **Strengths:** Direct computation avoids the overhead of long computational graphs inherent in AD. Efficient for dynamic volumetric density optimization [S1, unofficial].
- **Limitations:** Requires careful derivation of analytical forms for the radiance transfer equation. May be complex to implement for arbitrary scattering models compared to black-box AD [S1, unofficial].
- **When to Use:** When optimizing dynamic volumetric densities (e.g., fluid reconstruction) and performance is critical, and the radiance transfer equation can be analytically differentiated [S1, unofficial].
- **When Not to Use:** When the radiance transfer equation is too complex to derive analytical derivatives for, or when AD performance is sufficient and implementation simplicity is prioritized [S1, unofficial].

## Background
Originated from research into physics-based differentiable rendering for efficient and accurate volumetric simulation [S1, unofficial]. The work builds upon insights from previous AD-based differentiable rendering methods, which demonstrated that careful design of the backward pass can achieve excellent performance [S1, unofficial]. However, the authors identified that AD still fundamentally relies on propagating gradients through long computational graphs, carrying inherent overhead [S1, unofficial]. This method was developed to further improve performance by deriving an analytical form of the radiance transfer equation [S1, unofficial].

## Sources
[S1] Physics-Based Differentiable Rendering for Efficient and ... - MDPI, https://www.mdpi.com/1099-4300/25/9/1348, fetched 2026-10-27 (unofficial)
