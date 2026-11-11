# Point-Triangle Distance

Also called: Closest Point on Triangle, Point to Triangle Distance, closestPointOnTriangleToPoint, pointToTriangleDistSqr

## Overview
Computes the closest point on a triangle $[v_0, v_1, v_2]$ to a query point $p$, returning the barycentric coordinates of that point and optionally the squared distance [S1][S2, unofficial]. It solves the problem of finding the minimum Euclidean distance between a point and a triangular primitive, which is a fundamental operation for Signed Distance Field (SDF) evaluation, collision detection, and mesh voxelization [S1][S2, S3, S4, unofficial]. It sets itself apart from [Ray-Triangle Intersection](11312-ray-triangle-intersection.md) by not requiring a ray direction or parameter $t$; instead, it projects the point onto the triangle's Voronoi regions to determine if the closest feature is the interior, an edge, or a vertex [S1][S2, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**:
  - $v_0, v_1, v_2$: The three vertices of the triangle, represented as 3D vectors ($\mathbb{R}^3$) [S1][S2, unofficial].
  - $p$: The query point, represented as a 3D vector ($\mathbb{R}^3$) [S1][S2, unofficial].
- **Outputs**:
  - $t_0, t_1$: Barycentric coordinates such that the closest point $c = v_0 + t_0(v_1 - v_0) + t_1(v_2 - v_0)$ [S1].
  - $d^2$: The squared Euclidean distance $\|p - c\|^2$ [S1][S2, unofficial].

**Data Structures**
No complex data structures are required. The algorithm operates on scalar values derived from dot products of the input vectors.

**Steps**
The method uses Voronoi-region decomposition to determine which region of the triangle (interior, edge, or vertex) contains the closest point [S1]. The implementation follows the logic described in [S2, unofficial], which handles all cases via quadratic programming logic.

1.  **Compute Edge Vectors and Reference Vector**:
    $$ \text{edge0} = v_1 - v_0 $$
    $$ \text{edge1} = v_2 - v_0 $$
    $$ v_0 = v_0 - p $$
    *(Note: In [S2, unofficial], $v_0$ is reused as the vector from the query point to vertex 0. We distinguish them here as $\vec{e}_0, \vec{e}_1$ and $\vec{r}_0$ for clarity, but the calculation remains identical.)*

2.  **Compute Dot Products**:
    $$ a = \text{dot}(\text{edge0}, \text{edge0}) $$
    $$ b = \text{dot}(\text{edge0}, \text{edge1}) $$
    $$ c = \text{dot}(\text{edge1}, \text{edge1}) $$
    $$ d = \text{dot}(\text{edge0}, v_0) $$
    $$ e = \text{dot}(\text{edge1}, v_0) $$

3.  **Compute Determinant and Parameters**:
    $$ \text{det} = a \cdot c - b \cdot b $$
    $$ s = b \cdot e - c \cdot d $$
    $$ t = b \cdot d - a \cdot e $$

4.  **Determine Region and Clamp Barycentric Coordinates**:
    The parameters $s$ and $t$ correspond to the barycentric coordinates relative to the triangle's local basis. The condition $s + t < \text{det}$ indicates the closest point lies within the triangle's interior or on its boundary.

    **Case A: Interior or Boundary ($s + t < \text{det}$)**
    - If $s < 0$ and $t < 0$:
        - If $d < 0$:
            $$ s = \text{clamp}\left(\frac{-d}{a}, 0, 1\right) $$
            $$ t = 0 $$
        - Else:
            $$ s = 0 $$
            $$ t = \text{clamp}\left(\frac{-e}{c}, 0, 1\right) $$
    - If $s < 0$ and $t \ge 0$:
        $$ s = \text{clamp}\left(\frac{-d}{a}, 0, 1\right) $$
        $$ t = 0 $$
    - If $s \ge 0$ and $t < 0$:
        $$ s = 0 $$
        $$ t = \text{clamp}\left(\frac{-e}{c}, 0, 1\right) $$
    - Else ($s \ge 0, t \ge 0, s+t < \text{det}$):
        - The point is inside. No clamping needed for $s, t$ relative to the interior, but they must be normalized later if used as barycentrics for the full triangle. However, for the closest point calculation $v_0 + s \cdot \text{edge0} + t \cdot \text{edge1}$, these $s, t$ are the correct affine coordinates.

    **Case B: Outside Edge Opposite Vertex 0 ($s + t \ge \text{det}$)**
    - If $s < 0$:
        - Compute $tmp0 = b + d$ and $tmp1 = c + e$.
        - If $tmp1 > tmp0$:
            $$ s = \text{clamp}\left(\frac{tmp1 - tmp0}{a - 2b + c}, 0, 1\right) $$
            $$ t = 1 - s $$
        - Else:
            $$ s = 0 $$
            $$ t = \text{clamp}\left(\frac{-e}{c}, 0, 1\right) $$
    - If $s \ge 0$ and $t < 0$:
        - If $a + d > b + e$:
            $$ s = \text{clamp}\left(\frac{c + e - b - d}{a - 2b + c}, 0, 1\right) $$
            $$ t = 1 - s $$
        - Else:
            $$ s = \text{clamp}\left(\frac{-e}{c}, 0, 1\right) $$
            $$ t = 0 $$

5.  **Compute Closest Point**:
    $$ c = v_0 + s \cdot \text{edge0} + t \cdot \text{edge1} $$
    *(Note: Here $v_0$ is the original vertex position. In the variable naming of step 1, we used $v_0$ for the vector from $p$ to vertex 0. To avoid confusion, let $V_0$ be the vertex position. Then $c = V_0 + s \cdot \text{edge0} + t \cdot \text{edge1}$.)*

6.  **Compute Squared Distance**:
    $$ d^2 = \text{dot}(p - c, p - c) $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def closest_point_on_triangle(v0, v1, v2, p):
    """
    Computes the closest point on triangle [v0, v1, v2] to point p.
    Returns (closest_point, t0, t1, squared_distance).
    t0, t1 are barycentric coordinates such that closest_point = v0 + t0*(v1-v0) + t1*(v2-v0).
    """
    # Step 1: Edge vectors and reference vector
    edge0 = v1 - v0
    edge1 = v2 - v0
    v0_to_p = v0 - p  # Vector from vertex 0 to query point

    # Step 2: Dot products
    a = dot(edge0, edge0)
    b = dot(edge0, edge1)
    c = dot(edge1, edge1)
    d = dot(edge0, v0_to_p)
    e = dot(edge1, v0_to_p)

    # Step 3: Determinant and parameters
    det = a * c - b * b
    s = b * e - c * d
    t = b * d - a * e

    # Step 4: Determine region
    if s + t < det:
        if s < 0 and t < 0:
            if d < 0:
                s = clamp(-d / a, 0, 1)
                t = 0
            else:
                s = 0
                t = clamp(-e / c, 0, 1)
        elif s < 0:
            s = clamp(-d / a, 0, 1)
            t = 0
        elif t < 0:
            s = 0
            t = clamp(-e / c, 0, 1)
        # Else: inside triangle, s and t are valid
    else:
        if s < 0:
            tmp0 = b + d
            tmp1 = c + e
            if tmp1 > tmp0:
                denom = a - 2 * b + c
                s = clamp((tmp1 - tmp0) / denom, 0, 1)
                t = 1 - s
            else:
                s = 0
                t = clamp(-e / c, 0, 1)
        elif t < 0:
            if a + d > b + e:
                denom = a - 2 * b + c
                s = clamp((c + e - b - d) / denom, 0, 1)
                t = 1 - s
            else:
                s = clamp(-e / c, 0, 1)
                t = 0

    # Step 5: Closest point
    closest = v0 + s * edge0 + t * edge1

    # Step 6: Squared distance
    diff = p - closest
    dist_sq = dot(diff, diff)

    return closest, s, t, dist_sq

def clamp(x, min_val, max_val):
    return max(min_val, min(x, max_val))

def dot(u, v):
    return u.x * v.x + u.y * v.y + u.z * v.z

def sub(u, v):
    return Vec3(u.x - v.x, u.y - v.y, u.z - v.z)

def mul(s, v):
    return Vec3(s * v.x, s * v.y, s * v.z)

class Vec3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z
```

**Parameters**
- `Vec3T`: Any 3D vector type supporting arithmetic operations (`+`, `-`, `*`) and `dot()` [S1].
- `v0, v1, v2`: Triangle vertices [S1].
- `p`: Query point [S1].

**Edge Cases and Pitfalls**
- **Degenerate Triangles**: Triangles with zero area (collinear or coincident vertices) are handled implicitly. The face-interior test fails, and the code falls through to the nearest edge or vertex result [S1].
- **Precision**: Division by zero can occur if $a=0$ or $c=0$ in the clamping steps, which implies degenerate edges. In practice, robust implementations should check for near-zero denominators [S2, unofficial].
- **Barycentric Coordinates**: The returned $s$ and $t$ are affine coordinates relative to the basis $(v_1-v_0, v_2-v_0)$. To get standard barycentric coordinates $(\lambda_0, \lambda_1, \lambda_2)$ where $\lambda_0 + \lambda_1 + \lambda_2 = 1$, use $\lambda_1 = s, \lambda_2 = t, \lambda_0 = 1 - s - t$ [S1].

## Subtopics

### [Unsigned Triangle Distance](3587-unsigned-triangle-distance.md)
Computes the unsigned distance from a query point to a triangle defined by three vertices $a, b, c$. It checks if the point is inside the projected triangle to decide between edge distance and plane distance.

### [Mesh SDF Sampling](7159-mesh-sdf-sampling.md)
Mesh SDF Sampling creates a signed distance sampler function from a `THREE.BufferGeometry` and a world transform matrix, returning a function that computes the shortest signed distance to the mesh at any given world-space coordinate. It solves the problem of evaluating continuous signed distance fields (SDFs) for arbitrary triangle meshes without precomputing a voxel grid, enabling ray-marching or ray-casting against complex geometry in real-time volume rendering contexts. It differs from [Unsigned Triangle Distance](3587-unsigned-triangle-distance.md) by providing signed values (inside/outside) rather than only the magnitude of the distance, and from [Triangle-Box Intersection Voxelization](7408-triangle-box-intersection-voxelization.md) by operating in continuous world space rather than discrete voxel space.

### [Triangle-Box Intersection Voxelization](7408-triangle-box-intersection-voxelization.md)
This method voxelizes watertight triangular meshes into Signed Distance Functions (SDFs) or occupancy grids by computing the distance from voxel reference points to mesh faces or by testing for geometric intersection between triangles and voxel boxes. It distinguishes itself from pure SDF-based occupancy derivation by using explicit triangle-box intersection tests to guarantee that any voxel intersecting the mesh surface is marked as occupied, avoiding the "leaky" voxels common in SDF-only approaches. It builds on [Point-Triangle Distance](2776-point-triangle-distance.md) for SDF computation and relies on [Unsigned Triangle Distance](3587-unsigned-triangle-distance.md) concepts for spatial proximity checks.

### Unsigned Quadrilateral Distance
Computes the unsigned distance from a point to a quadrilateral defined by four vertices $a, b, c, d$. It checks if the point is inside the projected quad to decide between edge distance and plane distance [S3, unofficial]. If the point is inside the quad (sum of signs < 3), return the minimum squared distance to the four edges. Otherwise, return the squared distance to the plane containing the quad. Edge distance uses projection: $\text{proj}_{ba}(pa) = ba \cdot \text{clamp}(\frac{ba \cdot pa}{\|ba\|^2}, 0, 1)$ [S3, unofficial].

### Barycentric Coordinate Calculation
Computes the barycentric coordinates of a point relative to a triangle defined by three vertices, by projecting the point onto the triangle's plane and calculating areas of sub-triangles [S4, unofficial]. If the point is not on the triangle's plane, it is projected onto the plane before area calculation. If the triangle is degenerate (area zero), division by zero occurs in the final normalization [S4, unofficial].

### Triangle Coverage Test
A test to verify if a triangle $T$ is completely outside the volume defined by a set of empty spheres, ensuring the sampling of the initial volume is dense enough to create sphere intersections [S5]. The test fails (triangle not covered) if either of two cases is true: 1) one circle is disjoint from the other two, or 2) an intersection point of two circles lies inside $T$ and outside the third circle [S5].

## Operating characteristics
- **Time Complexity**: $O(1)$ per triangle. The algorithm involves a fixed number of vector subtractions, dot products, and comparisons.
- **Memory Complexity**: $O(1)$. No auxiliary data structures are required.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales linearly with the number of query points if batched, but each query is independent.
- **Strengths**: Robust handling of degenerate triangles; no need for normalization of triangle normals; efficient for GPU parallel execution.
- **Limitations**: Does not provide signed distance (inside/outside) without additional winding or normal checks; precision issues may arise with very large or very small triangles.
- **When to use**: When the closest point on a triangle is needed for collision detection, SDF evaluation, or mesh processing.
- **When not to use**: When signed distance is required without additional logic; when the triangle is part of a large mesh and acceleration structures (BVH, grid) are more appropriate for broad-phase culling.

## Background
The method is based on Voronoi-region decomposition as described in Christer Ericson's "Real-Time Collision Detection", Section 5.1.5 [S1]. It is widely used in computer graphics for collision detection, mesh processing, and SDF generation. The specific implementation details in [S2, unofficial] are from the Voxelizer project. The GLSL implementation for unsigned quadrilateral distance is from marklundin/glsl-sdf-primitives [S3, unofficial]. The barycentric coordinate calculation is from xandergos/unity-sparse-voxel-octrees [S4, unofficial]. The triangle coverage test is from "Sphere Carving: Bounding Volumes for Signed Distance Fields" [S5].

## Sources
[S1] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
[S2] itsyouyi/Voxelizer, https://github.com/itsyouyi/Voxelizer, fetched 2026-10-27 (unofficial)
[S3] marklundin/glsl-sdf-primitives, https://github.com/marklundin/glsl-sdf-primitives, fetched 2026-10-26 (unofficial)
[S4] xandergos/unity-sparse-voxel-octrees, https://github.com/xandergos/unity-sparse-voxel-octrees, fetched 2026-10-26 (unofficial)
[S5] Sphere Carving: Bounding Volumes for Signed Distance Fields, https://doi.org/10.1145/3730845, fetched 2026-10-27
