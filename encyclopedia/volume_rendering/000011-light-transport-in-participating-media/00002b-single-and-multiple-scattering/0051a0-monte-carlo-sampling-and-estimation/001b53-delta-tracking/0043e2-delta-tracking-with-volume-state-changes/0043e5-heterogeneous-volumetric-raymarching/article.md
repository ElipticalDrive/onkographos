# Heterogeneous Volumetric Raymarching

Also called: Delta/Ratio Tracking for Participating Media, Volume Raymarching

## Overview
Heterogeneous Volumetric Raymarching is a rendering technique for participating media (clouds, fog, godrays) that uses delta tracking and ratio tracking on density fields instead of sphere tracing on distance, allowing for heterogeneous (non-uniform) density distributions [S1]. It distinguishes itself from homogeneous volume methods by explicitly sampling a density field at each step to determine scattering probabilities, and from previous roadmap phases that relied on distance-based sphere tracing, by sharing the step-loop infrastructure while employing different sampling mathematics [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray origin and direction; a medium defined by an Axis-Aligned Bounding Box (AABB) and a reference to a density field; light sources (sun, environment map); rendering parameters (step limits, density scale, phase function anisotropy) [S1].
*   **Output:** Radiance contribution at the ray origin, including scattered light from the medium and inscatter from lights [S1].

**Data Structures:**
*   **SHAPE_VOLUME:** A new shape variant registered in the Bounding Volume Hierarchy (BVH) leaf nodes, defining a bounded medium with an AABB and a reference to a density field [S1].
*   **Density Field:** Accessed via a unified sampler interface that supports both 3D textures and procedural generation (FBM noise + Worley noise) [S1].

**Algorithm Steps:**
1. Register the medium in the BVH as a `SHAPE_VOLUME` leaf, defining a bounded medium with an AABB and a reference to a density field [S1].
2. Sample the density field at the current ray position using the unified sampler interface [S1].
3. Perform delta tracking along the ray to determine the distance to the next scattering event based on the local density [S1].
4. Apply ratio tracking to handle heterogeneous density, adjusting the probability of scattering based on the ratio of the current density to the maximum density in the medium [S1].
5. At a scattering event, compute the scattered radiance using the Henyey-Greenstein phase function, with an optional dual-lobe configuration for forward and back-scatter [S1].
6. For multi-scattering, use Russian roulette to terminate the path or continue with repeated delta tracking, up to the limit set by `r_vol_multiscatter_bounces` [S1].
7. Compute inscatter from the sun and environment map using Next Event Estimation (NEE): sample a light direction, and ratio-track transmittance back along that ray to the scattering point [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_volume_ray(ray_origin, ray_dir, medium, lights, params):
    # [G] Standard AABB intersection
    t_min, t_max = intersect_aabb(ray_origin, ray_dir, medium.aabb)
    if t_min > t_max:
        return 0.0

    t = t_min
    max_steps = params.r_vol_max_steps
    max_bounces = params.r_vol_multiscatter_bounces
    density_scale = params.r_vol_density_scale
    phase_g = params.r_vol_phase_g
    # [G] Query maximum density for ratio tracking bound
    density_max = medium.density_field_ref.get_max_density()
    radiance = 0.0
    throughput = 1.0
    bounces = 0

    for step in range(max_steps):
        if bounces >= max_bounces:
            break

        # Step 2: Sample density field at current position
        pos = ray_origin + ray_dir * t
        density = medium.density_field_ref.sample(pos) * density_scale

        # Step 3: Delta tracking to find distance to next event
        # [G] Standard delta tracking: dt = -ln(u) / sigma_t_max
        u = random()
        dt = -math.log(u) / (density_max * density_scale)

        # Step 4: Ratio tracking to adjust scattering probability
        if density > 0 and random() <= density / density_max:
            # Step 5: Scattering event
            # [G] Standard Henyey-Greenstein phase function sampling and evaluation
            # omega_s = sample_henyey_greenstein(phase_g)
            # phase_val = henyey_greenstein_eval(omega_s, omega_l, phase_g)

            # Step 7: NEE inscatter from lights
            for light in lights:
                omega_l = light.direction
                # [G] Standard ray-light intersection or directional sample
                t_light = ray_length(pos, omega_l, light)
                # [G] Standard ratio-track transmittance calculation along the path to the light
                transmittance = 1.0
                radiance += throughput * transmittance * light.radiance * phase_val

            # Step 6: Russian roulette for multi-scattering
            # [G] Standard survival probability
            q = max(0.05, 1.0 / (bounces + 1))
            if random() > q:
                break
            throughput /= q
            bounces += 1
        else:
            # No scattering, advance along ray
            t += dt

        # Terminate if ray exits medium bounds
        if t > t_max:
            break

    return radiance
```

**Parameters:**
*   `r_vol_max_steps`: Maximum number of steps per raymarch. Default: 64 [S1].
*   `r_vol_density_scale`: Global multiplier for medium density. Default: 1.0 [S1].
*   `r_vol_phase_g`: Anisotropy parameter for the Henyey-Greenstein phase function. Default: 0.8 (canonical for clouds) [S1].
*   `r_vol_multiscatter_bounces`: Number of multi-scattering bounces via Russian roulette. Default: 2 [S1].

**Pitfalls:**
*   **Undersampling:** A ray grazing a thin cloud shell at the limb has a long chord length [S1]. If the step size ($dt$) is too large relative to the feature size of the density field, it produces aliased noise [S1]. Fix: Use cone-footprint stepping ($dt = \text{clamp}(t \cdot \text{pixel\_cone\_angle}, dt\_min, dt\_max)$) and density LOD (drop FBM octaves proportionally to $\log_2(dt / \text{feature\_size})$) [S1].
*   **Performance Overhead:** A new marching loop in the hot path adds compile-time and runtime overhead to all scenes [S1]. Fix: Gate the feature behind a `PT_*_ENABLED` macro to exclude it from scenes that do not use volumes [S1].
*   **RHI Limitations:** The engine's RHI lacks 3D texture support (no depth field in `TextureDesc`, no samplers, bilinear interpolation is hand-written) [S1]. Using 3D textures requires ~400 LOC of new RHI code and unproven `RWTexture3D` lowering on Slang targets [S1]. Procedural density has no such cost [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of scattering events and steps per ray, influenced by density field evaluation cost and NEE samples [S1].
*   **Memory Complexity:** Depends on density field representation; 3D textures require memory proportional to resolution, while procedural generation requires minimal memory [S1].
*   **Measured Performance:** Target performance is $\le 6$ ms at 1080p for a single cloud layer on RTX-class hardware at 1 sample per pixel (spp) [S1].
*   **Strengths:** Handles heterogeneous density distributions effectively; integrates with existing BVH and raymarching infrastructure; supports both procedural and texture-based density fields [S1].
*   **Limitations:** Performance sensitive to step size and density field resolution; potential aliasing if not properly stepped; RHI limitations may hinder 3D texture usage; additional overhead in scenes without volumes if not gated [S1].
*   **When to Use:** Rendering clouds, fog, godrays, and other participating media with non-uniform density [S1].
*   **When Not to Use:** Simple homogeneous volumes (use simpler methods); scenes where performance is critical and volumes are not present (use gating) [S1].

## Background
Heterogeneous Volumetric Raymarching was introduced as Phase 4 of the "Roadmap: SDF + raymarching: surface and volumetric integration" (#96) in the demont-engine project [S1]. It was proposed by havokentity on May 16, 2026 [S1]. The method is designed to render cumulus cloud layers, godrays, and non-uniform fog, and is intended to pair with spherical-Earth atmospheric models (#51) for curved aerial perspective [S1]. It is independent of Phases 1-3 of the roadmap, which used sphere tracing on distance, but shares the step-loop pattern and Slang module structure [S1].

## Sources
[S1] renderer: heterogeneous volumetric raymarching -- clouds, godrays, non-uniform fog (Phase 4 of #96), https://github.com/havokentity/demont-engine/issues/100, fetched 2026-10-26
