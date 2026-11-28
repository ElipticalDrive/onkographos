# Soft Shadows via Area Lights

Also called: Area Light Shadow Sampling, Area Light Shadows, Partial Occlusion, Shadow Occlusion

## Overview
This method models shadows cast by light sources with spatial extent, allowing points in space to be partially occluded, unlike point sources which cast sharp shadows [S2, unofficial]. It differs from its parent [Shadow Ray Casting](2796-shadow-ray-casting.md) by replacing the single boolean visibility test with an integration over the light source area to compute natural soft shadows with penumbra [S1, unofficial].

## How it works
**Inputs:**
*   Point $\mathbf{p}$ in the volume or on the surface [G].
*   Light source geometry defining an area $A$ [S2, unofficial].
*   Volume density field (for participating media) or distance field (for solid geometry) [S2, unofficial].
*   Number of samples $N$ for the light source area [G].

**Outputs:**
*   Shadow factor $S \in [0, 1]$ representing the fraction of the light source visible from $\mathbf{p}$ [G].

**Data Structures:**
*   Light source area representation: A parametric surface or a set of sample points $\{\mathbf{l}_i\}_{i=1}^N$ distributed over $A$ [G].
*   Volume sampler: Provides extinction coefficient $\sigma_t(\mathbf{x})$ or distance to nearest surface $d(\mathbf{x})$ [G].

**Steps:**
1.  **Sample the Light Source:** Generate $N$ sample points $\mathbf{l}_i$ uniformly (or according to a distribution) over the light source area $A$ [S1, unofficial].
2.  **Cast Shadow Rays:** For each sample $\mathbf{l}_i$, cast a shadow ray from $\mathbf{p}$ towards $\mathbf{l}_i$ [G].
3.  **Evaluate Visibility:**
    *   If using a distance field (solid geometry): Compute the distance $d_i$ from $\mathbf{p}$ to the nearest surface along the direction $\mathbf{v}_i = \mathbf{l}_i - \mathbf{p}$. If $d_i < \|\mathbf{l}_i - \mathbf{p}\|$, the sample is occluded [S2, unofficial].
    *   If using a volume density field: Compute the transmittance $T_i$ from $\mathbf{p}$ to $\mathbf{l}_i$ using the method from [Shadow Ray Casting](2796-shadow-ray-casting.md) [G].
4.  **Accumulate Shadow Factor:**
    $$ S = \frac{1}{N} \sum_{i=1}^{N} V_i $$
    where $V_i = 1$ if the sample is unoccluded, and $V_i = 0$ (or $T_i$) if occluded [G].
5.  **Apply Shadow Factor:** Modulate the direct illumination contribution by $S$ [G].

**Parameters:**
*   $N$: Number of samples. Typical values range from 4 to 64 depending on desired quality and performance [G]. Higher $N$ reduces noise but increases cost [G].

**Edge Cases:**
*   **Small Light Sources:** If the light source is small relative to the distance, $N$ may need to be increased to avoid aliasing [G].
*   **Large Light Sources:** If the light source is very large, the shadow factor approaches 1, and soft shadows become negligible [G].
*   **Precision:** Floating-point precision issues may arise when $\mathbf{p}$ is very close to the light source or occluding geometry [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_area_light_shadow(p, light_samples, distance_field):
    """
    Computes soft shadow factor for point p given light samples and a distance field.
    
    Args:
        p: Point in space (tuple of 3 floats).
        light_samples: List of sample points on the light source area (list of tuples of 3 floats).
        distance_field: Object with method query(origin, direction) returning distance to nearest surface.
        
    Returns:
        Shadow factor S in [0, 1].
    """
    n_samples = len(light_samples)
    if n_samples == 0:
        return 1.0
    
    visible_count = 0.0
    
    for l in light_samples:
        direction = (l[0] - p[0], l[1] - p[1], l[2] - p[2])
        dist_to_light = math.sqrt(direction[0]**2 + direction[1]**2 + direction[2]**2)
        
        if dist_to_light < 1e-6:
            visible_count += 1.0
            continue
            
        unit_dir = (direction[0] / dist_to_light, direction[1] / dist_to_light, direction[2] / dist_to_light)
        
        # [G] Standard distance field ray marcher query
        dist_to_surface = distance_field.query(p, unit_dir)
        
        # If the surface is closer than the light, the sample is occluded
        if dist_to_surface < dist_to_light:
            visible_count += 0.0
        else:
            visible_count += 1.0
            
    return visible_count / n_samples
```

## Subtopics

### [Soft Shadows via Ray Tracing](5136-soft-shadows-via-ray-tracing.md)
This method generates soft shadows from area light sources by tracing coherent packets of shadow rays from primary ray hit points to samples on the light source. It differs from hard shadow casting by enabling smooth transitions between shadowed and unshadowed regions, providing visual cues about object positioning. Unlike rasterization-based impostor approaches, this technique integrates global effects easily due to its ray tracing basis.

### [Point Light Shadow Shader](13086-point-light-shadow-shader.md)
A shader function that applies shadow rays from surface points to a point light, handling plane intersections and soft shadows via Gaussian filtering. It differs from its parent [Soft Shadows via Area Lights](7320-soft-shadows-via-area-lights.md) by modeling a single point source rather than an area, and from [Soft Shadows via Ray Tracing](5136-soft-shadows-via-ray-tracing.md) by using a specific plane-intersection refinement and post-process Gaussian blur rather than direct area-light sampling.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of light samples and $M$ is the number of steps per shadow ray [G].
*   **Memory Complexity:** $O(1)$ auxiliary memory per sample [G].
*   **Strengths:** Produces physically plausible soft shadows with penumbra. Works well with dynamic geometry [S1, S2, unofficial].
*   **Limitations:** Computationally expensive due to multiple shadow rays per point. Noise can appear with low sample counts [S1, unofficial].
*   **When to use:** When realistic soft shadows are required and performance allows for multiple shadow rays [G].
*   **When not to use:** When real-time performance is critical and hard shadows are acceptable, or when the light source is effectively a point source [G].

## Background
The concept of soft shadows arises from the observation that real-world light sources have spatial extent, leading to partial occlusion and penumbra [S2, unofficial]. Early implementations in rasterization used distance-based low-pass filters on shadow maps [S2, unofficial]. In ray tracing, the approach of sampling the light source area instead of performing a single boolean test was introduced to achieve more natural appearing soft shadows [S1, unofficial]. This method is particularly relevant in volume rendering and raymarching, where distance fields can be used to approximate soft shadows efficiently [S2, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] (PDF) Raymarching Distance Fields with CUDA - ResearchGate, https://www.researchgate.net/publication/356081826_Raymarching_Distance_Fields_with_CUDA, fetched 2026-10-27 (unofficial)
