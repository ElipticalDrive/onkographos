# Progressive Photon Mapping
Also called: Multipass Photon Mapping
## Overview
Progressive Photon Mapping is a volumetric light transport method developed by Hachisuka et al. that uses a multipass rendering technique to continuously emit photons into the image, gradually reducing inaccuracy [S1]. It builds on [Volumetric Photon Mapping](2190-volumetric-photon-mapping.md) by applying a multipass strategy to photon emission and accumulation [S1]. It differs from sibling methods like [Beam Radiance Estimate](1283-beam-radiance-estimate.md) by refining illumination through iterative photon passes rather than beam-based approximations [G].

## How it works
**Inputs and Outputs:**
- Input: Scene geometry, participating media properties, camera parameters, and a total number of passes $N_{passes}$. Not given in the sources. [S1]
- Output: A rendered image with progressively refined illumination. [S1]

**Data Structures:**
- Photon Map: A dynamic list or array storing photon records with fields for position (3D vector), energy (scalar), and direction (3D vector). In this method, it is updated or rebuilt per pass [S1].
- [G] A standard spatial acceleration structure (e.g., KD-tree or grid) is typically used for neighbor search during density estimation, though specific implementation details are not provided in the source [G].

**Steps:**
1. Initialize the photon map. [S1]
2. For each pass $i$ from 1 to $N_{passes}$:
   a. Emit a batch of photons from light sources into the scene. [S1]
   b. Trace photons through the participating media, storing interactions in the photon map. [S1]
   c. Update the photon map with the new photons. [S1]
   d. Render the image using the current photon map state. [S1]
3. Gradually reduce inaccuracy by increasing the number of photons or refining the search radius in subsequent passes. [S1]

**Formulas:**
- Not given in the sources. [S1]

**Parameters:**
- Number of passes $N_{passes}$: Not given in the sources. [S1]
- Photons per pass: Not given in the sources. [S1]
- Search radius: Not given in the sources. [S1]

**Edge Cases and Pitfalls:**
- Convergence: The method relies on gradual reduction of inaccuracy; insufficient passes may result in noisy images [S1].
- Precision: Not given in the sources. [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_photon_mapping(scene, camera, num_passes):
    photon_map = []
    accumulated_image = zeros_like(camera.resolution)
    for pass_idx in range(1, num_passes + 1):
        # Emit and trace photons
        for light in scene.lights:
            for _ in range(photons_per_pass):
                ray_origin, ray_direction = sample_light_emission(light)
                pos = ray_origin
                while pos inside scene:
                    step = march_step(pos, scene.media)
                    pos += step * ray_direction
                    photon_map.append(Photon(pos, scene.media.extinction * step, ray_direction))
        # Render using current photon map
        for ray in camera.rays:
            radiance = 0.0
            for photon in photon_map:
                if distance(photon.position, ray.origin) < search_radius:
                    radiance += photon.energy / (pi * search_radius**2)
            accumulated_image += radiance
    return accumulated_image / num_passes
```

## Subtopics

### [Progressive Photon Beam Mapping](7795-progressive-photon-beam-mapping.md)
Progressive Photon Beam Mapping is a memory-efficient volumetric light transport method that replaces the large, static photon map with a series of small, iteratively generated photon maps. It addresses the high memory costs associated with storing huge volumetric photon maps in participating media. Unlike standard photon mapping which requires storing all photons, this method discards the photon map after each iteration, significantly improving practicality for large scenes.

## Operating characteristics
- Time Complexity: Not given in the sources. [S1]
- Memory Complexity: Not given in the sources. [S1]
- Measured Performance: Not given in the sources. [S1]
- Strengths: Gradual convergence through multipass photon emission [S1].
- Limitations: Requires multiple passes for high quality [S1].
- When to use: Real-time realistic volume rendering and interactive character updates [S1].
- When not to use: Not given in the sources. [S1]

## Background
Progressive Photon Mapping was developed by Hachisuka et al. [S1]. It was proposed as a multipass rendering technique to continuously emit photons into the image to gradually reduce inaccuracy [S1]. The method builds on earlier photon mapping techniques and adapts them for volumetric rendering [S1].

## Sources
[S1] Real-time volume rendering for three-dimensional fetal ultrasound using volumetric photon mapping., https://doi.org/10.1186/s42492-024-00177-4, fetched 2026-10-26
