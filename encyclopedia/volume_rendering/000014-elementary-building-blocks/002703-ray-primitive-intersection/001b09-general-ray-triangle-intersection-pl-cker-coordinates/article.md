# General Ray-Triangle Intersection (Plücker Coordinates)
Also called: rayIntersectsTriangle (Ray)
## Overview
This method determines if a ray intersects a triangle in 3D space using Plücker coordinates to evaluate the winding order of the triangle edges relative to the ray [S1, unofficial]. It handles arbitrary ray directions and is distinct from barycentric approaches like Möller-Trumbore by relying on cross-product-based Plücker line coordinates rather than solving a linear system for barycentric weights [S1, unofficial].
## How it works
**Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, triangle normal $\mathbf{n}$, and triangle vertices $\mathbf{a}, \mathbf{b}, \mathbf{c}$ [S1, unofficial].
**Outputs:** Boolean indicating intersection [S1, unofficial].
**Data Structures:** `vec3f` (3-component floating-point vector), `float` (32-bit floating-point scalar) [G].
**Steps:**
1. Extract ray origin $\mathbf{o}$ and direction $\mathbf{d}$ [S1, unofficial].
2. Compute the dot product $\cos(\mathbf{n}, \mathbf{d}) = \mathbf{n} \cdot \mathbf{d}$ [S1, unofficial].
3. Calculate the intersection parameter $t = -(\mathbf{o} - \mathbf{a}) \cdot \mathbf{n} / (\mathbf{n} \cdot \mathbf{d})$ [S1, unofficial].
4. Translate triangle vertices to ray origin space: $\mathbf{a} \leftarrow \mathbf{a} - \mathbf{o}$, $\mathbf{b} \leftarrow \mathbf{b} - \mathbf{o}$, $\mathbf{c} \leftarrow \mathbf{c} - \mathbf{o}$ [S1, unofficial].
5. Compute edge Plücker coordinates: $s_x = \text{pluecker}(\mathbf{0}, \mathbf{d}, \mathbf{a}, \mathbf{b})$, $s_y = \text{pluecker}(\mathbf{0}, \mathbf{d}, \mathbf{b}, \mathbf{c})$, $s_z = \text{pluecker}(\mathbf{0}, \mathbf{d}, \mathbf{c}, \mathbf{a})$ [S1, unofficial].
6. Check winding order consistency: if $\min(s_x, s_y, s_z) \ge 0$ or $\max(s_x, s_y, s_z) \le 0$, return `true` [S1, unofficial].
**Formula:**
The source defines the generic Plücker coordinate calculation as $\text{pluecker}(\mathbf{a}_0, \mathbf{a}_1, \mathbf{b}_0, \mathbf{b}_1) = \text{dot}(\mathbf{a}_1 - \mathbf{a}_0, \text{cross}(\mathbf{b}_1, \mathbf{b}_0)) + \text{dot}(\mathbf{b}_1 - \mathbf{b}_0, \text{cross}(\mathbf{a}_1, \mathbf{a}_0))$ [S1, unofficial].
For the specific call $\text{pluecker}(\mathbf{0}, \mathbf{d}, \mathbf{a}, \mathbf{b})$, substituting $\mathbf{a}_0=\mathbf{0}$, $\mathbf{a}_1=\mathbf{d}$, $\mathbf{b}_0=\mathbf{a}$, $\mathbf{b}_1=\mathbf{b}$ simplifies the expression to $s_x = \text{dot}(\mathbf{d}, \text{cross}(\mathbf{b}, \mathbf{a}))$ [S1, unofficial]. The terms for $s_y$ and $s_z$ follow identically for edges $\mathbf{b} \to \mathbf{c}$ and $\mathbf{c} \to \mathbf{a}$ [S1, unofficial].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_triangle_intersection_plucker(ray_origin, ray_direction, tri_normal, tri_a, tri_b, tri_c):
    org = ray_origin
    dir = ray_direction
    cos_nd = dot(tri_normal, dir)
    if cos_nd == 0.0:
        return False
    t = -dot(org - tri_a, tri_normal) / cos_nd
    a = tri_a - org
    b = tri_b - org
    c = tri_c - org
    sx = dot(dir, cross(b, a))
    sy = dot(dir, cross(c, b))
    sz = dot(dir, cross(a, c))
    if min(sx, sy, sz) >= 0.0 or max(sx, sy, sz) <= 0.0:
        return True
    return False
```
**Parameters:**
* `tri_normal`: Precomputed triangle normal. Not given in the sources how it is derived, but required for the plane check [S1, unofficial].
**Edge Cases:**
* **Parallel Ray:** If $\mathbf{n} \cdot \mathbf{d} = 0$, the method returns `false` [S1, unofficial]. This ignores the degenerate case where the ray lies within the triangle's plane [S1, unofficial].
* **Precision:** Floating-point precision issues may affect the winding check boundaries ($\ge 0$ or $\le 0$) [G].
## Subtopics

### [Axis-Aligned Ray-Triangle Intersection (Plücker Coordinates)](911-axis-aligned-ray-triangle-intersection-pl-cker-coordinates.md)
An algorithm optimized for axis-aligned rays that uses Plücker coordinates to simplify cross products. Choose this variant when rays are strictly aligned with coordinate axes to reduce computational overhead.

### [Möller-Trumbore Ray-Triangle Intersection](1274-m-ller-trumbore-ray-triangle-intersection.md)
Computes intersection by solving for parametric distance and barycentric coordinates directly, avoiding explicit plane equation storage. Choose this variant when meshes lack precomputed planes or when handling arbitrary ray directions without axis-specific optimizations.

## Operating characteristics
* **Time Complexity:** $O(1)$ per triangle, requiring a fixed number of dot and cross products [S1, unofficial].
* **Memory Complexity:** $O(1)$, using only scalar and vector registers [G].
* **Strengths:** The Plücker winding test is robust for determining if a ray passes through the triangle's infinite prism [S1, unofficial].
* **Limitations:** Requires a precomputed triangle normal for the initial plane check [S1, unofficial]. The implementation ignores coplanar rays [S1, unofficial].
* **Performance:** Not given in the sources.
## Background
This method originates from implementations using Plücker coordinates for ray-triangle intersection found in the `isoext` repository [S1, unofficial].
## Sources
[S1] GuangyanCai/isoext, https://github.com/GuangyanCai/isoext, fetched 2026-10-27 (unofficial)
