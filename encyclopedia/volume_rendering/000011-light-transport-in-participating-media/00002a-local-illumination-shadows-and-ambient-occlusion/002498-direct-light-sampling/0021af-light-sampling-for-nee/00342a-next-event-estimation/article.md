# Next Event Estimation


## Overview
Next Event Estimation (NEE) is a Monte Carlo rendering technique used to reduce variance by estimating direct illumination via sampling a point on the emitter and testing its visibility by casting a shadow ray [S1]. It differs from its parent [Light Sampling for NEE](2616-light-sampling-for-nee.md) by explicitly decoupling the light source sampling from the path vertex generation, thereby ensuring that light sources are always sampled directly rather than relying on random walk termination or indirect visibility [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A current path vertex position $\mathbf{x}$, the outgoing direction $\omega_o$ (or incoming direction $\omega_i$ depending on formulation), the scene geometry (emitters and occluders), and a random number generator.
*   **Output:** An estimate of the direct illumination contribution at $\mathbf{x}$ from the sampled light point.

**Data Structures:**
*   **Scene Representation:** A structure allowing efficient ray-scene intersection queries (e.g., BVH, grid) to test visibility between $\mathbf{x}$ and the sampled light point.
*   **Light Distribution:** A representation of the emitter's spatial and angular distribution to facilitate non-uniform sampling.

**Steps:**
1.  **Sample Light Position:** Generate a sample point $\mathbf{y}$ on the surface of a light emitter. This is done by sampling the light's position distribution $p_L(\mathbf{y})$ [S1].
2.  **Sample Light Direction:** Generate a direction $\omega_i$ from $\mathbf{y}$ towards $\mathbf{x}$. For area lights, this direction is typically $\omega_i = \frac{\mathbf{x} - \mathbf{y}}{\|\mathbf{x} - \mathbf{y}\|}$ [S1].
3.  **Compute Distance:** Calculate the distance $d = \|\mathbf{x} - \mathbf{y}\|$ [S1].
4.  **Visibility Test:** Cast a shadow ray from $\mathbf{x}$ in direction $\omega_i$ (or from $\mathbf{y}$ towards $\mathbf{x}$) to test for occlusion.
    *   If the ray hits an occluder at distance $d' < d$, the light is blocked. The contribution is zero.
    *   If the ray misses all occluders or hits the light source at distance $d$, the light is visible.
5.  **Estimate Contribution:** If visible, compute the direct illumination estimate using the rendering equation term for direct light:
    $$ L_{direct} = \frac{L_e(\mathbf{y}, -\omega_i) f_r(\mathbf{x}, \omega_i, \omega_o) \cos \theta_i}{p_L(\mathbf{y}) p(\omega_i)} $$
    Where:
    *   $L_e(\mathbf{y}, -\omega_i)$ is the radiance emitted by the light at $\mathbf{y}$ in direction $-\omega_i$ [S1].
    *   $f_r(\mathbf{x}, \omega_i, \omega_o)$ is the BRDF at $\mathbf{x}$ [S1].
    *   $\cos \theta_i$ is the cosine of the angle between $\omega_i$ and the surface normal at $\mathbf{x}$ [S1].
    *   $p_L(\mathbf{y})$ is the probability density of sampling point $\mathbf{y}$ on the light [S1].
    *   $p(\omega_i)$ is the probability density of the direction [S1].

**Parameters:**
*   **Number of NEE samples:** Typically one sample per path vertex per pixel per frame, but can be increased for high-quality renders.
*   **Light Sampling Strategy:** Uniform, stratified, or importance-sampled depending on the light distribution.

**Edge Cases and Pitfalls:**
*   **Refractive Boundaries:** NEE is not suitable for participating media with refractive boundaries [S1]. Standard NEE assumes straight-line visibility, which fails when light bends at interfaces. For such cases, Manifold Next Event Estimation (MNEE) is required [S1].
*   **Small Light Sources:** If the light source is small relative to the scene, uniform sampling of the light may result in many shadow rays missing the light entirely, leading to high variance. Importance sampling techniques are recommended [S1].
*   **Displaced Surfaces:** Techniques relying on geometric derivatives (like MNEE) do not handle detailed or displaced surfaces well [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def next_event_estimation(vertex_position, vertex_normal, outgoing_direction, scene, light, rng):
    # 1. Sample a point on the light source
    # Sample position y on light surface according to light's spatial distribution
    light_point, pdf_light = light.sample_position(rng)
    
    # 2. Compute direction from light to vertex
    direction_to_vertex = vertex_position - light_point
    distance = length(direction_to_vertex)
    
    # Avoid division by zero
    if distance < epsilon:
        return 0.0
    
    direction_to_vertex = normalize(direction_to_vertex)
    
    # 3. Visibility test (Shadow Ray)
    # Cast ray from vertex towards light
    shadow_hit = scene.ray_intersect(vertex_position, direction_to_vertex, max_distance=distance)
    
    if shadow_hit is not None and shadow_hit.distance < distance:
        # Occluded
        return 0.0
    
    # 4. Compute Direct Illumination
    # Get emitted radiance from light at the sampled point
    emitted_radiance = light.eval_radiance(light_point, -direction_to_vertex)
    
    # Compute BRDF at vertex
    brdf = scene.material_brdf(vertex_position, direction_to_vertex, outgoing_direction)
    
    # Compute cosine term
    cos_theta = max(0.0, dot(vertex_normal, direction_to_vertex))
    
    # Compute PDF for direction (usually 1/(pi * distance^2) for area lights if sampled uniformly)
    # Note: pdf_light is the spatial PDF. The directional PDF is derived from the spatial sampling.
    # For a simple area light sampled uniformly, the combined PDF is often handled by the light's sample_pdf function.
    # Here we assume light.sample_position returns the combined PDF for the point and direction.
    pdf = pdf_light
    
    if pdf < epsilon:
        return 0.0
    
    # Rendering Equation term
    contribution = (emitted_radiance * brdf * cos_theta) / pdf
    
    return contribution
```

## Subtopics

### [Importance-Sampled Next Event Estimation](7684-importance-sampled-next-event-estimation.md)
Importance-Sampled Next Event Estimation (NEE) is a Monte Carlo technique that reduces variance in direct illumination estimation by decoupling the sample budget from ray casting and using pre-computed importance structures to guide sampling of light paths. It differs from standard NEE by explicitly sampling high-frequency boundary light paths using grid-based or proxy structures rather than uniform emitter sampling, and differs from [Random-Walk Subsurface Scattering](9028-random-walk-subsurface-scattering.md) by focusing on direct boundary interactions rather than volumetric scattering.

### [Random-Walk Subsurface Scattering](9028-random-walk-subsurface-scattering.md)
Random-walk subsurface scattering is a Monte Carlo technique for rendering translucent materials like skin, wax, or marble by simulating light transport within the mesh volume. It is distinguished from standard volume rendering by its efficiency and controllability for materials with well-defined surfaces, avoiding the high cost of volumetric path tracing for multiple scattering events.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per vertex per light sample, assuming constant-time ray intersection. However, the cost is dominated by the ray intersection query, which depends on the scene complexity (e.g., $O(\log N)$ for BVH).
*   **Memory Complexity:** $O(1)$ additional memory per sample, excluding the scene representation.
*   **Strengths:** Significantly reduces variance for direct illumination compared to pure path tracing without NEE, especially for small or distant light sources.
*   **Limitations:** Not suitable for participating media with refractive boundaries [S1]. Can be computationally expensive if many shadow rays are cast.
*   **When to Use:** When rendering scenes with direct illumination from area lights or point lights, especially where variance reduction is critical.
*   **When Not to Use:** When rendering participating media with refractive boundaries, or when the scene geometry is too complex for efficient ray intersection.

## Background
Next Event Estimation (NEE) is a fundamental technique in Monte Carlo rendering, widely used in modern path tracers to improve the efficiency of direct illumination estimation [S1]. It was popularized in the context of path tracing to reduce the noise associated with sampling light sources indirectly [S1]. Hanika et al. [37] proposed Manifold Next Event Estimation (MNEE) to extend NEE to participating media with refractive boundaries, leveraging manifold exploration to connect to the light source through multiple refracting surfaces [S1]. This was later improved by Koerner et al. [38] by searching for a point in the volume which satisfies Fermat’s principle instead of walking over the surface [S1].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
