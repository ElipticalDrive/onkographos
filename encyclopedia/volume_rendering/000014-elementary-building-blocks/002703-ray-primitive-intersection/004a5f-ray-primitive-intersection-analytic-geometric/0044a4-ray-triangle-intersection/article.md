# Ray-Triangle Intersection
Also called: Möller-Trumbore algorithm (specific variant), Triangle intersection test

## Overview
Computes the intersection of a ray with a single triangle in 3D space, returning the intersection point or distance if one exists [S1, unofficial]. It solves the fundamental ray-primitive intersection problem for mesh geometry in ray tracing and volume rendering [S1, unofficial]. Unlike [Sphere Intersection](1620-sphere-intersection.md) which relies on quadratic solutions for curved surfaces, this method uses linear algebra to solve for barycentric coordinates directly, making it efficient for planar primitives [S1, unofficial]. It is distinct from [Point-Triangle Distance](2776-point-triangle-distance.md) as it requires a ray direction and returns a scalar distance $t$ rather than a minimum Euclidean distance metric [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ (vec3)
- Ray direction $\mathbf{d}$ (vec3, normalized)
- Triangle vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$ (vec3)
- Parameters: `small_t` (float, default $10^{-6}$), `weight_tolerance` (float, default $10^{-4}$) [S1, unofficial]

**Outputs:**
- Boolean `hit`: true if intersection exists.
- Float `t`: distance from ray origin to intersection point.
- Float `u, v`: barycentric coordinates (optional, for shading).

**Data Structures:**
- Vectors are 3-component floating-point arrays.
- No complex data structures; operates on local vertex data.

**Steps:**
1.  **Compute Edge Vectors:**
    Calculate two edge vectors of the triangle:
    $$ \mathbf{e}_1 = \mathbf{v}_1 - \mathbf{v}_0 $$
    $$ \mathbf{e}_2 = \mathbf{v}_2 - \mathbf{v}_0 $$

2.  **Compute Normal and Determinant:**
    Calculate the triangle normal $\mathbf{n}$ and the determinant $D$ which relates to the angle between the ray and the triangle plane:
    $$ \mathbf{n} = \mathbf{e}_1 \times \mathbf{e}_2 $$
    $$ D = \mathbf{n} \cdot \mathbf{d} $$
    [S1, unofficial]

3.  **Check Parallelism and Back-face Culling:**
    If $|D| < \epsilon$ (where $\epsilon$ is a small machine epsilon, e.g., $10^{-8}$), the ray is parallel to the triangle plane; return false [S1, unofficial].
    Optionally, if back-face culling is enabled, check if $\mathbf{n} \cdot \mathbf{d} > 0$ (assuming outward normals); if so, return false.

4.  **Compute Intersection Distance ($t$):**
    Calculate the vector from $\mathbf{v}_0$ to the ray origin:
    $$ \mathbf{v}_0\mathbf{o} = \mathbf{o} - \mathbf{v}_0 $$
    Compute $t$:
    $$ t = \frac{\mathbf{n} \cdot \mathbf{v}_0\mathbf{o}}{D} $$
    [S1, unofficial]

5.  **Check Minimum Distance:**
    If $t < \text{small\_t}$, the intersection is too close (self-intersection or shadow acne); return false [S1, unofficial].

6.  **Compute Barycentric Coordinates ($u, v$):**
    Calculate the barycentric coordinates of the intersection point relative to the triangle:
    $$ \mathbf{P} = \mathbf{o} + t\mathbf{d} $$
    $$ \mathbf{v}_0\mathbf{P} = \mathbf{P} - \mathbf{v}_0 $$
    $$ u = \frac{(\mathbf{v}_0\mathbf{P} \times \mathbf{e}_2) \cdot \mathbf{n}}{\mathbf{n} \cdot \mathbf{n}} $$
    $$ v = \frac{(\mathbf{e}_1 \times \mathbf{v}_0\mathbf{P}) \cdot \mathbf{n}}{\mathbf{n} \cdot \mathbf{n}} $$
    [S1, unofficial]

7.  **Check Barycentric Bounds:**
    The point is inside the triangle if:
    $$ u \ge -\text{weight\_tolerance} \quad \text{and} \quad v \ge -\text{weight\_tolerance} \quad \text{and} \quad u + v \le 1 + \text{weight\_tolerance} $$
    [S1, unofficial]
    If all conditions are met, return true with $t, u, v$. Otherwise, return false.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_triangle_intersect(o, d, v0, v1, v2, small_t=1e-6, weight_tolerance=1e-4):
    # Step 1: Edge vectors
    e1 = v1 - v0
    e2 = v2 - v0
    
    # Step 2: Normal and determinant
    n = cross(e1, e2)
    D = dot(n, d)
    
    # Step 3: Check parallelism
    if abs(D) < 1e-8:
        return False, None, None, None
        
    # Step 4: Compute t
    v0o = o - v0
    t = dot(n, v0o) / D
    
    # Step 5: Check min distance
    if t < small_t:
        return False, None, None, None
        
    # Step 6: Compute barycentric coordinates
    P = o + d * t
    v0P = P - v0
    n_dot_n = dot(n, n)
    
    u = dot(cross(v0P, e2), n) / n_dot_n
    v = dot(cross(e1, v0P), n) / n_dot_n
    
    # Step 7: Check bounds
    if u >= -weight_tolerance and v >= -weight_tolerance and (u + v) <= 1 + weight_tolerance:
        return True, t, u, v
    else:
        return False, None, None, None
```

## Subtopics

### [General Ray-Triangle Intersection (Plücker Coordinates)](909-general-ray-triangle-intersection-pl-cker-coordinates.md)
Uses Plücker coordinates to decouple intersection logic from explicit plane geometry, offering a different computational path for general rays.

### [Point-Triangle Distance](2776-point-triangle-distance.md)
Computes the closest point on a triangle to a query point, returning barycentric coordinates and squared distance, used for SDF evaluation and collision detection.

### [Triangle-Triangle Intersection Test](11175-triangle-triangle-intersection-test.md)
Determines if two triangles intersect using the Separating Axis Theorem (SAT), used for mesh collision detection.

### [GPU Mesh Voxelization via SAT Ray Casting](11666-gpu-mesh-voxelization-via-sat-ray-casting.md)
Converts triangle meshes to voxel grids using SAT broad-phase and Möller-Trumbore intersection tests.

## Operating characteristics
- **Time Complexity:** $O(1)$ per triangle. Involves a constant number of vector operations (cross, dot, add, sub).
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Performance:** Highly efficient for single triangle tests. Performance degrades linearly with the number of triangles in a naive brute-force approach.
- **Strengths:** Simple implementation, numerically stable with tolerance parameters, widely supported.
- **Limitations:** Does not handle degenerate triangles (zero area) gracefully without explicit checks. Precision issues may arise with very large or very small triangles if not normalized.
- **When to use:** Single triangle intersection tests, ray tracing against mesh geometry, voxelization [S1, unofficial].
- **When not to use:** Large-scale scene traversal (use BVH/SAH instead), curved surfaces (use Sphere Intersection) [S1, unofficial].

## Background
Originated from the need for efficient ray-triangle intersection in ray tracing algorithms. The specific implementation details described here are derived from practical implementations in volumetric cloud rendering and chunk-based world generation systems [S1, unofficial]. The use of barycentric coordinates and tolerance parameters is a standard technique to prevent gaps between adjacent triangles in mesh rendering [S1, unofficial].

## Sources
[S1] lijiemark/floating-islands-generator, https://github.com/lijiemark/floating-islands-generator, fetched 2026-10-27 (unofficial)
