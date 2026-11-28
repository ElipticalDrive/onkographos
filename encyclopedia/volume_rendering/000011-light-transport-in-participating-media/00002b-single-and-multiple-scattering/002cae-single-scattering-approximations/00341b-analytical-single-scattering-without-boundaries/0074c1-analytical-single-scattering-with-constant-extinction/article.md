# Analytical Single Scattering with Constant Extinction

Also called: Analytical solution assuming constant extinction, Constant extinction analytical integration

## Overview
This method computes single scattering contributions for volumetric particles by assuming constant extinction ($\sigma_t$) and full visibility ($Vis=1$) within a tetrahedron, enabling closed-form integration of the scattering equation [S1, unofficial]. It differs from variable extinction methods (e.g., Simpson's rule) by trading physical accuracy for computational simplicity, though it introduces visible geometric artifacts at tetrahedron boundaries [S1, unofficial].

## How it works
**Inputs:**
- Tetrahedron vertices defining the integration interval $[z_{min}, z_{max}]$ along a view ray.
- Light irradiance $R$.
- Particle albedo $\rho$.
- Phase function $p(L, V)$.
- Light transmittance optical depths at interval endpoints: $\tau_{min} = \tau_L(z_{min})$ and $\tau_{max} = \tau_L(z_{max})$.
- Constant extinction coefficient $\sigma$ (assumed uniform throughout the tetrahedron).

**Outputs:**
- Scattering coefficient $C_{scat}$.
- View transmittance $A = T_V(z_{min}, z_{max})$.

**Data Structures:**
No complex data structures are required; the method operates on scalar values derived from the ray-tetrahedron intersection and precomputed optical depths.

**Steps:**
1. Calculate the light transmittance approximation $T_L(z)$ assuming linear optical depth variation between $z_{min}$ and $z_{max}$:
   $$ T_L(z) = e^{\left(-\tau_{min} - \frac{(\tau_{max} - \tau_{min})(z - z_{min})}{z_{max} - z_{min}}\right)} $$
   where $\tau_{min} = \tau_L(z_{min})$ and $\tau_{max} = \tau_L(z_{max})$ [S1, unofficial].

2. Compute the scattering coefficient $C_{scat}$ using the analytical integration result for constant extinction $\sigma$ and $Vis(z)=1$:
   $$ C_{scat} = R\rho \times p(L, V) \times \frac{\sigma(z_{max} - z_{min})(e^{(\sigma z_{max} + \tau_{max})} - e^{(\sigma z_{min} + \tau_{min})})e^{(-\sigma z_{max} - \tau_{max} - \tau_{min})}}{\sigma z_{max} - \sigma z_{min} + \tau_{max} - \tau_{min}} $$
   [S1, unofficial].

3. Compute the view transmittance $A$ across the interval:
   $$ A = T_V(z_{min}, z_{max}) = e^{-\sigma(z_{max} - z_{min})} $$
   [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def analytical_constant_extinction_scattering(
    z_min: float,
    z_max: float,
    tau_min: float,
    tau_max: float,
    sigma: float,
    R: float,
    rho: float,
    phase_function: float
) -> tuple[float, float]:
    """
    Computes scattering coefficient and view transmittance assuming constant extinction.
    
    Args:
        z_min: Entry point of ray into tetrahedron.
        z_max: Exit point of ray from tetrahedron.
        tau_min: Light optical depth at z_min.
        tau_max: Light optical depth at z_max.
        sigma: Constant extinction coefficient.
        R: Light irradiance.
        rho: Particle albedo.
        phase_function: Phase function value p(L, V).
        
    Returns:
        (C_scat, A): Scattering coefficient and view transmittance.
    """
    # Handle degenerate case where z_min == z_max to avoid division by zero
    if z_max == z_min:
        return 0.0, 1.0

    # Denominator for the scattering integral
    denom = sigma * (z_max - z_min) + (tau_max - tau_min)
    
    # Numerator components
    # Term 1: sigma * (z_max - z_min)
    term1 = sigma * (z_max - z_min)
    
    # Term 2: exp(sigma * z_max + tau_max) - exp(sigma * z_min + tau_min)
    term2 = math.exp(sigma * z_max + tau_max) - math.exp(sigma * z_min + tau_min)
    
    # Term 3: exp(-sigma * z_max - tau_max - tau_min)
    term3 = math.exp(-sigma * z_max - tau_max - tau_min)
    
    # Calculate C_scat
    # C_scat = R * rho * p(L,V) * (term1 * term2 * term3) / denom
    if denom == 0:
        # Fallback for degenerate denominator, though physically unlikely with non-zero sigma
        C_scat = 0.0
    else:
        C_scat = R * rho * phase_function * (term1 * term2 * term3) / denom

    # Calculate View Transmittance A
    A = math.exp(-sigma * (z_max - z_min))
    
    return C_scat, A
```

**Edge Cases and Pitfalls:**
- **Visible Edges:** Assuming constant extinction fills the entire tetrahedron with participating medium, causing sharp geometric edges to be visible. This artifact does not exist in real smoke or clouds [S1, unofficial].
- **Division by Zero:** If $\sigma(z_{max} - z_{min}) + (\tau_{max} - \tau_{min}) = 0$, the denominator in the $C_{scat}$ formula becomes zero. This requires a guard clause or limit evaluation [G].

## Subtopics

### Hiding of Tetrahedron Edges
A follow-up technique that fades extinction exponentially from the tetrahedron center to hide the geometric artifacts caused by constant extinction [S1, unofficial]. See [Analytical Single Scattering with Variable Extinction](7211-analytical-single-scattering-with-variable-extinction.md).

## Operating characteristics
- **Time Complexity:** $O(1)$ per tetrahedron, as it involves only exponential and arithmetic operations.
- **Memory Complexity:** $O(1)$, requiring only scalar storage for optical depths and coefficients.
- **Strengths:** Trivial to compute and output in an interval shader; provides a cloud-like appearance with volumetric shadows [S1, unofficial].
- **Limitations:** Produces visible tetrahedron edges due to the uniform medium assumption, which is physically inaccurate for smoke or clouds [S1, unofficial].
- **When to Use:** When computational performance is critical and slight geometric artifacts are acceptable or can be mitigated by other means (e.g., edge fading).
- **When Not to Use:** When high physical accuracy is required, or when the underlying mesh geometry (tetrahedra) is likely to be visible.

## Background
This method was proposed by Sandilya Jandhyala in his dissertation on volumetric rendering of particles [S1, unofficial]. It serves as a baseline analytical solution before introducing more complex variable extinction models or numerical integration techniques like Simpson's rule [S1, unofficial].

## Sources
[S1] PDF Volumetric Rendering of Particle, https://jysandy.github.io/Sandilya-Jandhyala-Dissertation.pdf, fetched 2026-10-29 (unofficial)
