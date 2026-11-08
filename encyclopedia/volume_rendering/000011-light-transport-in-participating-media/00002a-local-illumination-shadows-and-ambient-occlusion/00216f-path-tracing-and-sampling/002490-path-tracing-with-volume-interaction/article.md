# Path Tracing with Volume Interaction

Also called: Path Tracing for Volume Lighting

## Overview
Path Tracing with Volume Interaction is a path tracing algorithm that handles both surface and participating media intersections, using Russian Roulette for termination and power heuristic for light sampling [S1, unofficial]. It evaluates multiple scattering in participating media, where light takes random paths through the medium until reaching a point, analogous to indirect illumination in surface rendering [S2, unofficial]. It builds on [Path Tracing and Sampling](2526-path-tracing-and-sampling.md) by integrating volume intersection logic and isotropic scattering models directly into the path generation loop.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene geometry, volume data, light sources, camera parameters, `settings.maxDepth`, `settings.minDepth`, `settings.gradientFactor`, `settings.stepSize`, and a random number sampler [S1, unofficial].
*   **Outputs:** Final pixel colour (radiance).

**Data Structures:**
*   `ScatterEvent`: Stores intersection position, normal, material ID, index of refraction (`iorI`, `iorO`), and input direction [S1, unofficial].
*   `LightHit`: Stores light colour, direction, distance, and probability density function (PDF) [S1, unofficial].

**Parameters:**
*   `settings.maxDepth`: Maximum number of bounces allowed for the path [S1, unofficial].
*   `settings.minDepth`: Minimum number of bounces before Russian Roulette can terminate the path [S1, unofficial].
*   `settings.gradientFactor`: Factor used to determine the probability of treating a volume interaction as a surface [S1, unofficial].
*   `settings.stepSize`: Small offset distance to shift position along the normal for surface interactions [S1, unofficial].
*   `RAY_EPS`: Minimum ray segment length to prevent self-intersection [S1, unofficial].

**Algorithm Steps:**
1. Initialize `throughput` to $(1.0, 1.0, 1.0)$ and `colour` to $(0, 0, 0)$. Set `interaction.iorI` to $1.0$ (air). Loop from `bounces = 0` to `maxDepth - 1` [S1, unofficial].
2. Intersect the ray with the scene. If no intersection is found, add `throughput * settings.backgroundLight` to `colour` and break [S1, unofficial].
3. If the intersection is a Volume type, compute the local position relative to the volume origin. Get the gradient of the volume at that position to determine the surface normal and magnitude. Sample the volume intensity [S1, unofficial].
4. Calculate the probability of treating this as a surface interaction using the formula:
   $$ p_{\text{BRDF}} = 1.0 - \exp(-\text{gradientFactor} \cdot \frac{\|\mathbf{g}\|}{\rho}) $$
   where $\|\mathbf{g}\|$ is the gradient magnitude and $\rho$ is the sampled intensity [S1, unofficial].
5. If `sampler.nextFloat() < p_BRDF`, treat as surface: shift position along normal by `settings.stepSize`, multiply `throughput` by the transfer function's RGBA output, and get material from transfer function [S1, unofficial].
6. If `sampler.nextFloat() >= p_BRDF`, treat as isotropic: multiply `throughput` by $2.0$ and create an isotropic material with the transfer function's color [S1, unofficial].
7. If the intersection is a Surface type, retrieve the material from `scene.materials` using `intersection.materialID` [S1, unofficial].
8. If the material has a light source and (`bounces == 0` or sampled lobe is Specular), add `throughput * hitLight->getEmittance(interaction)` to `colour` [S1, unofficial].
9. Add direct lighting contribution: `colour += throughput * SampleOneLight(interaction, scene, material, hitLight, settings, sampler)` [S1, unofficial].
10. Sample the next direction from the material. If the PDF is not a valid number (e.g., parallel direction), resample until a valid PDF is obtained [S1, unofficial].
11. Update `throughput *= material->evaluate(interaction) / pdf` [S1, unofficial].
12. If the sampled lobe is SpecularTransmission, update `interaction.iorI = interaction.iorO` [S1, unofficial].
13. Create the new ray: `origin = interaction.position`, `direction = interaction.inputDir`, `minT = RAY_EPS` [S1, unofficial].
14. If `bounces >= minDepth - 1`, calculate $p$ as the maximum component of `throughput`. If `sampler.nextFloat() > p`, break the loop. Otherwise, divide `throughput` by $p$ [S1, unofficial].
15. Return the final `colour` divided by $(1 + \text{bounces})$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def path_trace_volume(scene, camera, settings, sampler):
    throughput = [1.0, 1.0, 1.0]
    colour = [0.0, 0.0, 0.0]
    ior_i = 1.0
    current_ray = camera.get_ray()

    for bounces in range(settings.max_depth):
        hit = scene.intersect(current_ray)

        if hit is None:
            colour = [colour[i] + throughput[i] * scene.background_light[i] for i in range(3)]
            break

        interaction = hit.event
        interaction.ior_i = ior_i

        if hit.type == "Volume":
            local_pos = hit.position - volume.origin
            gradient = volume.gradient(local_pos)
            magnitude = math.sqrt(sum(g * g for g in gradient))
            intensity = volume.sample(local_pos)
            normal = [g / magnitude for g in gradient]

            prob_brdf = 1.0 - math.exp(-settings.gradient_factor * (magnitude / intensity))

            if sampler.next_float() < prob_brdf:
                hit.position = [hit.position[i] + normal[i] * settings.step_size for i in range(3)]
                throughput = [throughput[i] * volume.transfer_function_rgba(local_pos)[i] for i in range(3)]
                material = volume.get_material(local_pos)
            else:
                throughput = [t * 2.0 for t in throughput]
                material = IsotropicMaterial(volume.transfer_function_color(local_pos))

        elif hit.type == "Surface":
            material = scene.materials[hit.material_id]

        if material.has_emission and (bounces == 0 or material.sampled_lobe == "Specular"):
            light_hit = material.get_emittance(interaction)
            colour = [colour[i] + throughput[i] * light_hit.colour[i] for i in range(3)]

        light_hit = sample_one_light(interaction, scene, material, settings, sampler)
        colour = [colour[i] + throughput[i] * light_hit.colour[i] for i in range(3)]

        while True:
            next_dir = material.sample_direction(interaction, sampler)
            pdf = material.pdf(interaction, next_dir)
            if pdf is not None and pdf > 0:
                break

        eval_val = material.evaluate(interaction)
        throughput = [throughput[i] * eval_val[i] / pdf for i in range(3)]

        if material.sampled_lobe == "SpecularTransmission":
            ior_i = interaction.ior_o

        current_ray = Ray(interaction.position, next_dir, min_t=settings.RAY_EPS)

        if bounces >= settings.min_depth - 1:
            p = max(throughput)
            if sampler.next_float() > p:
                break
            throughput = [t / p for t in throughput]

    return [c / (1 + bounces) for c in colour]

def sample_one_light(interaction, scene, material, settings, sampler):
    # Standard direct lighting integration over scene lights
    hit_light = scene.sample_light(interaction.position, sampler)
    return hit_light
```

**Edge Cases and Pitfalls:**
*   **Invalid PDF:** If the sampled direction results in a zero or invalid PDF (e.g., parallel to normal), the algorithm must resample [S1, unofficial].
*   **Self-Intersection:** Use `RAY_EPS` for the minimum ray segment length to prevent the ray from intersecting the surface it just left [S1, unofficial].
*   **Heterogeneous Media:** Woodcock tracking can be used during integration over the sphere for each step along the ray to get unbiased results for heterogeneous media [S2, unofficial].
*   **Precision:** Not given in the sources.
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.

## Subtopics

### [Emissive Volume Rendering](3398-emissive-volume-rendering.md)
Extends path tracing algorithms (`volpath`, `volpathmis`, `prbvolpath`) to support participating media that emit light, enabling simulation of radiative transfer within volumetric emitters. It distinguishes itself by integrating volume emission sampling directly into medium interaction routines, requiring minimal code changes to existing integrators, while avoiding performance penalties of direct solid-angle-to-volume PDF conversion for Multiple Importance Sampling (MIS).

### [Neural Photon Field Rendering](3427-neural-photon-field-rendering.md)
A GPU-based path tracing method that estimates direct illumination via standard ray-marching techniques and approximates indirect global illumination by querying a pre-trained neural field. Unlike traditional path tracers that recursively trace secondary rays for global illumination, this method computes only the first volume interaction and uses the neural network to predict the incoming radiance $L_i$ from the rest of the scene.

### Woodcock Tracking
A technique used during integration over the sphere for each step along the ray to get unbiased results for heterogeneous media [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Path tracing for volumes is expensive because light can take random paths in the media until it reaches the point, analogous to indirect illumination in surface rendering [S2, unofficial].
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
Path tracing for volume interaction was developed as an extension of surface path tracing to handle participating media [S1, unofficial]. The specific implementation details are derived from the `thewildnath/PathTracer` repository [S1, unofficial]. The concept of using path tracing for volumes to handle multiple scattering is discussed in community resources such as Computer Graphics Stack Exchange [S2, unofficial].

## Sources
[S1] thewildnath/PathTracer, https://github.com/thewildnath/PathTracer, fetched 2026-10-26 (unofficial)
[S2] Volume Rendering (Path Tracing & Real-Time), https://computergraphics.stackexchange.com/questions/4412, fetched 2026-10-30 (unofficial)
