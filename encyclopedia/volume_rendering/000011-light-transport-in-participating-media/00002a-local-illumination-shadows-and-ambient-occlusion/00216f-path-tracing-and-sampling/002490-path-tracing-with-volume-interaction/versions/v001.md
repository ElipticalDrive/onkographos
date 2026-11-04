# Path Tracing with Volume Interaction

Also called: Volume-Aware Path Tracing, Mixed Surface-Volume Path Tracing

## Overview
Path Tracing with Volume Interaction is a unified path tracing algorithm that handles both surface and participating media intersections within a single loop, using Russian Roulette for path termination and power heuristic for light sampling [S1, unofficial]. It distinguishes itself from [Single Scattering Volume Rendering with Ray Marching](1532-single-scattering-volume-rendering-with-ray-marching.md) by supporting multiple bounces and indirect illumination through recursive path extension, while avoiding the complexity of [Explicit Path Tracing with Russian Roulette](2209-explicit-path-tracing-with-russian-roulette.md) by integrating volume scattering directly into the main traversal logic rather than treating it as a separate pass.

## How it works
**Inputs:** Scene geometry, volume data (density/gradient fields), material definitions, light sources, camera ray, sampler state, and settings (`minDepth`, `maxDepth`, `gradientFactor`, `stepSize`, `backgroundLight`).
**Outputs:** Final pixel color.

**Data Structures:**
*   `Vec3f`: 3D vector (x, y, z), typically float32.
*   `Ray`: `origin` (Vec3f), `direction` (Vec3f, normalized), `minT` (float).
*   `Intersection`: `position` (Vec3f), `normal` (Vec3f), `materialID` (int), `isVolume` (bool), `iorI` (float, index of refraction incoming), `iorO` (float, index of refraction outgoing).
*   `Material`: Properties including BRDF/BTDF evaluation functions, emission data, and specular flags.
*   `LightHit`: `colour` (Vec3f), `direction` (Vec3f), `distance` (float), `pdf` (float).

**Steps:**
1.  Initialize `throughput` to $(1.0, 1.0, 1.0)$ and `colour` to $(0, 0, 0)$. Set `interaction.iorI` to $1.0$ (air).
2.  Loop for `bounces` from $0$ to `settings.maxDepth` - $1$:
    a.  Intersect the current ray with the scene.
    b.  If no intersection is found:
        i.  Add `throughput * settings.backgroundLight` to `colour`.
        ii. Break the loop.
    c.  If the intersection is a **Volume** type:
        i.  Compute the local position relative to the volume origin.
        ii. Get the gradient of the volume at that position to determine the surface normal magnitude.
        iii. Sample the volume intensity (density) at that position.
        iv. Calculate the probability of treating this as a surface interaction using the formula:
        $$probBRDF = 1.0 - \exp(-settings.gradientFactor \cdot \frac{magnitude}{intensity})$$
        v.  If `sampler.nextFloat() < probBRDF`:
            1.  Treat as a surface (BRDF) interaction.
            2.  Shift the position along the normal by `settings.stepSize` to avoid self-intersection.
            3.  Multiply `throughput` by the transfer function's RGBA output (attenuation).
            4.  Retrieve the material from the transfer function.
        vi. Else (treat as isotropic volume scattering):
            1.  Multiply `throughput` by $2.0$ (isotropic scattering assumption).
            2.  Create an isotropic material using the transfer function's color.
    d.  If the intersection is a **Surface** type:
        i.  Retrieve the material from `scene.materials` using `intersection.materialID`.
    e.  If the material has a light source and (`bounces == 0` or the sampled lobe is Specular):
        i.  Add `throughput * hitLight->getEmittance(interaction)` to `colour`.
    f.  Add direct lighting contribution:
        $$colour += throughput \cdot SampleOneLight(interaction, scene, material, hitLight, settings, sampler)$$
    g.  Sample the next direction from the material.
        i.  If the PDF is not a number (e.g., parallel direction), resample until a valid PDF is obtained [G].
    h.  Update `throughput`:
        $$throughput *= \frac{material->evaluate(interaction)}{pdf}$$
    i.  If the sampled lobe is SpecularTransmission:
        i.  Update the index of refraction: `interaction.iorI = interaction.iorO`.
    j.  Create the new ray:
        i.  `origin = interaction.position`
        ii. `direction = interaction.inputDir` (the sampled direction)
        iii. `minT = RAY_EPS`
    k.  Perform Russian Roulette if `bounces >= settings.minDepth - 1`:
        i.  Calculate $p$ as the maximum component of `throughput`.
        ii. If `sampler.nextFloat() > p`, break the loop.
        iii. Otherwise, divide `throughput` by $p$.
3.  Return the final `colour` divided by $(1 + bounces)$ [G, normalization factor].

**Parameters:**
*   `settings.minDepth`: Minimum number of bounces before Russian Roulette can terminate the path.
*   `settings.maxDepth`: Maximum number of bounces allowed for the path.
*   `settings.gradientFactor`: Factor controlling the probability of treating volume gradients as surfaces.
*   `settings.stepSize`: Offset distance applied when treating a volume interaction as a surface to prevent self-intersection.
*   `settings.backgroundLight`: Color of the background environment.

**Edge Cases and Pitfalls:**
*   **Parallel Directions:** When sampling a direction parallel to the surface normal or previous ray, the PDF may become invalid or zero. The algorithm must resample until a valid PDF is obtained [S1, unofficial].
*   **Volume Gradient Magnitude:** If the gradient magnitude is zero (uniform volume), `probBRDF` becomes $0$, forcing isotropic scattering.
*   **Precision:** Floating-point precision errors in ray intersection or position shifting can cause self-intersection artifacts. Using `RAY_EPS` and `settings.stepSize` mitigates this.

## Subtopics

### [Emissive Volume Rendering](3398-emissive-volume-rendering.md)
Extends path tracing to support participating media that emit light, integrating volume emission sampling directly into medium interaction routines.

### [Neural Photon Field Rendering](3427-neural-photon-field-rendering.md)
Uses a pre-trained neural field to approximate indirect global illumination, computing only the first volume interaction and querying the network for incoming radiance.

## Operating characteristics
**Time Complexity:** $O(N \cdot B)$, where $N$ is the number of pixels and $B$ is the average number of bounces. Each bounce involves ray-scene intersection and potential volume sampling.
**Memory Complexity:** $O(V)$ for volume data storage, where $V$ is the volume resolution.
**Measured Performance:** Not given in the sources.
**Scaling:** Performance scales linearly with resolution and average path length. Volume sampling cost depends on the density field evaluation complexity.
**Strengths:** Unified handling of surfaces and volumes; supports multiple bounces and indirect illumination; relatively simple integration into existing path tracers.
**Limitations:** Isotropic scattering assumption for volume interactions may not be accurate for all media; Russian Roulette termination can introduce noise if not tuned properly.
**When to Use:** Scenes with mixed surface and volumetric elements where indirect illumination is important.
**When Not to Use:** Scenes requiring highly anisotropic volume scattering or real-time performance constraints where simpler single-scattering methods are preferred.

## Background
Originated in the `thewildnath/PathTracer` repository [S1, unofficial]. The method is part of a broader effort to integrate participating media into standard path tracing frameworks, building on [Path Tracing and Sampling](2526-path-tracing-and-sampling.md).

## Sources
[S1] thewildnath/PathTracer, https://github.com/thewildnath/PathTracer, fetched 2026-10-26 (unofficial)
