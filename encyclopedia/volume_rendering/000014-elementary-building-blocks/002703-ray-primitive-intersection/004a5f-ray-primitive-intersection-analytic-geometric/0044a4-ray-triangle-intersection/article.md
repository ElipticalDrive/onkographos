# Ray-Triangle Intersection
Also called: Ray-triangle test, Möller-Trumbore algorithm (common variant)

## Overview
Computes the intersection of a ray with a single triangle in 3D space, determining if the ray hits the triangle's interior and returning the intersection distance [S1, unofficial]. It solves the fundamental problem of ray tracing against mesh geometry, enabling volume rendering to sample density fields defined by triangular primitives [S1, unofficial]. Unlike [Sphere Intersection](1620-sphere-intersection.md) which uses algebraic distance checks, or [Ray-Plane/Rectangle Intersection](6498-ray-plane-rectangle-intersection.md) which handles infinite or axis-aligned bounds, this method explicitly tests against a finite, arbitrary triangular surface using barycentric coordinate validation [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (typically normalized), triangle vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$.
- **Outputs**: Boolean indicating intersection; if true, the distance $t$ along the ray to the intersection point.

**Data Structures**
- Vertices are stored as 3D vectors ($\mathbb{R}^3$).
- Ray is defined by origin $\mathbf{o} \in \mathbb{R}^3$ and direction $\mathbf{d} \in \mathbb{R}^3$.

**Parameters**
- `weight_tolerance`: A small value (e.g., $10^{-4}$) used to relax the barycentric coordinate check, allowing rays to pass slightly outside the triangle edges to prevent gaps between adjacent triangles [S1, unofficial].
- `small_t`: A minimum distance threshold to avoid self-intersection or self-shadowing bugs [S1, unofficial].

**Steps**
1. **Compute Triangle Normal**: Calculate the normal vector $\mathbf{n}$ of the triangle plane.
   $$ \mathbf{n} = \text{normalize}(\text{cross}(\mathbf{v}_1 - \mathbf{v}_0, \mathbf{v}_2 - \mathbf{v}_0)) $$
2. **Ray-Plane Intersection**: Calculate the numerator and denominator for the intersection distance $t$.
   $$ \text{numerator} = \text{dot}(\mathbf{n}, \mathbf{v}_0 - \mathbf{o}) $$
   $$ \text{denominator} = \text{dot}(\mathbf{n}, \mathbf{d}) $$
3. **Parallel Check**: If `denominator` is zero (or very close to zero), the ray is parallel to the triangle plane. Return false.
4. **Distance Check**: Compute $t = \text{numerator} / \text{denominator}$. If $t < \text{small\_t}$, the intersection is too close (behind the ray origin or self-intersecting). Return false.
5. **Intersection Point**: Compute the intersection point $\mathbf{p}$:
   $$ \mathbf{p} = \mathbf{o} + t \cdot \mathbf{d} $$
6. **Barycentric Coordinates**: Compute the barycentric coordinates $(\alpha, \beta, \gamma)$ of $\mathbf{p}$ relative to the triangle. This can be done by projecting $\mathbf{p}$ onto the triangle's local basis or using area ratios. For implementation simplicity, one common approach is:
   $$ \mathbf{e}_1 = \mathbf{v}_1 - \mathbf{v}_0 $$
   $$ \mathbf{e}_2 = \mathbf{v}_2 - \mathbf{v}_0 $$
   $$ \mathbf{v} = \mathbf{p} - \mathbf{v}_0 $$
   $$ \text{dot00} = \text{dot}(\mathbf{e}_1, \mathbf{e}_1) $$
   $$ \text{dot01} = \text{dot}(\mathbf{e}_1, \mathbf{e}_2) $$
   $$ \text{dot02} = \text{dot}(\mathbf{e}_1, \mathbf{v}) $$
   $$ \text{dot11} = \text{dot}(\mathbf{e}_2, \mathbf{e}_2) $$
   $$ \text{dot12} = \text{dot}(\mathbf{e}_2, \mathbf{v}) $$
   $$ \text{invDenom} = 1 / (\text{dot00} \cdot \text{dot11} - \text{dot01} \cdot \text{dot01}) $$
   $$ \alpha = (\text{dot11} \cdot \text{dot02} - \text{dot01} \cdot \text{dot12}) \cdot \text{invDenom} $$
   $$ \beta = (\text{dot00} \cdot \text{dot12} - \text{dot01} \cdot \text{dot02}) \cdot \text{invDenom} $$
   $$ \gamma = 1 - \alpha - \beta $$
7. **Inside Check**: Check if the barycentric coordinates are within the valid range, relaxed by `weight_tolerance`.
   $$ \text{if } \alpha \ge -\text{weight\_tolerance} \text{ and } \beta \ge -\text{weight\_tolerance} \text{ and } \gamma \ge -\text{weight\_tolerance} \text{ and } \alpha \le 1 + \text{weight\_tolerance} \text{ and } \beta \le 1 + \text{weight\_tolerance} \text{ and } \gamma \le 1 + \text{weight\_tolerance} $$
   Return true. Otherwise, return false.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_triangle_intersect(o, d, v0, v1, v2, small_t=1e-6, weight_tolerance=1e-4):
    # Step 1: Compute triangle normal
    e1 = v1 - v0
    e2 = v2 - v0
    n = normalize(cross(e1, e2))
    
    # Step 2: Ray-plane intersection parameters
    numerator = dot(n, v0 - o)
    denominator = dot(n, d)
    
    # Step 3: Parallel check
    if abs(denominator) < 1e-8:
        return False, 0.0
    
    # Step 4: Distance check
    t = numerator / denominator
    if t < small_t:
        return False, 0.0
    
    # Step 5: Intersection point
    p = o + t * d
    
    # Step 6: Barycentric coordinates
    v = p - v0
    dot00 = dot(e1, e1)
    dot01 = dot(e1, e2)
    dot02 = dot(e1, v)
    dot11 = dot(e2, e2)
    dot12 = dot(e2, v)
    
    invDenom = 1.0 / (dot00 * dot11 - dot01 * dot01)
    alpha = (dot11 * dot02 - dot01 * dot12) * invDenom
    beta = (dot00 * dot12 - dot01 * dot02) * invDenom
    gamma = 1.0 - alpha - beta
    
    # Step 7: Inside check with tolerance
    if (alpha >= -weight_tolerance and beta >= -weight_tolerance and gamma >= -weight_tolerance and
        alpha <= 1.0 + weight_tolerance and beta <= 1.0 + weight_tolerance and gamma <= 1.0 + weight_tolerance):
        return True, t
    else:
        return False, 0.0
```

## Subtopics

### [General Ray-Triangle Intersection (Plücker Coordinates)](909-general-ray-triangle-intersection-pl-cker-coordinates.md)
Uses Plücker coordinates to decouple intersection logic from explicit plane geometry, offering a different computational path for general rays.

### [Point-Triangle Distance](2776-point-triangle-distance.md)
Computes the closest point on a triangle to a query point, returning barycentric coordinates and squared distance, used for SDF evaluation and collision detection.

### [Triangle-Triangle Intersection Test](11175-triangle-triangle-intersection-test.md)
Determines if two triangles intersect using the Separating Axis Theorem (SAT), used for mesh collision detection.

### [GPU Mesh Voxelization via SAT Ray Casting](11666-gpu-mesh-voxelization-via-sat-ray-casting.md)
Converts a triangle mesh into a 3D voxel grid using SAT broad-phase and Möller-Trumbore intersection, used for brute-force voxelization.

## Operating characteristics
- **Time Complexity**: $O(1)$ per triangle. The number of operations is constant, involving a few vector cross products, dot products, and divisions.
- **Memory Complexity**: $O(1)$ auxiliary space.
- **Performance**: Highly efficient for single triangle tests. Performance scales linearly with the number of triangles tested. On GPUs, parallel execution across threads allows testing many triangles simultaneously [G].
- **Strengths**: Simple to implement, robust for most standard ray tracing scenarios, handles arbitrary triangle orientations.
- **Limitations**: Sensitive to numerical precision near triangle edges, which is mitigated by `weight_tolerance`. Not suitable for hierarchical acceleration structures without additional bounding volume checks.
- **When to Use**: When testing a ray against a single triangle or a small set of triangles.
- **When Not to Use**: When testing against large meshes; use acceleration structures like BVH or KD-trees instead.

## Background
The method is a fundamental building block in ray tracing and volume rendering, enabling the intersection of rays with triangular mesh geometry [S1, unofficial]. It is widely used in computer graphics for rendering 3D models, collision detection, and voxelization [S1, S2, S3, unofficial]. The specific implementation described here, using barycentric coordinates and a tolerance for edge cases, is common in practical applications to ensure robustness [S1, unofficial].

## Sources
[S1] lijiemark/floating-islands-generator, https://github.com/lijiemark/floating-islands-generator, fetched 2026-10-27 (unofficial)
