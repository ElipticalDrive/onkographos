# Infinite Plane Ray Intersection

Also called: Analytical ray-plane intersection, Infinite Ground Plane, InfinitePlan::rayIntersect, InfinitePlan::rayIntersectP, Ray-Plane Intersection, planeIntr

## Overview
Computes the intersection of a ray with an infinite plane defined by a point and a normal, typically used as a ground plane separate from the scene's BVH for ray intersection testing [S1, unofficial]. It returns the intersection distance and fills intersection details, or rejects if the ray is parallel to the plane or falls outside the valid ray segment [S2, unofficial]. Unlike [Ray-box intersection](776-ray-box-intersection.md) which handles bounded volumes, this method handles an unbounded surface, and unlike [General Ray-Triangle Intersection (Plücker Coordinates)](909-general-ray-triangle-intersection-pl-cker-coordinates.md) which handles finite triangles, this handles the entire plane.

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ (world space).
- Ray direction $\mathbf{d}$ (world space, assumed normalized or treated as a vector).
- Plane point $\mathbf{P}$ (world space).
- Plane normal $\mathbf{N}$ (world space, assumed normalized).
- Ray segment bounds $t_{min}$ and $t_{max}$.
- Epsilon $\epsilon$ for parallelism check.

**Outputs:**
- Intersection distance $t_{hit}$.
- Intersection point $\mathbf{p}$.
- Intersection normal $\mathbf{n}$.
- Boolean success flag.

**Data Structures:**
- `Intersection` struct:
  - `itsPoint`: vector3 (intersection point).
  - `normal`: vector3 (plane normal).
  - `shape`: reference to the plane object.
  - `tHit`: float (distance along ray).

**Steps:**
1. Compute the denominator: $denom = \mathbf{N} \cdot \mathbf{d}$.
2. Check for parallelism: If $|denom| < \epsilon$, the ray is parallel to the plane. Return failure.
3. Compute the intersection distance $t$:
   $$ t = \frac{\mathbf{N} \cdot (\mathbf{P} - \mathbf{o})}{\mathbf{N} \cdot \mathbf{d}} $$
   Alternatively expressed as:
   $$ t = \frac{-\mathbf{N} \cdot \mathbf{o} + \mathbf{N} \cdot \mathbf{P}}{\mathbf{N} \cdot \mathbf{d}} $$
4. Check segment bounds: If $t < t_{min}$ or $t > t_{max}$, return failure.
5. Compute intersection point: $\mathbf{p} = \mathbf{o} + \mathbf{d} \cdot t$.
6. Fill intersection details:
   - `its.itsPoint` = $\mathbf{p}$
   - `its.normal` = $\mathbf{N}$
   - `its.shape` = this plane
   - `its.tHit` = $t$
7. Update ray bounds: Set $t_{max} = t$ (if the implementation updates the ray's maxT to cull further intersections) [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_plane_intersect(ray_o, ray_d, plane_p, plane_n, t_min, t_max, epsilon):
    """
    Intersect a ray with an infinite plane.
    
    Args:
        ray_o: Vector3, ray origin
        ray_d: Vector3, ray direction
        plane_p: Vector3, point on the plane
        plane_n: Vector3, plane normal (assumed normalized)
        t_min: float, minimum ray distance
        t_max: float, maximum ray distance
        epsilon: float, threshold for parallelism check
        
    Returns:
        tuple: (success: bool, t_hit: float, point: Vector3, normal: Vector3)
    """
    # Step 1: Compute denominator
    denom = dot(plane_n, ray_d)
    
    # Step 2: Check for parallelism
    if abs(denom) < epsilon:
        return (False, 0.0, Vector3(0,0,0), Vector3(0,0,0))
    
    # Step 3: Compute intersection distance t
    # Formula: t = (N . (P - O)) / (N . D)
    # Equivalent to: t = (-N . O + N . P) / (N . D)
    numerator = dot(plane_n, plane_p - ray_o)
    t_hit = numerator / denom
    
    # Step 4: Check segment bounds
    if t_hit < t_min or t_hit > t_max:
        return (False, 0.0, Vector3(0,0,0), Vector3(0,0,0))
    
    # Step 5: Compute intersection point
    p_hit = ray_o + ray_d * t_hit
    
    # Step 6: Fill intersection details
    normal = plane_n
    
    # Step 7: Update ray maxT (optional, depending on implementation)
    # In some implementations, maxT is updated to cull further intersections
    # t_max = t_hit 
    
    return (True, t_hit, p_hit, normal)
```

**Parameters:**
- $\epsilon$: Threshold for parallel ray/plane test. Value not specified in the sources [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Parallel Ray:** If the ray direction is perpendicular to the plane normal (dot product is zero), the ray never intersects the plane. This is handled by the epsilon check.
- **Degenerate Input:** If the plane normal is not normalized, the results will be incorrect. The sources assume $\mathbf{N}$ is normalized [S3, unofficial].
- **Precision:** Floating-point precision issues may arise when the ray is nearly parallel to the plane. Using a small epsilon is necessary but the value is not specified [S2, unofficial].

## Subtopics

### Infinite Ground Plane
An infinite ground plane kept separate from the scene's BVH for ray intersection testing, often used for shadowing and ground reflections [S1, unofficial]. It is chosen when a simple, unbounded surface is needed for the scene base.

## Operating characteristics
**Time Complexity:** $O(1)$ constant time, involving a few dot products and a division.
**Memory Complexity:** $O(1)$ constant space.
**Measured Performance:** Not given in the sources.
**Strengths:** Extremely fast, simple to implement, no complex data structures required.
**Limitations:** Only handles infinite planes, not bounded primitives. Requires careful handling of parallel rays.
**When to Use:** When a simple ground plane is needed, or as a primitive in a larger intersection system.
**When Not to Use:** When bounded primitives are required, or when the plane is part of a complex mesh that should be handled by a BVH or other acceleration structure.

## Background
The method is used in path tracers and ray tracers to handle ground planes. It is mentioned in the context of adding a sun, a uniform sky, and an infinite ground plane to a renderer [S1, unofficial]. The analytical solution for ray-plane intersection is a fundamental building block in computer graphics, often taught in introductory graphics courses.

## Sources
[S1] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
[S2] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S3] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
