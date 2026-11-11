# Scene SDF Evaluation

Also called: SDF Scene Composition, Signed Distance Function (SDF) Scene Composition

## Overview
Scene SDF Evaluation computes the minimum signed distance from a query point to the nearest surface among all primitives in a scene, effectively performing a union operation [S1, unofficial]. It distinguishes itself from other constructive SDF operations by aggregating individual primitive distances via the minimum function, enabling the representation of complex, disjoint geometries as a single continuous field.

## How it works
**Inputs and Outputs**
- **Input**: A query point $P$ (3D vector) and a scene definition containing a list of sphere primitives and a list of cuboid primitives.
- **Output**: A scalar signed distance value $d$, representing the distance to the closest surface.

**Data Structures**
The scene is represented by two arrays:
1. `spheres`: An array of sphere primitives. Each sphere contains a center position $C$ (3D vector) and a radius $r$ (scalar).
2. `cuboids`: An array of cuboid primitives. Each cuboid contains a center position $C$ (3D vector) and a half-extent vector $H$ (3D vector), defining the bounds $[C_x - H_x, C_x + H_x]$, etc.

**Algorithm Steps**
1. Initialize a variable `min_dist` to a large positive value representing infinity (e.g., $10000.0$ or `f32::MAX`) [S1, unofficial].
2. Iterate through each sphere in the `spheres` array:
   a. Calculate the distance from $P$ to the sphere surface using the sphere SDF function.
   b. Update `min_dist` to the minimum of its current value and the sphere's distance.
3. Iterate through each cuboid in the `cuboids` array:
   a. Calculate the distance from $P$ to the cuboid surface using the cuboid SDF function.
   b. Update `min_dist` to the minimum of its current value and the cuboid's distance.
4. Return `min_dist`.

**Formulas**
The distance to a sphere $i$ with center $C_i$ and radius $r_i$ is:
$$ d_{sphere}(P) = \| P - C_i \| - r_i $$

The distance to a cuboid $j$ with center $C_j$ and half-extents $H_j$ is calculated by clamping the vector from the center to the point within the box bounds:
$$ d_{cuboid}(P) = \| \max(|P - C_j| - H_j, 0) \| - \min(\max(|P - C_j| - H_j, 0)) $$
*Note: The standard implementation often simplifies this by computing the closest point on the box to P, then taking the Euclidean distance if outside, or 0 if inside.*

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_scene_sdf(point, scene):
    """
    Computes the minimum signed distance from point to any primitive in the scene.
    
    Args:
        point: Vec3, the query point in world space.
        scene: Object containing 'spheres' (list of Sphere) and 'cuboids' (list of Cuboid).
        
    Returns:
        float: The signed distance to the nearest surface.
    """
    # Initialize with a value larger than any possible distance in the scene
    min_dist = 10000.0
    
    # Check all spheres
    for sphere in scene.spheres:
        # Vector from sphere center to point
        diff = point - sphere.center
        # Distance to surface: distance to center minus radius
        dist_to_sphere = diff.length() - sphere.radius
        if dist_to_sphere < min_dist:
            min_dist = dist_to_sphere
            
    # Check all cuboids
    for cuboid in scene.cuboids:
        # Vector from cuboid center to point
        diff = point - cuboid.center
        # Absolute difference
        abs_diff = abs(diff)
        # Check if point is inside the cuboid
        inside = abs_diff <= cuboid.half_extents
        # Distance to surface
        if all(inside):
            dist_to_cuboid = 0.0
        else:
            # Distance to closest point on box surface
            outside = abs_diff - cuboid.half_extents
            # Clamp negative values to 0 (inside parts)
            clamped_outside = max(outside, 0.0)
            dist_to_cuboid = clamped_outside.length()
            
        if dist_to_cuboid < min_dist:
            min_dist = dist_to_cuboid
            
    return min_dist
```

**Parameters**
- `min_dist` initial value: Typically set to a value larger than the scene bounds (e.g., $10000.0$) [S1, unofficial].

**Edge Cases**
- **Empty Scene**: If both `spheres` and `cuboids` are empty, the function returns the initial `min_dist` value. This should be handled by the caller or by initializing to a specific error code or infinity.
- **Precision**: Floating-point precision issues may arise when points are very close to surfaces or when combining many primitives.

## Subtopics

## Operating characteristics
- **Time Complexity**: $O(N + M)$, where $N$ is the number of spheres and $M$ is the number of cuboids. Each primitive requires a constant number of arithmetic operations.
- **Memory Complexity**: $O(1)$ auxiliary space, excluding the storage of the scene primitives themselves.
- **Strengths**: Simple to implement; efficient for scenes with a moderate number of primitives; supports arbitrary combinations of basic shapes.
- **Limitations**: Performance degrades linearly with the number of primitives; not suitable for highly complex scenes without spatial acceleration structures (which are not part of this basic evaluation method).

## Background
This method is a fundamental component of Signed Distance Field (SDF) ray marching, specifically within the context of Constructive SDF Modeling using Min Combination. It allows for the creation of complex scenes from simple geometric primitives by leveraging the property that the union of shapes corresponds to the minimum of their distance functions. The implementation details provided are derived from a practical ray marching example [S1, unofficial].

## Sources
[S1] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
