# Sphere Tracing with Represent Hit

Also called: RayMarch, Sphere Tracing with Surface/Represent Passes, SphereTracingFPass, SphereTracingKPass, Surface/Represent Ray Marching

## Overview
Sphere Tracing with Represent Hit is a deferred rendering technique that extends standard sphere tracing by tracking two distinct hit points per ray: a primary **surface hit** (the exact geometric intersection) and a **represent hit** (a proxy for the surface at the pixel's radius) to handle transparency and aliasing at object edges [S1, unofficial]. It selects the ray with greater visual impact (higher alpha) to trace, writing results to two separate render targets [S1, unofficial]. Unlike standard sphere tracing which terminates at the first surface contact, this method continues to evaluate proximity to the pixel radius to capture edge softness, distinguishing it from siblings like [Uniform Sphere Tracing](4406-uniform-sphere-tracing.md) which uses fixed step sizes, and [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md) which focuses on gradient computation.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin $ro$, ray direction $v$ (normalized), SDF map function $Map(p)$ returning a float2 (distance, material ID), clipping planes $ClippingPlanes = [near, far]$, maximum sphere tracing steps $SphereTracingSteps$, pixel radius $RadiusPixel$, and anti-aliasing flag $DisableAntiAliasing$ [S1, unofficial].
*   **Outputs:** Two hit structures, `surfaceHit` and `representHit`, written to render targets index 0 and 1 respectively [S1, unofficial].

**Data Structures:**
*   **Hit Structure:** Contains fields: `DistanceToWorld` (float), `TraceDistance` (float), `MaterialId` (int), `Position` (float3), and `Alpha` (float) [S1, unofficial].
*   **SDF Map Return:** A float2 where `.x` is the signed distance and `.y` is the material ID [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:**
    *   Initialize `surfaceHit` and `representHit` with `DistanceToWorld` set to `ClippingPlanes.x` (near) and `ClippingPlanes.y` (far) respectively [S1, unofficial].
    *   Set `surfaceHit.MaterialId = -1` and `representHit.MaterialId = -1` [S1, unofficial].
    *   Set `representHit.DistanceToWorld = 1000000` and `representHit.Alpha = 0` [S1, unofficial].
    *   Initialize ray distance $t = ClippingPlanes.x$ [S1, unofficial].
    *   Sample the SDF at the starting point: `currentMap = Map(ro + t * v)` [S1, unofficial].
    *   Initialize loop counter $i = 0$ and boolean `representFound = false` [S1, unofficial].

2.  **Sphere Tracing Loop:**
    *   Iterate $i$ from $0$ to $SphereTracingSteps - 1$ [S1, unofficial].
    *   Calculate adaptive epsilon: $\epsilon = 0.0015 \cdot t$ [S1, unofficial].
    *   **Surface Hit Check:** If `currentMap.x` $\le \epsilon$:
        *   Set `surfaceHit.DistanceToWorld = t`.
        *   Set `surfaceHit.TraceDistance = t`.
        *   Set `surfaceHit.MaterialId = currentMap.y`.
        *   Set `surfaceHit.Position = ro + t * v`.
        *   Break loop [S1, unofficial].
    *   **Represent Hit Logic:**
        *   Sample next point: `nextMap = Map(ro + (t + currentMap.x) * v)` [S1, unofficial].
        *   Check if inside pixel radius: If `currentMap.x < (t * RadiusPixel)`:
            *   If `nextMap.x > currentMap.x` (moving away from surface) and `representFound == false`:
                *   Set `representFound = true` [S1, unofficial].
            *   If `currentMap.x < representHit.DistanceToWorld` and (`representHit.MaterialId == -1` or `representHit.MaterialId == currentMap.y`):
                *   Update `representHit.DistanceToWorld = t`.
                *   Update `representHit.MaterialId = currentMap.y`.
                *   Update `representHit.Position = ro + t * v` [S1, unofficial].
    *   **Advance Ray:**
        *   $t \leftarrow t + currentMap.x$ [S1, unofficial].
        *   `currentMap = nextMap` [S1, unofficial].
    *   **Far Clipping Check:**
        *   If $t > ClippingPlanes.y$:
            *   Set `surfaceHit.TraceDistance = ClippingPlanes.y`.
            *   Set `surfaceHit.MaterialId = -1`.
            *   Break loop [S1, unofficial].

3.  **Post-Loop Processing:**
    *   If `representHit.MaterialId >= 0`:
        *   Calculate `representHit.Alpha = 1.0 - representHit.DistanceToWorld / (representHit.TraceDistance * RadiusPixel)` [S1, unofficial].
        *   Update `representHit.Position` (if not already set) [S1, unofficial].
    *   If loop finished without a surface hit ($i == SphereTracingSteps$):
        *   Use latest `currentMap` data for `surfaceHit` as a fallback hit [S1, unofficial].
    *   If `surfaceHit.MaterialId < 0` or `representHit.MaterialId < 0`:
        *   Set corresponding `Alpha` to $0$ [S1, unofficial].

4.  **Final Ray Marching and Evaluation:**
    *   If $DisableAntiAliasing$ is true, call `RayMarchNoAA(r, surfaceHit, representHit)`; else call `RayMarch(r, surfaceHit, representHit)` [S1, unofficial].
    *   Evaluate hits: Call `EvaluateHit(id.xy, r, surfaceHit, surface)` and `EvaluateHit(id.xy, r, representHit, represent)` [S1, unofficial].
    *   Write final data: Call `WriteSphereTracingData(id.xy, 0, surface)` and `WriteSphereTracingData(id.xy, 1, represent)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sphere_tracing_with_represent_hit(ro, v, map_func, clipping_near, clipping_far, max_steps, radius_pixel, disable_aa):
    # Initialization
    surface_hit = {
        'DistanceToWorld': clipping_near,
        'TraceDistance': clipping_near,
        'MaterialId': -1,
        'Position': ro + clipping_near * v,
        'Alpha': 0.0
    }
    represent_hit = {
        'DistanceToWorld': 1000000.0,
        'TraceDistance': clipping_far,
        'MaterialId': -1,
        'Position': ro + clipping_far * v,
        'Alpha': 0.0
    }
    
    t = clipping_near
    current_map = map_func(ro + t * v)
    represent_found = False
    
    for i in range(max_steps):
        epsilon = 0.0015 * t
        
        # Surface Hit Check
        if current_map[0] <= epsilon:
            surface_hit['DistanceToWorld'] = t
            surface_hit['TraceDistance'] = t
            surface_hit['MaterialId'] = current_map[1]
            surface_hit['Position'] = ro + t * v
            break
            
        # Represent Hit Logic
        next_map = map_func(ro + (t + current_map[0]) * v)
        
        if current_map[0] < (t * radius_pixel):
            if next_map[0] > current_map[0] and not represent_found:
                represent_found = True
            if current_map[0] < represent_hit['DistanceToWorld']:
                if represent_hit['MaterialId'] == -1 or represent_hit['MaterialId'] == current_map[1]:
                    represent_hit['DistanceToWorld'] = t
                    represent_hit['MaterialId'] = current_map[1]
                    represent_hit['Position'] = ro + t * v
                    
        # Advance Ray
        t += current_map[0]
        current_map = next_map
        
        # Far Clipping Check
        if t > clipping_far:
            surface_hit['TraceDistance'] = clipping_far
            surface_hit['MaterialId'] = -1
            break
            
    # Post-Loop Processing
    if represent_hit['MaterialId'] >= 0:
        if surface_hit['TraceDistance'] > 0:
            represent_hit['Alpha'] = 1.0 - represent_hit['DistanceToWorld'] / (surface_hit['TraceDistance'] * radius_pixel)
        else:
            represent_hit['Alpha'] = 0.0
            
    if i == max_steps:
        # Fallback hit
        surface_hit['Position'] = ro + t * v
        surface_hit['MaterialId'] = current_map[1]
        
    if surface_hit['MaterialId'] < 0:
        surface_hit['Alpha'] = 0.0
    if represent_hit['MaterialId'] < 0:
        represent_hit['Alpha'] = 0.0
        
    # Final Evaluation (Placeholder for specific shader logic)
    if disable_aa:
        ray_march_no_aa(ro, v, surface_hit, represent_hit)
    else:
        ray_march(ro, v, surface_hit, represent_hit)
        
    evaluate_hit(surface_hit, 'surface')
    evaluate_hit(represent_hit, 'represent')
    
    write_sphere_tracing_data(0, surface_hit)
    write_sphere_tracing_data(1, represent_hit)
```

**Parameters:**
*   $SphereTracingSteps$: Typically 32 to 128 [S1, unofficial].
*   $RadiusPixel$: Depends on camera FOV and resolution; scales with distance [S1, unofficial].
*   $ClippingPlanes$: Defined by camera near and far planes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If `TraceDistance` is zero or negative, the alpha calculation for `representHit` must be guarded [S1, unofficial].
*   **Material ID Mismatch:** The represent hit only updates if the material ID matches the current one or is uninitialized, preventing mixing of different materials in the represent pass [S1, unofficial].
*   **Precision:** Adaptive epsilon $\epsilon = 0.0015 \cdot t$ scales with distance, preventing precision issues at far clipping planes [S1, unofficial].

## Subtopics

### RayMarchNoAA
A variant that skips anti-aliasing steps for performance, used when `DisableAntiAliasing` is true [S1, unofficial].

### RayMarch
The standard anti-aliased ray marching pass, used when `DisableAntiAliasing` is false [S1, unofficial].

### EvaluateHit
Post-processing step that calculates lighting and shading for the hit points [S1, unofficial].

### WriteSphereTracingData
Output step that writes the hit data to the respective render targets (index 0 for surface, index 1 for represent) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$ where $N$ is the number of pixels and $S$ is the maximum number of sphere tracing steps [S1, unofficial]. The represent hit logic adds constant overhead per step.
*   **Memory Complexity:** $O(N)$ for storing the two hit structures per pixel (surface and represent) [S1, unofficial].
*   **Performance:** Depends heavily on the SDF evaluation cost and the number of steps required to reach the surface or pixel radius [S1, unofficial].
*   **Strengths:** Handles transparency and aliasing at object edges effectively by tracking the represent hit [S1, unofficial].
*   **Limitations:** Requires two render targets and additional logic per step, increasing computational cost compared to standard sphere tracing [S1, unofficial].
*   **When to Use:** When rendering SDF-based geometry with soft edges or transparency is required [S1, unofficial].
*   **When Not to Use:** When performance is critical and edge softness is not needed, or when the scene is fully opaque [S1, unofficial].

## Background
Sphere Tracing with Represent Hit was developed as part of a deferred rendering pipeline for SDF-based graphics, implemented in Unity using Compute Shaders [S1, unofficial]. It addresses the limitation of standard sphere tracing in handling anti-aliasing and transparency at object boundaries by introducing a secondary "represent" hit point [S1, unofficial]. The method is part of a larger system that includes ambient occlusion, environment mapping, and bilateral filtering [S1, unofficial].

## Sources
[S1] SphereTracing, https://github.com/nicoell/SphereTracing, fetched 2026-10-26 (unofficial)
