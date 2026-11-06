# Absorption Attenuation

Also called: Absorption Coefficient Model, Absorption Coefficient, Absorption Model

## Overview
Absorption Attenuation describes the exponential decay of radiance as light travels through a participating medium where particles absorb energy from the ray without scattering or emitting [S2, unofficial]. It solves the problem of calculating the remaining light intensity after traversing a distance $d$ in a volume with a given absorption coefficient [S2, unofficial]. Unlike the Prism Model, which approximates volume interaction via geometric primitives, or X-ray simulation which may account for transmission through dense matter, this method strictly models the probabilistic loss of photons due to absorption [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: Radiance $L(p, \omega)$ at position $p$ in direction $\omega$; Absorption coefficient $\sigma_a(p, \omega)$ (or $\kappa(x, \omega)$); Distance $d$ along the ray.
*   Output: Radiance $L(p+d\omega, \omega)$ at the end of the segment.

**Data Structures:**
The absorption coefficient is typically stored as a scalar field (e.g., a 3D texture or grid) where each voxel stores the value $\sigma_a$ or $\kappa$ at that spatial location [S1, S2, unofficial].

**Formulas:**
The differential change in radiance $dL$ over a differential distance $dt$ is defined as [S2, unofficial]:
$$ dL(p, \omega) = -\sigma_a(p, \omega) L(p, \omega) dt $$
where $\sigma_a(p, \omega)$ is the absorption coefficient representing the rate of attenuation per unit distance [S2, unofficial]. Alternatively, using $\kappa$ notation, the change is [S1, unofficial]:
$$ dL(x, \omega) = -\kappa(x, \omega) L(x, \omega) ds $$
where $\kappa(x, \omega)$ is the probability of a photon being absorbed at $x$ in direction $\omega$ per unit length [S1, unofficial].

The solution to this differential equation yields the transmittance (remaining light fraction) after traveling distance $d$ [S2, unofficial]:
$$ \frac{L(p+d\omega, \omega)}{L(p, \omega)} = e^{-\int_0^d \sigma_a(p(t), \omega) dt} $$
where $p(t) = p(0) + t\omega$ [S2, unofficial].

The optical depth $\tau(s)$ is defined as the integral of the absorption coefficient along the path [S1, unofficial]:
$$ \tau(s) = \int_0^s \kappa(x + t\omega, \omega) dt $$
Thus, the outgoing radiance is [S1, unofficial]:
$$ L(x + s\omega, \omega) = e^{-\tau(s)} L(x, \omega) $$

**Parameters:**
*   $\sigma_a$ or $\kappa$: Absorption coefficient. Typical values depend on the medium density and material properties. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def absorption_attenuation(L_in, sigma_a_field, p_start, direction, distance):
    """
    Calculate radiance after absorption attenuation.
    
    Args:
        L_in: Float, incoming radiance at p_start.
        sigma_a_field: Function or data structure providing sigma_a(p, direction).
        p_start: Vector3, starting position.
        direction: Vector3, normalized ray direction.
        distance: Float, total distance to travel.
        
    Returns:
        Float, outgoing radiance.
    """
    # Calculate optical depth tau by integrating sigma_a along the ray
    # For uniform medium, tau = sigma_a * distance
    # For non-uniform, numerical integration (e.g., trapezoidal rule) is required.
    
    # Example for uniform absorption coefficient sigma_a_const:
    # tau = sigma_a_const * distance
    
    # General case: numerical integration
    num_steps = 10 # Implementation detail [G]
    tau = 0.0
    dt = distance / num_steps
    for i in range(num_steps):
        t = i * dt
        p = p_start + direction * t
        sigma_a = sigma_a_field.sample(p, direction)
        tau += sigma_a * dt
        
    # Apply Beer-Lambert law
    transmittance = exp(-tau)
    L_out = L_in * transmittance
    
    return L_out
```

**Edge Cases and Pitfalls:**
*   If $\sigma_a$ is zero, transmittance is 1.0 (no attenuation).
*   If $\sigma_a$ is very large, transmittance approaches 0.0.
*   Precision issues may arise if $\tau$ is extremely large, causing underflow to zero.

## Subtopics

### Optical Depth
A measure of how far light travels before being absorbed, defined as the integral of the absorption coefficient along the path [S1, unofficial].

### Transmittance
The fraction of light that remains after passing through the volume, calculated as $e^{-\tau}$ [S1, S2, unofficial].

### Opacity
The fraction of light blocked by the volume, calculated as $1 - T(s)$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of samples taken along the ray to integrate the absorption coefficient.
*   **Memory Complexity**: Depends on the storage of the absorption coefficient field.
*   **Strengths**: Physically accurate for non-scattering media; computationally simple if $\sigma_a$ is constant.
*   **Limitations**: Does not account for scattering or emission; requires integration along the ray.
*   **When to use**: When modeling media where absorption is the dominant light interaction (e.g., smoke, fog with negligible scattering, or X-ray attenuation).
*   **When not to use**: When scattering or emission is significant.

## Background
The absorption model is a fundamental component of light transport in participating media, described in standard computer graphics literature such as PBRT [S2, unofficial]. It is derived from the physical principle that light intensity decreases exponentially with distance in an absorbing medium. The concept is widely used in offline rendering and volume visualization.

## Sources
[S1] Volume Rendering, https://docslib.org/doc/1400821/volume-rendering, fetched 2026-10-28 (unofficial)
[S2] Volume Rendering in Offline Renderer · A Graphics Guy's Note, https://agraphicsguynotes.com/posts/volume_rendering_in_offline_renderer/, fetched 2026-10-28 (unofficial)
