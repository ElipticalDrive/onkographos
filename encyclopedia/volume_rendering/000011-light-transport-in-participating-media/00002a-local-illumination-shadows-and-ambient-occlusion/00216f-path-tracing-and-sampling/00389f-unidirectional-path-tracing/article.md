# Unidirectional Path Tracing

Also called: PathTracing, Standard Path Tracing Lighting

## Overview
Unidirectional Path Tracing is a Monte Carlo method for evaluating visibility and illumination at points within or on the boundary of a volume, tracing rays from the camera into the scene [S1, S2, unofficial]. It handles surface interactions (BSDF sampling) and volumetric scattering within a single loop, terminated by Russian Roulette [S2, unofficial]. Unlike explicit path tracing with Russian Roulette which separates the termination logic into a distinct post-loop phase, this method integrates survival probability checks directly into the main traversal loop to handle path termination dynamically [S2, unofficial].

## How it works
**Inputs:**
- `maxDepth`: Maximum path length (default 100) [S2, unofficial].
- `rayBounces`: Parameter controlling bounce limits in scene data [S1, unofficial].
- Sun direction (`sunDirX`, `sunDirY`, `sunDirZ` as vec3f) and intensity (`sunIntensity` as f32), enabled via `sunEnabled` (u32) [S1, unofficial].
- Random number generator (Sampler) [S2, unofficial].
- Standard scene inputs: camera parameters, geometry, materials, volumes, and lights [G].

**Outputs:**
- Radiance $L$ for each pixel [S2, unofficial].

**Data Structures:**
- **Ray:** Struct containing `origin` (Vec3f), `direction` (Vec3f), `throughput` (Vec3f), and `mediums` (stack of Medium pointers via `pushMedium`/`popMedium`) [S2, unofficial].
- **Materials:** Defined in a constant array `MATERIALS` of size 5. Each material has `specular` (f32) and `emissive` (bool) fields. Index 0: specular 0.0, not emissive. Index 1: specular 0.0, not emissive. Index 2: specular 0.0, emissive. Index 3: specular 0.9, not emissive. Index 4: specular 1.0, not emissive [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:** Set radiance $L = 0$, ray throughput $T = (1, 1, 1)$, and path depth $d = 0$ [S2, unofficial].
2. **Traversal Loop:** While $d < \text{maxDepth}$:
   a. **Russian Roulette:** If $d > 0$, calculate survival probability $p = \min\left(\frac{T_r + T_g + T_b}{3}, 1.0\right)$ [S2, unofficial]. If `sampler.getNext1D()` $\ge p$, break the loop. Otherwise, update $T \leftarrow T / p$ [S2, unofficial].
   b. **Intersection:** Cast ray against scene geometry. If no hit, break loop.
   c. **Area Light Check:** If the hit primitive is an area light, add $T \cdot L_e$ to $L$ and break loop [S2, unofficial].
   d. **Surface Interaction:** If the hit primitive has a surface:
      i. **Shading Normal Check:** Let $\mathbf{n}_g$ be the geometric normal and $\mathbf{n}_s$ be the shading normal. Let $\omega_i$ be the incoming direction ($-\text{ray.direction}$) and $\omega_o$ be the outgoing direction [S2, unofficial]. If $\omega_i \cdot \mathbf{n}_g \cdot \omega_i \cdot \mathbf{n}_s \le 0$ or $\omega_o \cdot \mathbf{n}_g \cdot \omega_o \cdot \mathbf{n}_s \le 0$, return 0 to prevent light leaks [S2, unofficial].
      ii. **BSDF Sampling:** Sample the BSDF to get outgoing direction $\omega_o$ and PDF $p(\omega_o)$ [S2, unofficial].
      iii. **Throughput Update:** Update throughput:
      $$ T_{new} = T_{old} \cdot \frac{f(\omega_i, \omega_o) \cdot |\cos \theta|}{p(\omega_o)} $$
      where $\theta$ is the angle between $\mathbf{n}_s$ and $\omega_o$ [S2, unofficial].
      iv. **Medium Update:** Update the medium stack based on surface crossing using `updateMedium(ray, dir, info)` [S2, unofficial].
      v.  **Ray Update:** Set ray origin to surface position and direction to $\omega_o$ [S2, unofficial]. Set flag `is_reflected_or_refracted = true` [S2, unofficial].
   e.  **Depth Increment:** If `is_scattered` or `is_reflected_or_refracted` is true, increment $d$ [S2, unofficial].
   f.  **Reset Flags:** Reset `is_scattered` and `is_reflected_or_refracted` to false for the next iteration [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def unidirectional_path_tracing(camera, scene, sampler, max_depth=100):
    radiance = [0.0, 0.0, 0.0]
    throughput = [1.0, 1.0, 1.0]
    depth = 0
    ray_origin = camera.origin
    ray_direction = camera.direction
    medium_stack = []
    
    while depth < max_depth:
        # Russian Roulette
        if depth > 0:
            survival_prob = min((throughput[0] + throughput[1] + throughput[2]) / 3.0, 1.0)
            if sampler.get_next_1d() >= survival_prob:
                break
            throughput = [t / survival_prob for t in throughput]
        
        # Intersection
        hit = scene.intersect(ray_origin, ray_direction)
        if not hit:
            break
        
        # Area Light
        if hit.is_area_light:
            radiance = [radiance[i] + throughput[i] * hit.emitted_radiance[i] for i in range(3)]
            break
        
        # Surface Interaction
        if hit.has_surface:
            # Shading Normal Check
            wi = [-ray_direction[i] for i in range(3)]
            wo = hit.sample_bsdf(wi, sampler)
            pdf = hit.bsdf_pdf(wi, wo)
            
            ng = hit.geometric_normal
            ns = hit.shading_normal
            
            if (dot(wi, ng) * dot(wi, ns) <= 0) or (dot(wo, ng) * dot(wo, ns) <= 0):
                break # Light leak prevention
            
            f = hit.bsdf(wi, wo)
            cos_theta = abs(dot(wo, ns))
            throughput = [throughput[i] * (f[i] * cos_theta) / pdf for i in range(3)]
            
            # Medium Update based on surface crossing
            # Implementation details not provided in sources.
            
            # Ray Update
            ray_origin = hit.position
            ray_direction = wo
            ray_tmax = scene.world_size
            
            is_reflected_or_refracted = True
        else:
            is_reflected_or_refracted = False
            
        if is_reflected_or_refracted or is_scattered:
            depth += 1
            
        is_scattered = False
        is_reflected_or_refracted = False
        
    return radiance

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]
```

**Parameters:**
- `maxDepth`: Maximum path length. Default value is 100 [S2, unofficial].
- `rayBounces`: Controls the number of ray bounces in `SceneData` [S1, unofficial].
- Sun direction: Defined by `sunDirX`, `sunDirY`, `sunDirZ` (vec3f) [S1, unofficial].
- Sun intensity: Defined by `sunIntensity` (f32) [S1, unofficial].
- Sun enabled: Defined by `sunEnabled` (u32) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Light Leaks:** Prevented by checking the asymmetry of the BSDF using shading normals ($\mathbf{n}_s$) and geometric normals ($\mathbf{n}_g$). If $\omega_i \cdot \mathbf{n}_g \cdot \omega_i \cdot \mathbf{n}_s \le 0$ or $\omega_o \cdot \mathbf{n}_g \cdot \omega_o \cdot \mathbf{n}_s \le 0$, the contribution is zeroed out [S2, unofficial].
- **Precision:** Throughput division by survival probability can lead to variance explosion if probability is very low. Clamping probability to a minimum value is recommended [G].
- **Boundary Conditions:** When crossing surface boundaries, the medium stack must be updated correctly to ensure the correct extinction coefficient is used for subsequent volume interactions [S2, unofficial].

## Subtopics

### [Volumetric Path Tracing](1863-volumetric-path-tracing.md)
Volumetric Path Tracing is a Monte Carlo method for simulating light transport in participating media by extending classical path tracing to allow rays to enter volumetric regions, sample scattering events, and continue random walks. It solves the problem of computing radiance in scenes with fog, smoke, or clouds by modeling absorption, emission, and scattering within the medium. Unlike its sibling [Volumetric Radiance Estimators](818-volumetric-radiance-estimators.md), which may use deterministic or semi-analytic approaches, this method relies on stochastic sampling. It differs from [Multiple Scattering Volume Path Tracing with Phase Function Sampling](1043-multiple-scattering-volume-path-tracing-with-phase-function-.md) by lacking explicit light sampling (Next-Event Estimation) and surface scattering support, relying instead on delta tracking for distance selection and phase function sampling for direction. It is often used as a ground-truth baseline for debugging more sophisticated algorithms due to its simplicity.

### [Unidirectional Volume Boundary Estimator](3538-unidirectional-volume-boundary-estimator.md)
The Unidirectional Volume Boundary Estimator is a Monte Carlo method for estimating radiance from volume boundary paths in participating media. It solves the problem of efficiently sampling paths where a scattering event occurs exactly on a volume boundary (interface) by building a main path from the camera and attaching side paths to the boundary. It sets itself apart from its sibling, the [Bidirectional Volume Boundary Estimator](4773-bidirectional-volume-boundary-estimator.md), by constructing the path unidirectionally: it samples the main path first and then the boundary vertex, whereas the bidirectional variant samples the boundary vertex first and connects it to both light and camera.

### [Volumetric-Primitives Path Tracer](4107-volumetric-primitives-path-tracer.md)
A forward rendering method for scattering and emissive media that models the scene as a collection of 3D kernel-based volumetric primitives and uses ray tracing to compute light transport, supporting multiple scattering via Next-Event Estimation. Unlike [Volumetric Path Tracing](1863-volumetric-path-tracing.md) which typically relies on voxel grids or implicit fields requiring numerical integration, VPPT provides closed-form analytical solutions for transmittance and free-flight sampling, eliminating Monte Carlo noise for single-scattering events and enabling efficient hardware-accelerated traversal. It maintains 3D volumetric density and integrates physically, preserving reciprocity and avoiding the view-dependent ordering artifacts and popping seen in delta-function approximations like [3D Gaussian Splatting](https://nvlabs.github.io/3dgs/).

### [Unidirectional Differentiable Path Tracing](7780-unidirectional-differentiable-path-tracing.md)
An algorithm to estimate the interior integral of the differential path integral using unidirectional path tracing, computing both radiometric measurement $L$ and its scene derivative $\dot{L}$.

## Operating characteristics
- **Time Complexity:** $O(N \cdot D)$, where $N$ is the number of pixels and $D$ is the average path depth. Each bounce involves a scene intersection and a BSDF/volume sampling [G].
- **Memory Complexity:** $O(D)$ for the path stack and medium stack. No need to store multiple paths as in bidirectional methods [G].
- **Measured Performance:** On an RTX 3090, a related voxel raymarching implementation achieves around 120fps at 4K, dipping to 60fps or lower depending on scene complexity [S1, unofficial].
- **Strengths:** Handles surface reflections and volumetric scattering within a single loop, terminated by Russian Roulette [S2, unofficial].
- **Limitations:** High variance in scenes with complex lighting or deep shadows due to the unidirectional nature. Requires many samples for convergence [S2, unofficial].
- **When to Use:** Standard offline rendering and real-time applications with hardware acceleration [G].
- **When Not to Use:** Scenes requiring efficient bidirectional light transport or complex caustics where path space exploration would be more efficient [G].

## Background
Not given in the sources.

## Sources
[S1] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
[S2] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S3] Fast Voxelization and Level of Detail for Microgeometry Rendering, https://doi.org/10.48550/arxiv.2604.13191, fetched 2026-10-26
