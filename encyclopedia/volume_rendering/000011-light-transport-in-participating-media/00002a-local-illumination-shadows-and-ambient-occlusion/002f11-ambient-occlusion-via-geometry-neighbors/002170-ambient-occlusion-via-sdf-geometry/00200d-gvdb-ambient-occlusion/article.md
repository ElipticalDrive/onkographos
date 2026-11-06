# GVDB Ambient Occlusion

Also called: GVDB AO, computeAmbientOcclusion

## Overview
GVDB Ambient Occlusion computes local occlusion at a surface point by sampling a hemisphere of directions against a volume density field, using either point sampling or ray casting with distance-based falloff [S1, unofficial]. It differs from SDF-based AO by querying a volumetric density field directly rather than a signed distance function, and from SSAO by operating in world-space volume coordinates rather than screen-space depth buffers [S1, unofficial].

## How it works
**Inputs:**
- Volume data structure supporting density queries.
- Surface position `pos` (world space).
- Surface normal `normal` (world space, normalized).
- `aoSamples`: Number of hemisphere samples (integer, max 512).
- `aoRadius`: World-space radius for AO sampling (float).
- `aoBias`: Backtrack distance to reduce acne artifacts (float, typically $10^{-3}$).
- `AMBIENT_OCCLUSION_MODE`: Integer switch (0 for point sampling, 1 for ray sampling).
- `noise`: A precomputed random rotation vector (float3).

**Outputs:**
- Occlusion factor (float, range [0, 1]).

**Data Structures:**
- The volume is accessed via a query function `queryVolume(pos)` returning a scalar density value.
- Vectors are 3D floats (`float3`).

**Steps:**
1. If `aoSamples == 0`, return 1.0 [S1, unofficial].
2. Compute tangent and bitangent vectors to form a local basis aligned with the surface normal [S1, unofficial]:
   $$ \text{tangent} = \frac{\text{noise} - \text{normal} \cdot \text{noise}}{\|\text{noise} - \text{normal} \cdot \text{noise}\|} $$
   $$ \text{bitangent} = \text{normal} \times \text{tangent} $$
   [S1, unofficial]
3. Determine the query position by backtracking along the normal to avoid self-intersection/acne:
   $$ \text{queryPos} = \text{pos} + \text{normal} \times \text{aoBias} $$
   [S1, unofficial]
4. Initialize `occlusionSum = 0.0` [S1, unofficial].
5. For `i` from 0 to `aoSamples - 1`:
   a. Generate a sample direction `sampleLocal` in the hemisphere defined by the tangent-bitangent-normal basis. (Note: The specific hemisphere sampling algorithm is not detailed in the source, but it implies uniform or stratified sampling over the upper hemisphere relative to the normal) [S1, unofficial].
   b. Transform `sampleLocal` to world space:
      $$ \text{sampleWorld} = \text{tangent} \times \text{sampleLocal.x} + \text{bitangent} \times \text{sampleLocal.y} + \text{normal} \times \text{sampleLocal.z} $$
      [S1, unofficial]
   c. If `AMBIENT_OCCLUSION_MODE == 0` (Point Sampling):
      i. Compute sample point: `samplePoint = queryPos + sampleWorld * aoRadius` [S1, unofficial].
      ii. Query volume: `density = queryVolume(samplePoint)` [S1, unofficial].
      iii. If `density <= aoBias`, add 1.0 to `occlusionSum` [S1, unofficial].
   d. If `AMBIENT_OCCLUSION_MODE == 1` (Ray Sampling):
      i. Shoot a ray from `queryPos` in direction `sampleWorld` with length `aoRadius` [S1, unofficial].
      ii. If a hit is found at distance `dist`:
          $$ \text{weight} = \text{smoothstep}(1, 0, \frac{\text{aoRadius}}{\text{dist}}) $$
          [S1, unofficial]
          iii. Add `weight` to `occlusionSum` [S1, unofficial].
      iii. If no hit is found, add 0.0 to `occlusionSum` [S1, unofficial].
6. Compute final occlusion:
   $$ \text{occlusion} = \frac{\text{occlusionSum}}{\text{aoSamples}} $$
   [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def computeGVDBAO(pos, normal, aoSamples, aoRadius, aoBias, mode, noise, queryVolume):
    if aoSamples == 0:
        return 1.0

    # Compute tangent and bitangent
    # Note: Source uses 'noise' as a random vector for rotation
    proj = noise - normal * dot(noise, normal)
    tangent = normalize(proj)
    bitangent = cross(normal, tangent)

    # Backtrack to avoid acne
    queryPos = pos + normal * aoBias

    occlusionSum = 0.0

    for i in range(aoSamples):
        # Generate hemisphere sample in local space
        # Assuming standard hemisphere sampling (e.g., uniform or stratified)
        # sampleLocal is a unit vector with z >= 0 in tangent-bitangent-normal basis
        sampleLocal = generateHemisphereSample(i, aoSamples) 
        
        # Transform to world space
        sampleWorld = tangent * sampleLocal.x + bitangent * sampleLocal.y + normal * sampleLocal.z

        if mode == 0:
            # Point sampling
            samplePoint = queryPos + sampleWorld * aoRadius
            density = queryVolume(samplePoint)
            if density <= aoBias:
                occlusionSum += 1.0
        elif mode == 1:
            # Ray sampling
            hitDist = rayMarch(queryPos, sampleWorld, aoRadius, queryVolume)
            if hitDist < aoRadius:
                # smoothstep(1, 0, aoRadius / dist)
                t = aoRadius / hitDist
                # clamp t to [0, 1] for smoothstep if necessary, though formula implies t > 1 if dist < radius
                # smoothstep(edge0, edge1, x) returns 0 if x <= edge0, 1 if x >= edge1
                # Here edge0=1, edge1=0. So if t >= 1 (dist <= radius), result is 1.
                # If t <= 0 (dist >= infinity), result is 0.
                # Standard smoothstep implementation:
                t_clamped = max(0.0, min(1.0, t))
                weight = t_clamped * t_clamped * (3.0 - 2.0 * t_clamped)
                occlusionSum += weight

    return occlusionSum / aoSamples
```

**Parameters:**
- `aoSamples`: Max 512 [S1, unofficial].
- `aoBias`: Typically $10^{-3}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- Acne artifacts from sampling at the surface are mitigated by backtracking position by `aoBias` before querying the volume [S1, unofficial].
- If `aoSamples` is 0, the function returns 1.0 (no occlusion) [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The method is part of the GVDB (GPU Volume Database) library, specifically within the context of adaptive sampling and reconstruction for volume visualization [S1, unofficial]. It is implemented as a device function for GPU execution [S1, unofficial].

## Sources
[S1] Learning adaptive sampling and reconstruction for Volume Visualization, https://github.com/shamanDevel/AdaptiveSampling, fetched 2026-10-27 (unofficial)
