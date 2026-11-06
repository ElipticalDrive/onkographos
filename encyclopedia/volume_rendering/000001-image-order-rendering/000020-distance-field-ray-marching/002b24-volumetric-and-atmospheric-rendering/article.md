# Volumetric and Atmospheric Rendering

Also called: Volumetric Rendering, Atmospheric Rendering

## Overview
Volumetric and Atmospheric Rendering is a family of techniques that render participating media (clouds, fog, smoke) and planetary atmospheres by integrating light transport along rays. It differs from [SDF Cloud Rendering](2064-sdf-cloud-rendering.md) by solving the radiative transfer equation (or approximations like single scattering) rather than constructing explicit geometry via Signed Distance Fields, and from [Ray Marching with Single Scattering](2057-ray-marching-with-single-scattering.md) by focusing on global atmospheric phenomena rather than localized procedural clouds.

## Shared principles
The method renders geometry by shooting a ray through each pixel and repeatedly querying a density field or analytic function [S1, unofficial][S10][S14, unofficial]. Unlike surface rendering, the ray accumulates color and opacity as it traverses the medium.

The core loop initializes a ray with an origin point $ro$ and a normalized direction vector $v$ [S1, unofficial][S10][S12, unofficial]. It maintains a current position $p$ (initially $ro$) [S10]. In each iteration:
1. The density $\sigma_t$ (extinction coefficient) is evaluated at $p$ [S1, unofficial].
2. The transmittance is updated using the Beer-Lambert law: $T \leftarrow T \cdot \exp(-\sigma_t \cdot \Delta s)$, where $\Delta s$ is the step size [S1, unofficial].
3. The scattered light contribution is calculated based on the phase function and light source visibility [S1, unofficial].
4. The accumulated color is updated: $C \leftarrow C + T \cdot \text{scattered\_light} \cdot \Delta s$ [S1, unofficial].
5. The position is advanced: $p \leftarrow p + \Delta s \cdot v$ [S10].
6. The loop continues while the step count is less than $\text{MAX\_STEPS}$ [S1, unofficial][S7, unofficial].

If the loop terminates without significant opacity (e.g., $T$ remains high), the ray contributes the background sky color [S1, unofficial].

Parameters include $\text{MAX\_STEPS}$ (a safety cap, typically 32 to 128 depending on implementation [S1, unofficial][S4, unofficial]) and $\text{HIT\_THRESHOLD}$ (epsilon) [S1, unofficial][S12, unofficial]. A maximum range or accumulated travel distance limit can also bound rays that miss [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_volume(ro, v, density_func, max_steps=40, step_size=0.1, max_range=100.0):
    p = ro
    transmittance = 1.0
    accumulated_color = vec3(0.0)
    
    for i in range(max_steps):
        dist = dot(p - ro, v)
        if dist > max_range:
            break
            
        sigma_t = density_func(p)
        
        # Beer-Lambert Law
        transmittance *= exp(-sigma_t * step_size)
        
        # Early exit if fully opaque
        if transmittance < 0.01:
            break
            
        # Scattering calculation placeholder
        # scattered_light = calculate_scattering(p, v, sigma_t)
        # accumulated_color += transmittance * scattered_light * step_size
        
        p += v * step_size
        
    return accumulated_color + transmittance * sky_color(p)
```

## Subtopics

### [Ray Marching with Single Scattering](2057-ray-marching-with-single-scattering.md)
Ray marching with single scattering is a real-time technique for rendering volumetric clouds using OpenGL/WebGL ray marching, procedural noise for density, and physically-based single scattering illumination. It solves the problem of rendering translucent, animated volumetric media efficiently by approximating light transport with a single scattering event per sample, balancing visual fidelity against computational cost. It differs from [SDF Cloud Rendering](2064-sdf-cloud-rendering.md) by using procedural noise instead of signed distance fields for density, and from [Atmospheric Ray Marching](4874-atmospheric-ray-marching.md) by focusing on localized cloud volumes with explicit bounding boxes and wind animation rather than global atmospheric scattering.

### [SDF Cloud Rendering](2064-sdf-cloud-rendering.md)
SDF Cloud Rendering is a technique for rendering volumetric clouds using Signed Distance Fields (SDFs) via ray marching in a fragment shader, utilizing warped sphere primitives and smooth unions to create complex shapes. It differs from [Ray Marching with Single Scattering](2057-ray-marching-with-single-scattering.md) and [Atmospheric Ray Marching](4874-atmospheric-ray-marching.md) by constructing the volume geometry explicitly through analytic distance functions rather than sampling a density field or solving the radiative transfer equation with phase functions.

### [Atmospheric Ray Marching](4874-atmospheric-ray-marching.md)
A ray marching technique for rendering planetary atmospheres using single scattering, exponential density falloff, and Rayleigh phase function.

### Wax Volume Raymarching
#### step
- [S1, unofficial] Raymarch the interior volume of the glass container.
- [S1, unofficial] Compute surface normal using the gradient of the density field.
- [S1, unofficial] Set fragment depth explicitly using gl_FragDepth in the opaque pass.
- [S1, unofficial] Shading model includes: wrap lighting, soft Fresnel, fake subsurface scattering based on traversed thickness, and emission driven by temperature and proximity to the light bulb.
#### number
- [S1, unofficial] Artifacts: 0.00% of surface voxels cause a round-trip to the threshold, even at 60 Hz with ±30% jitter.

## Background
The technique originated in real-time graphics for rendering clouds and fog. The specific implementation details for wax volume raymarching, including the use of a 4³ occupancy grid to skip empty blocks and reduce texture reads by 3x, were documented in a GitHub pull request for a lava lamp simulation [S1, unofficial]. The method employs a separable kernel for density generation and temporal smoothing [S1, unofficial].

## Sources
[S1] Étape 4 : surface de cire raymarchée et liquide volumique, https://github.com/nathanddrl/lava-lamp/pull/4, fetched 2026-10-27 (unofficial)
