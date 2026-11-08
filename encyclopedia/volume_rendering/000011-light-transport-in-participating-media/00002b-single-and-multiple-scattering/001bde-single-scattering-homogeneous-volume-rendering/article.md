# Single-scattering homogeneous volume rendering

Also called: Single monochromatic homogeneous volume with absorption and single-scattering, SingleScatterTracer, Vol path tracing 2

## Overview
This method renders a participating medium by solving the radiative transfer equation (RTE) under the assumption that light scatters exactly once within a homogeneous volume before reaching the camera [S2, S3]. It extends a basic absorption-only model by adding in-scattering from surface-emitting geometry, assuming the surfaces do not reflect or transmit light [S2, S3]. It is distinguished from [Single-scattering atmosphere rendering](3289-single-scattering-atmosphere-rendering.md) by its general applicability to arbitrary scene manifolds rather than specific atmospheric layers, and from [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md) by its integration over the entire scene manifold $\mathcal{M}$ rather than discrete point lights.

## How it works
**Inputs:**
*   Camera ray origin $\mathbf{o}$ and direction $\boldsymbol{\omega}$.
*   Scene geometry $\mathcal{M}$, assumed to consist of surfaces that only emit light with radiance $L_e(\mathbf{p}', \boldsymbol{\omega}')$ [S2, S3].
*   Homogeneous medium coefficients: absorption $\sigma_a$, scattering $\sigma_s$ (constant in space) [S2, S3].
*   Phase function $\rho(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$ [S2, S3].
*   Random number generator `rng` [S2, S3].

**Outputs:**
*   Radiance $L_2(\mathbf{o}, \boldsymbol{\omega})$ at the camera [S2, S3].

**Data Structures:**
The method is encapsulated in a volume integration method struct [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```julia
struct SingleScatterTracer <: VolumeIntegrationMethod end
```
[G] In implementation, $\sigma_t$ is typically precomputed as $\sigma_a + \sigma_s$.

**Parameters:**
*   $\sigma_a$: Absorption coefficient ($m^{-1}$) [S2, S3].
*   $\sigma_s$: Scattering coefficient ($m^{-1}$) [S2, S3].
*   $\sigma_t$: Extinction coefficient, $\sigma_t = \sigma_a + \sigma_s$ [S2, S3].
*   The volume is monochromatic: $\sigma_a$ and $\sigma_s$ have identical values across color channels [S2, S3].

**Steps:**
1.  **Ray Intersection:** Intersect the camera ray with the scene geometry to find the nearest surface intersection distance $t_{\text{hit}}$ [S2, S3].
2.  **Distance Sampling:** Sample a scattering distance $t$ from the probability density function (PDF) $p(t) = \sigma_t \exp(-\sigma_t t)$ using inverse transform sampling [S2, S3].
    *   Generate a uniform random number $u \in [0, 1]$ [S2, S3].
    *   Compute $t = \frac{\log(1-u)}{-\sigma_t}$ [S2, S3].
3.  **Branching:**
    *   **Case A: Surface Hit ($t \geq t_{\text{hit}}$):** The ray hits the surface before scattering. The contribution is the surface emission attenuated by the transmittance to the hit point [S2, S3].
        $$ L = \exp(-\sigma_t t_{\text{hit}}) L_e(\mathbf{p}(t_{\text{hit}}), \boldsymbol{\omega}) $$
        The probability of this event is $P(t \geq t_{\text{hit}}) = \exp(-\sigma_t t_{\text{hit}})$ [S2, S3].
    *   **Case B: In-scattering ($t < t_{\text{hit}}$):** The ray scatters at point $\mathbf{p}(t) = \mathbf{o} + t\boldsymbol{\omega}$ [S2, S3].
        1.  Compute the single-scattering radiance $L_{\text{scatter1}}(\mathbf{p}(t), \boldsymbol{\omega})$ by integrating over the scene manifold $\mathcal{M}$ [S2, S3]:
            $$ L_{\text{scatter1}}(\mathbf{p}, \boldsymbol{\omega}) = \int_{\mathcal{M}} \rho(\boldsymbol{\omega}', \boldsymbol{\omega}) L_e(\mathbf{p}', \boldsymbol{\omega}') \exp(-\sigma_t \|\mathbf{p} - \mathbf{p}'\|) \frac{|\boldsymbol{\omega}' \cdot \mathbf{n}_{\mathbf{p}'}|}{\|\mathbf{p} - \mathbf{p}'\|^2} \text{visible}(\mathbf{p}, \mathbf{p}') d\mathbf{p}' $$
            where $\mathbf{p}'$ is a point on the surface, $\mathbf{n}_{\mathbf{p}'}$ is its normal, $\boldsymbol{\omega}' = \frac{\mathbf{p}' - \mathbf{p}}{\|\mathbf{p}' - \mathbf{p}\|}$, and $\text{visible}$ is the visibility term [S2, S3].
        2.  Weight the in-scattering term by the scattering probability and transmittance:
            $$ L = \frac{\exp(-\sigma_t t) \sigma_s L_{\text{scatter1}}(\mathbf{p}(t), \boldsymbol{\omega})}{p(t)} $$
            Substituting $p(t) = \sigma_t \exp(-\sigma_t t)$:
            $$ L = \frac{\sigma_s}{\sigma_t} L_{\text{scatter1}}(\mathbf{p}(t), \boldsymbol{\omega}) $$
            [G] Note: The source [S2] implies the estimator $\frac{\text{integrand}}{p(t)}$. The integrand in Eq 9 is $\exp(-\sigma_t t) \sigma_s L_{\text{scatter1}}$. Thus the estimator is $\frac{\exp(-\sigma_t t) \sigma_s L_{\text{scatter1}}}{\sigma_t \exp(-\sigma_t t)} = \frac{\sigma_s}{\sigma_t} L_{\text{scatter1}}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_scatter_homogeneous_volume(camera_ray, scene, sigma_a, sigma_s, phase_func, rng):
    sigma_t = sigma_a + sigma_s
    
    # 1. Intersect ray with scene
    isect = intersect(scene, camera_ray)
    t_hit = isect.distance
    
    # 2. Sample scattering distance
    u = rng.uniform(0.0, 1.0)
    t = log(1.0 - u) / -sigma_t
    
    # 3. Branch based on hit
    if t >= t_hit:
        # Surface hit: attenuated emission
        p_hit = camera_ray.origin + t_hit * camera_ray.direction
        # Assuming surface emits towards camera
        L_e = scene.get_emission(p_hit, camera_ray.direction)
        transmittance = exp(-sigma_t * t_hit)
        return transmittance * L_e
    else:
        # In-scattering
        p_scatter = camera_ray.origin + t * camera_ray.direction
        
        # Compute L_scatter1 by integrating over scene manifold M
        # This requires a separate Monte Carlo integration or analytic solution
        # depending on scene complexity. Here we denote the integral result.
        L_s1 = integrate_scene_manifold(scene, p_scatter, camera_ray.direction, 
                                        sigma_t, sigma_a, sigma_s, phase_func)
        
        # Estimator: Integrand / PDF
        # Integrand from Eq 9: exp(-sigma_t * t) * sigma_s * L_s1
        # PDF: sigma_t * exp(-sigma_t * t)
        # Ratio: sigma_s / sigma_t * L_s1
        return (sigma_s / sigma_t) * L_s1
```

**Edge Cases and Pitfalls:**
*   **Ray Differentials:** Ray differentials for volumetric scattering are an unsolved problem; they must be disabled (set to 0) to avoid artifacts [S3].
*   **Environment Maps:** The method assumes no environment maps are present in the scene [S2, S3].
*   **Monochromatic Assumption:** The coefficients $\sigma_a$ and $\sigma_s$ are assumed constant across all color channels [S2, S3].

## Operating characteristics
*   **Complexity:** The method requires intersecting the camera ray with the scene ($O(N)$ for naive traversal) and performing a Monte Carlo integration over the scene manifold for in-scattering. The in-scattering integral itself is expensive as it requires shooting rays from every scattering point to the scene geometry [S2, S3].
*   **Memory:** Low memory footprint as it does not require storing volumetric grids or complex acceleration structures for the medium itself, only the scene geometry [S1, unofficial][S2].
*   **Strengths:** Physically plausible single-scattering effects in homogeneous media; simple implementation compared to multiple scattering methods [S2, S3].
*   **Limitations:** Only accounts for one scattering event; inaccurate for optically thick media or dense fog where multiple scattering dominates [S2, S3].
*   **When to use:** Thin to moderately dense homogeneous volumes with emissive surfaces [S2, S3].
*   **When not to use:** Heterogeneous volumes, media with significant multiple scattering, or scenes with complex surface reflections [S2, S3].

## Background
This method originates from the course "CSE 272: Volumetric Path Tracing" at the University of California, San Diego [S2, S3]. It is presented as an extension of a basic absorption-only volume renderer, introducing single scattering to solve the radiative transfer equation for a homogeneous medium with emissive surfaces [S2, S3]. The approach is also referenced in the `Lyr.jl` library documentation as `SingleScatterTracer` [S1, unofficial].

## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] PDF Volumetric Path Tracing - University of California, San Diego, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S3] Volumetric Path Tracing, https://cseweb.ucsd.edu/~tzli/cse272/wi2022/homework2.pdf, fetched 2026-10-30
