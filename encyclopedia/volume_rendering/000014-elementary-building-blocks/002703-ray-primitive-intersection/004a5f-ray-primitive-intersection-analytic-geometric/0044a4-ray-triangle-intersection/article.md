# Ray-Triangle Intersection
Also called: Ray-triangle test, Möller-Trumbore (variant)

## Overview
Computes the intersection of a ray with a single triangle in 3D space, returning the intersection point or distance [S1, unofficial]. It is the fundamental primitive test for ray tracing against mesh geometry and is used to extract triangle meshes from binary density fields in volume rendering pipelines [S1, unofficial]. Unlike [Sphere Intersection](1620-sphere-intersection.md) which uses algebraic distance, or [Ray-Plane/Rectangle Intersection](6498-ray-plane-rectangle-intersection.md) which tests infinite or bounded planes, this method explicitly handles the triangular boundary constraints using barycentric coordinates.

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (typically normalized).
- Triangle vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$.
- Parameters: `small_t` (minimum distance threshold, e.g., $10^{-4}$), `weight_tolerance` (barycentric relaxation, e.g., $10^{-4}$) [S1, unofficial].

**Outputs:**
- Boolean indicating intersection.
- If intersecting: intersection point $\mathbf{p}$ or distance $t$.

**Data Structures:**
- Vectors $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$ are 3D points.
- Ray is defined by $\mathbf{o}$ and $\mathbf{d}$.

**Steps:**
1. Compute two edge vectors: $\mathbf{e}_1 = \mathbf{v}_1 - \mathbf{v}_0$ and $\mathbf{e}_2 = \mathbf{v}_2 - \mathbf{v}_0$.
2. Compute the triangle normal: $\mathbf{n} = \text{normalize}(\mathbf{e}_1 \times \mathbf{e}_2)$.
3. Calculate the denominator for the ray-plane intersection: $D = \mathbf{n} \cdot \mathbf{d}$.
4. Check if the ray is parallel to the plane. If $|D| < \epsilon$ (where $\epsilon$ is a small value like $10^{-8}$), return false [S1, unofficial].
5. Calculate the numerator for the ray-plane intersection: $N = \mathbf{n} \cdot (\mathbf{v}_0 - \mathbf{o})$.
6. Compute the intersection distance: $t = N / D$.
7. Check if the intersection is too close to the ray origin. If $t < \text{small\_t}$, return false [S1, unofficial].
8. Compute the intersection point: $\mathbf{p} = \mathbf{o} + t \mathbf{d}$.
9. Compute barycentric coordinates of $\mathbf{p}$ relative to the triangle. This can be done by projecting $\mathbf{p}$ onto the edge vectors or using the area method. Let the barycentric coordinates be $(\beta, \gamma)$.
10. Check if the point is inside the triangle. The point is inside if $\beta \ge -\text{weight\_tolerance}$, $\gamma \ge -\text{weight\_tolerance}$, and $\beta + \gamma \le 1 + \text{weight\_tolerance}$ [S1, unofficial].
11. If inside, return true and the intersection details. Otherwise, return false.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_triangle_intersect(o, d, v0, v1, v2, small_t=1e-4, weight_tolerance=1e-4):
    # Step 1: Edge vectors
    e1 = v1 - v0
    e2 = v2 - v0
    
    # Step 2: Normal
    n = normalize(cross(e1, e2))
    
    # Step 3: Denominator
    D = dot(n, d)
    
    # Step 4: Parallel check
    if abs(D) < 1e-8:
        return False, None
    
    # Step 5: Numerator
    N = dot(n, v0 - o)
    
    # Step 6: Distance
    t = N / D
    
    # Step 7: Near origin check
    if t < small_t:
        return False, None
    
    # Step 8: Intersection point
    p = o + t * d
    
    # Step 9: Barycentric coordinates
    # Using the area method or projection. 
    # Let's use the projection method for clarity.
    # We need to solve: p - v0 = beta * e1 + gamma * e2
    # This is a 2x2 system in the plane of the triangle.
    # A robust way is to use the cross products to find areas.
    
    # Vector from v0 to p
    v0p = p - v0
    
    # Compute barycentric coordinates using cross products
    # Area of triangle (v0, v1, v2) is proportional to |n|
    # Area of (p, v1, v2) corresponds to beta
    # Area of (v0, p, v2) corresponds to gamma
    
    # Note: The normal n might not be unit length if we didn't normalize in step 2, 
    # but we did. So areas are half the magnitude of cross products.
    
    # Beta: area of triangle (p, v1, v2) / area of (v0, v1, v2)
    # Gamma: area of triangle (v0, p, v2) / area of (v0, v1, v2)
    
    # Cross products
    c1 = cross(v1 - p, v2 - p)
    c2 = cross(v0 - p, v2 - p) # Note: order matters for sign
    
    # Actually, standard barycentric calculation:
    # beta = dot(cross(v1-p, v2-p), n) / dot(n, n) ? No, simpler:
    
    # Let's use the standard Möller-Trumbore logic for barycentrics if we want to be precise,
    # but the source describes a generic barycentric check.
    # Let's implement a simple projection-based barycentric check.
    
    # Solve for beta, gamma in: v0p = beta * e1 + gamma * e2
    # Dot with e1 and e2 to get a 2x2 system:
    # v0p . e1 = beta (e1 . e1) + gamma (e2 . e1)
    # v0p . e2 = beta (e1 . e2) + gamma (e2 . e2)
    
    dot00 = dot(e2, e2)
    dot01 = dot(e1, e2)
    dot02 = dot(e1, v0p)
    dot11 = dot(e1, e1)
    dot12 = dot(e2, v0p)
    
    denom = dot00 * dot11 - dot01 * dot01
    
    if abs(denom) < 1e-8:
        return False, None # Degenerate triangle
        
    invDenom = 1.0 / denom
    gamma = (dot11 * dot12 - dot01 * dot02) * invDenom
    beta = (dot00 * dot02 - dot01 * dot12) * invDenom
    
    # Step 10: Inside check
    if beta >= -weight_tolerance and gamma >= -weight_tolerance and (beta + gamma) <= 1.0 + weight_tolerance:
        return True, p
    else:
        return False, None
```

**Edge Cases and Pitfalls:**
- **Degenerate Triangles:** If the triangle vertices are collinear, the normal is zero or near-zero. The denominator in the barycentric calculation will be near-zero. Handle by checking the magnitude of the normal or the barycentric denominator [S1, unofficial].
- **Precision:** Floating-point errors can cause rays to miss triangles due to gaps. Using `weight_tolerance` helps mitigate this [S1, unofficial].
- **Self-Intersection:** Rays originating from a surface may intersect the same surface due to precision errors. Using `small_t` prevents this [S1, unofficial].

## Subtopics

### [General Ray-Triangle Intersection (Plücker Coordinates)](909-general-ray-triangle-intersection-pl-cker-coordinates.md)
Uses Plücker coordinates to evaluate winding order, distinct from barycentric approaches.

### [Point-Triangle Distance](2776-point-triangle-distance.md)
Computes the closest point on a triangle to a query point, used for SDF evaluation.

### [Triangle-Triangle Intersection Test](11175-triangle-triangle-intersection-test.md)
Determines if two triangles intersect, using the Separating Axis Theorem.

### [GPU Mesh Voxelization via SAT Ray Casting](11666-gpu-mesh-voxelization-via-sat-ray-casting.md)
Converts a triangle mesh into a 3D voxel grid using SAT and ray casting.

## Operating characteristics
- **Time Complexity:** $O(1)$ per triangle.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Simple to implement, fast for single triangle tests.
- **Limitations:** Inefficient for large scenes without acceleration structures. Sensitive to floating-point precision.
- **When to use:** When testing against a small number of triangles or as a leaf node in an acceleration structure.
- **When not to use:** For large meshes without spatial partitioning.

## Background
The method is a standard technique in computer graphics for ray tracing. It is used in various rendering pipelines, including those for volumetric clouds and dynamic world generation [S1, unofficial]. The specific implementation details, such as the use of barycentric coordinates and tolerance parameters, are derived from practical implementations in open-source projects [S1, unofficial].

## Sources
[S1] lijiemark/floating-islands-generator, https://github.com/lijiemark/floating-islands-generator, fetched 2026-10-27 (unofficial)
