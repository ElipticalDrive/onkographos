# Differential Ray Marching

Also called: Differential Form RTE Solver, ODE-based Volume Rendering

## Overview
Differential Ray Marching solves the Radiative Transfer Equation (RTE) in its differential form as an initial value problem, rather than approximating its integral form [S1]. Unlike traditional ray marching, which approximates integrals using the rectangle method (equivalent to Euler's method) and requires computing the double integral for inner transmittance, this method applies numerical ODE solvers directly to the differential equation [S1]. This approach allows for higher-order accuracy and adaptive step sizing, reducing the sample count or increasing image quality compared to standard techniques [S1].

## How it works
**Inputs:**
*   Ray origin $x_0$ and direction $\omega$.
*   Medium properties at any position $x$: absorption coefficient $\sigma_a(x)$, scattering coefficient $\sigma_s(x)$, extinction coefficient $\sigma_t(x) = \sigma_a(x) + \sigma_s(x)$, phase function $p(x, \omega_0, \omega)$, and emission $L_e(x, \omega)$ [S1].
*   Initial radiance $y_0 = L(x_0, \omega)$, derived from surface material properties and light distribution at the entry point [S1].
*   Integration interval $[0, t_{end}]$, where $t$ is the distance along the ray [S1].

**Outputs:**
*   Radiance $L(x_{end}, \omega)$ at the exit of the medium [S1].

**Data Structures:**
*   The state vector $y$ represents the radiance $L(x, \omega)$ at the current step [S1].
*   The independent variable $t$ represents the distance traveled along the ray [S1].
*   The derivative function $f(y, t)$ computes the rate of change of radiance [S1].

**Formulas:**
The Radiative Transfer Equation is defined as [S1]:

$$
\frac{\partial L(x, \omega)}{\partial t} = \sigma_a (x)L_e (x, \omega) - \sigma_t (x)L(x, \omega) + \sigma_s (x) \int_{\Omega} p(x, \omega_0 , \omega)L(x, \omega_0 ) d\omega_0
$$

This is formulated as an initial value problem [S1]:

$$
y'(t) = f(y, t)
$$

Where:
*   $y(t) = L(x(t), \omega)$ is the radiance [S1].
*   $t$ is the distance along the path of light [S1].
*   $f(y, t)$ is the right-hand side of the RTE [S1].
*   $y_0 = L(x_0, \omega)$ is the known initial value [S1].

The term $\int_{\Omega} p(x, \omega_0 , \omega)L(x, \omega_0 ) d\omega_0$ represents the in-scattered radiance contribution from all incoming directions $\omega_0$ [S1]. In practice, this integral is often approximated or precomputed depending on the specific ODE solver and scattering model used, but the core differential formulation avoids the explicit computation of the transmittance integral $T_r$ required in the integral form [S1].

**Steps:**
1.  **Initialize:** Set $t = 0$ and $y = y_0$ [S1].
2.  **Evaluate Derivative:** Compute $k = f(y, t)$ using the current radiance $y$ and position $x(t) = x_0 + t\omega$ [S1].
    *   $f(y, t) = \sigma_a(x) L_e(x, \omega) - \sigma_t(x) y + \sigma_s(x) L_{in}(x, \omega)$, where $L_{in}$ is the in-scattered radiance [S1].
3.  **Advance Step:** Use a numerical ODE method (e.g., Euler, Runge-Kutta) to update $y$ and $t$ based on the step size $\Delta t$ [S1].
    *   *Note:* The choice of numerical method determines how $y$ and $t$ are updated. Traditional ray marching uses Euler's method (rectangle rule) [S1]. Higher-order methods (e.g., Runge-Kutta) provide better accuracy for the same step size [S1].
4.  **Repeat:** Increment $t$ by $\Delta t$. If $t < t_{end}$, return to Step 2. Otherwise, return $y$ as the final radiance [S1].

**Parameters:**
*   Step size $\Delta t$: Can be fixed or adaptive. Adaptive step sizing is a key advantage of ODE solvers, allowing larger steps in homogeneous regions and smaller steps in high-gradient areas [S1].
*   Numerical method order: Determines accuracy and computational cost per step [S1].

**Edge Cases and Pitfalls:**
*   **In-scattering Integral:** The term $\int_{\Omega} p(x, \omega_0 , \omega)L(x, \omega_0 ) d\omega_0$ is computationally expensive. In differential ray marching, this is still required at each step unless approximated. The method does not eliminate the need to estimate in-scattering, but it changes how the transmittance and absorption are handled [S1].
*   **Precision:** Standard floating-point precision may be insufficient for very small or very large extinction coefficients. [G] Clamping $\sigma_t$ to a minimum non-zero value can prevent division by zero or numerical instability in adaptive step size controllers.
*   **Boundary Conditions:** The initial value $y_0$ must be correctly computed from the surface intersection. If the ray starts inside the medium, $y_0$ is typically zero or the background radiance [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def differential_ray_marching(ray_origin, ray_direction, medium, t_end, initial_radiance):
    """
    Solves the RTE in differential form using an ODE solver.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray.
        medium: Object providing methods to query sigma_a, sigma_s, sigma_t, L_e, and phase function.
        t_end: Float, total distance to integrate.
        initial_radiance: Float, L(x_0, omega).
        
    Returns:
        Float, final radiance L(x_end, omega).
    """
    t = 0.0
    y = initial_radiance
    
    # Determine step size. For simplicity, using a fixed step here.
    # In practice, an adaptive step size controller would be used.
    delta_t = medium.get_step_size() 
    
    while t < t_end:
        # Current position
        x = ray_origin + t * ray_direction
        
        # Evaluate the derivative f(y, t)
        # f(y, t) = sigma_a * L_e - sigma_t * y + sigma_s * L_in
        sigma_a = medium.get_sigma_a(x)
        sigma_s = medium.get_sigma_s(x)
        sigma_t = sigma_a + sigma_s
        L_e = medium.get_emission(x, ray_direction)
        
        # Compute in-scattered radiance L_in
        # This requires integrating over the sphere. 
        # For this pseudocode, we assume a helper function exists.
        L_in = compute_in_scattered_radiance(x, ray_direction, medium)
        
        derivative = sigma_a * L_e - sigma_t * y + sigma_s * L_in
        
        # Update using Euler's method (equivalent to traditional ray marching)
        # Higher-order methods (e.g., RK4) would require multiple evaluations
        y = y + delta_t * derivative
        t = t + delta_t
        
    return y

def compute_in_scattered_radiance(x, view_direction, medium):
    """
    Approximates the integral of in-scattered light.
    This is a placeholder for the specific scattering model used.
    """
    # Example: Single scattering approximation or precomputed light field
    # The exact implementation depends on the specific application.
    # The differential form does not change this requirement.
    return 0.0 # Placeholder
```

## Subtopics

### Forward Scattering Dipole Model
A specific scattering model that can be used within the differential ray marching framework to approximate the in-scattering term efficiently. It differs from general phase function integration by using a dipole approximation for the Green's function of the diffusion equation. Choose this when single scattering is insufficient but full multiple scattering is too expensive. [Link](10173-forward-scattering-dipole-model.md)

## Operating characteristics
*   **Time Complexity:** Depends on the number of steps $n$ and the cost of evaluating the derivative $f(y, t)$ at each step. Higher-order ODE methods require more function evaluations per step but may allow larger step sizes, potentially reducing $n$ [S1].
*   **Memory Complexity:** Low, as it only stores the current state $y$ and $t$ [S1].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with the number of steps. Adaptive step sizing can improve scaling in complex media by reducing steps in homogeneous regions [S1].
*   **Strengths:** Avoids the double integral of transmittance required in integral form methods. Allows the use of advanced ODE solvers for better accuracy and adaptive step sizing [S1].
*   **Limitations:** Still requires computation of the in-scattering integral at each step, which can be expensive. The benefit is most pronounced when the in-scattering term is approximated or precomputed [S1].
*   **When to use:** When high accuracy is needed with fewer samples, or when adaptive step sizing is beneficial. [S1]
*   **When not to use:** When the in-scattering term is extremely expensive to compute and cannot be approximated, the overhead of the ODE solver may not justify the benefits over simpler methods. [S1]

## Background
Differential Ray Marching was introduced in the paper "Differential Ray Marching" (2012) [S1]. It revisits the concept of ray marching not as an integration technique, but as the application of numerical methods to solve an initial value differential equation [S1]. The work analyzes a wide range of numerical methods and their applicability to participating media rendering, showing improvements over traditional ray marching in terms of sample count and accuracy [S1].

## Sources
[S1] Differential Ray Marching, https://doi.org/10.2312/localchapterevents/ceig/ceig12/091-098, fetched 2026-10-26
