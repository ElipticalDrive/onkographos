# Spectral+NEE MIS (Shadow Connection Routine)

Also called: MIS for chromatic media, Spectral MIS, Spectral Multiple Importance Sampling (MIS), Spectral Multiple Importance Sampling with Next-Event Estimation, Spectral NEE with MIS

## Overview
Spectral+NEE MIS is a shadow connection routine used in volumetric path tracing to estimate direct illumination in chromatic participating media [S2, unofficial]. It reduces variance by combining unidirectional path sampling and next-event estimation (NEE) using Multiple Importance Sampling (MIS) across spectral samples [S1]. Unlike standard spectral MIS, which can suffer from high variance or color noise in heterogeneous media, this method uses ratio tracking to handle null scattering events, allowing robust connections through dense or thin media while maintaining color fidelity [S2, unofficial].

## How it works
### Inputs and Outputs
- **Inputs**:
  - Current path vertex position $x$ and incoming direction $\omega$ [S2, unofficial].
  - Selected component index $\tilde{c}$ for distance sampling [S2, unofficial].
  - Current path contribution accumulator $f$ (vector of spectral values) [S2, unofficial].
  - Current path probability density function (pdf) $p$ [S2, unofficial].
- **Outputs**:
  - Updated path contribution $f$ [S2, unofficial].
  - Updated path pdf $p$ [S2, unofficial].
  - MIS weight for combining unidirectional and NEE estimators [S2, unofficial].

### Data Structures
- **Spectral Vector**: A vector of size $n$ (number of spectral samples) representing radiance or contribution at each wavelength [S2, unofficial].
- **Extinction Coefficients**: $\mu_t^c(x)$ for each component $c \in \{1, \dots, N\}$ at position $x$ [S2, unofficial].
- **Null Scattering Coefficients**: $\mu_n^c(x)$ for each component $c$ [S2, unofficial].
- **Phase Function**: $\rho_m(\omega, x, \omega')$ for the medium [S2, unofficial].

### Steps
1. **Sample Light Direction**: Sample a direction $\omega'$ toward the light source and compute its pdf $p_{\omega'}$ [S2, unofficial].
2. **Update Contributions**: Update the path contribution accumulator $f$ by multiplying with the medium phase function value $\rho_m(\omega, x, \omega')$ [S2, unofficial].
3. **Initialize PDFs**:
   - Initialize the next-event path pdf $p_{nee}$ as $p \cdot p_{\omega'}$ [S2, unofficial].
   - Initialize the unidirectional path pdf $p_{uni}$ as $p \cdot \rho_m(\omega, x, \omega')$ [S2, unofficial].
4. **Ratio Tracking Loop**: Begin ratio tracking along the shadow ray in direction $\omega'$ [S2, unofficial].
   - Sample distance $t$ using the extinction coefficient of the selected component $\tilde{c}$:
     $$ t = -\frac{\ln(1-\xi)}{\mu_t^{\tilde{c}}} $$
     where $\xi \in [0, 1)$ is a uniform random number [S2, unofficial].
   - Intersect the ray at distance $t$ to find point $y$ [S2, unofficial].
   - **If surface hit**: If $y$ exists (surface hit), return the average of $p_{uni}$ and $p_{nee}$ (2N techniques) [S2, unofficial].
   - **If no hit**:
     - Update contributions $f$ with null-scattering transmittance for all components $c$:
       $$ f \leftarrow f \circ \langle T_c^1(t)\mu_n^1(x), \dots, T_c^N(t)\mu_n^N(x) \rangle $$
       where $\circ$ denotes component-wise multiplication and $T_c^k(t)$ is the transmittance for component $k$ over distance $t$ [S2, unofficial].
     - Update $p_{nee}$ with extinction/transmittance ratios for all components [S2, unofficial].
     - Update $p_{uni}$ with null-scattering ratios for all components [S2, unofficial].
     - Repeat loop [S2, unofficial].

### Formulas
- **MIS Weight**: The balance heuristic weight for wavelength sample $\lambda_1$ is:
  $$ w_{\lambda_1}(x) = \frac{p_{\lambda_1}(x)}{\sum_{i=1}^{n} p_{\lambda_i}(x)} $$
  [S1]
- **Estimator**: The combined estimator for spectral samples is:
  $$ \frac{p_{\lambda_1}(x)}{\frac{1}{n} \sum_{i=1}^{n} p_{\lambda_i}(x)} \left[ \frac{f_{\lambda_1}(x)}{p_{\lambda_1}(x)}, \frac{f_{\lambda_2}(x)}{p_{\lambda_1}(x)}, \dots, \frac{f_{\lambda_n}(x)}{p_{\lambda_1}(x)} \right] = \frac{\left[ f_{\lambda_1}(x), f_{\lambda_2}(x), \dots, f_{\lambda_n}(x) \right]}{\frac{1}{n} \sum_{i=1}^{n} p_{\lambda_i}(x)} $$
  [S1]

### Parameters
- **$n$**: The number of spectral samples [S1].
- **$q$**: The discrete probability of sampling using wavelength $\lambda_1$, uniform at $1/n$ [S1].

### Edge Cases and Pitfalls
- **High Variance in Heterogeneous Media**: Estimating every component separately can lead to color noise, which the MIS combination remedies [S2, unofficial].
- **Extreme Variance**: Sampling distances according to a single component with low extinction can lead to extreme variance if other components have much higher extinction, which ratio tracking mitigates [S2, unofficial].
- **Surface Hit**: If a surface is hit, the routine returns the average of the two path pdfs, effectively combining 2N techniques [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spectral_nee_mis_shadow_connection(x, omega, c_tilde, f, p, n_spectral_samples):
    """
    Perform Spectral+NEE MIS shadow connection.
    
    Args:
        x: Current vertex position.
        omega: Incoming direction.
        c_tilde: Selected component index for distance sampling.
        f: Current path contribution (vector of size n_spectral_samples).
        p: Current path pdf.
        n_spectral_samples: Number of spectral samples.
        
    Returns:
        f: Updated path contribution.
        p: Updated path pdf.
        weight: MIS weight.
    """
    # 1. Sample light direction
    omega_prime, p_omega_prime = sample_light_dir(x)
    
    # 2. Update path contributions with phase function
    rho_m = medium_phase_function(omega, x, omega_prime)
    f = f * rho_m
    
    # 3. Initialize PDFs
    p_nee = p * p_omega_prime
    p_uni = p * rho_m
    
    # 4. Ratio tracking loop
    while True:
        # Sample distance using selected component
        xi = random_uniform()
        mu_t_c_tilde = extinction_coefficient(x, c_tilde)
        t = -log(1 - xi) / mu_t_c_tilde
        
        # Intersect ray
        y = intersect_ray(x, omega_prime, t)
        
        # If surface hit, return average of pdfs
        if y is not None:
            weight = 0.5 * (p_uni + p_nee)
            return f, p, weight
        
        # Update contributions with null-scattering transmittance
        null_scattering_factors = []
        for c in range(1, n_spectral_samples + 1):
            T_c_t = transmittance(x, omega_prime, t, c)
            mu_n_c = null_scattering_coefficient(x, c)
            null_scattering_factors.append(T_c_t * mu_n_c)
        f = f * null_scattering_factors
        
        # Update p_nee with extinction/transmittance ratios
        p_nee_ratios = []
        for c in range(1, n_spectral_samples + 1):
            mu_t_c = extinction_coefficient(x, c)
            T_c_t = transmittance(x, omega_prime, t, c)
            # Ratio tracking update for NEE pdf [G]
            ratio = mu_t_c / (mu_t_c_tilde * T_c_t)
            p_nee_ratios.append(ratio)
        p_nee = p_nee * p_nee_ratios
        
        # Update p_uni with null-scattering ratios
        p_uni_ratios = []
        for c in range(1, n_spectral_samples + 1):
            mu_n_c = null_scattering_coefficient(x, c)
            T_c_t = transmittance(x, omega_prime, t, c)
            # Ratio tracking update for unidirectional pdf [G]
            ratio = mu_n_c / (mu_t_c_tilde * T_c_t)
            p_uni_ratios.append(ratio)
        p_uni = p_uni * p_uni_ratios

def sample_light_dir(x):
    """Sample direction toward light source and return pdf."""
    # Standard light sampling strategy
    return (vec3(0.0), 1.0)

def medium_phase_function(omega, x, omega_prime):
    """Compute medium phase function value."""
    # Standard phase function evaluation
    return 1.0

def extinction_coefficient(x, c):
    """Get extinction coefficient for component c at x."""
    return 1.0

def null_scattering_coefficient(x, c):
    """Get null scattering coefficient for component c at x."""
    return 0.0

def transmittance(x, omega, t, c):
    """Compute transmittance for component c over distance t."""
    mu_t = extinction_coefficient(x, c)
    return exp(-mu_t * t)

def intersect_ray(x, omega, t):
    """Intersect ray at distance t. Returns point or None."""
    # Standard ray-scene intersection
    return None

def random_uniform():
    """Return uniform random number in [0, 1)."""
    return 0.5

def log(x):
    """Natural logarithm."""
    import math
    return math.log(x)

def exp(x):
    """Exponential function."""
    import math
    return math.exp(x)
```

## Subtopics

### Spectral MIS with Unidirectional Path Tracing
A sibling method that uses spectral MIS with unidirectional path tracing but without the NEE shadow connection routine. It is chosen when NEE is not required or when unidirectional sampling is sufficient. [Spectral MIS with Unidirectional Path Tracing](1964-spectral-mis-with-unidirectional-path-tracing.md)

## Operating characteristics
- **Time Complexity**: The ratio tracking loop length varies depending on the medium density and extinction coefficients [S2, unofficial].
- **Memory Complexity**: Requires storage for spectral vectors and extinction/null scattering coefficients for each component [S2, unofficial].
- **Measured Performance**: In a test scene (Fig. 9 right plume), Spectral+NEE MIS achieved RMSE: 0.016 and LTUV: 0.08M lookups, outperforming Spectral MIS (RMSE: 0.029, LTUV: 1.09M) [S2, unofficial].
- **Strengths**: Reduces variance significantly in chromatic media and handles both back-lit thin media and side-lit dense media robustly [S2, unofficial].
- **Limitations**: Can be computationally expensive due to the ratio tracking loop and spectral calculations [S2, unofficial].
- **When to Use**: When rendering chromatic volumetric media with complex lighting and where variance reduction is critical [S1].
- **When Not to Use**: When computational resources are limited and approximate results are acceptable, or when the medium is non-chromatic [G].

## Background
- **Origin**: The method is part of the VolPathIntegrator, which uses multiple importance sampling for direct illumination in volumetric media [S1].
- **History**: The specific combination of spectral MIS with NEE and ratio tracking was integrated into null-scattering path integral formulations for light transport to handle heterogeneous media and reduce color noise [S2, unofficial].
- **Usage**: Implemented in the VolPathIntegrator [S1].

## Sources
[S1] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
[S2] A Null-Scattering Path Integral Formulation of Light Transport, https://www.scribd.com/document/1067625811/A-Null-scattering-Path-Integral-Formulation-of-Light-Transport, fetched 2026-10-29 (unofficial)
