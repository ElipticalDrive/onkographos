# Scene SDF Evaluation

Also called: SDF Scene Composition, Signed Distance Function (SDF) Scene Composition

## Overview
Scene SDF Evaluation computes the minimum signed distance from a query point to any primitive in a scene, effectively performing a union operation [S1, unofficial]. It aggregates individual primitive distances into a single continuous field, enabling the ray marching algorithm to step through complex scenes composed of multiple objects [S2, unofficial]. Unlike [Smooth SDF Composition](2685-smooth-sdf-composition.md), which handles the mathematical blending of primitives, this method manages the spatial composition and transformation of distinct geometric entities within a scene graph.

## How it works
**Inputs:**
- A scene structure containing lists of primitives (e.g., spheres, cuboids) [S1, unofficial].
- A query point $P$ in world space [S1, unofficial].
- Transformations mapping local primitive coordinates to world space [S2, unofficial].

**Outputs:**
- A scalar value representing the signed distance from $P$ to the nearest surface in the scene [S1, unofficial].

**Data Structures:**
- `Scene`: Contains arrays for primitive types.
  - `spheres`: Array of sphere primitives. Each sphere has a center $C$ and radius $R$.
  - `cuboids`: Array of cuboid primitives. Each cuboid has a center $C$, orientation $Q$, and half-extents $H$.
  - `sphere_count`: Integer count of spheres.
  - `cuboid_count`: Integer count of cuboids.
- `ComposedSDF` (Variant): A container holding a list of SDF objects and a corresponding stack of transformation matrices (one per SDF) [S2, unofficial].

**Steps:**
1. Initialize `min_dist` to a large value representing infinity, typically `MAX_SIGNED_DISTANCE` (e.g., 10000.0) [S1, unofficial].
2. Iterate through all spheres in the scene:
   a. Transform the query point $P$ into the local coordinate system of the current sphere.
   b. Evaluate the sphere's SDF: $d_{sphere} = \|P_{local} - C\| - R$.
   c. Update `min_dist = min(min_dist, d_{sphere})`.
3. Iterate through all cuboids in the scene:
   a. Transform the query point $P$ into the local coordinate system of the current cuboid using its orientation $Q$ and center $C$.
   b. Evaluate the cuboid's SDF: $d_{cuboid} = \text{distance to box surface}$.
   c. Update `min_dist = min(min_dist, d_{cuboid})`.
4. Return `min_dist` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_scene_sdf(scene, point):
    """
    Computes the minimum signed distance from point to any primitive in the scene.
    """
    # Initialize with a value larger than any expected distance in the scene
    min_dist = 10000.0
    
    # Evaluate all spheres
    for i in range(scene.sphere_count):
        sphere = scene.spheres[i]
        # Transform point to sphere's local space
        # Assuming sphere center is in world space, we translate
        local_point = point - sphere.center
        # SDF for sphere: distance to center minus radius
        dist = length(local_point) - sphere.radius
        if dist < min_dist:
            min_dist = dist
            
    # Evaluate all cuboids
    for i in range(scene.cuboid_count):
        cuboid = scene.cuboids[i]
        # Transform point to cuboid's local space
        # Translate to center, then rotate by inverse orientation
        translated = point - cuboid.center
        local_point = cuboid.orientation.inverse() * translated
        
        # SDF for axis-aligned box centered at origin
        # d = max(|x|-wx, |y|-wy, |z|-wz)
        abs_local = abs(local_point)
        dist = max(abs_local.x - cuboid.half_extents.x, 
                   abs_local.y - cuboid.half_extents.y, 
                   abs_local.z - cuboid.half_extents.z)
        
        if dist < min_dist:
            min_dist = dist
            
    return min_dist
```

**Parameters:**
- `MAX_SIGNED_DISTANCE`: Default 10000.0 [S1, unofficial].

**Edge Cases and Pitfalls:**
- If objects overlap or share faces, artifacts may occur in determining interior-ness [S2, unofficial].
- Precision issues may arise if `MAX_SIGNED_DISTANCE` is not sufficiently large relative to the scene bounds.

## Subtopics

### Composed SDF
A variant that combines multiple SDFs into a single queryable SDF by passing a list of SDFs and a stack of transforms (one per SDF) to a constructor [S2, unofficial]. It is useful for scenes with multiple objects where each object may have its own complex SDF definition. Pitfall: Objects should not overlap or share faces, otherwise there will be artifacts in the SDF query in determining interior-ness [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the total number of primitives in the scene. Each primitive must be evaluated once per query point.
- **Memory Complexity:** $O(N)$ to store the primitive definitions and their transforms.
- **Strengths:** Simple to implement; supports arbitrary combinations of primitives; efficient for scenes with a moderate number of objects.
- **Limitations:** Performance degrades linearly with the number of primitives; not suitable for scenes with thousands of objects without spatial acceleration structures (e.g., BVH, which is outside the scope of this specific evaluation method).
- **When to use:** Small to medium-sized scenes with procedural or mathematical primitives.
- **When not to use:** Large-scale scenes requiring hierarchical culling or complex mesh-based SDFs.

## Background
Scene SDF Evaluation is a fundamental technique in ray marching for rendering scenes defined by constructive solid geometry (CSG) or collections of primitives. It builds on the concept of [Smooth SDF Composition](2685-smooth-sdf-composition.md) by applying the union operation (minimum) across a set of transformed primitives. The method is widely used in real-time ray marching demos and educational graphics code due to its simplicity and effectiveness for procedural geometry.

## Sources
[S1] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S2] GitHub - UM-ARM-Lab/pytorch_volumetric: Volumetric structures such as ..., https://github.com/UM-ARM-Lab/pytorch_volumetric, fetched 2026-10-30 (unofficial)
