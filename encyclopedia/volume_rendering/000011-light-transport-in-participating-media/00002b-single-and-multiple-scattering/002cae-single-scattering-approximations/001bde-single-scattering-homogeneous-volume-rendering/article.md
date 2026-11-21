# Single-scattering homogeneous volume rendering

Also called: Single monochromatic homogeneous volume with absorption and single-scattering, SingleScatterTracer, Vol path tracing 2

## Overview
This method renders a participating medium by solving the radiative transfer equation under the assumption that light scatters exactly once within the volume before reaching the camera [S2, S3]. It extends an absorption-only model by adding single-scattering contributions from surface-emitting lights, assuming a single homogeneous medium where absorption and scattering coefficients are spatially constant and monochromatic [S2, S3]. It is distinguished from multiple scattering models by ignoring light that bounces more than once within the medium, and from atmosphere rendering by assuming a bounded volume with emitting surfaces rather than an infinite or unbounded medium [S2, S3].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera ray origin $\mathbf{o}$ and direction $\omega$, scene geometry (manifold $\mathcal{M}$), homogeneous medium coefficients $\sigma_a$ (absorption) and $\sigma_s$ (scattering), phase function $\rho$, and a random number generator.
*   **Output:** Radiance $L_2(\mathbf{o}, \omega)$ at the camera.

**Data Structures:**
The method is encapsulated in a volume integration method struct, typically defined as:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```julia
struct SingleScatterTracer <: VolumeIntegrationMethod end
```
[S1, unofficial]

**Parameters:**
*   $\sigma_a$: Absorption coefficient (constant).
*   $\sigma_s$: Scattering coefficient (constant).
*   $\sigma_t = \sigma_a + \sigma_s$: Extinction coefficient.
*   $\rho(\mathbf{p}, \omega, \omega')$: Phase function describing the probability of scattering from direction $\omega'$ to $\omega$ at point $\mathbf{p}$.
*   $L_e(\mathbf{p}', \omega')$: Emission radiance from surface point $\mathbf{p}'$.
*   The medium is monochromatic: $\sigma_a$ and $\sigma_s$ have identical values across color channels [S2, S3].

**Assumptions:**
1.  There is a single, homogeneous volume.
2.  Surfaces only emit light ($L_e$) and do not reflect or transmit light [S2, S3].
3.  Light scatters exactly once within the volume [S2, S3].
4.  No environment maps are present in the scene [S2, S3].

**Algorithm Steps:**
1.  **Ray Intersection:** Intersect the camera ray with the scene geometry to find the hit distance $t_{\text{hit}}$ and the hit point $\mathbf{p}(t_{\text{hit}})$ [S2, S3].
2.  **Sample Scattering Distance:** Generate a uniform random number $u \in [0, 1]$ [S2, S3].
3.  **Inverse Transform Sampling:** Compute the sampled scattering distance $t$ using the inverse CDF of the exponential distribution derived from importance sampling the transmittance term $\exp(-\sigma_t t)$:
    $$ t = \frac{\log(1-u)}{-\sigma_t} $$
    [S2, S3]
    *Note: Since $u$ is uniform in $[0,1]$, $1-u$ is also uniform. Using $\log(u)/-\sigma_t$ is mathematically equivalent and avoids the subtraction [G].*
4.  **Branching:**
    *   **Case A: Surface Hit ($t \geq t_{\text{hit}}$):** The ray hits the surface before scattering. The contribution is the surface emission attenuated by the transmittance to the surface:
        $$ \text{Contribution} = \exp(-\sigma_t t_{\text{hit}}) L_e(\mathbf{p}(t_{\text{hit}})) $$
        [S2, S3]
    *   **Case B: Volume Scattering ($t < t_{\text{hit}}$):** The ray scatters at point $\mathbf{p}(t) = \mathbf{o} + t\omega$. The contribution is the single-scattering radiance $L_{\text{scatter1}}$ at $\mathbf{p}(t)$, attenuated by the transmittance from the camera to the scattering point, weighted by the scattering coefficient and the PDF of the sampling distribution:
        $$ \text{Contribution} = \frac{\exp(-\sigma_t t) \sigma_s L_{\text{scatter1}}(\mathbf{p}(t), \omega)}{p(t)} $$
        where the PDF is $p(t) = \sigma_t \exp(-\sigma_t t)$ [S2, S3].
        The term $\frac{\exp(-\sigma_t t) \sigma_s}{p(t)}$ simplifies to $\frac{\sigma_s}{\sigma_t}$ [G].
5.  **Compute Single-Scattering Radiance:** Calculate $L_{\text{scatter1}}(\mathbf{p}(t), \omega)$ by integrating over the scene manifold $\mathcal{M}$:
    $$ L_{\text{scatter1}}(\mathbf{p}, \omega) = \int_{\mathcal{M}} \rho(\mathbf{p}, \omega, \omega') L_e(\mathbf{p}', \omega') \exp(-\sigma_t \|\mathbf{p} - \mathbf{p}'\|) \frac{|\omega' \cdot \mathbf{n}_{\mathbf{p}'}|}{\|\mathbf{p} - \mathbf{p}'\|^2} \text{visible}(\mathbf{p}, \mathbf{p}') d\mathbf{p}' $$
    [S2, S3]
    Here, $\mathbf{p}'$ is a point on the emitting surface, $\mathbf{n}_{\mathbf{p}'}$ is its normal, $\omega' = \frac{\mathbf{p}' - \mathbf{p}}{\|\mathbf{p}' - \mathbf{p}\|}$, and $\text{visible}$ is the visibility function (1 if unoccluded, 0 otherwise). This integral is typically evaluated using Monte Carlo sampling over the scene geometry [S2, S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_scatter_homogeneous_volume_render(camera_ray, scene, sigma_a, sigma_s, rng):
    sigma_t = sigma_a + sigma_s
    
    # 1. Intersect ray with scene
    isect = intersect(scene, camera_ray)
    t_hit = isect.distance
    p_hit = isect.point
    
    # 2. Sample scattering distance
    u = rng.uniform(0.0, 1.0)
    # Inverse CDF: t = log(1-u) / -sigma_t
    # Using log(u) is equivalent and numerically stable for u in (0,1]
    t = -log(1.0 - u) / sigma_t
    
    if t >= t_hit:
        # Case A: Hit surface
        # Attenuated surface emission
        transmittance = exp(-sigma_t * t_hit)
        return transmittance * scene.get_emission(p_hit, camera_ray.dir)
    else:
        # Case B: Scatter in volume
        p_scatter = camera_ray.origin + t * camera_ray.direction
        
        # PDF of the sampled distance
        pdf = sigma_t * exp(-sigma_t * t)
        
        # Compute single scattering radiance L_s1 at p_scatter
        # This requires integrating over the scene manifold M
        # L_s1 = integral over M of ...
        L_s1 = compute_single_scattering_radiance(p_scatter, camera_ray.direction, scene, sigma_t)
        
        # Weight by scattering coefficient and divide by PDF
        # Contribution = exp(-sigma_t * t) * sigma_s * L_s1 / pdf
        # Simplifies to: (sigma_s / sigma_t) * L_s1
        weight = sigma_s / sigma_t
        
        return weight * L_s1

def compute_single_scattering_radiance(p, omega, scene, sigma_t):
    # Monte Carlo integration over scene manifold M
    # L_s1(p, omega) = int_M rho(p, omega, omega') * L_e(p', omega') *
    #                  exp(-sigma_t * ||p - p'||) * |omega' . n_p'| / ||p - p'||^2 *
    #                  visible(p, p') dp'
    
    # Sample a point p' on the scene geometry
    p_prime, omega_prime, normal, pdf_surface = sample_scene_point(scene, p)
    
    if p_prime is None:
        return 0.0
    
    # Visibility check
    if not is_visible(p, p_prime, scene):
        return 0.0
    
    # Distance and transmittance
    dist = distance(p, p_prime)
    transmittance = exp(-sigma_t * dist)
    
    # Phase function
    # omega_prime is direction from p to p_prime? 
    # Note: In the formula, omega' is direction FROM surface TO p? 
    # The formula uses omega' . n_p'. Usually omega' is direction of incoming light at p'.
    # In the integral change of variable, omega' = (p' - p) / ||p' - p||.
    # So omega_prime in sample_scene_point should be direction from p to p'.
    
    # Radiance from surface
    L_e = scene.get_emission(p_prime, -omega_prime) # Assuming L_e is emitted in direction -omega_prime towards p
    
    # Jacobian term: |omega' . n_p'| / dist^2
    # Note: omega_prime here is direction from p to p'. 
    # The normal n_p' is at p'.
    cos_theta = abs(dot(omega_prime, normal))
    jacobian = cos_theta / (dist * dist)
    
    # Phase function rho(p, omega, omega_in)
    # omega is direction towards camera. omega_in is direction from light source (p') to p.
    # omega_in = -omega_prime
    omega_in = -omega_prime
    phase = phase_function(p, omega, omega_in)
    
    # Contribution
    contribution = phase * L_e * transmittance * jacobian
    
    # Divide by PDF of sampling p'
    if pdf_surface > 0:
        return contribution / pdf_surface
    else:
        return 0.0
```

**Edge Cases and Pitfalls:**
*   **Ray Differentials:** Determining ray differentials for volumetric scattering is an unsolved problem in the referenced context. Ray differentials should be disabled by setting `RayDifferential ray_diff = RayDifferential{Real(0), Real(0)}` [S2, S3].
*   **Zero Extinction:** If $\sigma_t = 0$, the sampling formula involves division by zero. This case is not explicitly handled in the sources but implies no attenuation [G].
*   **Monochromatic Assumption:** The method assumes monochromatic coefficients. Applying it to RGB requires running the calculation for each channel independently or ensuring $\sigma_a$ and $\sigma_s$ are scalar values applied to all channels [S2, S3].
*   **No Environment Maps:** The method assumes no environment maps; all light comes from surface emission $L_e$ [S2, S3].

## Operating characteristics
**Time Complexity:** The method requires one ray intersection per pixel for the primary ray, plus additional ray intersections for sampling the single-scattering integral over the scene manifold. The complexity depends on the number of samples used to estimate the integral in $L_{\text{scatter1}}$ [S2, S3].

**Memory Complexity:** Low, as it only requires storing the scene geometry and medium coefficients. No volumetric grids or large buffers are needed [S1, unofficial][S2, S3].

**Strengths:**
*   Physically plausible single-scattering effects.
*   Simple implementation compared to multiple scattering methods [S2, S3].
*   Efficient for optically thin volumes or when single scattering dominates [S2, S3].

**Limitations:**
*   Ignores multiple scattering, leading to inaccuracies in optically thick volumes [S2, S3].
*   Limited to homogeneous media [S2, S3].
*   Limited to scenes with only emitting surfaces (no reflection) [S2, S3].
*   Ray differentials are not supported [S2, S3].

**When to Use:**
*   When rendering homogeneous participating media with single scattering dominance.
*   When computational resources are limited and multiple scattering is not required.
*   When the scene consists of emitting surfaces and no environment maps [S2, S3].

**When Not to Use:**
*   For optically thick volumes where multiple scattering is significant.
*   For heterogeneous media.
*   For scenes with reflective surfaces or environment maps (as per the specific implementation constraints) [S2, S3].

## Background
This method is part of a series of volumetric path tracing homework assignments from the University of California, San Diego (CSE 272) [S2, S3]. It builds upon the absorption-only model by adding single-scattering contributions. The derivation follows the radiative transfer equation, converting the differential form into an integral form suitable for Monte Carlo sampling [S2, S3]. The specific implementation details, such as the handling of ray differentials and the absence of environment maps, are specific to the homework context [S2, S3]. The term "SingleScatterTracer" appears in the Lyr.jl library documentation [S1, unofficial].

## Sources
[S1] Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] PDF Volumetric Path Tracing - University of California, San Diego, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S3] Volumetric Path Tracing, https://cseweb.ucsd.edu/~tzli/cse272/wi2022/homework2.pdf, fetched 2026-10-30
