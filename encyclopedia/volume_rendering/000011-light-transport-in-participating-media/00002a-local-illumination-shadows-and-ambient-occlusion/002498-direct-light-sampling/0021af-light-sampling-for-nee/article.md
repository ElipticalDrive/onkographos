# Light Sampling for NEE

Also called: sampleDirectionToLight

## Overview
Light Sampling for NEE is a technique for estimating direct illumination in participating media by explicitly sampling light sources and their surfaces to generate connection points for path tracing. It solves the problem of noisy direct lighting estimates by concentrating samples on bright emitters rather than relying solely on unidirectional shadow rays. It differs from [Shadow Ray Casting](2796-shadow-ray-casting.md) by using probabilistic sampling with importance weighting and from [Direct Light Sampling](3337-direct-light-sampling.md) by focusing specifically on the geometry of light emission and the associated probability density functions (PDFs) required for unbiased Nearest Energy Estimation (NEE) in media.

## How it works
**Inputs:**
- Current interaction point in the medium: $pos$ (Vec3).
- Surface normal at the interaction point (if on a boundary): $ns$ (Vec3).
- Sampler state: $sampler$.
- Scene light sources: $scene$.

**Outputs:**
- Sampled light radiance: $L$ (Vec3).
- Direction to light: $dir$ (Vec3).
- Probability density: $pdf$ (float).

**Data Structures:**
- Light Source Object: Contains methods `samplePoint`, `Le`, and properties for position and normal.
- Sampler: Provides random numbers for uniform sampling.

**Steps:**
1. **Select Light Source:** Choose a light source from the scene using a probability distribution $p_{light}$.
   $$ light = scene.sampleLight(sampler, p_{light}) $$
   Where $p_{light}$ is the probability of selecting this specific light source.

2. **Sample Light Surface:** Generate a point on the selected light's emitting surface.
   $$ light\_surf = light.samplePoint(sampler, p_{pos}) $$
   Where $p_{pos}$ is the PDF of the position on the light surface.

3. **Calculate Direction:** Compute the vector from the interaction point to the sampled light point.
   $$ dir = normalize(light\_surf.position - pos) $$

4. **Calculate Distance:** Compute the Euclidean distance to the light.
   $$ dist\_to\_light = length(light\_surf.position - pos) $$

5. **Calculate Cosine Term:** Compute the cosine of the angle between the surface normal and the direction to the light.
   $$ cos = |dot(-dir, light\_surf.ns)| $$
   Note: $-dir$ points from the light to the surface, aligning with the normal convention for emission.

6. **Calculate Directional PDF:** Combine the light selection probability, the position PDF, and geometric terms to form the total PDF for the direction.
   $$ pdf = p_{light} \cdot \frac{dist\_to\_light^2}{cos} \cdot p_{pos} $$
   This formula accounts for the solid angle subtended by the light source as seen from $pos$.

7. **Return Radiance:** Evaluate the light's emitted radiance in the direction towards the interaction point.
   $$ L = light.Le(light\_surf, -dir) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_direction_to_light(pos, ns, sampler, scene):
    # Step 1: Select light source
    # p_light is the probability of choosing this light
    light, p_light = scene.sample_light(sampler)
    
    # Step 2: Sample point on light surface
    # p_pos is the PDF of the position on the light
    light_surf, p_pos = light.sample_point(sampler)
    
    # Step 3: Calculate direction
    dir_vec = normalize(light_surf.position - pos)
    
    # Step 4: Calculate distance
    dist_to_light = length(light_surf.position - pos)
    
    # Step 5: Calculate cosine
    # -dir_vec points from light to pos, aligning with light_surf.normal
    cos = abs(dot(-dir_vec, light_surf.normal))
    
    # Step 6: Calculate PDF
    # Formula: p_light * (dist^2 / cos) * p_pos
    pdf = p_light * (dist_to_light ** 2 / cos) * p_pos
    
    # Step 7: Return radiance
    # Le evaluates emitted radiance in direction -dir_vec
    radiance = light.Le(light_surf, -dir_vec)
    
    return radiance, dir_vec, pdf
```

**Parameters:**
- $p_{light}$: Probability of selecting a specific light source. Typically uniform or importance-weighted.
- $p_{pos}$: Probability density of the sampled point on the light surface. Depends on the light's geometry (e.g., area, spherical).

**Edge Cases and Pitfalls:**
- **Zero Cosine:** If $cos \approx 0$, the PDF becomes undefined. Guard against grazing angles or ensure normals are well-defined.
- **Distance Zero:** If $pos$ is on the light surface, $dist\_to\_light \approx 0$. Handle self-intersection or use a small epsilon.
- **PDF Consistency:** The PDF must match the sampling strategy exactly for unbiased results. Mismatched PDFs cause bias.

## Subtopics

### Importance-Weighted Light Sampling with MIS
Selects per-vertex lights proportional to each light's estimated contribution plus a 15% uniform floor [S2, unofficial]. Reuses the same probability mass function (PMF) for Multiple Importance Sampling (MIS) when a BSDF ray hits an emitter [S2, unofficial]. Maps hit shapes to light indices for O(1) emitter PDF calculation, removing the need for dynamic_cast scans over all lights [S2, unofficial]. The uniform floor weight is 15% of the total probability mass [S2, unofficial].

### Importance Sampling of Multiple Light Sources
An attempt to use importance sampling to choose which light source to sample (e.g., sun vs. sky) to reduce noise [S3, unofficial]. Tested with 16 and 64 samples per light [S3, unofficial]. However, importance sampling increased visual noise because the darker sky was rarely sampled but had high weight when sampled, causing bright blue pixels amidst unaffected ones [S3, unofficial].

## Operating characteristics
- **Time Complexity:** O(1) per light sample if light selection and point sampling are O(1). With importance weighting and MIS, overhead increases slightly due to contribution estimation.
- **Memory Complexity:** O(L) where L is the number of light sources, for storing light properties and indices.
- **Performance:** Importance-weighted sampling reduces variance for scenes with dominant lights [S2, unofficial]. However, poor importance estimates (e.g., for dark emitters) can increase noise [S3, unofficial].
- **Strengths:** Unbiased, handles complex light geometries, integrates well with MIS.
- **Limitations:** Requires accurate PDFs, sensitive to light selection strategy, can be noisy if importance weights are poor.

## Background
Originated in the context of path tracing and volumetric rendering, specifically in implementations like volpt [S1, unofficial] and photon-engine [S2, unofficial]. The technique is fundamental to unbiased rendering in participating media, allowing for accurate direct lighting estimation.

## Sources
[S1] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S2] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
[S3] CS348b Final Project - Rendering Underwater Scenes, http://www.graphics.stanford.edu/courses/cs348b-competition/cs348b-04/underwater/index.html, fetched 2026-10-30 (unofficial)
