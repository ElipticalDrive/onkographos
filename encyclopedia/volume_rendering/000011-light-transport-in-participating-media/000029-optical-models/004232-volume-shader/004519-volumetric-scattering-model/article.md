# Volumetric Scattering Model

Also called: Participating Media Physics, Participating Medium Rendering

## Overview
Volumetric scattering models light interaction with participating media—substances like fog, smoke, clouds, and translucent solids where space participates in lighting [S2, unofficial][S3]. Unlike surface rendering where radiance is constant between surfaces, participating media modify radiance through emission, absorption, out-scattering, and in-scattering [S2, unofficial]. This method distinguishes itself by explicitly modeling light transport through volume using the radiative transfer equation, decomposed into extinction and in-scattering terms, rather than approximating volume effects via surface shaders [S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Ray origin $x$ and direction $\omega$ [S2, unofficial].
*   Volume definition (homogeneous or heterogeneous) providing spatially varying coefficients [S2, unofficial].
*   Light sources for direct in-scattering [S2, unofficial].
*   Environment map or indirect lighting cache for indirect in-scattering [S2, unofficial].
*   Phase function parameters (e.g., anisotropy $g$, wavelength $\lambda$) [S2, unofficial].

**Outputs:**
*   Radiance $L(x, \omega)$ accumulated along the ray segment [S2, unofficial].

### Data Structures
The implementation relies on two primary abstractions [S2, unofficial]:
1.  **Volume Class**:
    *   `evaluate(x)`: Returns a struct containing absorption coefficient $\sigma_a(x)$, scattering coefficient $\sigma_s(x)$, emission $E(x)$, and a phase function object.
    *   `intersect(ray)`: Returns entry and exit distances or a bounding volume test.
2.  **Phase Function Class**:
    *   `evaluate(cos_theta)`: Returns probability density value.
    *   `sample(cos_theta)`: Returns a sampled direction and PDF.

### Formulas and Definitions
The change in radiance $dL$ along a differential path length $ds$ is governed by four processes [S2, unofficial]:

1.  **Emission**:
    $$ dL(x, \omega) = \frac{E(x) ds}{4\pi} $$
    Where $E(x)$ is isotropic emission per unit length [S2, unofficial].

2.  **Absorption**:
    $$ dL(x, \omega) = -\sigma_a(x) L(x, \omega) ds $$
    Where $\sigma_a$ is the absorption coefficient in units of $1/m$ [S2, unofficial].

3.  **Out-Scattering**:
    $$ dL(x, \omega) = -\sigma_s(x) L(x, \omega) ds $$
    Where $\sigma_s$ is the scattering coefficient [S2, unofficial].

4.  **Extinction**:
    Combining absorption and out-scattering:
    $$ \sigma_t(x) = \sigma_a(x) + \sigma_s(x) $$
    $$ dL(x, \omega) = -\sigma_t(x) L(x, \omega) ds $$

5.  **Transmittance**:
    The fraction of light surviving distance $s$:
    $$ T(s) = e^{-\tau(s)} = e^{-\int_0^s \sigma_t(x + s'\omega) ds'} $$
    For homogeneous media, $\tau(s) = \sigma_t s$ [S2, unofficial].

6.  **In-Scattering**:
    Light scattered *into* the ray direction from other directions $\omega_i$:
    $$ dL(x, \omega) = \sigma_s(x) ds \int_{4\pi} p(\omega_i \cdot \omega) L(x, \omega_i) d\omega_i $$
    Where $p(\cos \theta)$ is the phase function [S2, unofficial].

### Phase Functions
The phase function $p(\cos \theta)$ describes the angular distribution of scattered light. It must be reciprocal and energy-conserving [S2, unofficial].

*   **Isotropic**:
    $$ p(\cos \theta) = \frac{1}{4\pi} $$
    Scatters equally in all directions [S2, unofficial].

*   **Henyey-Greenstein**:
    $$ p(\cos \theta) = \frac{1 - g^2}{4\pi (1 + g^2 - 2g\cos \theta)^{1.5}} $$
    Uses anisotropy parameter $g \in [-1, 1]$. $g=0$ is isotropic; $g>0$ is forward-scattering; $g<0$ is back-scattering [S1, unofficial].

*   **Rayleigh** (for particles $d \ll \lambda$):
    $$ p(\cos \theta) = \frac{3}{16\pi} (1 + \cos^2 \theta) $$
    Scattering coefficient: $\sigma_s = \frac{2\pi^5 d^6}{3 \lambda^4} \frac{(n^2 - 1)^2}{(n^2 + 2)^2}$ [S2, unofficial].

*   **Empirical Mie Approximation**:
    $$ p(\cos \theta) = \frac{1}{4\pi} \frac{z+1}{2} \left( \frac{1 + \cos \theta}{2} \right)^z $$
    Controls forward scattering sharpness via parameter $z$ (e.g., $z=8, 32$) [S2, unofficial].

### Algorithm: Ray Marching
The integral form of the radiative transfer equation is solved numerically via ray marching [S2, unofficial].

1.  **Discretization**: Divide the ray segment inside the volume into finite segments.
2.  **Step Size**: Use a maximum step size $\Delta s_{max}$. Each actual step taken is a random number from $0$ to $1$ times the maximum step size [S2, unofficial].
3.  **Accumulation**: For each segment, compute the change in radiance due to emission, absorption, out-scattering, and in-scattering [S2, unofficial].
4.  **In-Scattering Approximation**:
    $$ L_i(x, \omega) \approx \frac{4\pi \sigma_s(x)}{N} \sum_{N} p(\omega_i \cdot \omega) L(x, \omega_i) $$
    Computed by tracing shadow rays to light sources (direct) and generating sample scattering rays traced into the environment (indirect) [S2, unofficial].
5.  **Shadow Ray Calculation**: Trace a ray to the light. First test for opaque surfaces. If nothing blocks, test for volumes. If the ray passes through volume, attenuate light by extinction, resulting in a $[0, 1]$ scale factor. For homogeneous volumes, attenuate by total length. For inhomogeneous volumes, ray march through volume applying local extinction to each segment [S2, unofficial].
6.  **Update Rule**:
    $$ L(x + s\omega, \omega) \approx e^{-\sigma_t(x) s} L(x, \omega) + E(x) s + L_i(x) s $$
    Absorption and out-scattering are combined into extinction. Emission is approximated as emission at point $x$ times length $s$. In-scattering at point $x$ is represented by $L_i(x)$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def ray_march_volume(ray_origin, ray_direction, volume, light_sources, env_map, max_step=0.1, num_samples=16):
    # 1. Find entry and exit distances along the ray
    t_entry, t_exit = volume.intersect(ray_origin, ray_direction)
    if t_entry is None:
        return 0.0

    t_current = t_entry
    current_radiance = 0.0  # Background radiance [G]
    PI = math.pi

    # 2. Ray march through the volume
    while t_current < t_exit:
        # 3. Actual step is a random number from 0 to 1 times the maximum step size
        step_size = random.uniform(0.0, 1.0) * max_step

        # Clamp step to not exceed volume exit
        remaining = t_exit - t_current
        if step_size > remaining:
            step_size = remaining

        current_pos = ray_origin + t_current * ray_direction
        vol_props = volume.evaluate(current_pos)
        sigma_a = vol_props.sigma_a
        sigma_s = vol_props.sigma_s
        sigma_t = sigma_a + sigma_s
        emission = vol_props.emission

        # 4. Compute extinction (transmittance) for this segment
        tau = sigma_t * step_size
        segment_transmittance = math.exp(-tau)

        # 5. Compute In-Scattering Li(x) via Monte Carlo approximation
        # Li(x, ω) ≈ (4π σ_s(x) / N) Σ p(ωi · ω) L(x, ωi)
        direct_sum = 0.0
        indirect_sum = 0.0

        for _ in range(num_samples):
            # Direct in-scattering: trace shadow rays to light sources
            for light in light_sources:
                shadow_dist = volume.intersect(current_pos, light.direction)
                if shadow_dist is not None and shadow_dist < light.distance:
                    # Attenuate light by extinction through volume
                    light_atten = math.exp(-sigma_t * shadow_dist)
                    direct_sum += light.intensity * light_atten

            # Indirect in-scattering: generate sample scattering rays traced into environment
            env_dir = env_map.sample_direction(current_pos)
            indirect_sum += env_map.sample(current_pos, env_dir)

        # Evaluate phase function p(ωi · ω) for sampled directions
        phase_val_direct = vol_props.phase_function.evaluate(ray_direction, light_sources[0].direction)
        phase_val_indirect = vol_props.phase_function.evaluate(ray_direction, env_dir)

        li_x = (4.0 * PI * sigma_s / num_samples) * (
            phase_val_direct * direct_sum + phase_val_indirect * indirect_sum
        )

        # 6. Update radiance using discrete approximation
        # L(x + sω) ≈ e^(-σ_t s) L(x) + E(x) s + Li(x) s
        emission_contrib = emission * step_size
        in_scattering_contrib = li_x * step_size

        current_radiance = segment_transmittance * current_radiance + emission_contrib + in_scattering_contrib
        t_current += step_size

    return current_radiance
```

### Edge Cases and Pitfalls
*   **Homogeneous vs. Heterogeneous**: Homogeneous volumes allow computational shortcuts and closed-form transmittance ($e^{-\sigma_t s}$). Heterogeneous volumes vary across space and require ray marching for every shadow ray and in-scattering calculation, significantly increasing cost [S2, unofficial].
*   **Single vs. Multiple Scattering**: Single scattering (one event) is sufficient for thin haze and light beams, and for thicker volumes with high absorption like smoke. It is not sufficient for dense, low-absorption volumes like clouds where multiple scattering dominates [S2, unofficial].
*   **Precision**: Small step sizes are needed for high-frequency heterogeneous volumes to avoid aliasing, but increase computational cost [S2, unofficial].
*   **Phase Function Sampling**: Importance sampling the phase function is required for efficient Monte Carlo integration of in-scattering [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Strengths**: Physically accurate representation of light transport in participating media; handles complex lighting effects like light beams and subsurface scattering [S2, unofficial].
*   **Limitations**: Computationally expensive due to the need for in-scattering integration; single scattering approximations fail for dense media [S2, unofficial].
*   **When to Use**: Clouds, smoke, fog, translucent materials [S2, unofficial].
*   **When Not to Use**: Not given in the sources.

## Background
The volumetric scattering model is rooted in the physics of radiative transfer in participating media. Physicists call fog, smoke, and clouds participating media because the space itself participates in the lighting instead of leaving light alone until it hits a surface [S2, unofficial][S3]. Key developments include the Henyey-Greenstein phase function, a classic model from a 1941 astrophysics paper originally designed to model scattering in galactic dust clouds [S1, unofficial]. Rayleigh scattering describes scattering by particles much smaller than the wavelength [S2, unofficial]. Mie scattering approximates the true scattering of water droplets [S1, unofficial]. The ray marching algorithm is a standard numerical solution for the radiative transfer equation in rendering [S2, unofficial].

## Sources
[S1] What Is Volumetric Rendering? Volume Shaders Explained, https://www.maxon.net/en/article/volumetric-rendering, fetched 2026-10-26 (unofficial)
[S2] CSE 168: Rendering Algorithms, https://cseweb.ucsd.edu/classes/sp17/cse168-a/CSE168_14_Volumetric.pdf, fetched 2026-10-27 (unofficial)
[S3] Introduction to Volumetric Rendering, https://help.maxon.net/c4d/2027/en-us/Content/html/What_is_Volumetric_Rendering.html, fetched 2026-10-29
