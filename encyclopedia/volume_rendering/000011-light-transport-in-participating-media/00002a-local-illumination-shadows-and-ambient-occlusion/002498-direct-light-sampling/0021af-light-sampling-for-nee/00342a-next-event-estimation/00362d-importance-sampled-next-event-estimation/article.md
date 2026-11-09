# Importance-Sampled Next Event Estimation

Also called: NEE with Importance Sampling, Next-Habits-Event Estimation

## Overview
Importance-Sampled Next Event Estimation (NEE) is a Monte Carlo technique that reduces variance in direct illumination estimation by decoupling the sample budget from ray casting and using pre-computed importance structures to guide sampling of light paths [S1, S2, unofficial]. It differs from standard NEE by explicitly sampling high-frequency boundary light paths using grid-based or proxy structures rather than uniform emitter sampling, and differs from [Random-Walk Subsurface Scattering](9028-random-walk-subsurface-scattering.md) by focusing on direct boundary interactions rather than volumetric scattering [S2, unofficial].

## How it works
**Inputs:** Scene geometry (implicit or explicit), light sources, pre-computed importance data (grid cells or proxy surfaces), and a surface point $p$ on the boundary.
**Outputs:** A sampled point on the light source and the corresponding radiance contribution.

**Data Structures:**
1.  **Grid-based Structure (for explicit/voxelized scenes):** A regular grid where each cell stores an estimated importance value derived from photon mapping (from lights) and importon mapping (from sensor) [S2, unofficial].
2.  **Proxy Surface (for implicit surfaces):** A proxy mesh $F$ approximating the implicit surface $\phi$, used to sample points $q$ which are then projected onto $\phi$ [S3].
3.  **ABSDF Components:** Factored appearance scattering distribution function components (specular and diffuse) with weights $w_j$ and convolved lobes [S1, unofficial].

**Steps:**
1.  **Pre-processing (Grid Method):** Perform a photon mapping pass from light sources and an importon mapping pass from the sensor to assign an estimated importance value to each grid cell [S2, unofficial].
2.  **Pre-processing (Implicit Method):** For environment maps, pre-compute plane-sphere intersections with uniformly distributed normals to determine plane orientation and build a 1D distribution conditioned on the normal direction [S3].
3.  **Component Selection:** Pick one component between specular and diffuse via uniform sampling [S1, unofficial].
4.  **Specular Sampling:**
    *   Pick one convolved lobe from the set based on weights $w_j$ [S1, unofficial].
    *   Sample the corresponding SGGX distribution [S1, unofficial].
5.  **Diffuse Sampling:**
    *   Assume a fixed $\kappa$ during importance sampling because $\kappa_{ij}$ cannot be determined without $\omega_i$ [S1, unofficial].
    *   Pick one convolved lobe with the fixed $\kappa$ and sample the corresponding SGGX distribution [S1, unofficial].
6.  **Light Path Completion (Grid Method):** Utilize the spatial importance information from the grid to sample the edge and complete the light path [S2, unofficial].
7.  **Light Path Completion (Implicit Method):** Intersect the tangent plane $T_p B$ at point $p$ with the light source $L$. Sample points from the resulting intersection [S3].
8.  **PDF Computation:** Compute the corresponding probability density function (PDF) for the selected sample [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def importance_sample_nee(surface_point_p, light_source, precomputed_data, component_type):
    """
    Performs importance-sampled NEE.
    :param surface_point_p: Point on the boundary surface.
    :param light_source: Light source object.
    :param precomputed_data: Grid importance map or proxy surface data.
    :param component_type: 'specular' or 'diffuse'.
    :return: Sampled light point, radiance, PDF.
    """
    # Step 3: Component Selection (Uniform)
    # In practice, this might be driven by weights w_j for specular components
    if component_type == 'specular':
        # Step 4: Specular Sampling
        # Pick lobe j based on weights w_j
        lobe_index = sample_lobe_by_weight(precomputed_data.lobes, precomputed_data.weights)
        # Sample SGGX distribution for the chosen lobe
        direction = sample_sggx(precomputed_data.lobes[lobe_index])
        light_point = intersect_ray_with_light(surface_point_p, direction, light_source)
        pdf = compute_sggx_pdf(direction, precomputed_data.lobes[lobe_index])
        
    elif component_type == 'diffuse':
        # Step 5: Diffuse Sampling
        # Assume fixed kappa
        fixed_kappa = 1.0 # [G] Placeholder for fixed kappa value; exact value not specified in sources
        # Pick lobe with fixed kappa
        lobe_index = sample_lobe_by_weight(precomputed_data.lobes, precomputed_data.weights, fixed_kappa)
        # Sample SGGX distribution
        direction = sample_sggx(precomputed_data.lobes[lobe_index])
        light_point = intersect_ray_with_light(surface_point_p, direction, light_source)
        pdf = compute_sggx_pdf(direction, precomputed_data.lobes[lobe_index])
        
    else:
        # Fallback for implicit surfaces (Tangent Plane Intersection)
        # Step 7: Intersect tangent plane with light
        tangent_plane = get_tangent_plane(surface_point_p)
        intersection_segment = intersect_plane_with_light(tangent_plane, light_source)
        
        # Sample from intersection
        if light_source.is_environment_map():
            # Use pre-computed 1D distribution conditioned on normal
            direction = sample_from_precomputed_distribution(surface_point_p.normal, precomputed_data.env_map_dist)
        else:
            # Sample uniformly from intersection segment
            direction = sample_from_segment(intersection_segment)
            
        light_point = intersect_ray_with_light(surface_point_p, direction, light_source)
        pdf = compute_pdf(direction, light_source, surface_point_p)

    return light_point, light_point.radiance, pdf
```

**Parameters:**
*   **Sample Budget:** Decoupled from the ray casting sample budget [S1, unofficial].
*   **Grid Resolution:** Must be high for complex scenes, leading to scalability issues [S2, unofficial].
*   **Number of Lights:** Experiments assume one light; handling large numbers of polygons is left for future work [S3].

**Edge Cases and Pitfalls:**
*   **Complex Scenes:** When the scene becomes more complicated, the resolution of the regular grid has to be impractically high to achieve desirable results, leading to scalability issues [S2, unofficial].
*   **Kappa Determination:** For diffuse components, $\kappa_{ij}$ cannot be determined without $\omega_i$, necessitating the assumption of a fixed $\kappa$ [S1, unofficial].
*   **Large Polygon Counts:** Efficient plane intersection with a large number of polygons is not handled in the implicit surface variant [S3].

## Subtopics

### Tangent Plane Light Intersection for NEE
An efficient sampling strategy for implicit surfaces that intersects the tangent plane at a sampled surface point with the light source, then samples points from the resulting intersection, rather than uniformly sampling tangent directions [S3]. It pre-computes plane-sphere intersections for environment maps to build a 1D distribution conditioned on the normal direction, avoiding on-the-fly distribution building [S3]. It assumes one light source in experiments [S3].

## Operating characteristics
*   **Time Complexity:** Pre-processing requires photon/importon mapping passes [S2, unofficial]. Sampling is efficient if the importance structure is accurate, but grid resolution requirements can make pre-processing costly for complex scenes [S2, unofficial].
*   **Memory Complexity:** Grid-based methods require memory proportional to the grid resolution, which can be impractically high for complex scenes [S2, unofficial].
*   **Strengths:** Decouples sample budget from ray casting [S1, unofficial]. Reduces variance for high-frequency boundary light paths [S2, unofficial].
*   **Limitations:** Scalability issues with regular grids in complex scenes [S2, unofficial]. Limited handling of multiple lights in implicit variants [S3].
*   **When to Use:** When direct illumination variance is high due to small or complex light sources, and pre-processing time is acceptable [S1, S2, unofficial].
*   **When Not to Use:** For very complex scenes where grid resolution becomes prohibitive [S2, unofficial].

## Background
Originating from techniques like Next-Habits-Event Estimation (NEE) introduced by Zhang et al. [ZMY* 20] for differentiable rendering, this method uses pre-processed grid-based structures to sample high-frequency boundary light paths [S2, unofficial]. It was further developed to address scalability issues with kd-trees [YLB* 22] and adapted for implicit surfaces using tangent plane intersections [S3]. It is used in physics-based differentiable rendering and level-of-detail rendering pipelines [S1, S2, unofficial].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
[S2] A Survey on Physics-based Differentiable Rendering - arXiv.org, https://arxiv.org/pdf/2504.01402, fetched 2026-10-29 (unofficial)
[S3] Path-Space Differentiable Rendering of Implicit Surfaces, https://doi.org/10.1145/3641519.3657473, fetched 2026-10-30
