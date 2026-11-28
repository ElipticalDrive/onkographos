# Analytical Single Scattering without Boundaries
Also called: Analytic Single Scattering, Semi-analytic solutions

## Overview
This method computes the exact single-scattered radiance for a homogeneous participating medium illuminated by a point or spot light, assuming no occlusion or boundary interactions [S2, unofficial]. It differs from numerical ray-marching approaches by evaluating a closed-form solution, eliminating sampling noise and discretization errors while enabling real-time performance [S1].

## How it works
The method evaluates the radiative transfer equation analytically for a single light path in an unbounded homogeneous medium [S3].

**Inputs:**
*   $x_t$: The 3D position of the scattering point.
*   $x_0$: The 3D position of the light source.
*   $\sigma_t$: The constant extinction coefficient of the medium.
*   $L(x_0, \omega)$: The incoming radiance at the path start (background or direct light if boundaries exist).
*   $L(\omega_i)$: The constant radiance of the point light source.
*   $p(\omega_i, \omega)$: The phase function value for the scattering angle between light direction $\omega_i$ and view direction $\omega$.

**Outputs:**
*   $L(x_t, \omega)$: The computed single-scattered radiance at $x_t$ in direction $\omega$.

**Data Structures:**
No complex data structures are required; the method relies on scalar arithmetic and vector distance calculations [G].

**Steps:**
1.  Compute the geometric distance $t = \|x_t - x_0\|$ [G].
2.  Compute the light direction $\omega_i = (x_t - x_0) / t$ [G].
3.  Evaluate the optical depth $\tau = t \cdot \sigma_t$ [G].
4.  Apply the closed-form radiance equation [S3].

**Formula:**
The radiance $L(x_t, \omega)$ is given by [S3]:
$$
L(x_t, \omega) = e^{-t\sigma_t} L(x_0, \omega) + p(\omega_i, \omega) L(\omega_i) \frac{1 - e^{-t\sigma_t}}{\sigma_t}
$$
Where:
*   $L(x_0, \omega)$ is the incoming radiance at the start of the light path [S3].
*   $t$ is the geometric distance between the light and the scattering point [S3].
*   $\sigma_t$ is the extinction coefficient [S3].
*   $p(\omega_i, \omega)$ is the phase function value [S3].
*   $L(\omega_i)$ is the constant light radiance for the path [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def analytical_single_scattering(x_t, x_light, sigma_t, L_bg, L_light, phase_func, omega_view):
    """
    Computes analytical single scattering for a point light in a homogeneous medium.
    Args:
        x_t: list/tuple of 3 floats, scattering point position.
        x_light: list/tuple of 3 floats, light source position.
        sigma_t: float, extinction coefficient.
        L_bg: float, incoming radiance at path start (background).
        L_light: float, point light radiance.
        phase_func: callable, returns p(omega_i, omega_view).
        omega_view: list/tuple of 3 floats, view direction.
    Returns:
        float, single-scattered radiance.
    """
    # 1. Geometric distance and light direction
    delta = [x_t[i] - x_light[i] for i in range(3)]
    t = math.sqrt(sum(d*d for d in delta))
    
    # Handle degenerate case where point coincides with light source
    if t < 1e-6:
        return L_bg
        
    omega_i = [d / t for d in delta]
    
    # 2. Optical depth and transmittance
    tau = t * sigma_t
    transmittance = math.exp(-tau)
    
    # 3. Scattering integral term: (1 - e^(-t*sigma_t)) / sigma_t
    # Handle zero extinction limit analytically to avoid division by zero
    if sigma_t < 1e-6:
        scattering_integral = t
    else:
        scattering_integral = (1.0 - transmittance) / sigma_t
        
    # 4. Phase function evaluation
    p_val = phase_func(omega_i, omega_view)
    
    # 5. Combine direct/background term and single scattering term
    direct_term = transmittance * L_bg
    scatter_term = p_val * L_light * scattering_integral
    
    return direct_term + scatter_term
```

**Edge Cases:**
*   **Point at Light Source:** If $t \approx 0$, the transmittance approaches 1 and the scattering integral approaches 0, yielding $L(x_0, \omega)$ [G].
*   **Zero Extinction:** If $\sigma_t = 0$, the medium is transparent. The term $\frac{1 - e^{-t\sigma_t}}{\sigma_t}$ approaches $t$ as $\sigma_t \to 0$ [G]. Implementation must check for $\sigma_t < \epsilon$ to prevent division by zero [G].
*   **Implementation Complexity:** The method is noted as computationally intensive and difficult to implement ("nightmare") due to complex phase functions and the complete lack of shadow support [S2, unofficial].

## Subtopics

### Analytical Ground Truth
An analytical solution to the radiative transfer equation used as ground truth for numerical error analysis [S3]. It is applicable only under specific conditions: a homogeneous medium (constant absorption and scattering, no emission), uniform lighting without shadows (e.g., infinite directional light with non-occluding geometry), and no multiple scattering [S3]. The formula is identical to the base method: $L(x_t, \omega) = e^{-t\sigma_t} L(x_0, \omega) + p(\omega_i, \omega) L(\omega_i) \frac{1 - e^{-t\sigma_t}}{\sigma_t}$ [S3]. It serves to validate numerical methods like differential ray marching by providing a noise-free reference [S3].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample/pixel, requiring only basic arithmetic and exponential functions [G].
*   **Memory Complexity:** $O(1)$, as no auxiliary buffers or data structures are required [G].
*   **Performance:** Capable of achieving real-time frame rates [S1].
*   **Strengths:** Produces noise-free results; provides an exact solution for the specified configuration; extremely fast evaluation [S1].
*   **Limitations:** Strictly limited to point or spot lights; assumes no occlusion or shadow casting; requires a homogeneous medium; implementation is noted as difficult and computationally heavy for complex phase functions [S2, unofficial].
*   **When to use:** Real-time visualization of point lights in homogeneous fog or atmospheric volumes where noise is unacceptable and boundaries are negligible [S1].
*   **When not to use:** Scenes with area lights, complex shadowing, heterogeneous media, or significant multiple scattering [S2, unofficial].

## Background
Sun et al. proposed this analytical solution for single scattering by participating media without boundaries under a point light [S1]. The approach is cited alongside work by Pegoraro et al. [S2, unofficial]. It has been applied to real-time atmospheric cloud rendering and serves as a reference solution for error analysis in differential ray marching techniques [S1, S3].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S2] Participating media, https://imaging.cs.cmu.edu/15-468/2024_spring/lectures/lecture_12.pdf, fetched 2026-10-29 (unofficial)
[S3] Differential Ray Marching, https://doi.org/10.2312/localchapterevents/ceig/ceig12/091-098, fetched 2026-10-26
