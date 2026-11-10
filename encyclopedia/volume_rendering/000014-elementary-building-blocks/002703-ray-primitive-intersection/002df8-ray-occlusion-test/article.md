# Ray Occlusion Test

Also called: hitSomething, hitSomething8, rtcOccluded1, rtcOccluded8

## Overview
The Ray Occlusion Test determines whether a ray intersects any geometry within a specified distance range, returning only a binary result (hit or miss) rather than detailed intersection data [S1, unofficial]. It is optimized for simple occlusion checks where the specific hit location or primitive ID is unnecessary, distinguishing it from sibling methods like [Ray-Triangle Intersection](11312-ray-triangle-intersection.md) which compute precise hit points.

## How it works
**Inputs and Outputs**
- **Input**: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, minimum distance $t_{near}$, maximum distance $t_{far}$ (typically infinity), and an acceleration structure context.
- **Output**: Boolean indicating if any geometry was intersected within $[t_{near}, t_{far}]$.

**Data Structures**
The method relies on a ray structure containing:
- `origin`: 3D vector $(x, y, z)$
- `direction`: 3D vector $(x, y, z)$, typically normalized.
- `tnear`: float, minimum intersection distance.
- `tfar`: float, maximum intersection distance.
- `hit`: boolean or flag indicating intersection status.

**Steps**
1. Initialize the ray structure with origin $\mathbf{o}$ and direction $\mathbf{d}$.
2. Set $t_{near}$ to the specified minimum distance (e.g., $0.0$ or a small epsilon to avoid self-intersection).
3. Set $t_{far}$ to the maximum valid distance (e.g., infinity or scene bounds).
4. Initialize the acceleration structure context if not already done.
5. Invoke the occlusion query function (e.g., `rtcOccluded1` for single rays or `rtcOccluded8` for batched rays).
6. The function updates the ray's `hit` status or returns a boolean.
7. Return `true` if an intersection occurred within the range, `false` otherwise.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_occlusion_test(origin, direction, min_dist, max_dist, context):
    """
    Tests if a ray intersects any geometry.
    
    Args:
        origin (Vec3): Ray origin.
        direction (Vec3): Ray direction (normalized).
        min_dist (float): Minimum distance for intersection.
        max_dist (float): Maximum distance for intersection.
        context: Acceleration structure context.
        
    Returns:
        bool: True if geometry is occluded, False otherwise.
    """
    # Initialize ray structure
    ray = Ray()
    ray.origin = origin
    ray.direction = direction
    ray.tnear = min_dist
    ray.tfar = max_dist
    ray.hit = False
    
    # Perform occlusion test
    # The specific implementation depends on the library (e.g., Embree)
    # This step queries the acceleration structure for any hit
    hit = query_occlusion(ray, context)
    
    return hit
```

**Parameters**
- `min_dist`: Minimum ray distance for intersection. Typical value: $0.0$ or a small epsilon [S1, unofficial].

**Edge Cases**
- If $t_{near} > t_{far}$, the test should return false immediately.
- Degenerate rays (zero-length direction) should be handled or rejected.

## Subtopics

### [Ray-cast Occlusion with Sampling Masks](11726-ray-cast-occlusion-with-sampling-masks.md)
An optimized variant using pre-generated sampling masks to efficiently update occlusion values along a ray direction, designed to accelerate local density averages in 3D volumes.

## Operating characteristics
- **Time Complexity**: Depends on the acceleration structure traversal depth. Generally $O(\log N)$ for balanced structures, but optimized for early exit on miss.
- **Memory Complexity**: Requires storage for the acceleration structure. The ray structure itself is minimal.
- **Strengths**: High performance for binary occlusion queries; avoids overhead of computing hit points, normals, or UVs.
- **Limitations**: Cannot provide detailed intersection information; requires a pre-built acceleration structure.
- **When to use**: When only binary occlusion is needed (e.g., visibility checks, shadow testing).
- **When not to use**: When hit details (position, normal, primitive ID) are required.

## Background
The method is commonly associated with the Embree ray tracing kernel, which provides `rtcOccluded1` and `rtcOccluded8` functions for efficient occlusion queries [S1, unofficial]. It is a fundamental building block in real-time rendering and offline rendering pipelines for visibility culling.

## Sources
[S1] whu-lyh/flod, https://github.com/whu-lyh/flod, fetched 2026-10-27 (unofficial)
