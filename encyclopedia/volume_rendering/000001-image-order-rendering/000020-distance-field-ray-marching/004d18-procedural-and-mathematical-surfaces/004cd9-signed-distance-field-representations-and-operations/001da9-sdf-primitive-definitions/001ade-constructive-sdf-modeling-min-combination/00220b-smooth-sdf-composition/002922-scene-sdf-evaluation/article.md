# Scene SDF Evaluation

Also called: SDF Scene Composition, Signed Distance Function (SDF) Scene Composition

## Overview
Scene SDF Evaluation computes the minimum signed distance from a query point to any primitive in a scene, effectively performing a union operation [S1, unofficial].

## How it works
**Inputs:**
- A scene structure containing lists of primitives (e.g., spheres, cuboids) [S1, unofficial].
- A query point $P$ [S1, unofficial].
- Transformations mapping local primitive coordinates to world space [S2, unofficial].

**Outputs:**
- A scalar value representing the signed distance from $P$ to the nearest surface in the scene [S1, unofficial].

**Data Structures:**
- `Scene`: Contains arrays for primitive types and integer counts [S1, unofficial].
  - `spheres`: Array of sphere primitives. Each sphere has a center $C$ and radius $R$ [G].
  - `cuboids`: Array of cuboid primitives. Each cuboid has a center $C$, orientation $Q$, and half-extents $H$ [G].
  - `sphere_count`: Integer count of spheres [S1, unofficial].
  - `cuboid_count`: Integer count of cuboids [S1, unofficial].
- `ComposedSDF` (Variant): A container holding a list of SDF objects and a corresponding stack of transformation matrices (one per SDF) [S2, unofficial].

**Steps:**
1. Initialize `min_dist` to `MAX_SIGNED_DISTANCE` (10000.0) [S1, unofficial].
2. Iterate through all spheres (0 to `scene.sphere_count`): transform $P$ into the local coordinate system of the current sphere [G], evaluate the sphere's SDF [G], and update `min_dist = min(min_dist, d_{sphere})` [S1, unofficial].
3. Iterate through all cuboids (0 to `scene.cuboid_count`): transform $P$ into the local coordinate system of the current cuboid using its orientation $Q$ and center $C$ [G], evaluate the cuboid's SDF [G], and update `min_dist = min(min_dist, d_{cuboid})` [S1, unofficial].
4. Return `min_dist` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_scene_sdf(scene, point):
    """
    Computes the minimum signed distance from point to any primitive in the scene.
    """
    # Initialize with MAX_SIGNED_DISTANCE [S1, unofficial]
    min_dist = 10000.0
    
    # Evaluate all spheres [S1, unofficial]
    for i in range(scene.sphere_count):
        sphere = scene.spheres[i]
        # Transform point to sphere's local space [G]
        local_point = point - sphere.center
        # Evaluate sphere SDF: distance to center minus radius [G]
        dist = (local_point.x**2 + local_point.y**2 + local_point.z**2)**0.5 - sphere.radius
        if dist < min_dist:
            min_dist = dist
            
    # Evaluate all cuboids [S1, unofficial]
    for i in range(scene.cuboid_count):
        cuboid = scene.cuboids[i]
        # Transform point to cuboid's local space [G]
        translated = point - cuboid.center
        local_point = cuboid.orientation.inverse() * translated
        # Evaluate cuboid SDF: max(|x|-wx, |y|-wy, |z|-wz) [G]
        abs_local = (abs(local_point.x), abs(local_point.y), abs(local_point.z))
        dist = max(abs_local[0] - cuboid.half_extents.x, 
                   abs_local[1] - cuboid.half_extents.y, 
                   abs_local[2] - cuboid.half_extents.z)
        if dist < min_dist:
            min_dist = dist
            
    return min_dist
```

**Parameters:**
- `MAX_SIGNED_DISTANCE`: Default 10000.0 [S1, unofficial].

**Edge Cases and Pitfalls:**
- If objects overlap or share faces, artifacts may occur in determining interior-ness [S2, unofficial].

## Subtopics

### Composed SDF
A variant that combines multiple SDFs into a single queryable SDF by passing a list of SDFs and a stack of transforms (one per SDF) to a constructor [S2, unofficial]. Pitfall: Objects should not overlap or share faces, otherwise there will be artifacts in the SDF query in determining interior-ness [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S2] GitHub - UM-ARM-Lab/pytorch_volumetric: Volumetric structures such as ..., https://github.com/UM-ARM-Lab/pytorch_volumetric, fetched 2026-10-30 (unofficial)
