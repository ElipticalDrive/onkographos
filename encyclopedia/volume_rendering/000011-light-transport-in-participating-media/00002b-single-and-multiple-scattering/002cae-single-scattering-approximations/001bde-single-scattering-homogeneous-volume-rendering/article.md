# Single-scattering homogeneous volume rendering

Also called: Single monochromatic homogeneous volume with absorption and single-scattering, SingleScatterTracer

## Overview
This method renders participating media by solving the radiative transfer equation under the assumption that light scatters exactly once within a single, homogeneous volume before reaching the camera [S2]. It extends an absorption-only model by adding in-scattering from surface-emitting geometry, assuming monochromatic coefficients and surfaces that do not reflect or transmit light [S2]. It is distinguished from multiple scattering methods by ignoring secondary bounces within the medium and from heterogeneous methods by assuming constant extinction and scattering coefficients throughout the volume [S2].

## How it works
### Inputs and outputs
**Inputs:**
*   Camera ray origin $\mathbf{p}(0)$ and direction $\omega$.
*   Scene geometry $\mathcal{M}$, consisting of surfaces that emit light with radiance $L_e$. Surfaces do not reflect or transmit light [S2].
*   Volume parameters: extinction coefficient $\sigma_t$, scattering coefficient $\sigma_s$, and absorption coefficient $\sigma_a$ (where $\sigma_t = \sigma_a + \sigma_s$). These are constant (homogeneous) and monochromatic (same value for all color channels) [S2].
*   Phase function $\rho(\mathbf{p}(t), \omega, \omega')$ [S2].
*   Random number generator state [S2].

**Outputs:**
*   Radiance $L_2(\mathbf{p}(0), \omega)$ at the camera origin [S2].

### Data structures
The method is encapsulated in a type representing the integration strategy [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
struct SingleScatterTracer <: VolumeIntegrationMethod
    # No specific fields required for the algorithm logic itself,
    # as parameters are passed or accessed from scene context.
    pass
end
```

### Formulas
The total radiance $L_2$ at the camera is the sum of the attenuated surface emission and the single-scattering in-scattering term [S2]:

$$
L_2(\mathbf{p}(0), \omega) = \int_0^{t_{\text{hit}}} \exp(-\sigma_t t) \sigma_s L_{\text{scatter1}}(\mathbf{p}(t), \omega) \, dt + \exp(-\sigma_t t_{\text{hit}}) L_e(\mathbf{p}(t_{\text{hit}}))
$$

where $t_{\text{hit}}$ is the distance to the first surface intersection along the ray [S2].

The single-scattering radiance $L_{\text{scatter1}}$ at a point $\mathbf{p}(t)$ in the direction $\omega$ is the integral over the scene manifold $\mathcal{M}$ [S2]:

$$
L_{\text{scatter1}}(\mathbf{p}(t), \omega) = \int_{\mathcal{M}} \rho(\mathbf{p}(t), \omega, \omega') L_e(\mathbf{p}', \omega') \exp(-\sigma_t \|\mathbf{p}(t) - \mathbf{p}'\|) \frac{|\omega' \cdot \mathbf{n}_{\mathbf{p}'}|}{\|\mathbf{p}(t) - \mathbf{p}'\|^2} \text{visible}(\mathbf{p}(t), \mathbf{p}') \, d\mathbf{p}'
$$

where:
*   $\mathbf{p}'$ is a point on the emitting surface.
*   $\omega'$ is the direction from $\mathbf{p}(t)$ to $\mathbf{p}'$, i.e., $\omega' = \frac{\mathbf{p}' - \mathbf{p}(t)}{\|\mathbf{p}' - \mathbf{p}(t)\|}$.
*   $\mathbf{n}_{\mathbf{p}'}$ is the normal at $\mathbf{p}'$.
*   $\text{visible}(\mathbf{p}(t), \mathbf{p}')$ is 1 if the path is unoccluded, 0 otherwise.
*   $\exp(-\sigma_t \|\mathbf{p}(t) - \mathbf{p}'\|)$ is the transmittance between the scattering point and the light source point [S2].

### Steps
1.  **Intersect Ray:** Compute $t_{\text{hit}}$, the distance to the nearest surface intersection along the camera ray. If no intersection exists, $t_{\text{hit}} = \infty$ [S2].
2.  **Sample Scattering Distance:** Generate a uniform random number $u \in [0, 1]$. Sample the scattering distance $t$ using inverse transform sampling based on the PDF $p(t) = \sigma_t \exp(-\sigma_t t)$ [S2]:
    $$
    t = \frac{\log(1 - u)}{-\sigma_t}
    $$
    This samples from the transmittance distribution $\exp(-\sigma_t t)$ [S2].
3.  **Evaluate Contribution:**
    *   **Case 1: $t < t_{\text{hit}}$ (Scattering occurs inside volume):**
        *   Calculate the scattering point $\mathbf{p}(t) = \mathbf{p}(0) + t\omega$ [S2].
        *   Compute $L_{\text{scatter1}}(\mathbf{p}(t), \omega)$ by integrating over the scene geometry $\mathcal{M}$. This typically involves Monte Carlo sampling of the surface points $\mathbf{p}'$ [S2].
        *   The contribution is $\frac{\sigma_s L_{\text{scatter1}}(\mathbf{p}(t), \omega)}{p(t)} = \frac{\sigma_s L_{\text{scatter1}}(\mathbf{p}(t), \omega)}{\sigma_t \exp(-\sigma_t t)}$ [S2]. Note: The source pseudocode implies computing the term $\exp(-\sigma_t t) \sigma_s L_{\text{scatter1}}$ and dividing by the PDF $p(t)$ for the estimator [S2].
    *   **Case 2: $t \geq t_{\text{hit}}$ (Ray hits surface):**
        *   The contribution is the attenuated surface emission: $\frac{\exp(-\sigma_t t_{\text{hit}}) L_e(\mathbf{p}(t_{\text{hit}}))}{P(t \geq t_{\text{hit}})}$ [S2].
        *   The probability of this event is $P(t \geq t_{\text{hit}}) = \exp(-\sigma_t t_{\text{hit}})$ [S2].
        *   Thus, the contribution simplifies to $L_e(\mathbf{p}(t_{\text{hit}}))$ [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_single_scatter_homogeneous(camera_ray, scene, sigma_t, sigma_s, phase_function, rng):
    """
    Renders a single pixel using single-scattering homogeneous volume approximation.
    
    Args:
        camera_ray: Ray with origin p0 and direction omega.
        scene: Object with intersect() and sample_light() methods.
        sigma_t: Extinction coefficient (scalar, monochromatic).
        sigma_s: Scattering coefficient (scalar, monochromatic).
        phase_function: Function rho(p, omega_in, omega_out).
        rng: Random number generator.
        
    Returns:
        Radiance L2.
    """
    # 1. Intersect ray with scene geometry
    isect = scene.intersect(camera_ray)
    t_hit = isect.distance if isect.hit else float('inf')
    
    # 2. Sample scattering distance t from PDF p(t) = sigma_t * exp(-sigma_t * t)
    u = rng.uniform(0.0, 1.0)
    # t = log(1-u) / -sigma_t
    t = -math.log(1.0 - u) / sigma_t
    
    # 3. Evaluate contribution based on whether scattering happens before or after surface hit
    if t < t_hit:
        # Scattering occurs inside the volume
        p_scatter = camera_ray.origin + t * camera_ray.direction
        
        # Compute single-scattering radiance L_s1 at p_scatter
        # This requires integrating over the scene manifold M (light sources)
        # Using Monte Carlo estimation for the integral over M
        L_s1 = estimate_single_scattering_radiance(p_scatter, camera_ray.direction, scene, phase_function, rng)
        
        # PDF value for the sampled t
        pdf_t = sigma_t * math.exp(-sigma_t * t)
        
        # Contribution: (sigma_s * L_s1 * exp(-sigma_t * t)) / pdf_t
        # Note: exp(-sigma_t * t) / pdf_t = 1 / sigma_t
        # So contribution is (sigma_s / sigma_t) * L_s1
        # However, strictly following the integral form L2 = int(exp(-st*t)*ss*L_s1 dt) + ...
        # The estimator for the integral term int_0^thit exp(-st*t) ss L_s1 dt
        # using importance sampling with p(t) is:
        # (exp(-st*t) * ss * L_s1) / p(t)
        transmittance = math.exp(-sigma_t * t)
        contribution = (transmittance * sigma_s * L_s1) / pdf_t
        
        return contribution
        
    else:
        # Ray hits the surface before scattering
        # Contribution: exp(-sigma_t * t_hit) * L_e(p(t_hit)) / P(t >= t_hit)
        # P(t >= t_hit) = exp(-sigma_t * t_hit)
        # Contribution simplifies to L_e(p(t_hit))
        
        # Get surface emission at intersection point
        # Assuming scene.intersect returns emission if hit
        L_e = isect.emission if isect.hit else 0.0
        
        return L_e

def estimate_single_scattering_radiance(p_scatter, omega, scene, phase_function, rng):
    """
    Estimates L_s1(p, omega) by integrating over the scene manifold M.
    L_s1(p, omega) = int_M rho(p, omega, omega') L_e(p', omega') exp(-sigma_t ||p-p'||) 
                     * |omega' . n_p'| / ||p-p'||^2 * visible(p, p') dA'
    """
    # Monte Carlo integration over light sources/surfaces
    # Sample a point p' on the scene geometry
    light_sample = scene.sample_light(rng)
    p_light = light_sample.position
    L_e_light = light_sample.emission
    n_light = light_sample.normal
    
    # Direction from scatter point to light
    to_light = p_light - p_scatter
    dist_sq = dot(to_light, to_light)
    dist = math.sqrt(dist_sq)
    
    if dist < 1e-6:
        return 0.0
        
    omega_prime = to_light / dist
    
    # Visibility check
    if not scene.is_visible(p_scatter, p_light):
        return 0.0
        
    # Phase function
    rho = phase_function(p_scatter, omega, omega_prime)
    
    # Geometry term
    # |omega' . n_p'| / ||p - p'||^2
    # Note: omega' is direction FROM p_scatter TO p_light.
    # The formula uses omega' as direction from p(t) to p'.
    # The normal n_p' is at p'.
    cos_theta = abs(dot(omega_prime, n_light))
    geometry_term = cos_theta / dist_sq
    
    # Transmittance
    transmittance = math.exp(-scene.sigma_t * dist)
    
    # Integrand
    integrand = rho * L_e_light * transmittance * geometry_term
    
    # Monte Carlo estimator: integrand / PDF of sampling p'
    # The PDF of sampling p' depends on the scene's sampling strategy.
    # Assuming scene.sample_light returns a sample with associated PDF p(p')
    pdf_light = light_sample.pdf
    
    if pdf_light == 0:
        return 0.0
        
    return integrand / pdf_light
```

### Parameters
*   $\sigma_t$: Extinction coefficient. Must be constant and positive [S2].
*   $\sigma_s$: Scattering coefficient. Must be constant and positive [S2].
*   $\sigma_a$: Absorption coefficient. $\sigma_a = \sigma_t - \sigma_s$ [S2].
*   $\rho$: Phase function. Describes the angular distribution of scattered light [S2].

### Edge cases and pitfalls
*   **$t_{\text{hit}} = \infty$:** If the ray does not intersect any surface, the second term (surface emission) is zero. The algorithm relies entirely on the scattering term. If no light sources are visible, the result is black [S2].
*   **Precision:** Computing $\log(1-u)$ can suffer from precision loss if $u$ is very close to 1. It is safer to compute $1-u$ first or use `log1p(-u)` if available [S2].
*   **Monochromatic Assumption:** The method assumes $\sigma_a$ and $\sigma_s$ are scalar values, not vectors. For RGB rendering, the same logic applies to each channel independently, but the coefficients must be identical across channels [S2].
*   **No Environment Map:** The method assumes no environment map; all light comes from surface emission $L_e$ [S2].

## Subtopics

### [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md)
Uses Monte Carlo sampling for point lights specifically, potentially simplifying the integration over $\mathcal{M}$ compared to general geometry.

### [Single-Scattering Atmosphere Rendering](3289-single-scattering-atmosphere-rendering.md)
Specialized for atmospheric volumes, often using spherical geometry and specific phase functions (e.g., Henyey-Greenstein).

### [Single Scattering Approximation](5264-single-scattering-approximation.md)
A broader category that may include heterogeneous volumes or different sampling strategies.

## Operating characteristics
*   **Time Complexity:** Dominated by the cost of computing $L_{\text{scatter1}}$, which requires integrating over the scene geometry. This is typically $O(N)$ where $N$ is the number of light sources or triangles, per pixel per sample [S2].
*   **Memory Complexity:** $O(1)$ per pixel, as no volume grid is stored; the volume is defined by coefficients [S2].
*   **Strengths:** Physically plausible single-bounce scattering; efficient compared to multiple scattering methods; simple implementation [S2].
*   **Limitations:** No multiple scattering; assumes homogeneous medium; assumes monochromatic coefficients; assumes surfaces only emit light (no reflections) [S2].
*   **When to use:** Scenes with thin or optically thin volumes where single scattering dominates; educational purposes; real-time applications where multiple scattering is too expensive [S2].
*   **When not to use:** Dense volumes (fog, smoke) where multiple scattering is significant; scenes with reflective surfaces interacting with the volume [S2].

## Background
The method is derived from the radiative transfer equation, specifically the single-scattering approximation [S2]. It is presented in the context of volumetric path tracing homework assignments at the University of California, San Diego [S2]. The implementation structure is inspired by the `Lyr.jl` library's stencil API, which provides a framework for volume integration methods [S1, unofficial].

## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] PDF Volumetric Path Tracing - University of California, San Diego, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
