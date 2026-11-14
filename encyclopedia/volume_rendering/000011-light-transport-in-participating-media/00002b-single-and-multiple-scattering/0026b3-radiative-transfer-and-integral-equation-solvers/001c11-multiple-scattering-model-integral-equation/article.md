# Multiple Scattering Model (Integral Equation)

Also called: General integral equation for radiation transport, Radiation transport integral equation, Transport Theory Illumination Model

## Overview
The Multiple Scattering Model (Integral Equation) is a physically based illumination model that computes the radiance arriving at a pixel by integrating the contributions of light along a ray, explicitly accounting for emission, absorption, and multiple scattering events within a participating medium [S2, book]. It solves for the directional intensity $I(X, \omega)$ at every point $X$ and direction $\omega$ by treating the in-scattering term as a source that depends on the unknown intensity itself, resulting in a Fredholm integral equation of the second kind [S1, unofficial]. Unlike single-scattering approximations or differential solvers that may neglect angular complexity, this model provides the exact formal solution to the radiative transfer problem, serving as the foundational reference against which approximations like the Diffusion Approximation or Monte Carlo methods are compared [S1, unofficial][S2, book].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   Ray origin $X$ and direction $\omega$ (unit vector).
    *   Ray length $D$ (distance to the far clip plane or volume boundary).
    *   Extinction coefficient $\tau(X)$ (or $\sigma_t(X) = \sigma_a(X) + \sigma_{sc}(X)$) [S1, unofficial][S2, book].
    *   Scattering phase function $r(X, \omega, \omega')$ (or $\rho_{sc}(\omega' \to \omega)$), defining the probability of scattering from direction $\omega'$ to $\omega$ [S1, unofficial][S2, book].
    *   Emission coefficient $e(X)$ (or $E(t)$), representing non-directional glow or black-body thermal radiation [S1, unofficial][S2, book].
    *   External illumination $I_0(X + D\omega, \omega)$, the radiance entering the volume from the background or light sources [S1, unofficial].
*   **Output:**
    *   Radiance $I(X, \omega)$ at the ray origin $X$ in direction $\omega$.

**Data Structures**
The method requires storing volumetric fields for the extinction coefficient $\tau$, emission $e$, and phase function parameters. The phase function $r(X, \omega, \omega')$ is typically stored as a parametric function (e.g., Henyey-Greenstein) or a lookup table, as integrating over the full $4\pi$ solid angle for every sample is computationally prohibitive without approximation [S1, unofficial][S2, book].

**Mathematical Formulation**
The general integral equation for radiation transport is given by [S1, unofficial]:

$$
I(X, \omega) = I_0(X + D\omega, \omega) \exp\left(-\int_0^D \tau(X + t\omega) \, dt\right) + \int_0^D \left[ \int_{4\pi} r(X + s\omega, \omega, \omega') I(X + s\omega, \omega') \, d\omega' + e(X + s\omega) \right] \exp\left(-\int_0^s \tau(X + t\omega) \, dt\right) \, ds
$$

Where:
*   $I(X, \omega)$ is the unknown intensity at position $X$ in direction $\omega$.
*   The first term represents the background illumination attenuated by the total path length $D$.
*   The second term integrates the "source term" $g(s)$ along the ray from $0$ to $D$.
*   The source term $g(s)$ at distance $s$ along the ray is defined as [S1, unofficial]:
    $$
    g(s) = \int_{4\pi} r(X + s\omega, \omega, \omega') I(X + s\omega, \omega') \, d\omega' + e(X + s\omega)
    $$
*   The exponential terms represent the transmittance (Beer-Lambert law) from the sample point to the observer (for the inner integral's contribution) or from the ray origin to the sample point (for the outer integral's attenuation).

An alternative formulation often used in computer graphics contexts defines the intensity at the near clip plane $t_0$ as [S2, book]:

$$
I(t_0, \vec{\omega}) = \int_{t_0}^{\infty} Q(\tau) e^{\left(-\int_{t_0}^{t} \sigma_a(\tau) + \sigma_{sc}(\tau) \, d\tau\right)} \, d\tau
$$

Where the contribution $Q(t)$ at distance $t$ is [S2, book]:

$$
Q(t) = E(t) + \sigma_{sc}(t) \int_{\Omega} \rho_{sc}(\omega' \to \omega) I(t, \omega') \, d\omega'
$$

**Algorithm Steps**
Because the unknown $I(X, \omega)$ appears on both sides of the integral equation (inside the integral over $\omega'$), it cannot be solved directly in a single pass [S1, unofficial][S2, book]. The standard approach involves iterative solution methods or angular expansions [S1, unofficial]:

1.  **Initialization:** Initialize the intensity field $I^{(0)}(X, \omega)$ to zero or to the direct illumination term (single-scattering approximation).
2.  **Iteration:** For iteration $k = 1, 2, \dots$:
    a.  Compute the source term $g^{(k-1)}(s)$ using the intensity from the previous iteration $I^{(k-1)}$:
        $$
        g^{(k-1)}(s) = \int_{4\pi} r(X + s\omega, \omega, \omega') I^{(k-1)}(X + s\omega, \omega') \, d\omega' + e(X + s\omega)
        $$
    b.  Update the intensity $I^{(k)}(X, \omega)$ by integrating the source term along the ray:
        $$
        I^{(k)}(X, \omega) = I_0(X + D\omega, \omega) T(0, D) + \int_0^D g^{(k-1)}(s) T(s, D) \, ds
        $$
        where $T(s_1, s_2) = \exp\left(-\int_{s_1}^{s_2} \tau(X + t\omega) \, dt\right)$ is the transmittance.
3.  **Convergence:** Repeat until the change in $I$ between iterations is below a threshold, or a fixed number of bounces is reached.

**Pseudocode**
The following pseudocode illustrates the iterative solution for a single ray, assuming a discrete set of directions $\Omega_{discrete}$ for the angular integration.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_multiple_scattering_integral_equation(ray_origin, ray_direction, ray_length, 
                                                extinction_field, emission_field, 
                                                phase_function, background_radiance, 
                                                max_iterations, tolerance):
    """
    Solves the integral equation for multiple scattering iteratively.
    
    Args:
        ray_origin: 3D vector X
        ray_direction: 3D unit vector omega
        ray_length: float D
        extinction_field: function tau(X) returning extinction coefficient
        emission_field: function e(X) returning emission coefficient
        phase_function: function r(X, omega_in, omega_out) returning scattering probability
        background_radiance: function I0(X, omega) returning external illumination
        max_iterations: int
        tolerance: float
        
    Returns:
        radiance: float I(X, omega)
    """
    # 1. Initialize intensity to zero (or direct illumination for faster convergence)
    # Direct illumination term (single scattering from background)
    # T(0, D) is transmittance from origin to end of ray
    transmittance_total = compute_transmittance(ray_origin, ray_direction, ray_length, extinction_field)
    direct_term = background_radiance(ray_origin + ray_direction * ray_length, ray_direction) * transmittance_total
    
    current_intensity = direct_term # Initial guess
    
    # Discrete directions for angular integration (e.g., from a spherical grid)
    # In practice, this integral is approximated via quadrature
    discrete_directions = get_spherical_quadrature_points() 
    
    for iteration in range(max_iterations):
        previous_intensity = current_intensity
        source_integral = 0.0
        
        # 2. Integrate along the ray (outer integral)
        # Sample points along the ray
        num_samples = 100 # Implementation detail: adaptive sampling preferred
        for s in linspace(0, ray_length, num_samples):
            sample_point = ray_origin + ray_direction * s
            
            # Compute transmittance from sample point to observer (origin)
            # Note: In the integral equation, the source at s is attenuated by the path from s to 0
            transmittance_sample_to_origin = compute_transmittance(sample_point, -ray_direction, s, extinction_field)
            
            # Compute source term g(s) (inner integral over 4pi)
            source_term_g = emission_field(sample_point)
            
            for omega_prime in discrete_directions:
                # Get intensity from previous iteration at this point and direction
                # Note: current_intensity is a field in this general case, 
                # but for a single ray trace, we assume we have access to I(X, omega') from other rays or a global field.
                # Here we assume a global intensity field is maintained.
                intensity_at_sample = get_intensity_from_field(sample_point, omega_prime, previous_intensity)
                
                # Add scattered contribution
                scattering_prob = phase_function(sample_point, omega_prime, ray_direction)
                source_term_g += scattering_prob * intensity_at_sample
            
            # Accumulate weighted by transmittance and differential length ds
            ds = ray_length / num_samples
            source_integral += source_term_g * transmittance_sample_to_origin * ds
            
        # 3. Update intensity
        new_intensity = direct_term + source_integral
        
        # 4. Check convergence
        if abs(new_intensity - previous_intensity) < tolerance:
            return new_intensity
            
        current_intensity = new_intensity
        
    return current_intensity

def compute_transmittance(start_point, direction, distance, extinction_field):
    """Computes exp(-integral of extinction along path)."""
    # Simplified: assume constant extinction over small steps or use midpoint rule
    # For exact implementation, integrate tau along the line
    mid_point = start_point + direction * (distance / 2.0)
    avg_extinction = extinction_field(mid_point)
    return exp(-avg_extinction * distance)
```

**Parameters**
*   **Albedo ($\eta$):** Defined as $\eta = \sigma_s / \sigma_t$ (or $\mu_s / \mu_t$) [S3, unofficial]. High albedo implies strong multiple scattering.
*   **Mean Free Path ($l$):** $l = 1/\sigma_t$ [S3, unofficial].
*   **Phase Function:** Common choices include Henyey-Greenstein or Mie theory, defined by asymmetry parameter $g$ [S1, S3, unofficial].

**Edge Cases and Pitfalls**
*   **Circular Dependency:** The unknown $I(X, \omega)$ appears on both sides of the equation, requiring iterative solvers or angular expansions (like P-N method) [S1, unofficial]. Direct evaluation is impossible without approximation [S2, book].
*   **Computational Complexity:** The inner integral over $4\pi$ makes the evaluation expensive. Discretizing the angular domain (Discrete Ordinates) or expanding in Spherical Harmonics are common approximations [S1, unofficial].
*   **Precision:** In highly scattering media (high albedo), convergence of iterative methods can be slow.
*   **Boundary Conditions:** Accurate specification of $I_0$ (background illumination) is critical, especially for thin volumes where background light dominates [S1, unofficial].

## Subtopics

### [Full Propagator Approximation for Multiply-Scattered Radiance](1298-full-propagator-approximation-for-multiply-scattered-radianc.md)
Approximates the path integral by integrating over a Gaussian support perpendicular to the Most Probable Path (MPP), reducing the intractable integration over complete path space.

### [P-N Method (Spherical Harmonics)](1395-p-n-method-spherical-harmonics.md)
Expands directional radiance into a truncated spherical harmonics basis, converting the integral equation into a coupled system of PDEs for the coefficients, handling anisotropic scattering better than diffusion.

### [Monte Carlo Method for Multiple Scattering](1940-monte-carlo-method-for-multiple-scattering.md)
Solves the integral equation stochastically by tracing random walks, addressing visibility problems in dense media where deterministic integration is too costly.

### [Diffusion Approximation](2372-diffusion-approximation.md)
Replaces the integral RTE with a PDE based on the first two angular moments (irradiance and flux), assuming near-isotropic radiance in highly scattering media.

### Neural Network RTE Simplification via Albedo Series
Simplifies the classical radiation transfer equation by expressing the co-scattering term $F(p, \omega)$ as an infinite series of albedo powers $\eta^i$, enabling prediction via neural networks. The co-scattering term is defined as $F(p, \omega) = \sum_{i=0}^{\infty} \eta^i F_i(p, \omega)$, where $F_0 = f_p T I$ and $F_{i+1} = \Pi F_i$, with $\Pi$ being an integral operator involving transmittance $T$ and phase function $f_p$ [S3, unofficial]. This allows the complex integral equation to be approximated by a neural network predicting the series coefficients, leveraging the pre-calculated sampling features (density, transmittance, phase) [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** The direct evaluation of the integral equation is $O(N \cdot M)$, where $N$ is the number of spatial samples along the ray and $M$ is the number of discrete directions in the angular integration. Iterative solutions multiply this by the number of iterations required for convergence.
*   **Memory Complexity:** Requires storage of volumetric fields for extinction, emission, and potentially the full angular radiance field if solving globally.
*   **Strengths:** Physically accurate; captures complex lighting effects like color bleeding and subsurface scattering in dense media; serves as the ground truth for approximations.
*   **Limitations:** Computationally expensive due to the nested integrals (spatial and angular); requires iterative solvers which can be slow to converge for high-albedo media; difficult to implement efficiently on GPUs without approximations [S2, book].
*   **When to Use:** When physical accuracy is paramount and computational resources allow for iterative or expanded solutions (e.g., offline rendering, scientific visualization).
*   **When Not to Use:** For real-time applications with high-albedo media, where approximations like Diffusion or Neural Network simplifications are preferred [S3, unofficial].

## Background
The integral equation for radiation transport originates from thermal engineering and neutron transport theory, where it is used to model the flow of energy or particles through absorbing and scattering media [S1, unofficial]. In computer graphics, it was adapted as a physically based illumination model to accurately simulate light transport in participating media, such as fog, clouds, and subsurface scattering materials [S2, book]. The formulation is attributed to the general theory of radiative transfer, with specific applications in volume rendering discussed by authors like Kajiya and Von Herzen, who explored spherical harmonic expansions to solve the equation [S1, unofficial]. The model is distinct from the differential form of the RTE (Radiative Transfer Equation) in that it expresses the solution as a path integral rather than a differential balance, making it suitable for global illumination techniques that trace paths [S1, unofficial][S2, book].

## Sources
[S1] Mathematical models for volume rendering and neutron transport, https://digital.library.unt.edu/ark:/67531/metadc1274949/?q=volume rendering equation, fetched 2026-10-28 (unofficial)
[S2] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S3] A real-time rendering method for high albedo anisotropic materials with multiple scattering, https://doi.org/10.48550/arxiv.2401.14051, fetched 2026-10-26 (unofficial)
