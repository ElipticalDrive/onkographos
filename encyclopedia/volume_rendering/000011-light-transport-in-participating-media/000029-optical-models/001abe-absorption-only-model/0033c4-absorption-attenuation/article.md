# Absorption Attenuation

Also called: Absorption Coefficient Model, Absorption Coefficient, Absorption Model

## Overview
Absorption Attenuation is the process by which light radiance is reduced as it travels through a participating medium due to the conversion of photon energy into other forms, such as heat [S3, book]. It solves the problem of calculating the surviving radiance along a ray segment in an absorption-only medium, where no scattering or emission occurs. Unlike scattering-based models, this method considers only the exponential decay of incident light, making it the foundational component for compositing in volume rendering and radiography where emission or scattering is negligible or handled separately [8074-transparency-and-opacity.md].

## How it works
**Inputs and Outputs**
- Input: Incident radiance $L_i$ at the start of the segment, the absorption coefficient field $\sigma_a(p, \omega)$, the start position $p$, the direction $\omega$, and the segment length $d$.
- Output: Exitant radiance $L_o$ at the end of the segment.

**Data Structures**
The absorption coefficient $\sigma_a$ is typically stored as a scalar field (e.g., a 3D texture or grid) where each voxel contains a non-negative value representing the probability density of absorption per unit distance [S3, book]. The units are reciprocal distance ($m^{-1}$) [S3, book].

**Formulas**
The differential change in radiance $dL$ over a differential distance $dt$ is given by:
$$ dL(p, \omega) = -\sigma_a(p, \omega) L(p, \omega) dt $$
where $L(p, \omega)$ is the radiance at position $p$ in direction $\omega$, and $\sigma_a(p, \omega)$ is the absorption coefficient [S2, unofficial][S3, book].

The surviving radiance after traveling distance $d$ in direction $\omega$ starting at point $p$ is:
$$ L_{surviving} = L_{initial} \cdot e^{-\int_0^d \sigma_a(p + t\omega, \omega) dt} $$
The term in the exponent is the optical depth $\tau$ [S1, unofficial]:
$$ \tau(d) = \int_0^d \sigma_a(p + t\omega, \omega) dt $$
Thus, the transmittance $T(d)$ is:
$$ T(d) = e^{-\tau(d)} $$
And the exitant radiance is:
$$ L_o = T(d) L_i $$

**Parameters**
- $\sigma_a$: Absorption coefficient. It can take any non-negative value and is not required to be between 0 and 1 [S3, book]. It may vary with position $p$ and direction $\omega$, though many implementations (like pbrt) model it as purely a function of position [S3, book].

**Steps**
1. Initialize the accumulated optical depth $\tau = 0$.
2. Sample the absorption coefficient $\sigma_a$ at the current position along the ray.
3. Add $\sigma_a \cdot dt$ to $\tau$, where $dt$ is the step size.
4. Repeat steps 2-3 for all steps along the ray segment of length $d$.
5. Compute the transmittance $T = e^{-\tau}$.
6. Compute the output radiance $L_o = L_i \cdot T$.

**Edge Cases and Pitfalls**
- If $\sigma_a$ is zero, $T=1$ and no attenuation occurs.
- If $\sigma_a$ is very large, $T$ approaches 0, resulting in complete opacity.
- Numerical precision issues may arise if $\tau$ is extremely large, causing $e^{-\tau}$ to underflow to zero.
- The absorption coefficient is distinct from the absorption coefficient used in specifying indices of refraction of conductors [S3, book].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def absorption_attenuation(L_initial, start_pos, direction, length, get_sigma_a):
    """
    Calculate radiance after absorption attenuation.
    
    Args:
        L_initial: Incident radiance at start_pos.
        start_pos: 3D vector, starting point of the ray segment.
        direction: Normalized 3D vector, direction of propagation.
        length: Float, total distance to travel.
        get_sigma_a: Function that takes (position, direction) and returns sigma_a.
        
    Returns:
        L_exitant: Radiance after attenuation.
    """
    tau = 0.0
    num_steps = 100  # Not given in sources; implementation detail for integration
    dt = length / num_steps
    
    for i in range(num_steps):
        # Current position along the ray
        current_pos = start_pos + direction * (i * dt)
        
        # Sample absorption coefficient
        # Note: get_sigma_a may ignore direction if isotropic
        sigma_a = get_sigma_a(current_pos, direction)
        
        # Accumulate optical depth
        tau += sigma_a * dt
        
    # Compute transmittance
    transmittance = math.exp(-tau)
    
    # Compute exitant radiance
    L_exitant = L_initial * transmittance
    
    return L_exitant
```

## Subtopics

### [Transparency and Opacity](8074-transparency-and-opacity.md)
Defines transparency (transmittance) $T(s)$ as the fraction of light remaining and opacity $O(s)$ as the fraction absorbed ($O(s) = 1 - T(s)$). It is distinct from scattering-based models by considering only absorption.

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of sampling steps along the ray. If the absorption coefficient is constant, it is $O(1)$.
- **Memory Complexity**: Depends on the storage of the absorption coefficient field. For a 3D grid, it is $O(N^3)$ where $N$ is the resolution.
- **Strengths**: Physically accurate for non-scattering media; computationally inexpensive compared to scattering models.
- **Limitations**: Does not account for light scattering or emission; requires accurate sampling of $\sigma_a$ for inhomogeneous media.
- **When to use**: When modeling media where scattering is negligible (e.g., certain types of smoke, fog, or X-ray attenuation) or when compositing pre-computed attenuation maps.

## Background
Absorption is one of the three main physical processes affecting radiance in participating media, alongside emission and scattering [S3, book]. It was formally described in the context of volume rendering in early works on participating media [S1, unofficial]. The differential equation governing absorption is derived from the Beer-Lambert law [S2, unofficial][S3, book]. The concept is fundamental to offline rendering, as detailed in PBRT [S3, book], and is also relevant in real-time solutions where biased approximations like particle systems are used [S2, unofficial].

## Sources
[S1] Volume Rendering Volume Rendering, https://docslib.org/doc/1400821/volume-rendering, fetched 2026-10-28 (unofficial)
[S2] Volume Rendering in Offline Renderer · A Graphics Guy's Note, https://agraphicsguynotes.com/posts/volume_rendering_in_offline_renderer/, fetched 2026-10-28 (unofficial)
[S3] 11.1 Volume Scattering Processes, https://pbr-book.org/4ed/Volume_Scattering/Volume_Scattering_Processes, fetched 2026-10-29 (book)
