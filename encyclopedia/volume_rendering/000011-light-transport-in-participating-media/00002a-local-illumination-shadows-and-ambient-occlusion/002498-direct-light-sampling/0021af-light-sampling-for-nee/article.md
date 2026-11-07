# Light Sampling for NEE

Also called: sampleDirectionToLight

## Overview
Light Sampling for NEE estimates direct illumination in participating media by explicitly sampling light sources and their surfaces to generate connection points for path tracing [S1, unofficial]. It reduces noise in direct lighting estimates by concentrating samples on bright emitters rather than relying solely on unidirectional shadow rays [G]. It differs from [Shadow Ray Casting](2796-shadow-ray-casting.md) by using probabilistic sampling with importance weighting, and from [Direct Light Sampling](3337-direct-light-sampling.md) by explicitly computing the geometry and probability density functions (PDFs) required for unbiased Nearest Energy Estimation (NEE) in media [G].

## How it works
**Inputs:**
- `pos`: Current interaction point in the medium (Vec3) [S1, unofficial].
- `ns`: Surface normal at the interaction point (Vec3) [S1, unofficial].
- `sampler`: Random number generator state [S1, unofficial].
- `scene`: Scene object containing light sources [S1, unofficial].

**Outputs:**
- `radiance`: Sampled light radiance (Vec3) [S1, unofficial].
- `dir`: Direction to the sampled light point (Vec3) [S1, unofficial].
- `pdf`: Directional probability density (float) [S1, unofficial].

**Data Structures:**
- Light object: Contains methods `samplePoint` and `Le`, and properties `position` and `ns` [S1, unofficial].

**Steps:**
1. **Select Light Source:** Choose a light source from the scene using a probability distribution $p_{light}$ [S1, unofficial].
   $$ light, p_{light} = scene.sampleLight(sampler) $$
2. **Sample Light Surface:** Generate a point on the selected light's emitting surface [S1, unofficial].
   $$ light\_surf, p_{pos} = light.samplePoint(sampler) $$
3. **Calculate Direction:** Compute the vector from the interaction point to the sampled light point [S1, unofficial].
   $$ dir = normalize(light\_surf.position - pos) $$
4. **Calculate Distance:** Compute the Euclidean distance to the light [S1, unofficial].
   $$ dist\_to\_light = length(light\_surf.position - pos) $$
5. **Calculate Cosine Term:** Compute the cosine of the angle between the surface normal and the direction to the light [S1, unofficial].
   $$ cos = |dot(-dir, light\_surf.ns)| $$
6. **Calculate Directional PDF:** Combine the light selection probability, the position PDF, and geometric terms to form the total PDF for the direction [S1, unofficial].
   $$ pdf = p_{light} \cdot \frac{dist\_to\_light^2}{cos} \cdot p_{pos} $$
   This formula accounts for the solid angle subtended by the light source as seen from $pos$ [G].
7. **Return Radiance:** Evaluate the light's emitted radiance in the direction towards the interaction point [S1, unofficial].
   $$ radiance = light.Le(light\_surf, -dir) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_direction_to_light(pos, ns, sampler, scene):
    # Step 1: Select light source
    light, light_choose_prob = scene.sample_light(sampler)
    
    # Step 2: Sample point on light surface
    light_surf, light_pos_pdf = light.sample_point(sampler)
    
    # Step 3: Calculate direction
    dir_vec = normalize(light_surf.position - pos)
    
    # Step 4: Calculate distance
    dist_to_light = length(light_surf.position - pos)
    
    # Step 5: Calculate cosine
    cos = abs(dot(-dir_vec, light_surf.ns))
    
    # Step 6: Calculate PDF
    pdf = light_choose_prob * (dist_to_light ** 2 / cos) * light_pos_pdf
    
    # Step 7: Return radiance
    radiance = light.Le(light_surf, -dir_vec)
    
    return radiance, dir_vec, pdf
```

**Parameters:**
- $p_{light}$: Probability of selecting a specific light source [S1, unofficial].
- $p_{pos}$: Probability density of the sampled point on the light surface [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Cosine:** If $cos \approx 0$, the PDF becomes undefined [G]. Guard against grazing angles or ensure normals are well-defined [G].
- **Distance Zero:** If $pos$ is on the light surface, $dist\_to\_light \approx 0$ [G]. Handle self-intersection or use a small epsilon [G].
- **PDF Consistency:** The PDF must match the sampling strategy exactly for unbiased results [G]. Mismatched PDFs cause bias [G].

## Subtopics

### Importance-Weighted Light Sampling with MIS
Selects per-vertex lights proportional to each light's estimated contribution plus a 15% uniform floor [S2, unofficial]. Reuses the same probability mass function (PMF) for Multiple Importance Sampling (MIS) when a BSDF ray hits an emitter [S2, unofficial]. Maps hit shapes to light indices for O(1) emitter PDF calculation, removing the need for dynamic_cast scans over all lights [S2, unofficial]. The uniform floor weight is 15% of the total probability mass [S2, unofficial].

### Importance Sampling of Multiple Light Sources
An attempt to use importance sampling to choose which light source to sample (e.g., sun vs. sky) to reduce noise [S3, unofficial]. Tested with 16 and 64 samples per light [S3, unofficial]. However, importance sampling increased visual noise because most of the time the darker sky was not sampled, but when it was it was very strongly weighted, causing bright blue pixels amidst unaffected ones [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Importance-weighted sampling reduces variance for scenes with dominant lights [S2, unofficial]. However, poor importance estimates (e.g., for the darker sky) can increase visual noise because the sky is rarely sampled but carries a high weight when selected [S3, unofficial].
- **Strengths:** Unbiased, handles complex light geometries, integrates well with MIS [G].
- **Limitations:** Requires accurate PDFs, sensitive to light selection strategy, can be noisy if importance weights are poor [G].

## Background
The technique is implemented in volumetric path tracers such as volpt [S1, unofficial] and photon-engine [S2, unofficial]. The exact historical origin of the method is not specified in the sources. It is fundamental to unbiased rendering in participating media, allowing for accurate direct lighting estimation [G].

## Sources
[S1] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S2] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
[S3] CS348b Final Project - Rendering Underwater Scenes, http://www.graphics.stanford.edu/courses/cs348b-competition/cs348b-04/underwater/index.html, fetched 2026-10-30 (unofficial)
