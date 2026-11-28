# Reference Path Tracing

Also called: Path Traced Subsurface Scattering, Random Walk SSS, ReferencePathTracer, Volume SSS, Volumetric Path Tracing for Subsurface Scattering, path tracing based subsurface scattering

## Overview
Reference Path Tracing is an unbiased volumetric path tracer used as a ground-truth reference renderer for subsurface scattering [S1, unofficial]. It solves the problem of accurately modeling light transport in participating media by tracing full random walks through the volume, handling zero-scatter, single-scatter, and multiple-scattering events within a unified framework [S2, unofficial]. Unlike dipole approximations which require separate handling for zero- and single-scatter events, this method treats the medium as a general volume, allowing for explicit handling of high-curvature details and accurate shadowing [S3, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Surface intersection point, surface normal, material properties (extinction coefficient $\sigma_t$, scattering albedo $\omega$, phase function parameters), and scene geometry.
- **Output:** Radiance contribution of the subsurface scattering path.

**Data Structures:**
The method is encapsulated in a struct `ReferencePathTracer` [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```julia
struct ReferencePathTracer <: VolumeIntegrationMethod
    max_bounces::Int
    rr_start::Int
end
```
- `max_bounces`: Maximum number of scattering events per path. Default: 64 [S1, unofficial].
- `rr_start`: Number of bounces after which Russian roulette activation begins. Default: 3 [S1, unofficial].

**Algorithm Steps:**

1. **Initialization:** When a primary ray hits a surface, calculate the BTDF (or diffuse BRDF pointing inward) [S2, unofficial]. For a diffuse BRDF, use cosine importance sampling to determine the direction of the internal ray, choosing the initial direction from a Lambertian distribution around the inverse surface normal [S3, unofficial].

2. **Distance Sampling:** Advance along the ray by a distance $s$ sampled from the transmittance distribution. For a homogeneous medium with extinction coefficient $\sigma_t$, the transmittance is governed by Beer's Law. To sample the distance, draw a uniform random number $\xi \in [0, 1)$ and invert the transmittance function:
   $$ s = -\frac{\ln(1 - \xi)}{\sigma_t} $$
   [S3, unofficial].

3. **Random Walk:**
   - **Step Advancement:** Move the current position by $s$.
   - **Direction Update:** In the isotropic scattering case, the direction of the next step is unrelated to the previous step. Draw a new direction from a spherical distribution [S3, unofficial].
   - **Wavelength Sampling:** Perform random walk inside the volume, sampling wavelengths using Multiple Importance Sampling (MIS) based on throughput and albedo to reduce noise [S2, unofficial]. The weights for MIS are proportional to $|\text{throughput} \times \text{albedo}|$ [S2, unofficial].

4. **Boundary Check:** Trace a ray against the geometry for the current step. If the step crosses the exterior interface, terminate the random walk [S3, unofficial].

5. **Exit and Illumination:**
   - If the exit point is found, compute direct illumination at the exit point.
   - Weigh the incoming radiance by a Lambertian response, without applying surface coloring effects (such as Fresnel or BRDF shading) at the exit point [S3, unofficial].

6. **Termination:**
   - If the path continues to a fixed depth (default 256 steps) without finding an exit point, deem the photon lost and set the final contribution to zero [S3, unofficial].
   - Use Russian Roulette to terminate paths with low weight stochastically in practice, starting after `rr_start` bounces [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reference_path_trace(entry_point, normal, material, scene, rng):
    """
    entry_point: Point3D
    normal: Vector3D (surface normal)
    material: Object containing sigma_t, albedo, phase_function
    scene: Object with intersect() method
    rng: Random number generator
    """
    max_bounces = 64
    rr_start = 3
    
    # 1. Initialize direction (Lambertian around inverse normal)
    # Note: Inverse normal points inward for a surface hit from outside
    inward_normal = -normal
    direction = sample_cosine_hemisphere(inward_normal, rng)
    
    current_position = entry_point
    throughput = Vector3D(1.0, 1.0, 1.0)
    radiance = Vector3D(0.0, 0.0, 0.0)
    
    for bounce in range(max_bounces):
        # 2. Sample distance using Beer's Law inversion
        xi = rng.uniform(0.0, 1.0)
        sigma_t = material.sigma_t
        # Handle potential division by zero or invalid sigma_t
        if sigma_t <= 0:
            break
        distance = -log(1.0 - xi) / sigma_t
        
        # 3. Advance position
        next_position = current_position + direction * distance
        
        # 4. Check for boundary intersection
        hit = scene.ray_intersect(current_position, direction, max_distance=distance)
        
        if hit is not None:
            # Hit interior geometry before exiting, continue walk
            # Update position to hit point
            current_position = hit.point
            # Update direction based on phase function (isotropic example)
            direction = sample_isotropic_phase_function(direction, rng)
            # Update throughput with scattering albedo
            throughput *= material.albedo
        else:
            # Ray exited the volume
            exit_point = next_position
            
            # 5. Compute direct illumination at exit point
            # Weigh incoming radiance by Lambertian response
            # Note: No surface coloring effects applied at exit
            incoming_radiance = scene.direct_lighting(exit_point, rng)
            # Lambertian weighting factor is 1/pi for diffuse, but source says "weigh by lambertian response"
            # Assuming standard diffuse exit weighting
            exit_weight = 1.0 / pi 
            
            radiance += throughput * incoming_radiance * exit_weight
            break
            
        # 6. Russian Roulette
        if bounce >= rr_start:
            p_survive = min(max(throughput), 0.95) # Heuristic for survival probability
            if rng.uniform(0.0, 1.0) > p_survive:
                break
            throughput /= p_survive
            
    return radiance
```

**Parameters:**
- `max_bounces`: Default 64 [S1, unofficial].
- `rr_start`: Default 3 [S1, unofficial].
- Fixed depth limit: 256 steps [S3, unofficial].

**Pitfalls:**
- Shapes must be watertight with thickness and no holes for correct calculation [S2, unofficial].
- Subsurface objects must not be separate if they share material properties; otherwise, correct calculation may fail [S2, unofficial].
- Handling parallel light sources through glossy BTDFs or Fresnel reflections can be difficult without advanced techniques like Subdivision Next-Event Estimation (SNEE) or manifold next-event estimation [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of bounces and scattering events. The cost is dominated by ray-scene intersections for each step of the random walk.
- **Memory Complexity:** Low, as it only requires storing the current path state and does not require storing large volume grids or precomputed light fields.
- **Strengths:** Unbiased ground-truth reference; handles zero, single, and multiple scattering uniformly; accurate shadowing and high-curvature detail preservation; no need for dipole approximations [S2, unofficial][S3, unofficial].
- **Limitations:** Computationally expensive due to the need for many samples to converge; sensitive to geometry watertightness; complex to implement efficiently for glossy/transmissive boundaries [S2, unofficial].
- **When to use:** When a ground-truth reference is needed or when high-curvature details and accurate shadowing are critical.
- **When not to use:** For real-time applications or when performance is prioritized over accuracy, where dipole approximations or other heuristic methods are preferred.

## Background
Reference Path Tracing was developed as a method to provide an unbiased reference for subsurface scattering, treating the medium as a general volume rather than relying on diffusion approximations [S3, unofficial]. It builds on the principles of volumetric path tracing and random walks [S1, unofficial]. The approach was highlighted in discussions and implementations related to production path tracing, such as those in Blender Cycles and research by Pixar [S2, unofficial][S3, unofficial].

## Sources
[S1] Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] 2018 年, ナウでヤングな path tracing based subsurface scattering を実装する, https://qiita.com/syoyo/items/a5469f9e399f80fc62df, fetched 2026-10-27 (unofficial)
[S3] Path Traced Subsurface Scattering Model, https://www.scribd.com/document/965284472/Pixar-Path-Traced-SSS-Paper, fetched 2026-10-29 (unofficial)
