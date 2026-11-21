# Main Ray Marching with Frostbite Integration

Also called: Frostbite Volume Integration

## Overview
Main Ray Marching with Frostbite Integration is a volume rendering technique that marches along the view direction through a participating medium, accumulating radiance using a physically based integration scheme from Frostbite that handles transmittance, inscatter, and early termination [S1, unofficial]. It differs from its parent method, [Volume Single Scattering (Monte Carlo with Sphere Light)](4549-volume-single-scattering-monte-carlo-with-sphere-light.md), by replacing stochastic sphere light sampling with a fixed-step deterministic ray march [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $O$ and direction $D$ (normalized) [S1, unofficial].
- Bounding volume of the participating medium [S1, unofficial].
- Volume density $\rho$ [S1, unofficial].
- Scattering coefficient $\sigma_s$ and extinction coefficient $\sigma_t$ [S1, unofficial].
- Step size $s$ [S1, unofficial].
- Light source parameters (position, intensity, phase function parameters) [S1, unofficial].

**Outputs:**
- Accumulated radiance $L$ [S1, unofficial].
- Total transmittance $T_{total}$ (passed by reference) [S1, unofficial].

**Data Structures:**
- Current position $P$ [S1, unofficial].
- Cumulative transmittance $T_{total}$ [S1, unofficial].
- Accumulated radiance $L$ [S1, unofficial].

**Step-by-Step Method:**
1. Determine ray intersection with bounding volume. If no intersection, return black [S1, unofficial].
2. Initialize $L = 0$. Initialize $T_{total}$ (passed by reference). Calculate phase function: $p(\theta) = 0.7 \cdot HG(\theta, g=-0.3) + 0.3 \cdot HG(\theta, g=0.3)$ [S1, unofficial].
3. If density $\rho > 0$: Calculate sample scattering coefficient $\sigma_s' = \sigma_s \cdot \rho$ and sample extinction coefficient $\sigma_t' = \sigma_t \cdot \rho$ [S1, unofficial].
4. Calculate emission (distance glow) and ambient light. Calculate inscatter: $L_{inscatter} = L_{emission} + L_{ambient} + L_{sun} \cdot p(\theta) \cdot \tau_{light}$ [S1, unofficial].
5. Scale inscatter by scattering coefficient and density: $L_{inscatter} \leftarrow L_{inscatter} \cdot \sigma_s'$ [S1, unofficial].
6. Calculate transmittance for the step: $T_{step} = \exp(-\sigma_t' \cdot s)$ [S1, unofficial].
7. Integrate radiance using Frostbite method: $L \leftarrow L + T_{total} \cdot (L_{inscatter} - L_{inscatter} \cdot T_{step}) / \sigma_t'$ [S1, unofficial].
8. Update total transmittance: $T_{total} \leftarrow T_{total} \cdot T_{step}$ [S1, unofficial].
9. Early termination: if $|T_{total}| \le 10^{-3}$, set $T_{total} = 0$ and return $L$ [S1, unofficial].
10. Step along ray: $P \leftarrow P + D \cdot s$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def frostbite_volume_integration(ray_origin, ray_dir, bounding_volume, sigma_s, sigma_t, step_size, light_source):
    # 1. Intersection Test
    t_min, t_max = intersect_bounding_volume(ray_origin, ray_dir, bounding_volume)
    if t_min is None:
        return 0.0, 0.0

    # Initialize
    L = 0.0
    T_total = 1.0
    epsilon = 1e-3
    current_pos = ray_origin + ray_dir * t_min

    # Loop
    while current_pos is within bounding_volume and T_total > epsilon:
        # 3. Sample Coefficients
        density = 1.0  # Not given in the sources.
        sigma_s_prime = sigma_s * density
        sigma_t_prime = sigma_t * density

        # 2. Phase Function
        cos_theta = dot(normalize(light_source.position - current_pos), -ray_dir)
        p_theta = 0.7 * heney_greenstein(cos_theta, -0.3) + 0.3 * heney_greenstein(cos_theta, 0.3)

        # 4. Insatter
        # lightRay is not given in the sources.
        tau_light = exp(-sigma_t_prime * length(light_source.position - current_pos))
        L_emission = 0.0  # Not given in the sources.
        L_ambient = 0.0   # Not given in the sources.
        L_sun = light_source.intensity
        L_inscatter = L_emission + L_ambient + L_sun * p_theta * tau_light

        # 5. Scale Insatter
        L_inscatter *= sigma_s_prime

        # 6. Step Transmittance
        T_step = exp(-sigma_t_prime * step_size)

        # 7. Integrate Radiance (Frostbite Method)
        if sigma_t_prime > 1e-6:
            term = L_inscatter * (1.0 - T_step) / sigma_t_prime
        else:
            term = L_inscatter * step_size
        L += T_total * term

        # 8. Update Total Transmittance
        T_total *= T_step

        # 9. Early Termination
        if abs(T_total) <= epsilon:
            T_total = 0.0
            return L

        # 10. Step Forward
        current_pos += ray_dir * step_size

    return L, T_total

def intersect_bounding_volume(origin, direction, volume):
    # Standard axis-aligned bounding box intersection
    # Not given in the sources.
    t_min = (volume.min - origin) / direction
    t_max = (volume.max - origin) / direction
    t_enter = max(min(t_min))
    t_exit = min(max(t_max))
    if t_enter > t_exit or t_exit < 0:
        return None, None
    return t_enter, t_exit

def heney_greenstein(cos_theta, g):
    numerator = 1.0 - g * g
    denominator = 4.0 * 3.141592653589793 * pow(1.0 + g * g - 2.0 * g * cos_theta, 1.5)
    if denominator == 0:
        return 0.0
    return numerator / denominator
```

**Edge Cases and Pitfalls:**
- **Early Termination Threshold:** The $10^{-3}$ threshold is a heuristic to save computation in transparent regions [S1, unofficial].
- **Phase Function Mix:** The specific weights ($0.7$ for $g=-0.3$, $0.3$ for $g=0.3$) are hardcoded in the source [S1, unofficial].

## Subtopics

### [Uniform Volume Ray Marching](12998-uniform-volume-ray-marching.md)
Uniform Volume Ray Marching is a rendering technique for participating media with constant opacity and density, using ray marching to estimate the integral of light scattering. It assumes a single light source and constant volume properties, allowing precomputation of attenuation factors. It is distinguished from its sibling, [Main Ray Marching with Frostbite Integration](4931-main-ray-marching-with-frostbite-integration.md), by its strict assumption of uniform volume properties, which simplifies the transmission calculation to a constant per-step factor.

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
