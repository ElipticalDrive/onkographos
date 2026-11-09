# Möller-Trumbore Ray-Triangle Intersection

Also called: Möller-Trumbore, MT Algorithm, Ray-Triangle Intersection (Möller-Trumbore)

## Overview
The Möller-Trumbore algorithm computes the intersection of a ray with a triangle by solving for the parametric distance $t$ and barycentric coordinates $(u, v)$ directly, avoiding the explicit calculation or storage of the triangle's plane equation [S7, unofficial]. It builds on [General Ray-Triangle Intersection (Plücker Coordinates)](909-general-ray-triangle-intersection-pl-cker-coordinates.md) by using a change of basis to decouple the intersection test from plane geometry, offering significant memory savings for meshes without precomputed planes [S7, unofficial].

## How it works
**Inputs:** Ray origin $\mathbf{O}$, ray direction $\mathbf{D}$, and triangle vertices $\mathbf{V}_1, \mathbf{V}_2, \mathbf{V}_3$ [S1, S5, unofficial].
**Outputs:** Boolean indicating intersection, and if true, the parametric distance $t$ along the ray and barycentric coordinates $u, v$ [S2, unofficial].

**Data Structures:**
*   Vectors $\mathbf{O}, \mathbf{D}, \mathbf{V}_1, \mathbf{V}_2, \mathbf{V}_3$: 3D floating-point vectors (typically `double` or `float`) [S2, S5, unofficial].
*   Scalars $t, u, v, \text{det}, \text{inv\_det}$: 32-bit or 64-bit floats [S2, S5, unofficial].

**Steps:**
1.  Compute edge vectors:
    $$ \mathbf{e}_1 = \mathbf{V}_2 - \mathbf{V}_1 $$
    $$ \mathbf{e}_2 = \mathbf{V}_3 - \mathbf{V}_1 $$
2.  Compute the vector from $\mathbf{V}_1$ to the ray origin:
    $$ \mathbf{T} = \mathbf{O} - \mathbf{V}_1 $$
3.  Compute the cross product of the ray direction and the second edge:
    $$ \mathbf{pvec} = \mathbf{D} \times \mathbf{e}_2 $$
4.  Compute the determinant of the system:
    $$ \text{det} = \mathbf{e}_1 \cdot \mathbf{pvec} $$
5.  Check for parallelism. If $|\text{det}| < \epsilon$, the ray is parallel to the triangle plane and does not intersect (or lies in it) [S6, unofficial].
    *   Typical $\epsilon$: $10^{-6}$ in [S1, S2, S5, unofficial] and $10^{-7}$ in [S6, unofficial] (some implementations use $10^{-8}$ [S3, unofficial]).
6.  Compute the inverse determinant:
    $$ \text{inv\_det} = 1.0 / \text{det} $$
7.  Compute the first barycentric coordinate $u$:
    $$ u = (\mathbf{T} \cdot \mathbf{pvec}) \times \text{inv\_det} $$
    If $u < 0.0$ or $u > 1.0$, the intersection point is outside the triangle edge [S1, unofficial]. Return false.
8.  Compute the cross product of $\mathbf{T}$ and the first edge:
    $$ \mathbf{qvec} = \mathbf{T} \times \mathbf{e}_1 $$
9.  Compute the second barycentric coordinate $v$:
    $$ v = (\mathbf{D} \cdot \mathbf{qvec}) \times \text{inv\_det} $$
    If $v < 0.0$ or $u + v > 1.0$, the intersection point is outside the triangle [S1, unofficial]. Return false.
10. Compute the parametric distance $t$:
    $$ t = (\mathbf{e}_2 \cdot \mathbf{qvec}) \times \text{inv\_det} $$
    If $t < 0.0$, the intersection is behind the ray origin [G]. Return false.
11. Return true with $t, u, v$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def muller_trumbore(O, D, V1, V2, V3, epsilon=1e-7):
    # 1. Edge vectors
    e1 = V2 - V1
    e2 = V3 - V1
    
    # 2. Vector from V1 to ray origin
    T = O - V1
    
    # 3. Cross product D x e2
    pvec = cross(D, e2)
    
    # 4. Determinant
    det = dot(e1, pvec)
    
    # 5. Check parallelism
    if abs(det) < epsilon:
        return False, None, None, None
    
    # 6. Inverse determinant
    inv_det = 1.0 / det
    
    # 7. Barycentric coordinate u
    u = dot(T, pvec) * inv_det
    if u < 0.0 or u > 1.0:
        return False, None, None, None
    
    # 8. Cross product T x e1
    qvec = cross(T, e1)
    
    # 9. Barycentric coordinate v
    v = dot(D, qvec) * inv_det
    if v < 0.0 or (u + v) > 1.0:
        return False, None, None, None
    
    # 10. Parametric distance t
    t = dot(e2, qvec) * inv_det
    
    # 11. Check if behind ray origin
    if t < 0.0:
        return False, None, None, None
        
    return True, t, u, v
```

**Edge Cases and Pitfalls:**
*   **Parallel Rays:** If $\text{det}$ is near zero, division by zero occurs. The epsilon check prevents this [S1, S2, S5, S6, unofficial].
*   **Degenerate Triangles:** If vertices are collinear, $\mathbf{e}_1$ and $\mathbf{e}_2$ are parallel, making $\text{det}$ zero. The algorithm correctly rejects these as parallel/non-intersecting [S1, S6, unofficial].
*   **Back-Facing:** The algorithm returns intersections for both front and back faces. The caller must check the dot product of the triangle normal and ray direction to cull back-faces if desired [S6, unofficial].
*   **Precision:** Using a fixed epsilon is standard, but for very large or very small scenes, relative epsilon based on triangle size may be more robust [G]. Precision issues can arise with extremely thin triangles or rays nearly parallel to the plane if epsilon is not chosen carefully [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per triangle [G]. It involves a constant number of vector operations (subtractions, cross products, dot products) and one division [G].
*   **Memory Complexity:** $O(1)$ auxiliary space [G]. No plane equation needs to be precomputed or stored, saving memory compared to plane-based methods [S7, unofficial].
*   **Performance:** It is widely considered the fastest general-purpose ray-triangle intersection routine for meshes without precomputed plane equations [S7, unofficial].
*   **Strengths:** Minimal storage, fast execution, numerically stable with epsilon checks.
*   **Limitations:** Does not handle back-face culling internally; requires caller to check normal if needed. Precision issues can arise with extremely thin triangles or rays nearly parallel to the plane if epsilon is not chosen carefully [S2, unofficial].

## Background
The algorithm was presented by Tomas Möller and Ben Trumbore in the paper "Fast, Minimum Storage Ray/Triangle Intersection" (cited as 1997 in implementation notes [S1, unofficial], published in 2005 [S7, unofficial]). It was developed to provide a clean, efficient alternative to methods requiring plane equation computation, leveraging a change of basis to solve for barycentric coordinates directly [S7, unofficial]. It is a fundamental building block in ray tracing and volume rendering for determining ray-primitive intersections [S1, S4, S5, unofficial].

## Sources
[S1] idealab-isu/GPView, https://github.com/idealab-isu/GPView, fetched 2026-10-26 (unofficial)
[S2] Mesh Voxelization - GitHub, https://github.com/davidstutz/mesh-voxelization, fetched 2026-10-26 (unofficial)
[S3] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S4] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
[S5] abhishektrip/SliceBasedVolumeRendering, https://github.com/abhishektrip/SliceBasedVolumeRendering, fetched 2026-10-27 (unofficial)
[S6] migaelstrydom/VoxelPhase, https://github.com/migaelstrydom/VoxelPhase, fetched 2026-10-28 (unofficial)
[S7] (PDF) Axis‐Normalized Ray‐Box Intersection - ResearchGate, https://www.researchgate.net/publication/390921688_Axis-Normalized_Ray-Box_Intersection, fetched 2026-10-30 (unofficial)
