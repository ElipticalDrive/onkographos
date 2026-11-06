# Free-path sampling
Also called: Homogeneous Volume Tracking, Single Scattering Volume Path Tracing with Transmittance Sampling, Single Scattering with Surface Emission, Tracking in Homogeneous Volumes, Vol Path Tracing 1

## Overview
Free-path sampling is a technique for simulating light transport in participating media by randomly determining the distance a ray travels before interacting with the medium, thereby avoiding explicit distance and spherical integrations for single scattering events [S1, unofficial]. It differs from sibling methods like [Majorant Iterator Ray Sampling](3075-majorant-iterator-ray-sampling.md) by directly sampling the interaction distance based on the extinction coefficient rather than iterating through a majorant bound, and it serves as the foundational step for [Homogeneous Medium Sampling](980-homogeneous-medium-sampling.md).

## How it works
**Inputs:** Ray origin $x$, direction $\omega$, medium extinction coefficient $\sigma_t$ (sum of absorption $\sigma_a$ and scattering $\sigma_s$), and scene geometry (surface intersection distance $d_{surf}$).
**Outputs:** Interaction distance $t$, interaction type (volume or surface), and resulting radiance contribution.

**Data Structures:**
*   `Ray`: struct containing `origin` (vec3), `direction` (vec3), `t_max` (float).
*   `Medium`: struct containing `sigma_t` (float), `sigma_s` (float), `sigma_a` (float).
*   `Intersection`: struct containing `point` (vec3), `distance` (float), `is_surface` (bool).

**Steps:**
1.  **Sample Free Flight Distance:** Generate a uniform random number $\xi \in [0, 1)$. Calculate the free flight distance $t_{free}$ using the inverse cumulative distribution function (CDF) of the exponential distribution derived from the Beer-Lambert law:
    $$ t_{free} = -\frac{\ln(1 - \xi)}{\sigma_t} $$
    [S1, unofficial]

2.  **Determine Interaction Type:** Compare $t_{free}$ with the distance to the nearest surface intersection $d_{surf}$ along the ray direction.
    *   If $t_{free} < d_{surf}$: The ray interacts with the volume.
    *   If $t_{free} \ge d_{surf}$: The ray hits the surface before interacting with the volume.

3.  **Handle Volume Interaction:**
    *   Compute the transmittance from the ray origin to the interaction point $x_t = x + t_{free}\omega$:
        $$ T(x, x_t) = e^{-\sigma_t t_{free}} $$
    *   Sample a new direction $\omega'$ from the phase function $p(\omega' \leftarrow \omega)$ at $x_t$ [S1, unofficial].
    *   Compute the in-scattered radiance contribution. For a single light source with flux $\Phi$ at distance $D$ and angle $\theta$, the contribution is:
        $$ L_{in} = \sigma_s T(x, x_t) T(x_t, K) Vis(x_t, K) \frac{\Phi}{D^2 + t^2} \frac{1}{pdf(t)} $$
        where $pdf(t)$ is the probability density of the sampling strategy used for the light direction [S1, unofficial].
    *   Recursively trace the new ray from $x_t$ in direction $\omega'$ to compute indirect lighting [S1, unofficial].

4.  **Handle Surface Interaction:**
    *   Proceed with standard path tracing code for the surface at $d_{surf}$ [S1, unofficial].
    *   Evaluate the BRDF and perform Next Event Estimation (NEE) if applicable [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def free_path_sample(ray, medium, scene):
    # Step 1: Sample free flight distance
    xi = random_uniform(0.0, 1.0)
    sigma_t = medium.sigma_a + medium.sigma_s
    
    # Avoid division by zero if medium is non-existent
    if sigma_t == 0:
        t_free = float('inf')
    else:
        t_free = -log(1.0 - xi) / sigma_t
        
    # Step 2: Find nearest surface intersection
    intersection = scene.intersect(ray)
    d_surf = intersection.distance if intersection else float('inf')
    
    # Step 3: Determine interaction type
    if t_free < d_surf:
        # Volume interaction
        t = t_free
        x_t = ray.origin + t * ray.direction
        
        # Calculate transmittance to interaction point
        transmittance = exp(-sigma_t * t)
        
        # Sample phase function for new direction
        omega_new = sample_phase_function(medium.phase, ray.direction)
        
        # Compute in-scattered radiance (simplified single light example)
        # Note: Full implementation requires NEE loop and specific pdf calculation
        # as described in [S1, unofficial] for equi-angular sampling or similar
        radiance = compute_inscatter(ray.origin, x_t, omega_new, medium, scene)
        
        # Recursive call for indirect lighting
        new_ray = Ray(x_t, omega_new)
        radiance += free_path_sample(new_ray, medium, scene)
        
        return radiance * transmittance
    else:
        # Surface interaction
        # Standard path tracing logic
        return surface_path_trace(ray, intersection, scene)
```

**Parameters:**
*   $\sigma_t$: Extinction coefficient. Typical values depend on the medium density.
*   $\xi$: Uniform random number.

**Edge Cases:**
*   If $\sigma_t = 0$, the medium is transparent; set $t_{free} = \infty$ to ensure surface intersection is handled [G].
*   Precision issues may arise if $t_{free}$ is very close to $d_{surf}$; a small epsilon bias may be needed to prevent self-intersection artifacts [G].

## Subtopics

### Unidirectional transmittance-based path sampling
A method to construct paths by incrementally adding vertices from one endpoint (light or eye) until a light source or sensor is hit, sampling propagation distance proportional to transmittance [S2]. It differs from standard free-path sampling by explicitly building the path vertex-by-vertex with importance sampling based on the phase function and transmittance PDF, rather than just tracking a single ray bounce [S2]. Choose this when importance sampling the path construction is required to reduce variance in complex scenes [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per step for distance sampling, but overall complexity depends on the path depth and number of light samples [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory per ray, excluding recursion stack [S1, unofficial].
*   **Performance:** Produces noticeable noise if multiple scattering is handled naively without proper variance reduction techniques [S1, unofficial].
*   **Strengths:** Simple to implement; avoids expensive integration for single scattering [S1, unofficial].
*   **Limitations:** Can be noisy; not suitable for scenes with significant multiple scattering without additional techniques [S1, unofficial].
*   **When to use:** Single scattering dominated scenes or as a base for more complex volume rendering algorithms [S1, unofficial].
*   **When not to use:** Scenes with high multiple scattering or where variance reduction is critical without additional sampling strategies [S1, unofficial].

## Background
Free-path sampling originates from the fundamental principles of light transport in participating media, specifically the Beer-Lambert law which governs transmittance [S1, unofficial]. It is a core component of volumetric path tracing algorithms, allowing for the simulation of light interacting with homogeneous volumes [S1, unofficial]. The method was discussed in the context of implementing volumetric effects in path tracers, highlighting its role in avoiding explicit integration for single scattering events [S1, unofficial].

## Sources
[S1] l0exy1h/VolumetricPathTracing-public, https://github.com/l0exy1h/VolumetricPathTracing-public, fetched 2026-10-26 (unofficial)
[S2] Joint importance sampling of low-order volumetric scattering, https://doi.org/10.1145/2508363.2508411, fetched 2026-10-28
