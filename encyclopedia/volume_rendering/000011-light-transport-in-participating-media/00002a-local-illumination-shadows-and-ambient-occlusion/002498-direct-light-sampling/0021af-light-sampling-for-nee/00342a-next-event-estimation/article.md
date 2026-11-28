# Next Event Estimation


## Overview
Next Event Estimation (NEE) is a Monte Carlo rendering technique used to reduce variance by estimating direct illumination via sampling a point on the emitter and testing its visibility by casting a shadow ray [S1]. It differs from its parent [Light Sampling for NEE](2616-light-sampling-for-nee.md) by explicitly targeting direct light paths rather than general light sampling strategies.

## How it works
**Inputs and Outputs:**
*   **Input:** Current path vertex position $\mathbf{x}$, outgoing direction $\omega_o$, scene geometry (emitters and occluders), and a random number generator [G].
*   **Output:** An estimate of the direct illumination contribution at $\mathbf{x}$ from the sampled light point [G].

**Data Structures:**
*   **Scene Representation:** A structure allowing efficient ray-scene intersection queries (e.g., BVH, grid) to test visibility between $\mathbf{x}$ and the sampled light point [G].
*   **Light Distribution:** A representation of the emitter's spatial distribution to facilitate non-uniform sampling [G].

**Steps:**
1.  **Sample Light Position:** Generate a sample point $\mathbf{y}$ on the surface of a light emitter according to its spatial distribution [G].
2.  **Compute Direction and Distance:** Calculate the direction $\omega_i = (\mathbf{x} - \mathbf{y}) / \|\mathbf{x} - \mathbf{y}\|$ and the distance $d = \|\mathbf{x} - \mathbf{y}\|$ [G].
3.  **Visibility Test:** Cast a shadow ray from $\mathbf{x}$ in direction $\omega_i$ to test for occlusion [S1].
4.  **Estimate Contribution:** If the ray is unoccluded, compute the direct illumination estimate using the Monte Carlo estimator [G].

**Formulas:**
$$ \hat{L}_{direct} = \frac{L_e(\mathbf{y}, -\omega_i) f_r(\mathbf{x}, \omega_i, \omega_o) \cos \theta_i}{p(\mathbf{y})} $$
Where:
*   $L_e(\mathbf{y}, -\omega_i)$ is the radiance emitted by the light at $\mathbf{y}$ in direction $-\omega_i$ [G].
*   $f_r(\mathbf{x}, \omega_i, \omega_o)$ is the BRDF at $\mathbf{x}$ [G].
*   $\cos \theta_i$ is the cosine of the angle between $\omega_i$ and the surface normal at $\mathbf{x}$ [G].
*   $p(\mathbf{y})$ is the probability density of sampling point $\mathbf{y}$ on the light [G].

**Parameters:**
*   **Number of NEE samples:** Typically one sample per path vertex per pixel per frame [G].
*   **Light Sampling Strategy:** Uniform, stratified, or importance-sampled depending on the light distribution [G].

**Edge Cases and Pitfalls:**
*   **Refractive Boundaries:** NEE is not suitable for participating media with refractive boundaries [S1]. Standard NEE assumes straight-line visibility, which fails when light bends at interfaces [G]. For such cases, Manifold Next Event Estimation (MNEE) is required [S1].
*   **Displaced Surfaces:** Techniques relying on geometric derivatives (like MNEE) do not handle detailed or displaced surfaces well [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def next_event_estimation(vertex_position, vertex_normal, outgoing_direction, scene, light, rng):
    EPSILON = 1e-6
    # 1. Sample Light Position: Generate a sample point y on the emitter surface
    # Light sampling implementation depends on emitter type [G]
    light_point, pdf_light = light.sample_position(rng)
    
    # 2. Compute Direction and Distance
    direction_to_vertex = vertex_position - light_point
    distance = length(direction_to_vertex)
    
    if distance < EPSILON:
        return 0.0
    
    direction_to_vertex = normalize(direction_to_vertex)
    
    # 3. Visibility Test: Cast a shadow ray from x in direction omega_i
    shadow_hit = scene.ray_intersect(vertex_position, direction_to_vertex, max_distance=distance)
    
    if shadow_hit is not None and shadow_hit.distance < distance:
        return 0.0
    
    # 4. Estimate Contribution: Compute direct illumination estimate
    emitted_radiance = light.eval_radiance(light_point, -direction_to_vertex)
    brdf = scene.material_brdf(vertex_position, direction_to_vertex, outgoing_direction)
    cos_theta = max(0.0, dot(vertex_normal, direction_to_vertex))
    
    if pdf_light < EPSILON:
        return 0.0
    
    contribution = (emitted_radiance * brdf * cos_theta) / pdf_light
    return contribution
```

## Subtopics

### [Importance-Sampled Next Event Estimation](7684-importance-sampled-next-event-estimation.md)
Importance-Sampled Next Event Estimation (NEE) is a Monte Carlo technique that reduces variance in direct illumination estimation by decoupling the sample budget from ray casting and using pre-computed importance structures to guide sampling of light paths [G]. It differs from standard [Next Event Estimation](7209-next-event-estimation.md) by explicitly sampling high-frequency boundary light paths using grid-based or proxy structures rather than uniform emitter sampling, and differs from [Random-Walk Subsurface Scattering](9028-random-walk-subsurface-scattering.md) by focusing on direct boundary interactions rather than volumetric scattering [G]. Choose it when rendering scenes with complex light distributions or high-frequency boundary interactions where uniform sampling causes excessive variance [G].

### [Random-Walk Subsurface Scattering](9028-random-walk-subsurface-scattering.md)
Random-walk subsurface scattering is a Monte Carlo technique for rendering translucent materials like skin, wax, or marble by simulating light transport within a mesh volume [G]. It differs from standard volume rendering by restricting the random walk to the interior of the geometry and terminating at a surface vertex, making it more efficient and easier to control for thin or bounded translucent objects [G]. Unlike [Importance-Sampled Next Event Estimation](7684-importance-sampled-next-event-estimation.md), which samples the light directly, this method samples the scattering path within the material to handle multiple scattering events that single-bounce approximations miss [G]. Choose it when rendering thin or bounded translucent objects where multiple scattering events within the material volume must be captured efficiently [G].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per vertex per light sample, assuming constant-time ray intersection [G]. The cost is dominated by the ray intersection query, which depends on scene complexity (e.g., $O(\log N)$ for BVH) [G].
*   **Memory Complexity:** $O(1)$ additional memory per sample, excluding the scene representation [G].
*   **Measured Performance and Scaling:** Not given in the sources. [G]
*   **Strengths:** Significantly reduces variance for direct illumination compared to pure path tracing without NEE, especially for small or distant light sources [G].
*   **Limitations:** Not suitable for participating media with refractive boundaries [S1]. Can be computationally expensive if many shadow rays are cast [G].
*   **When to Use:** When rendering scenes with direct illumination from area lights or point lights, especially where variance reduction is critical [G].
*   **When Not to Use:** When rendering participating media with refractive boundaries, or when the scene geometry is too complex for efficient ray intersection [G].

## Background
Next Event Estimation (NEE) is a fundamental technique in Monte Carlo rendering, widely used in modern path tracers to improve the efficiency of direct illumination estimation [S1]. Hanika et al. proposed Manifold Next Event Estimation (MNEE) to extend NEE to participating media with refractive boundaries, leveraging manifold exploration to connect to the light source through multiple refracting surfaces [S1]. This was later improved by Koerner et al. by searching for a point in the volume which satisfies Fermat’s principle instead of walking over the surface [S1].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
