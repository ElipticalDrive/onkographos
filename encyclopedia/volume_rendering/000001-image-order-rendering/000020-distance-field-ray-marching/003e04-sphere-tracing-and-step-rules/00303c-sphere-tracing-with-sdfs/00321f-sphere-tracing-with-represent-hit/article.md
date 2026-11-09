# Sphere Tracing with Represent Hit

Also called: RayMarch, Sphere Tracing with Surface/Represent Passes, SphereTracingFPass, SphereTracingKPass, Surface/Represent Ray Marching

## Overview
Sphere Tracing with Represent Hit is a deferred rendering technique that extends standard sphere tracing by tracking two distinct hit points per pixel: a primary **surface hit** (the exact geometric intersection) and a **represent hit** (a proxy for the surface at the pixel's radius) [S1, unofficial]. This dual-pass approach is designed to handle transparency and aliasing artifacts at object edges by evaluating visibility and occlusion from both points, ultimately selecting the ray with greater visual impact (higher alpha) to write to two separate render targets [S1, unofficial]. It differs from standard sphere tracing by explicitly maintaining a "representative" distance to preserve edge information that would otherwise be lost during the discrete stepping of the SDF [S1, unofficial].

## How it works
The method operates as a deferred rendering pass, writing results to two render targets: index 0 for the surface hit and index 1 for the represent hit [S1, unofficial]. It uses a compute shader environment with global properties such as `RadiusPixel`, `SphereTracingSteps`, and `ClippingPlanes` [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized) [G].
*   SDF Map: A texture or buffer storing `float2` values at each query point: distance $d$ and material ID $m$ [S1, unofficial].
*   Global Parameters:
    *   `ClippingPlanes`: Vector $(t_{near}, t_{far})$ [S1, unofficial].
    *   `SphereTracingSteps`: Maximum iteration count $N$ [S1, unofficial].
    *   `RadiusPixel`: The radius of the pixel in world units, used for anti-aliasing logic [S1, unofficial].
    *   `DisableAntiAliasing`: Boolean flag [S1, unofficial].

**Outputs:**
*   `surfaceHit`: Structure containing `DistanceToWorld`, `TraceDistance`, `MaterialId`, and `Position` [S1, unofficial].
*   `representHit`: Structure containing `DistanceToWorld`, `Alpha`, `MaterialId`, and `Position` [S1, unofficial].

### Data Structures
The hit structures are defined as follows [S1, unofficial]:
*   `surfaceHit`:
    *   `DistanceToWorld`: float, distance from camera origin to hit point.
    *   `TraceDistance`: float, distance along the ray for shading calculations.
    *   `MaterialId`: int, identifier of the material at the hit point.
    *   `Position`: float3, world position of the hit.
*   `representHit`:
    *   `DistanceToWorld`: float, distance to the represent point.
    *   `Alpha`: float, opacity value derived from represent logic.
    *   `MaterialId`: int, identifier of the material.
    *   `Position`: float3, world position of the represent point.

### Algorithm Steps
1.  **Initialization**:
    *   Set `surfaceHit.DistanceToWorld = ClippingPlanes.x` and `surfaceHit.TraceDistance = ClippingPlanes.x` [S1, unofficial].
    *   Set `surfaceHit.MaterialId = -1` [S1, unofficial].
    *   Set `representHit.DistanceToWorld = 1000000` (a large initial value) [S1, unofficial].
    *   Set `representHit.MaterialId = -1` and `representHit.Alpha = 0` [S1, unofficial].
    *   Initialize ray distance $t = ClippingPlanes.x$ [S1, unofficial].
    *   Initialize loop counter $i = 0$ [S1, unofficial].
    *   Initialize `representFound = false` [S1, unofficial].

2.  **Marching Loop**:
    *   Loop $i$ from 0 to `SphereTracingSteps - 1` [S1, unofficial].
    *   Calculate adaptive epsilon: $\epsilon = 0.0015 \times t$ [S1, unofficial].
    *   Sample SDF at current position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$:
        *   Let `currentMap` be the result: `currentMap.x` is distance $d$, `currentMap.y` is material ID $m$ [S1, unofficial].
    *   **Surface Hit Check**:
        *   If $d \le \epsilon$:
            *   Set `surfaceHit.DistanceToWorld = t + d` [S1, unofficial].
            *   Set `surfaceHit.TraceDistance = t + d` [S1, unofficial].
            *   Set `surfaceHit.MaterialId = m` [S1, unofficial].
            *   Set `surfaceHit.Position = \mathbf{p} + d\mathbf{d}` [S1, unofficial].
            *   Break loop [S1, unofficial].
    *   **Represent Hit Logic**:
        *   Sample next point: `nextMap` at $\mathbf{p}_{next} = \mathbf{o} + (t + d)\mathbf{d}$ [S1, unofficial].
        *   Check if inside pixel radius: If $d < t \times \text{RadiusPixel}$:
            *   If `nextMap.x > d` (moving away from surface) and `representFound == false`:
                *   Set `representFound = true` [S1, unofficial].
            *   If $d < \text{representHit.DistanceToWorld}$ and (`representHit.MaterialId == -1` or `representHit.MaterialId == m`):
                *   Update `representHit.DistanceToWorld = t + d` [S1, unofficial].
                *   Update `representHit.MaterialId = m` [S1, unofficial].
                *   Update `representHit.Position = \mathbf{p} + d\mathbf{d}` [S1, unofficial].
    *   **Advance Ray**:
        *   $t \leftarrow t + d$ [S1, unofficial].
        *   `currentMap` = `nextMap` [S1, unofficial].
    *   **Far Clipping Check**:
        *   If $t > \text{ClippingPlanes.y}$:
            *   Set `surfaceHit.TraceDistance = ClippingPlanes.y` [S1, unofficial].
            *   Set `surfaceHit.MaterialId = -1` [S1, unofficial].
            *   Break loop [S1, unofficial].

3.  **Post-Loop Processing**:
    *   If `representHit.MaterialId >= 0`:
        *   Calculate `representHit.Alpha`:
            $$ \text{Alpha} = 1.0 - \frac{\text{representHit.DistanceToWorld}}{\text{surfaceHit.TraceDistance} \times \text{RadiusPixel}} $$
            [S1, unofficial]
        *   Update `representHit.Position` if not already set [S1, unofficial].
    *   If loop finished without a surface hit ($i == \text{SphereTracingSteps}$):
        *   Use latest `currentMap` data for `surfaceHit` as a fallback [S1, unofficial].
    *   If `surfaceHit.MaterialId < 0`:
        *   Set `representHit.Alpha = 0` [S1, unofficial].

4.  **Final Evaluation**:
    *   If `DisableAntiAliasing` is true, call `RayMarchNoAA(r, surfaceHit, representHit)`; else call `RayMarch(r, surfaceHit, representHit)` [S1, unofficial].
    *   Evaluate hits:
        *   Call `EvaluateHit(id.xy, r, surfaceHit, surface)` [S1, unofficial].
        *   Call `EvaluateHit(id.xy, r, representHit, represent)` [S1, unofficial].
    *   Write final data:
        *   Call `WriteSphereTracingData(id.xy, 0, surface)` [S1, unofficial].
        *   Call `WriteSphereTracingData(id.xy, 1, represent)` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sphere_tracing_with_represent_hit(ray_origin, ray_direction, sdf_map, clipping_planes, sphere_tracing_steps, radius_pixel, disable_anti_aliasing):
    # Initialization
    surface_hit = {
        'DistanceToWorld': clipping_planes.x,
        'TraceDistance': clipping_planes.x,
        'MaterialId': -1,
        'Position': ray_origin + ray_direction * clipping_planes.x
    }
    represent_hit = {
        'DistanceToWorld': 1000000.0,
        'Alpha': 0.0,
        'MaterialId': -1,
        'Position': ray_origin
    }
    represent_found = False
    t = clipping_planes.x
    current_map = sdf_map.sample(ray_origin + t * ray_direction) # Returns (distance, material_id)

    for i in range(sphere_tracing_steps):
        epsilon = 0.0015 * t
        
        # Surface Hit Check
        if current_map.distance <= epsilon:
            surface_hit['DistanceToWorld'] = t + current_map.distance
            surface_hit['TraceDistance'] = t + current_map.distance
            surface_hit['MaterialId'] = current_map.material_id
            surface_hit['Position'] = ray_origin + (t + current_map.distance) * ray_direction
            break
        
        # Sample next point
        next_point = ray_origin + (t + current_map.distance) * ray_direction
        next_map = sdf_map.sample(next_point)
        
        # Represent Hit Logic
        if current_map.distance < t * radius_pixel:
            if next_map.distance > current_map.distance and not represent_found:
                represent_found = True
            
            if current_map.distance < represent_hit['DistanceToWorld']:
                if represent_hit['MaterialId'] == -1 or represent_hit['MaterialId'] == current_map.material_id:
                    represent_hit['DistanceToWorld'] = t + current_map.distance
                    represent_hit['MaterialId'] = current_map.material_id
                    represent_hit['Position'] = ray_origin + (t + current_map.distance) * ray_direction
        
        # Advance Ray
        t += current_map.distance
        current_map = next_map
        
        # Far Clipping Check
        if t > clipping_planes.y:
            surface_hit['TraceDistance'] = clipping_planes.y
            surface_hit['MaterialId'] = -1
            break

    # Post-Loop Processing
    if represent_hit['MaterialId'] >= 0:
        # Avoid division by zero if TraceDistance is 0 or very small
        if surface_hit['TraceDistance'] > 0:
            represent_hit['Alpha'] = 1.0 - (represent_hit['DistanceToWorld'] / (surface_hit['TraceDistance'] * radius_pixel))
        else:
            represent_hit['Alpha'] = 0.0
    
    if surface_hit['MaterialId'] < 0:
        represent_hit['Alpha'] = 0.0

    # Final Evaluation
    if disable_anti_aliasing:
        ray_march_no_aa(ray_origin, ray_direction, surface_hit, represent_hit)
    else:
        ray_march(ray_origin, ray_direction, surface_hit, represent_hit)
        
    evaluate_hit(surface_hit, 'surface')
    evaluate_hit(represent_hit, 'represent')
    
    write_sphere_tracing_data(0, surface_hit)
    write_sphere_tracing_data(1, represent_hit)
```

## Operating characteristics
*   **Time Complexity**: $O(N)$ per pixel, where $N$ is `SphereTracingSteps`. The represent hit logic adds constant overhead per step.
*   **Memory Complexity**: $O(1)$ per pixel for hit structures, plus storage for the SDF map and two render targets (surface and represent) [S1, unofficial].
*   **Performance**: Performance depends heavily on the SDF evaluation cost and the number of steps. The dual-pass nature increases memory bandwidth usage due to writing to two targets [S1, unofficial].
*   **Strengths**: Reduces aliasing at object edges by preserving represent hit information; handles transparency better than standard sphere tracing [S1, unofficial].
*   **Limitations**: Requires additional memory for the second render target; increased complexity in hit evaluation logic [S1, unofficial].
*   **When to use**: When high-quality anti-aliasing at SDF boundaries is required, especially in deferred rendering pipelines [S1, unofficial].
*   **When not to use**: When memory bandwidth is a critical bottleneck or when simple geometric primitives do not require edge anti-aliasing [S1, unofficial].

## Background
Sphere Tracing with Represent Hit is implemented in the "SphereTracing" project by Nico Ell [S1, unofficial]. It is part of a larger deferred rendering system that includes ambient occlusion, environment mapping, and bilateral filtering [S1, unofficial]. The method was developed to address aliasing issues in real-time SDF rendering by maintaining a secondary "represent" hit point [S1, unofficial].

## Sources
[S1] SphereTracing, https://github.com/nicoell/SphereTracing, fetched 2026-10-26 (unofficial)
