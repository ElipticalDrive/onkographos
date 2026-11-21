# In scattering
## Overview
In scattering accounts for the increase in radiance along a ray due to light scattering from other directions into the ray's current direction [S1, book]. It serves as the volumetric analog to the Bidirectional Scattering Distribution Function (BSDF), describing how radiance from the surrounding environment is redistributed into the viewing direction [S1, book]. Unlike out-scattering, which removes radiance from the ray, in-scattering adds radiance [S1, book].
## How it works
### Inputs and Outputs
**Inputs:**
*   Position $p$ in the medium [S1, book].
*   Outgoing direction $\omega$ (direction of the ray) [S1, book].
*   Incident radiance field $L_i(p, \omega_i)$ for all incoming directions $\omega_i \in \mathcal{S}^2$ [S1, book].
*   Phase function $p(p, \omega_i, \omega)$ [S1, book].
*   Medium coefficients: absorption $\sigma_a(p, \omega)$, scattering $\sigma_s(p, \omega)$, and total attenuation $\sigma_t(p, \omega) = \sigma_a(p, \omega) + \sigma_s(p, \omega)$ [S1, book].
*   Emitted radiance $L_e(p, \omega)$ [S1, book].

**Outputs:**
*   Added radiance per unit distance $dL_o^+(p, \omega)$ [S1, book].

### Data Structures
Not given in the sources.

### Steps
1.  **Verify Assumptions**: Inter-particle interactions can be ignored when describing scattering at a particular location if the separation between particles is at least a few times the lengths of their radii [S1, book].
2.  **Compute Source Function $L_s^+$**: Calculate the source function at position $p$ for direction $\omega$, which accounts for both volume emission and in-scattering [S1, book]:
    $$
    L_s^+(p, \omega) = \frac{\sigma_a(p, \omega)}{\sigma_t(p, \omega)} L_e(p, \omega) + \frac{\sigma_s(p, \omega)}{\sigma_t(p, \omega)} \int_{\mathcal{S}^2} p(p, \omega_i, \omega) L_i(p, \omega_i) d\omega_i
    $$
    The term $\frac{\sigma_s(p, \omega)}{\sigma_t(p, \omega)}$ is the single-scattering albedo, representing the probability of scattering versus absorption [S1, book].
3.  **Calculate Added Radiance**: Multiply the source function by the total attenuation coefficient and the differential distance $dt$ to obtain the total added radiance per unit distance [S1, book]:
    $$
    dL_o^+(p, \omega) = \sigma_t(p, \omega) L_s^+(p, \omega) dt
    $$

### Formulas
**Phase Function Normalization:**
The phase function $p(\omega, \omega')$ must satisfy the normalization constraint:
$$
\int_{\mathcal{S}^2} p(\omega, \omega') d\omega' = 1
$$
This constraint ensures that phase functions act as probability distributions for scattering into a particular direction [S1, book].

**Source Function:**
$$
L_s^+(p, \omega) = \frac{\sigma_a(p, \omega)}{\sigma_t(p, \omega)} L_e(p, \omega) + \frac{\sigma_s(p, \omega)}{\sigma_t(p, \omega)} \int_{\mathcal{S}^2} p(p, \omega_i, \omega) L_i(p, \omega_i) d\omega_i
$$
*   $L_s^+$: Source function (accounts for emission and in-scattering) [S1, book].
*   $\sigma_a$: Absorption coefficient [S1, book].
*   $\sigma_s$: Scattering coefficient [S1, book].
*   $\sigma_t$: Total attenuation coefficient ($\sigma_a + \sigma_s$) [S1, book].
*   $L_e$: Emitted radiance [S1, book].
*   $L_i$: Incident radiance [S1, book].
*   $\omega_i$: Incident direction [S1, book].
*   $p$: Phase function [S1, book].

**Added Radiance:**
$$
dL_o^+(p, \omega) = \sigma_t(p, \omega) L_s^+(p, \omega) dt
$$
*   $dL_o^+$: Total added radiance per unit distance [S1, book].
*   $dt$: Differential distance [S1, book].

### Parameters
*   **Single-Scattering Albedo**: Defined as $\sigma_s / \sigma_t$. Under radiometry assumptions, this value is always between 0 and 1 [S1, book]. It describes the probability of scattering versus absorption at a scattering event [S1, book].
*   **Mean Free Path**: Defined as $1 / \sigma_t(p, \omega)$ [S1, book]. This gives the average distance that a ray travels in a medium with attenuation coefficient $\sigma_t(p, \omega)$ before interacting with a particle [S1, book].

### Edge Cases and Pitfalls
*   **Inter-particle Interactions**: The model assumes particles are sufficiently separated (at least a few times their radii) so that inter-particle interactions can be ignored when describing scattering at a particular location [S1, book].
*   **Cosine Term Absence**: Unlike surface scattering equations (e.g., Equation 4.14 in [S1, book]), the in-scattering source function does not include a cosine term because the phase function operates on radiance rather than differential irradiance [S1, book].
*   **Normalization**: The phase function must be normalized to 1 over the sphere [S1, book].
*   **Absorption vs. Scattering**: If the single-scattering albedo is 1 (no absorption), light scatters many times inside the medium, giving it a substantially different appearance [S1, book].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def compute_in_scattering(p, omega, sigma_a, sigma_s, sigma_t, L_e, phase_function, incident_radiance_sampler, num_samples):
    """
    Computes the added radiance per unit distance due to in-scattering.
    
    Args:
        p: Position in the medium.
        omega: Outgoing direction.
        sigma_a: Absorption coefficient at p.
        sigma_s: Scattering coefficient at p.
        sigma_t: Total attenuation coefficient at p (sigma_a + sigma_s).
        L_e: Emitted radiance at p.
        phase_function: Function evaluating p(omega_i, omega).
        incident_radiance_sampler: Function that samples L_i(p, omega_i) for a given omega_i.
        num_samples: Number of Monte Carlo samples for the spherical integral.
        
    Returns:
        dL_o_plus: Added radiance per unit distance.
    """
    if sigma_t == 0:
        return 0.0
    single_scattering_albedo = sigma_s / sigma_t
    emission_contribution = (sigma_a / sigma_t) * L_e
    
    # The sources do not specify the integration method for the spherical integral.
    # Standard implementations use Monte Carlo sampling over S^2 [G].
    integral_sum = 0.0
    for _ in range(num_samples):
        omega_i = sample_uniform_sphere() # [G]
        integral_sum += phase_function(omega_i, omega) * incident_radiance_sampler(omega_i)
    spherical_integral = (4.0 * math.pi / num_samples) * integral_sum
    
    in_scattering_contribution = single_scattering_albedo * spherical_integral
    L_s_plus = emission_contribution + in_scattering_contribution
    
    # dL_o^+ = sigma_t * L_s^+ * dt
    # Returns added radiance per unit distance (dt = 1)
    return sigma_t * L_s_plus

def sample_uniform_sphere(): # [G]
    u1 = random.random()
    u2 = random.random()
    theta = 2.0 * math.pi * u1
    phi = math.acos(2.0 * u2 - 1.0)
    return (math.sin(phi) * math.cos(theta), math.sin(phi) * math.sin(theta), math.cos(phi))
```
## Subtopics

### Scattering Operator
An operator that acts on an incoming radiance field $g$ to produce a scattered outgoing radiance field, handling both surface scattering and volumetric scattering [S2, unofficial]. It is defined as $<\mathbf{K} g>(\bx, \bo) := \begin{cases} \displaystyle \int_{\mathcal{S}^2} f_s(\bx, \bo', \bo) g(\bx, \bo') |\vec{n} \cdot \bo'| \, d\bo' & \bx \in \mathcal{M} \\ \displaystyle \sigma_s(\bx) \int_{\mathcal{S}^2} f_p(\bx, -\bo', \bo) g(\bx, \bo') \, d\bo' & \bx \in \mathcal{V} \end{cases}$ [S2, unofficial]. Here $\mathcal{M}$ is the set of object surfaces, $\mathcal{V} := \Omega \setminus \mathcal{M}$ is the medium space, and $\Omega$ is the 3D simulation domain containing all surfaces and media [S2, unofficial].

## Operating characteristics
The computational cost is dominated by the spherical integral $\int_{\mathcal{S}^2} p(p, \omega_i, \omega) L_i(p, \omega_i) d\omega_i$ [S1, book]. Memory requirements are not given in the sources. The model accurately depicts the bright appearance of participating media like clouds and fog where light is scattered into the viewer's eye from other directions [S1, book].
## Background
In scattering is described in detail in "Physically Based Rendering: From Theory to Implementation" (PBR-book), specifically in Chapter 11 on Volume Scattering Processes [S1, book]. The concept is analogous to surface scattering but adapted for volumetric media, where the phase function replaces the BSDF [S1, book]. Cloud model examples illustrate the visual impact of in-scattering, particularly when absorption is negligible (single-scattering albedo of 1) [S1, book].
## Sources
[S1] 11.1 Volume Scattering Processes, https://pbr-book.org/4ed/Volume_Scattering/Volume_Scattering_Processes, fetched 2026-10-29 (book)
[S2] ボリュームレンダリング方程式(Volume Rendering Equation) 2, https://rayspace.xyz/notes/VLTE2/, fetched 2026-10-30 (unofficial)
