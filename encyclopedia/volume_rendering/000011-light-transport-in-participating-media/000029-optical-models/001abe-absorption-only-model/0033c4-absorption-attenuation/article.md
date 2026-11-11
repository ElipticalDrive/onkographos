# Absorption Attenuation

Also called: Absorption Coefficient Model, Absorption Coefficient, Absorption Model

## Overview
Absorption Attenuation is the optical model describing the exponential decay of radiance as light travels through a participating medium where particles absorb energy from the ray without scattering or emitting [S2, unofficial]. It solves the problem of calculating the remaining light intensity after traversing a distance $d$ in a volume with a given absorption coefficient [S2, unofficial].

## How it works
**Inputs:**
*   Position $p$ and direction $w$ of the ray origin [G].
*   Absorption coefficient $\sigma_a(p, w)$ (or $\kappa(x, \omega)$), defined as the probability of a photon being absorbed at position $x$ in direction $\omega$ per unit length [S1, unofficial].
*   Distance $d$ along the ray [G].
*   Initial radiance $L(p, w)$ at the start of the segment [G].

**Outputs:**
*   Attenuated radiance $L(p+d \cdot w, w)$ at the end of the segment [G].
*   Transmittance $T(d)$, the fraction of light surviving absorption [G].

**Data Structures:**
*   The absorption coefficient is stored as a scalar field indexed by spatial coordinates $(x, y, z)$ [G].
*   Radiance $L$ is a vector (e.g., RGB) or scalar value depending on the color space [G].

**Steps:**
1.  **Define the Ray:** Parameterize the ray as $p(t) = p(0) + t \cdot w$, where $t$ is the distance along the ray [S2, unofficial].
2.  **Formulate Differential Equation:** The change in radiance $dL$ over a differential distance $dt$ is proportional to the current radiance and the absorption coefficient [S2, unofficial]:
    $$ dL(p, w) = -\sigma_a(p, w) L(p, w) dt $$
    Note: The subscript distinguishing incoming/outgoing light is dropped for clarity [S2, unofficial].
3.  **Separate Variables:** Rearrange the equation to isolate the radiance terms:
    $$ \frac{dL(p(t), w)}{L(p(t), w)} = -\sigma_a(p(t), w) dt $$
4.  **Integrate:** Integrate both sides from $t=0$ to $t=d$ [S2, unofficial]:
    $$ \int_0^d \frac{dL(p(t), w)}{L(p(t), w)} = \int_0^d -\sigma_a(p(t), w) dt $$
5.  **Solve for Transmittance:** The left side integrates to the natural logarithm of the radiance ratio. Exponentiating both sides yields the transmittance formula [S2, unofficial]:
    $$ \frac{L(p+d \cdot w, w)}{L(p, w)} = e^{-\int_0^d \sigma_a(p(t), w) dt} $$
    Let $\tau(d) = \int_0^d \sigma_a(p(t), w) dt$ be the optical depth [S2, unofficial]. (Note: [S1, unofficial] uses the notation $\kappa(x,\omega)$ and defines optical depth as $\tau(s) = \int_0^s \kappa(x + t\omega,\omega) dt$.) Then:
    $$ T(d) = e^{-\tau(d)} $$
6.  **Compute Final Radiance:** Multiply the initial radiance by the transmittance [S1, unofficial]:
    $$ L(p+d \cdot w, w) = T(d) L(p, w) $$

**Parameters:**
*   $\sigma_a$ (or $\kappa$): Absorption coefficient. For a uniform volume, $\sigma_a$ is constant [S2, unofficial].
*   $d$: Distance along the ray.

**Edge Cases and Pitfalls:**
*   **Uniform vs. Non-Uniform:** If $\sigma_a$ is constant, the integral simplifies to $\sigma_a \cdot d$ [S2, unofficial]. If $\sigma_a$ varies, numerical integration is required along the ray segment [G].
*   **Precision:** For large optical depths ($\tau \gg 1$), $T(d)$ approaches 0. Floating-point underflow may occur, resulting in exactly 0.0, which is physically correct [G].
*   **Boundary Conditions:** The model assumes the medium is homogeneous or that the absorption coefficient is known at every point along the ray [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def absorption_attenuation(radiance_initial, ray_origin, ray_direction, distance, num_samples=10):
    """
    Calculates the radiance after absorption attenuation along a ray segment.
    Args:
        radiance_initial: float or vector, L(p, w) at the start.
        ray_origin: tuple (x, y, z), p(0).
        ray_direction: tuple (dx, dy, dz), w (assumed normalized).
        distance: float, d, the length of the segment.
        num_samples: int, number of integration steps [G].
    Returns:
        radiance_final: float or vector, L(p + d*w, w).
    """
    import math
    step = distance / num_samples
    optical_depth = 0.0
    for i in range(num_samples):
        t_mid = (i + 0.5) * step
        pos = (
            ray_origin[0] + ray_direction[0] * t_mid,
            ray_origin[1] + ray_direction[1] * t_mid,
            ray_origin[2] + ray_direction[2] * t_mid
        )
        # Sample absorption coefficient at pos from the field (e.g., trilinear interpolation) [G]
        sigma_a = sample_field(pos)
        optical_depth += sigma_a * step
    transmittance = math.exp(-optical_depth)
    radiance_final = radiance_initial * transmittance
    return radiance_final
```

## Subtopics

### [Transparency and Opacity](8074-transparency-and-opacity.md)
Defines transparency $T(s)$ as the transmittance $e^{-\tau(s)}$ and opacity $O(s)$ as $1-T(s)$, providing the fundamental components for calculating light transmission and blocking in the absorption-only model.

## Operating characteristics
Not given in the sources.

## Background
The absorption model is a fundamental component of light transport in participating media, described in standard computer graphics literature such as PBRT [S2, unofficial]. The differential form $dL = -\sigma_a L dt$ is widely used in offline renderers to simulate the loss of radiance due to absorption [S2, unofficial]. The concept of optical depth $\tau$ and transmittance $T$ is central to volume rendering techniques, allowing for the efficient calculation of light transmission through volumetric data [S1, unofficial].

## Sources
[S1] Volume Rendering, https://docslib.org/doc/1400821/volume-rendering, fetched 2026-10-28 (unofficial)
[S2] Volume Rendering in Offline Renderer · A Graphics Guy's Note, https://agraphicsguynotes.com/posts/volume_rendering_in_offline_renderer/, fetched 2026-10-28 (unofficial)
