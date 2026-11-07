# Absorption Attenuation

Also called: Absorption Coefficient Model, Absorption Coefficient, Absorption Model

## Overview
Absorption Attenuation is the optical model describing the exponential decay of radiance as light travels through a participating medium where particles absorb energy from the ray without scattering or emitting [S2, unofficial]. It solves the problem of calculating the remaining light intensity after traversing a distance $d$ in a volume with a given absorption coefficient [S2, unofficial]. It is distinguished from its sibling [X-ray simulation via ray integration](1445-x-ray-simulation-via-ray-integration.md) by its explicit use of the radiance differential equation and transmittance formulation, and from the [Prism Model](3153-prism-model.md) by its assumption of isotropic absorption rather than directional refraction or specific geometric prism constraints.

## How it works
**Inputs:**
*   Position $p$ and direction $w$ of the ray origin.
*   Absorption coefficient $\sigma_a(p, w)$ (or $\kappa(x, \omega)$), defined as the probability of a photon being absorbed at position $x$ in direction $\omega$ per unit length [S1, S2, unofficial].
*   Distance $d$ along the ray.
*   Initial radiance $L(p, w)$ at the start of the segment.

**Outputs:**
*   Attenuated radiance $L(p+d \cdot w, w)$ at the end of the segment.
*   Transmittance $T(d)$, the fraction of light surviving absorption.

**Data Structures:**
*   The absorption coefficient $\sigma_a$ is typically stored as a scalar field (e.g., a 3D texture or voxel grid) indexed by spatial coordinates $(x, y, z)$.
*   Radiance $L$ is a vector (e.g., RGB) or scalar value depending on the color space.

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
    Let $\tau(d) = \int_0^d \sigma_a(p(t), w) dt$ be the optical depth [S1, unofficial]. Then:
    $$ T(d) = e^{-\tau(d)} $$
6.  **Compute Final Radiance:** Multiply the initial radiance by the transmittance [S1, unofficial]:
    $$ L(p+d \cdot w, w) = T(d) L(p, w) $$

**Parameters:**
*   $\sigma_a$ (or $\kappa$): Absorption coefficient. Typical values depend on the medium density and material properties. For a uniform volume, $\sigma_a$ is constant [S2, unofficial].
*   $d$: Distance along the ray.

**Edge Cases and Pitfalls:**
*   **Uniform vs. Non-Uniform:** If $\sigma_a$ is constant, the integral simplifies to $\sigma_a \cdot d$ [S2, unofficial]. If $\sigma_a$ varies, numerical integration (e.g., midpoint rule) is required along the ray segment.
*   **Precision:** For large optical depths ($\tau \gg 1$), $T(d)$ approaches 0. Floating-point underflow may occur, resulting in exactly 0.0, which is physically correct but may cause issues if further operations expect non-zero values.
*   **Boundary Conditions:** The model assumes the medium is homogeneous or that $\sigma_a$ is known at every point along the ray. Discontinuities in $\sigma_a$ at voxel boundaries require careful interpolation or sampling [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def absorption_attenuation(radiance_initial, absorption_coefficient_field, ray_origin, ray_direction, distance):
    """
    Calculates the radiance after absorption attenuation along a ray segment.
    
    Args:
        radiance_initial: float or vector, L(p, w) at the start of the segment.
        absorption_coefficient_field: function or data structure, returns sigma_a at a given 3D position.
        ray_origin: tuple (x, y, z), p(0).
        ray_direction: tuple (dx, dy, dz), w (assumed normalized).
        distance: float, d, the length of the segment.
        
    Returns:
        radiance_final: float or vector, L(p + d*w, w).
    """
    # Calculate optical depth tau by integrating sigma_a along the ray
    # For simplicity, assume uniform sampling or constant sigma_a if not specified.
    # Here we use a simple midpoint approximation for non-uniform fields.
    
    num_samples = 10  # Not given in sources; sufficient for smooth fields [G]
    step = distance / num_samples
    optical_depth = 0.0
    
    for i in range(num_samples):
        # Midpoint of the current sub-segment
        t_mid = (i + 0.5) * step
        pos = (
            ray_origin[0] + ray_direction[0] * t_mid,
            ray_origin[1] + ray_direction[1] * t_mid,
            ray_origin[2] + ray_direction[2] * t_mid
        )
        # Sample absorption coefficient at midpoint
        sigma_a = absorption_coefficient_field(pos)
        optical_depth += sigma_a * step
        
    # Calculate transmittance
    transmittance = math.exp(-optical_depth)
    
    # Apply attenuation
    radiance_final = radiance_initial * transmittance
    
    return radiance_final
```

## Subtopics

### [Transparency and Opacity](8074-transparency-and-opacity.md)
Defines transparency $T(s)$ as the transmittance $e^{-\tau(s)}$ and opacity $O(s)$ as $1-T(s)$, providing the fundamental components for calculating light transmission and blocking in the absorption-only model.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of samples taken along the ray to integrate $\sigma_a$. For uniform media, it is $O(1)$.
*   **Memory Complexity:** $O(V)$ where $V$ is the size of the absorption coefficient field (voxel grid or texture).
*   **Strengths:** Computationally inexpensive for uniform media; physically accurate for non-scattering absorbing media; forms the basis for more complex models (e.g., adding scattering).
*   **Limitations:** Does not account for light redirection (scattering) or emission; requires accurate sampling of $\sigma_a$ for non-uniform media.
*   **When to Use:** When modeling media that primarily absorb light (e.g., smoke, fog with negligible scattering, or X-ray attenuation) and scattering effects are negligible or handled separately.
*   **When Not to Use:** When scattering (in-scattering or out-scattering) significantly contributes to the final image, as this model ignores these effects [S2, unofficial].

## Background
The absorption model is a fundamental component of light transport in participating media, described in standard computer graphics literature such as PBRT [S2, unofficial]. It is derived from the Beer-Lambert law, which describes the attenuation of light as it passes through a medium. The differential form $dL = -\sigma_a L dt$ is widely used in offline renderers to simulate the loss of radiance due to absorption [S2, unofficial]. The concept of optical depth $\tau$ and transmittance $T$ is central to volume rendering techniques, allowing for the efficient calculation of light transmission through volumetric data [S1, unofficial].

## Sources
[S1] Volume Rendering, https://docslib.org/doc/1400821/volume-rendering, fetched 2026-10-28 (unofficial)
[S2] Volume Rendering in Offline Renderer · A Graphics Guy's Note, https://agraphicsguynotes.com/posts/volume_rendering_in_offline_renderer/, fetched 2026-10-28 (unofficial)
