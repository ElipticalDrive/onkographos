# Triangle Coverage Test


## Overview
The Triangle Coverage Test verifies whether a triangle $T$ is completely covered by three spheres centered at its vertices, with radii determined by a scalar field function $f$ evaluated at those vertices [S1, S2]. It solves the problem of ensuring that a discrete sampling of an implicit volume is dense enough to guarantee no gaps exist between the approximating spheres, which is critical for sphere carving algorithms [S1, S2].

## How it works
**Inputs and Outputs:**
- Inputs: Triangle $T$ defined by three vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2 \in \mathbb{R}^3$, and a scalar field function $f: \mathbb{R}^3 \to \mathbb{R}$ [S1, S2].
- Output: Boolean `True` if the triangle is completely covered by the spheres; `False` otherwise [S1, S2].

**Data Structures:**
- Vertices: Array of 3 vectors $\mathbf{v}_i$.
- Radii: Array of 3 floats $r_i = f(\mathbf{v}_i)$.
- Circles: Implicitly defined by center $\mathbf{c}_i = \mathbf{v}_i$ and radius $r_i$ in the plane of $T$ [S1, S2].

**Numbered Steps:**
1. Evaluate radii $r_i = f(\mathbf{v}_i)$ for $i \in \{0, 1, 2\}$ [S1, S2].
2. Check disjointness: If any pair of circles is disjoint, the test fails and returns `False` [S1, S2].
3. Check intersection points: For each pair of circles, compute their intersection points. If any intersection point lies inside $T$ and outside the third circle, the test fails and returns `False` [S1, S2].
4. Return `True` if neither disjointness nor uncovered intersection points are found [S1, S2].

**Formulas:**
- Sphere coverage condition: $\forall \mathbf{p} \in T, \exists i \in \{0, 1, 2\}, \|\mathbf{p} - \mathbf{v}_i\| < f(\mathbf{v}_i)$, where $\mathbf{p}$ is any point on the triangle, $\mathbf{v}_i$ are the vertices, and $f(\mathbf{v}_i)$ are the sphere radii [S1, S2].
- Disjointness condition: $\|\mathbf{v}_i - \mathbf{v}_j\| > r_i + r_j$ [S1, S2].

**Parameters:**
- The test relies on the property that the initial volume sampling is dense enough to create sphere intersections [S1, S2]. Typical values are not specified in the sources.

**Edge Cases and Pitfalls:**
- Degenerate triangles (collinear vertices) have zero area and require explicit handling [G].
- Floating-point precision errors in intersection calculations may cause false negatives; epsilon thresholds should be used for comparisons [G].
- The test specifically checks for intersection points of two circles that lie inside the triangle but outside the third circle [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def triangle_coverage_test(v0, v1, v2, f):
    # Step 1: Evaluate radii
    r = [f(v0), f(v1), f(v2)]
    verts = [v0, v1, v2]

    # Helper: Euclidean distance
    def dist(a, b):
        return math.sqrt(sum((a[i] - b[i])**2 for i in range(3)))

    # Step 2: Check disjointness
    for i in range(3):
        for j in range(i + 1, 3):
            if dist(verts[i], verts[j]) > r[i] + r[j]:
                return False

    # Compute triangle normal for 3D circle intersection [G]
    v01 = [verts[1][k] - verts[0][k] for k in range(3)]
    v02 = [verts[2][k] - verts[0][k] for k in range(3)]
    normal = [
        v01[1]*v02[2] - v01[2]*v02[1],
        v01[2]*v02[0] - v01[0]*v02[2],
        v01[0]*v02[1] - v01[1]*v02[0]
    ]
    n_len = math.sqrt(sum(x**2 for x in normal))
    if n_len < 1e-9:
        return True # Degenerate triangle, assume covered [G]
    normal = [x / n_len for x in normal]

    # Helper: Point-in-triangle using barycentric coordinates [G]
    def point_in_triangle(p):
        v0v1 = [verts[1][k] - verts[0][k] for k in range(3)]
        v0v2 = [verts[2][k] - verts[0][k] for k in range(3)]
        v0p = [p[k] - verts[0][k] for k in range(3)]
        d00 = sum(v0v1[k]*v0v1[k] for k in range(3))
        d01 = sum(v0v1[k]*v0v2[k] for k in range(3))
        d11 = sum(v0v2[k]*v0v2[k] for k in range(3))
        d20 = sum(v0p[k]*v0v1[k] for k in range(3))
        d21 = sum(v0p[k]*v0v2[k] for k in range(3))
        denom = d00 * d11 - d01 * d01
        if abs(denom) < 1e-9:
            return False
        v = (d11 * d20 - d01 * d21) / denom
        w = (d00 * d21 - d01 * d20) / denom
        u = 1.0 - v - w
        return u >= 0 and v >= 0 and w >= 0

    # Helper: Circle-circle intersection in 3D [G]
    def circle_intersection(p1, r1, p2, r2, n):
        d = dist(p1, p2)
        if d > r1 + r2 or d < abs(r1 - r2):
            return []
        if d == 0:
            return []
        a = (r1**2 - r2**2 + d**2) / (2 * d)
        h_sq = r1**2 - a**2
        if h_sq < 0:
            return []
        h = math.sqrt(h_sq)
        p2_vec = [p2[k] - p1[k] for k in range(3)]
        p_base = [p1[k] + a * p2_vec[k] / d for k in range(3)]
        u = [p2_vec[k] / d for k in range(3)]
        v = [n[1]*u[2] - n[2]*u[1], n[2]*u[0] - n[0]*u[2], n[0]*u[1] - n[1]*u[0]]
        v_len = math.sqrt(sum(x**2 for x in v))
        if v_len < 1e-9:
            return []
        v = [x / v_len for x in v]
        return [[p_base[k] + h * v[k] for k in range(3)],
                [p_base[k] - h * v[k] for k in range(3)]]

    # Step 3: Check intersection points
    for i in range(3):
        for j in range(i + 1, 3):
            k = 3 - i - j
            pts = circle_intersection(verts[i], r[i], verts[j], r[j], normal)
            for p in pts:
                if point_in_triangle(p):
                    if dist(p, verts[k]) > r[k]:
                        return False

    return True
```

## Subtopics

### [Unsigned Triangle Distance](3587-unsigned-triangle-distance.md)
Computes the distance from a point to a triangle without sign. Use when you need metric distance rather than coverage.

### [Mesh SDF Sampling](7159-mesh-sdf-sampling.md)
Samples SDF values on a mesh. Use when generating the input $f$ for the coverage test.

### [Triangle-Box Intersection Voxelization](7408-triangle-box-intersection-voxelization.md)
Voxelizes a triangle. Use when rasterizing rather than checking sphere coverage.

### [Barycentric Coordinate Calculation](11142-barycentric-coordinate-calculation.md)
Computes barycentric coords. Use as a helper for point-in-triangle tests within the coverage test.

### [Zero Set Initialization of Meshes](11845-zero-set-initialization-of-meshes.md)
Initializes SDF zero sets. Use when setting up the initial volume for sphere carving.

## Operating characteristics
- Time complexity: Not given in the sources.
- Memory complexity: Not given in the sources.
- Measured performance: Not given in the sources.
- Scaling: Not given in the sources.
- Strengths: Efficiently verifies volume completeness without dense sampling [S1, S2].
- Limitations: Relies on dense initial sampling; fails if spheres are too small or sparse [S1, S2].
- When to use: Sphere carving algorithms, implicit volume reconstruction [S1, S2].
- When not to use: When spheres are not dense enough or when exact SDF values are not available [S1, S2].

## Background
The Triangle Coverage Test was introduced by Cédric Zanni in 2024 as part of the Sphere Carving algorithm for bounding volumes of signed distance fields [S1, S2]. It addresses the need to ensure that the discrete sampling of an implicit volume is dense enough to create sphere intersections, preventing gaps in the reconstructed volume [S1, S2]. The method is based on the observation that a triangle is not covered if either one circle is disjoint from the others or an intersection point of two circles lies inside the triangle but outside the third [S1, S2].

## Sources
[S1] Sphere Carving: Bounding Volumes for Signed Distance Fields, https://doi.org/10.1145/3730845, fetched 2026-10-27
[S2] Sphere Carving: Bounding Volumes for Signed Distance ..., https://research.adobe.com/publication/sphere-carving-bounding-volumes-for-signed-distance-fields/, fetched 2026-11-05
