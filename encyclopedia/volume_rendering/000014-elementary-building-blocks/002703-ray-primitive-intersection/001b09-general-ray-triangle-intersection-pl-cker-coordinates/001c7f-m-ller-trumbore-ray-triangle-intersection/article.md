# Möller-Trumbore Ray-Triangle Intersection

Also called: MT Algorithm, Möller-Trumbore, Ray-Triangle Intersection, SDF Triangle Distance and Ray Intersection, TriRayIntersectCUDA, TriRayIntersection, hitTriangle, udTriangle

## Overview
The Möller-Trumbore algorithm computes the intersection of a ray with a triangle defined by three vertices, returning the intersection distance $t$ and barycentric coordinates $(u, v)$ [S1, S4, unofficial]. It solves for the intersection point by transforming the ray origin and direction into a coordinate system defined by the triangle's edges, avoiding the explicit computation or storage of the triangle's plane equation [S4, unofficial]. This method is distinct from its siblings, such as the Plücker coordinate approach, by relying on Cramer's rule applied to the barycentric formulation, offering a balance of speed and minimal memory overhead [S4, unofficial]. It is particularly noted as the fastest routine for triangles that do not have precomputed plane equations [S4, unofficial].

## How it works
**Inputs:**
- Ray origin $O$ (float3).
- Ray direction $D$ (float3).
- Triangle vertices $V_0, V_1, V_2$ (float3).
- Optional: Epsilon threshold for numerical stability (default $10^{-6}$) [S1, unofficial].

**Outputs:**
- Intersection distance $t$ (float).
- Barycentric coordinates $u, v$ (float).
- Boolean hit status.

**Data Structures:**
- The algorithm operates on standard 3D vectors. No persistent data structures are required beyond the input vertices. In implementations storing triangles in textures (e.g., CUDA), vertices are often packed as 3 consecutive `float3` entries (9 floats total per triangle) [S3, unofficial].

**Steps:**

1.  **Compute Edge Vectors:**
    Calculate two edge vectors originating from $V_0$:
    $$ e_1 = V_1 - V_0 $$
    $$ e_2 = V_2 - V_0 $$

2.  **Compute Helper Vector P:**
    Calculate the cross product of the ray direction $D$ and edge $e_2$:
    $$ P = D \times e_2 $$

3.  **Compute Determinant:**
    Calculate the dot product of edge $e_1$ and vector $P$:
    $$ det = e_1 \cdot P $$

4.  **Check for Parallelism:**
    If $|det| < \epsilon$ (where $\epsilon$ is a small constant, typically $10^{-6}$ or $10^{-8}$), the ray is parallel to the triangle plane. Return no hit [S1, S2, unofficial].
    *Note:* If $det$ is very close to zero, the ray lies in or is parallel to the plane. The algorithm typically returns false/no hit in this case [S1, S2, unofficial].

5.  **Compute Inverse Determinant:**
    $$ inv\_det = 1.0 / det $$

6.  **Compute Vector T:**
    Calculate the vector from $V_0$ to the ray origin $O$:
    $$ T = O - V_0 $$

7.  **Compute Barycentric Coordinate u:**
    Calculate the dot product of $T$ and $P$:
    $$ u = (T \cdot P) \cdot inv\_det $$
    If $u < 0$ or $u > 1$, the intersection point lies outside the triangle's edge boundaries. Return no hit [S1, S2, unofficial].

8.  **Compute Helper Vector Q:**
    Calculate the cross product of $T$ and $e_1$:
    $$ Q = T \times e_1 $$

9.  **Compute Barycentric Coordinate v:**
    Calculate the dot product of the ray direction $D$ and vector $Q$:
    $$ v = (D \cdot Q) \cdot inv\_det $$
    If $v < 0$ or $u + v > 1$, the intersection point lies outside the triangle. Return no hit [S1, S2, unofficial].

10. **Compute Intersection Distance t:**
    Calculate the dot product of edge $e_2$ and vector $Q$:
    $$ t = (e_2 \cdot Q) \cdot inv\_det $$
    If $t < \epsilon$ (assuming the ray direction points outward from the origin and we only care about forward intersections), the intersection is behind the ray origin. Return no hit [G].

11. **Result:**
    The ray intersects the triangle at distance $t$. The intersection point $I$ is $O + tD$. The barycentric coordinates are $(u, v)$. The third barycentric coordinate is $w = 1 - u - v$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def moller_trumbore(ray_origin, ray_direction, v0, v1, v2, epsilon=1e-6):
    """
    Computes ray-triangle intersection using the Möller-Trumbore algorithm.
    
    Args:
        ray_origin: (float, float, float) Ray origin O.
        ray_direction: (float, float, float) Ray direction D.
        v0, v1, v2: (float, float, float) Triangle vertices.
        epsilon: float, threshold for parallelism check.
        
    Returns:
        tuple: (hit: bool, t: float, u: float, v: float)
    """
    # Step 1: Compute edge vectors
    e1 = (v1[0] - v0[0], v1[1] - v0[1], v1[2] - v0[2])
    e2 = (v2[0] - v0[0], v2[1] - v0[1], v2[2] - v0[2])
    
    # Step 2: Compute helper vector P = D x e2
    pvec = (
        ray_direction[1] * e2[2] - ray_direction[2] * e2[1],
        ray_direction[2] * e2[0] - ray_direction[0] * e2[2],
        ray_direction[0] * e2[1] - ray_direction[1] * e2[0]
    )
    
    # Step 3: Compute determinant det = e1 . P
    det = e1[0] * pvec[0] + e1[1] * pvec[1] + e1[2] * pvec[2]
    
    # Step 4: Check for parallelism
    if abs(det) < epsilon:
        return (False, 0.0, 0.0, 0.0)
    
    # Step 5: Compute inverse determinant
    inv_det = 1.0 / det
    
    # Step 6: Compute vector T = O - V0
    tvec = (
        ray_origin[0] - v0[0],
        ray_origin[1] - v0[1],
        ray_origin[2] - v0[2]
    )
    
    # Step 7: Compute barycentric coordinate u
    u = (tvec[0] * pvec[0] + tvec[1] * pvec[1] + tvec[2] * pvec[2]) * inv_det
    
    # Check if u is outside [0, 1]
    if u < 0.0 or u > 1.0:
        return (False, 0.0, 0.0, 0.0)
    
    # Step 8: Compute helper vector Q = T x e1
    qvec = (
        tvec[1] * e1[2] - tvec[2] * e1[1],
        tvec[2] * e1[0] - tvec[0] * e1[2],
        tvec[0] * e1[1] - tvec[1] * e1[0]
    )
    
    # Step 9: Compute barycentric coordinate v
    v = (ray_direction[0] * qvec[0] + ray_direction[1] * qvec[1] + ray_direction[2] * qvec[2]) * inv_det
    
    # Check if v is outside [0, 1] and u+v > 1
    if v < 0.0 or (u + v) > 1.0:
        return (False, 0.0, 0.0, 0.0)
    
    # Step 10: Compute intersection distance t
    t = (e2[0] * qvec[0] + e2[1] * qvec[1] + e2[2] * qvec[2]) * inv_det
    
    # Check if intersection is behind the ray origin
    if t < epsilon:
        return (False, 0.0, 0.0, 0.0)
    
    return (True, t, u, v)
```

**Edge Cases and Pitfalls:**
- **Parallel Rays:** If the ray is parallel to the triangle plane, the determinant is zero. This must be checked explicitly to avoid division by zero [S1, S2, unofficial].
- **Degenerate Triangles:** If the triangle vertices are collinear, the area is zero, and the determinant will be zero (or near zero). The parallelism check handles this [S1, unofficial].
- **Precision:** Using a small epsilon for comparisons is crucial for numerical stability, especially when the ray is nearly parallel to the triangle [S1, unofficial].
- **Winding Order:** The algorithm assumes a specific winding order (typically counter-clockwise) for the normal direction. If the ray direction and triangle normal are aligned such that the dot product is negative, the ray hits the back face. The algorithm as described above handles both front and back faces depending on the sign of $det$, but typically only forward intersections ($t > 0$) are considered [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per triangle. The algorithm performs a fixed number of arithmetic operations (cross products, dot products, divisions).
- **Memory Complexity:** $O(1)$. No additional memory is required beyond local variables. This is a key advantage over methods that require precomputed plane equations [S4, unofficial].
- **Performance:** It is considered one of the fastest ray-triangle intersection routines, particularly for meshes without precomputed plane equations [S4, unofficial].
- **Strengths:** Minimal storage requirements, clean implementation, no need for plane equation computation [S4, unofficial].
- **Limitations:** Sensitive to numerical precision in degenerate cases (parallel rays, degenerate triangles) [S1, S2, unofficial]. Does not handle back-face culling inherently without checking the sign of $det$ or $t$ [S3, unofficial].

## Background
The Möller-Trumbore algorithm was published by Tomas Möller and Ben Trumbore in 1997 in the article "Fast, Minimum Storage Ray-Triangle Intersection" [S4, unofficial]. It is widely used in ray tracing and volume rendering applications due to its efficiency and low memory footprint [S1, S3, unofficial]. It is a standard component in many graphics libraries and renderers [S1, unofficial].

## Sources

[S1] idealab-isu/GPView, https://github.com/idealab-isu/GPView, fetched 2026-10-26 (unofficial)
[S2] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S3] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
[S4] (PDF) Axis‐Normalized Ray‐Box Intersection - ResearchGate, https://www.researchgate.net/publication/390921688_Axis-Normalized_Ray-Box_Intersection, fetched 2026-10-30 (unofficial)
