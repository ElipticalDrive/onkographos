# Path Tracing with Volume Interaction

Also called: Path Tracing for Volume Lighting

## Overview
Path Tracing with Volume Interaction is a global illumination algorithm that handles light transport through participating media by treating volume intersections as potential scattering events within the path generation loop [S1, unofficial]. It distinguishes itself from pure surface path tracers by integrating volume sampling directly into the ray intersection logic, allowing for multiple scattering in heterogeneous media, and from pure ray marching methods by using stochastic path sampling rather than deterministic integration for indirect light [S2, unofficial].

## How it works
**Inputs:**
*   Scene geometry and volume data (density fields, transfer functions).
*   Camera ray $(o, d)$.
*   Sampler instance for random number generation.
*   Settings: `minBounces`, `maxBounces`, `gradientFactor`, `stepSize`, `backgroundLight`.

**Outputs:**
*   Radiance color $C$ for the pixel.

**Data Structures:**
*   `ScatterEvent`: Stores intersection position, normal, material ID, and index of refraction ($iorI, iorO$).
*   `LightHit`: Stores light color, direction, distance, and PDF.
*   `Material`: Defines BRDF/BSDF properties and emission.

**Algorithm Steps:**

1.  **Initialization:**
    *   Set throughput $T = (1.0, 1.0, 1.0)$.
    *   Set accumulated color $C = (0, 0, 0)$.
    *   Set initial index of refraction $interaction.iorI = 1.0$ (air).
    *   Loop $bounces$ from $0$ to $maxBounces - 1$.

2.  **Intersection:**
    *   Intersect the current ray with the scene.
    *   If no intersection is found:
        *   $C \leftarrow C + T \times backgroundLight$.
        *   Break loop.

3.  **Volume vs. Surface Decision:**
    *   If the intersection is a **Volume**:
        *   Compute local position relative to volume origin.
        *   Get volume gradient to determine normal magnitude and sample intensity.
        *   Calculate probability of treating as surface (BRDF) using [S1, unofficial]:
            $$probBRDF = 1.0 - \exp(-settings.gradientFactor \times \frac{magnitude}{intensity})$$
        *   If $sampler.nextFloat() < probBRDF$:
            *   Treat as surface.
            *   Shift position along normal by $settings.stepSize$.
            *   $T \leftarrow T \times TransferFunctionRGBA$.
            *   Retrieve material from transfer function.
        *   Else (treat as isotropic volume):
            *   $T \leftarrow T \times 2.0$.
            *   Create Isotropic material using transfer function color.
    *   If the intersection is a **Surface**:
        *   Retrieve material from $scene.materials$ using $intersection.materialID$.

4.  **Emission:**
    *   If the material is a light source AND ($bounces == 0$ OR sampled lobe is Specular):
        *   $C \leftarrow C + T \times hitLight.getEmittance(interaction)$.

5.  **Direct Lighting:**
    *   $C \leftarrow C + T \times SampleOneLight(interaction, scene, material, hitLight, settings, sampler)$.
    *   *Note:* `SampleOneLight` handles point, directional, and object lights with specific PDF calculations [S1, unofficial].

6.  **Next Direction Sampling:**
    *   Sample next direction from material's BSDF.
    *   If PDF is NaN (e.g., parallel direction), resample until valid.
    *   Update throughput:
        $$T \leftarrow T \times \frac{MaterialEvaluate(interaction)}{pdf}$$
    *   If sampled lobe is SpecularTransmission:
        *   $interaction.iorI \leftarrow interaction.iorO$.

7.  **Ray Update:**
    *   New ray origin: $interaction.position$.
    *   New ray direction: $interaction.inputDir$ (sampled direction).
    *   New ray minT: $RAY\_EPS$.

8.  **Russian Roulette:**
    *   If $bounces \ge minBounces - 1$:
        *   $p = \max(T_r, T_g, T_b)$.
        *   If $sampler.nextFloat() > p$:
            *   Break loop.
        *   Else:
            *   $T \leftarrow T / p$.

9.  **Final Output:**
    *   Return $C / (1 + bounces)$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def path_trace_volume(ray_o, ray_d, scene, sampler, settings):
    throughput = Vec3f(1.0, 1.0, 1.0)
    color = Vec3f(0.0, 0.0, 0.0)
    ior_i = 1.0
    
    for bounces in range(settings.max_bounces):
        # 1. Intersect
        hit = scene.intersect(ray_o, ray_d)
        
        if hit is None:
            color += throughput * settings.background_light
            break
            
        # 2. Volume/Surface Logic
        if hit.type == VOLUME:
            local_pos = hit.position - hit.volume_origin
            magnitude = hit.volume_gradient(local_pos).length()
            intensity = hit.volume_intensity(local_pos)
            
            prob_brdf = 1.0 - exp(-settings.gradient_factor * (magnitude / intensity))
            
            if sampler.next_float() < prob_brdf:
                # Treat as surface
                hit.position += hit.normal * settings.step_size
                throughput *= hit.transfer_function_rgba
                material = hit.transfer_function_material
            else:
                # Treat as isotropic volume
                throughput *= 2.0
                material = IsotropicMaterial(hit.transfer_function_color)
        else:
            material = scene.materials[hit.material_id]
            
        # 3. Emission
        if material.is_light and (bounces == 0 or material.sampled_lobe == SPECULAR):
            light_hit = material.get_emittance(hit)
            color += throughput * light_hit.color
            
        # 4. Direct Lighting
        direct = sample_one_light(hit, scene, material, settings, sampler)
        color += throughput * direct.color
        
        # 5. Next Direction
        pdf = 0.0
        while not isfinite(pdf):
            new_dir, pdf = material.sample(hit, sampler)
            
        throughput *= material.evaluate(hit, new_dir) / pdf
        
        if material.sampled_lobe == SPECULAR_TRANSMISSION:
            ior_i = hit.ior_o
            
        # 6. Update Ray
        ray_o = hit.position
        ray_d = new_dir
        ray_o += ray_d * RAY_EPS
        
        # 7. Russian Roulette
        if bounces >= settings.min_bounces - 1:
            p = max(throughput.r, throughput.g, throughput.b)
            if sampler.next_float() > p:
                break
            throughput /= p
            
    return color / (1.0 + bounces)
```

**Parameters:**
*   `settings.minDepth`: Minimum bounces before Russian Roulette applies [S1, unofficial].
*   `settings.maxDepth`: Maximum bounces allowed [S1, unofficial].
*   `settings.gradientFactor`: Controls the probability of treating a volume interaction as a surface [S1, unofficial].
*   `settings.stepSize`: Offset distance for surface shift in volume [S1, unofficial].

**Edge Cases:**
*   **NaN PDF:** If the sampled direction is parallel to the normal or otherwise invalid, the PDF may be NaN. The algorithm must resample until a valid PDF is obtained [S1, unofficial].
*   **Heterogeneous Media:** Standard homogeneous assumptions fail; Woodcock tracking is required for unbiased results in heterogeneous media [S2, unofficial].

## Subtopics

### [Emissive Volume Rendering](3398-emissive-volume-rendering.md)
Emissive Volume Rendering extends path tracing to support participating media that emit light, integrating volume emission sampling directly into medium interaction routines. It avoids the performance penalties of direct solid-angle-to-volume PDF conversion for Multiple Importance Sampling (MIS).

### [Neural Photon Field Rendering](3427-neural-photon-field-rendering.md)
Neural Photon Field Rendering is a GPU-based method that estimates direct illumination via ray marching and approximates indirect global illumination by querying a pre-trained neural field. It decouples direct and indirect terms, computing only the first volume interaction and using the network to predict incoming radiance, differing from Emissive Volume Rendering by its reliance on learned fields.

### Woodcock Tracking
Woodcock Tracking is a technique used during integration over the sphere for each step along the ray to obtain unbiased results for heterogeneous media [S2, unofficial].

## Operating characteristics
*   **Complexity:** Path tracing for volumes is expensive because light can take random paths in the media until it reaches the point, analogous to indirect illumination in surface rendering [S2, unofficial].
*   **Performance:** Multiple scattering increases computational cost significantly compared to single scattering methods [S2, unofficial].
*   **Strengths:** Handles multiple scattering and complex light transport in participating media [S1, unofficial].
*   **Limitations:** High computational cost for multiple scattering; requires careful handling of heterogeneous media [S2, unofficial].

## Background
Originating from general path tracing principles applied to participating media, this method addresses the need for physically based light transport in volumes [S1, S2, unofficial]. It builds on the foundation of [Path Tracing and Sampling](2526-path-tracing-and-sampling.md) and is discussed in contexts such as the SIGGRAPH 2015 presentation "Physically-based & Unified Volumetric Rendering in Frostbite" for proper integration [S2, unofficial].

## Sources
[S1] thewildnath/PathTracer, https://github.com/thewildnath/PathTracer, fetched 2026-10-26 (unofficial)
[S2] Volume Rendering (Path Tracing & Real-Time), https://computergraphics.stackexchange.com/questions/4412, fetched 2026-10-30 (unofficial)
[S3398] Emissive Volume Rendering, https://github.com/thewildnath/PathTracer (inferred from sibling link context), fetched 2026-10-26 (unofficial)
[S3427] Neural Photon Field Rendering, https://github.com/thewildnath/PathTracer (inferred from sibling link context), fetched 2026-10-26 (unofficial)
