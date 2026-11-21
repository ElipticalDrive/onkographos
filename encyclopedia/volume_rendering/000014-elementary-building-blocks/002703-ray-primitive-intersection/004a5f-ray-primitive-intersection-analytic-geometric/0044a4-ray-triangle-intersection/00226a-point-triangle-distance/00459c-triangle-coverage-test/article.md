# Triangle Coverage Test

Also called: Triangle Sphere Coverage Test, Circle Coverage Test

## Overview
The Triangle Coverage Test verifies whether a triangle $T$ is completely contained within the union of three spheres centered at its vertices, with radii determined by a scalar field function $f$ evaluated at those vertices. It solves the problem of ensuring that a discrete sampling of an implicit volume is dense enough to guarantee no gaps exist between the approximating spheres, which is critical for sphere carving algorithms [S1, S2]. Unlike general ray-triangle intersection which determines visibility, this test determines topological completeness of the volume approximation [S1, S2]. It is distinct from [Point-Triangle Distance](2776-point-triangle-distance.md) by focusing on set containment rather than proximity, and from [Barycentric Coordinate Calculation](11142-barycentric-coordinate-calculation.md) by using barycentric logic only implicitly to define the triangle domain [S1, S2].

## How it works
**Inputs:**
- Triangle $T$ defined by three vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2 \in \mathbb{R}^3$.
- Scalar field function $f: \mathbb{R}^3 \to \mathbb{R}$, where $f(\mathbf{p})$ represents the radius of the empty sphere at $\mathbf{p}$ [S1, S2].

**Outputs:**
- Boolean: `True` if the triangle is completely covered by the spheres; `False` otherwise.

**Data Structures:**
- Vertices: Array of 3 vectors $\mathbf{v}_i$.
- Radii: Array of 3 floats $r_i = f(\mathbf{v}_i)$.
- Circles: Implicitly defined by center $\mathbf{c}_i = \mathbf{v}_i$ and radius $r_i$ in the plane of $T$.

**Steps:**
1. **Evaluate Radii:** Compute $r_i = f(\mathbf{v}_i)$ for $i \in \{0, 1, 2\}$ [S1, S2].
2. **Check Disjointness:** Verify if any circle is disjoint from the other two. A circle $i$ is disjoint from circle $j$ if the distance between centers exceeds the sum of radii: $\|\mathbf{v}_i - \mathbf{v}_j\| > r_i + r_j$. If any pair is disjoint, the test fails (returns `False`) [S1, S2].
3. **Check Intersection Points:** For each pair of circles $(i, j)$, compute their intersection points. If an intersection point lies inside triangle $T$ and outside the third circle $k$ (where $k \neq i, j$), the test fails (returns `False`) [S1, S2].
   - To check if a point $\mathbf{p}$ is inside $T$, use barycentric coordinates or cross-product signs.
   - To check if $\mathbf{p}$ is outside circle $k$, verify $\|\mathbf{p} - \mathbf{v}_k\| > r_k$ [S1, S2].
4. **Return Result:** If neither disjointness nor uncovered intersection points are found, return `True` [S1, S2].

**Formulas:**
- Sphere coverage condition: $\forall \mathbf{p} \in T, \exists i \in \{0, 1, 2\}, \|\mathbf{p} - \mathbf{v}_i\| < f(\mathbf{v}_i)$ [S1, S2].
- Disjointness condition: $\|\mathbf{v}_i - \mathbf{v}_j\| > r_i + r_j$ [S1, S2].
- Outside circle condition: $\|\mathbf{p} - \mathbf{v}_k\| > r_k$ [S1, S2].

**Parameters:**
- None specified. The test relies on the property that the initial volume sampling is dense enough to create sphere intersections [S1, S2].

**Edge Cases and Pitfalls:**
- **Degenerate Triangles:** If vertices are collinear, the triangle has zero area. The test may behave unpredictably if not handled. [G] Check for zero area and return `True` or handle as a line segment.
- **Precision:** Floating-point errors in intersection calculations may cause false negatives. [G] Use a small epsilon for comparisons.
- **Intersection Calculation:** Computing circle-circle intersections in 3D requires projecting to the triangle's plane. [G] Ensure robust plane projection.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def triangle_coverage_test(v0, v1, v2, f):
    """
    v0, v1, v2: 3D vectors (tuples or lists of 3 floats)
    f: function that takes a 3D vector and returns a float (radius)
    Returns: True if covered, False otherwise
    """
    # Step 1: Evaluate radii
    r0 = f(v0)
    r1 = f(v1)
    r2 = f(v2)
    
    # Helper: distance squared
    def dist_sq(a, b):
        return sum((a[i] - b[i])**2 for i in range(3))
    
    # Helper: distance
    def dist(a, b):
        return math.sqrt(dist_sq(a, b))
    
    # Step 2: Check disjointness
    # Circle 0 disjoint from 1?
    if dist(v0, v1) > r0 + r1:
        return False
    # Circle 0 disjoint from 2?
    if dist(v0, v2) > r0 + r2:
        return False
    # Circle 1 disjoint from 2?
    if dist(v1, v2) > r1 + r2:
        return False
        
    # Step 3: Check intersection points
    # We need to check intersections of pairs (0,1), (0,2), (1,2)
    # For each pair, find intersection points, check if inside T and outside third circle
    
    # Function to find intersection points of two circles in 3D
    # Returns list of 3D points
    def circle_intersection(p1, r1, p2, r2):
        d = dist(p1, p2)
        # If d > r1 + r2 or d < |r1 - r2|, no intersection (already handled by disjoint check, but for completeness)
        if d > r1 + r2 or d < abs(r1 - r2):
            return []
        if d == 0 and r1 == r2:
            return [] # Concentric circles, infinite intersections, but disjoint check would have caught if r1 != r2? No, if r1==r2 and d=0, they are identical. 
                      # However, if they are identical, they cover each other. But we need to check if they cover the triangle.
                      # This case is degenerate. Assume no intersection points to check? 
                      # Actually, if circles are identical, any point on one is on the other.
                      # The intersection points are not unique. 
                      # For the purpose of this test, if circles are identical, they don't leave gaps between them.
                      # But we need to check if they cover the triangle. 
                      # The logic of "intersection points inside T and outside third circle" might not apply directly.
                      # However, if two circles are identical, the third circle must cover the rest.
                      # Let's assume general position for now.
            return []
        
        # Calculate the point P2 along the line from p1 to p2
        # a = (r1^2 - r2^2 + d^2) / (2*d)
        a = (r1**2 - r2**2 + d**2) / (2 * d)
        # h = sqrt(r1^2 - a^2)
        h_sq = r1**2 - a**2
        if h_sq < 0:
            return []
        h = math.sqrt(h_sq)
        
        # P2 = p1 + a * (p2 - p1) / d
        p2_vec = [p2[i] - p1[i] for i in range(3)]
        p2_point = [p1[i] + a * p2_vec[i] / d for i in range(3)]
        
        # Offset vector perpendicular to p1-p2
        # We need a vector perpendicular to p1-p2 in the plane of the triangle.
        # This is complex in 3D. 
        # Alternative: Project everything to 2D? 
        # Or use the fact that the intersection points lie on the line perpendicular to p1-p2 passing through P2.
        # Let's use a simpler approach: 
        # The intersection points are P2 +/- h * n, where n is a unit vector perpendicular to p1-p2.
        # But n is not unique in 3D. It must be in the plane of the triangle.
        # So we need the normal of the triangle.
        pass # This helper is incomplete without triangle normal. See below.

    # To properly compute intersection points, we need the triangle's plane.
    # Compute normal
    v01 = [v1[i] - v0[i] for i in range(3)]
    v02 = [v2[i] - v0[i] for i in range(3)]
    normal = [
        v01[1]*v02[2] - v01[2]*v02[1],
        v01[2]*v02[0] - v01[0]*v02[2],
        v01[0]*v02[1] - v01[1]*v02[0]
    ]
    norm_len = math.sqrt(sum(n**2 for n in normal))
    if norm_len == 0:
        return True # Degenerate triangle, assume covered? Or handle as edge case.
    normal = [n / norm_len for n in normal]
    
    # Helper to find intersection points in 3D given normal
    def circle_intersection_3d(p1, r1, p2, r2, n):
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
        
        # P2 point
        p2_vec = [p2[i] - p1[i] for i in range(3)]
        p2_point = [p1[i] + a * p2_vec[i] / d for i in range(3)]
        
        # Find a vector perpendicular to p1-p2 and n
        # u = p2_vec / d
        u = [p2_vec[i] / d for i in range(3)]
        # v = n x u
        v = [
            n[1]*u[2] - n[2]*u[1],
            n[2]*u[0] - n[0]*u[2],
            n[0]*u[1] - n[1]*u[0]
        ]
        v_len = math.sqrt(sum(vi**2 for vi in v))
        if v_len == 0:
            return [] # Should not happen if n is normal to triangle and u is in triangle
        v = [vi / v_len for vi in v]
        
        # Intersection points
        p_int1 = [p2_point[i] + h * v[i] for i in range(3)]
        p_int2 = [p2_point[i] - h * v[i] for i in range(3)]
        
        return [p_int1, p_int2]

    # Helper to check if point is inside triangle
    def point_in_triangle(p, v0, v1, v2):
        # Barycentric coordinates
        v0v1 = [v1[i] - v0[i] for i in range(3)]
        v0v2 = [v2[i] - v0[i] for i in range(3)]
        v0p = [p[i] - v0[i] for i in range(3)]
        
        d00 = sum(v0v1[i]*v0v1[i] for i in range(3))
        d01 = sum(v0v1[i]*v0v2[i] for i in range(3))
        d11 = sum(v0v2[i]*v0v2[i] for i in range(3))
        d20 = sum(v0p[i]*v0v1[i] for i in range(3))
        d21 = sum(v0p[i]*v0v2[i] for i in range(3))
        
        denom = d00 * d11 - d01 * d01
        if denom == 0:
            return False # Degenerate
        
        v = (d11 * d20 - d01 * d21) / denom
        w = (d00 * d21 - d01 * d20) / denom
        u = 1.0 - v - w
        
        return u >= 0 and v >= 0 and w >= 0

    # Helper to check if point is outside circle
    def point_outside_circle(p, center, radius):
        return dist(p, center) > radius

    # Check pairs
    pairs = [(0, 1, 2), (0, 2, 1), (1, 2, 0)]
    for i, j, k in pairs:
        vi = [v0[i] if i==0 else (v1[i] if i==1 else v2[i]) for i in range(3)] # This is wrong indexing. Let's fix.
        pass

    # Correct indexing
    vertices = [v0, v1, v2]
    radii = [r0, r1, r2]
    
    for i in range(3):
        for j in range(i + 1, 3):
            k = 3 - i - j
            # Intersection of circle i and j
            pts = circle_intersection_3d(vertices[i], radii[i], vertices[j], radii[j], normal)
            for p in pts:
                if point_in_triangle(p, v0, v1, v2):
                    if point_outside_circle(p, vertices[k], radii[k]):
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
- **Time Complexity:** Constant time $O(1)$ for a single triangle, involving a fixed number of distance calculations and intersection checks.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of triangles in the mesh.
- **Strengths:** Efficiently verifies volume completeness without dense sampling.
- **Limitations:** Relies on dense initial sampling; fails if spheres are too small or sparse.
- **When to use:** Sphere carving algorithms, implicit volume reconstruction.
- **When not to use:** When spheres are not dense enough or when exact SDF values are not available.

## Background
The Triangle Coverage Test was introduced by Cédric Zanni in 2024 as part of the Sphere Carving algorithm for bounding volumes of signed distance fields [S1, S2]. It addresses the need to ensure that the discrete sampling of an implicit volume is dense enough to create sphere intersections, preventing gaps in the reconstructed volume [S1, S2]. The method is based on the observation that a triangle is not covered if either one circle is disjoint from the others or an intersection point of two circles lies inside the triangle but outside the third [S1, S2].

## Sources
[S1] Sphere Carving: Bounding Volumes for Signed Distance Fields, https://doi.org/10.1145/3730845, fetched 2026-10-27
[S2] Sphere Carving: Bounding Volumes for Signed Distance ..., https://research.adobe.com/publication/sphere-carving-bounding-volumes-for-signed-distance-fields/, fetched 2026-11-05
