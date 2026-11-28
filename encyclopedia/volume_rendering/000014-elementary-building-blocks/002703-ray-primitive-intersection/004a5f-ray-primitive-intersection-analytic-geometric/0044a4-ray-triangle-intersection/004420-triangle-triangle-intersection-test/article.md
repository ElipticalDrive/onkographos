# Triangle-Triangle Intersection Test

Also called: SAT Triangle Intersection, Separating Axis Theorem (SAT) for triangles, testIntersection

## Overview
The Triangle-Triangle Intersection Test determines whether two triangles in 3D space intersect using the Separating Axis Theorem (SAT) [S1, S2, S3, unofficial]. It solves the problem of detecting collisions between triangular primitives by projecting them onto potential separating axes and checking for interval overlap [S2, unofficial]. Unlike [Ray-Triangle Intersection](11312-ray-triangle-intersection.md), which tests a ray against a single triangle, this method tests two arbitrary triangles against each other without a predefined viewing direction [S2, unofficial]. It is distinct from [General Ray-Triangle Intersection (Plücker Coordinates)](909-general-ray-triangle-intersection-pl-cker-coordinates.md) in that it relies on interval projection rather than line-plane duality, and from [Point-Triangle Distance](2776-point-triangle-distance.md) as it returns a boolean intersection status rather than a distance metric [S2, unofficial].

## How it works
### Inputs and Outputs
**Input:** Two triangles, $T_1$ and $T_2$ [S1, unofficial]. **Output:** A boolean value: `true` (or 1) if the triangles intersect, `false` (or 0) if they do not [S1, unofficial].

### Data Structures
Each triangle is defined by three vertices in 3D space, stored in a `vt[3][3]` array where each row is a 3D float vector [S3, unofficial]. Vertices are indexed $0, 1, 2$.

### Algorithm Steps
The test projects both triangles onto a set of potential separating axes [S2, unofficial]. If the projections (intervals) on any axis are disjoint, the triangles do not intersect [S2, unofficial]. If all tested axes show overlap, the triangles intersect [S2, unofficial].

1.  **Compute Edge Vectors:** For each triangle $T_i$ ($i \in \{1, 2\}$), compute the three edge vectors: $e_{i,0} = T_i.vt[1] - T_i.vt[0]$, $e_{i,1} = T_i.vt[2] - T_i.vt[1]$, $e_{i,2} = T_i.vt[0] - T_i.vt[2]$ [S2, S3, unofficial].
2.  **Compute Face Normals:** Compute the unnormalized face normals: $n_1 = e_{1,0} \times e_{1,1}$ and $n_2 = e_{2,0} \times e_{2,1}$ [S2, S3, unofficial].
3.  **Check for Co-planarity:** Compute $n = n_1 \times n_2$ [S2, S3, unofficial]. If $n$ is the zero vector, the triangles are co-planar [S1, S2, S3, unofficial].
    *   **Co-planar Case:** Check if the triangles lie in the same plane by comparing $n_1 \cdot T_1.vt[0]$ and $n_1 \cdot T_2.vt[0]$ [S1, S3, unofficial]. If $n_1 \cdot T_1.vt[0] \neq n_1 \cdot T_2.vt[0]$, the planes are distinct and the triangles do not intersect; return `false` [S3, unofficial]. If they are in the same plane, test separation along the 6 edge normals: $n_1 \times e_{1,i}$ and $n_2 \times e_{2,i}$ for $i \in \{0, 1, 2\}$ [S1, S3, unofficial]. If any axis separates the triangles, return `false`; otherwise return `true` [S1, S3, unofficial].
    *   **Non-Co-planar Case:** Test separation along the face normals $n_1$ and $n_2$ [S1, S3, unofficial]. If either separates them, return `false` [S1, S3, unofficial]. Test separation along the 9 cross products of every edge of $T_1$ with every edge of $T_2$: $axis_{i,j} = e_{1,i} \times e_{2,j}$ for $i,j \in \{0, 1, 2\}$ [S1, S3, unofficial]. If any separates them, return `false` [S1, S3, unofficial]. If no axis separates the triangles, return `true` [S1, S3, unofficial].
4.  **Projection and Separation Check:** For each candidate axis $A$, project both triangles to min/max intervals: $min_1 = A \cdot T_1.vt[0]$, $max_1 = min_1$; loop $k \in \{1, 2\}$: $temp = A \cdot T_1.vt[k]$; update $min_1 = \min(min_1, temp)$, $max_1 = \max(max_1, temp)$; repeat for $T_2$ [S2, unofficial]. If any axis shows disjoint intervals ($min_1 \ge max_2$ or $min_2 \ge max_1$), return `false`; else return `true` [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def triangle_triangle_intersect(t1, t2, epsilon=1e-5):
    # 1. Compute edge vectors
    e1 = [t1.vt[1] - t1.vt[0], t1.vt[2] - t1.vt[1], t1.vt[0] - t1.vt[2]]
    e2 = [t2.vt[1] - t2.vt[0], t2.vt[2] - t2.vt[1], t2.vt[0] - t2.vt[2]]

    # 2. Compute face normals
    n1 = cross(e1[0], e1[1])
    n2 = cross(e2[0], e2[1])

    # 3. Check co-planarity
    n = cross(n1, n2)
    if is_zero_vector(n, epsilon):
        # Co-planar case
        if dot(n1, t1.vt[0]) != dot(n1, t2.vt[0]):
            return False
        axes = []
        for i in range(3):
            axes.append(cross(n1, e1[i]))
            axes.append(cross(n2, e2[i]))
        for axis in axes:
            if test_separation(axis, t1, t2, epsilon):
                return False
        return True
    else:
        # Non-co-planar case
        if test_separation(n1, t1, t2, epsilon):
            return False
        if test_separation(n2, t1, t2, epsilon):
            return False
        for i in range(3):
            for j in range(3):
                axis = cross(e1[i], e2[j])
                if test_separation(axis, t1, t2, epsilon):
                    return False
        return True

def test_separation(axis, t1, t2, epsilon):
    min1 = dot(axis, t1.vt[0])
    max1 = min1
    for k in range(1, 3):
        proj = dot(axis, t1.vt[k])
        if proj < min1:
            min1 = proj
        if proj > max1:
            max1 = proj

    min2 = dot(axis, t2.vt[0])
    max2 = min2
    for k in range(1, 3):
        proj = dot(axis, t2.vt[k])
        if proj < min2:
            min2 = proj
        if proj > max2:
            max2 = proj

    if min1 > max2 + epsilon or min2 > max1 + epsilon:
        return True
    return False

def is_zero_vector(v, epsilon):
    return abs(v.x) < epsilon and abs(v.y) < epsilon and abs(v.z) < epsilon
```
*Note: `dot` and `cross` are standard vector operations.*

### Parameters
- **Epsilon:** A small floating-point tolerance (e.g., `0.00001f`) used to handle numerical instability in separation checks and zero-vector detection [S2, S3, unofficial].

### Edge Cases and Pitfalls
- **Co-planar Triangles:** Handling co-planar triangles correctly is critical [S1, S2, S3, unofficial]. Some implementations use a hack where co-planar triangles are explicitly regarded as not intersecting to avoid degenerate cases [S1, S2, S3, unofficial]. The full SAT test requires testing the 6 edge normals in the co-planar case [S1, S3, unofficial].
- **Numerical Precision:** Floating-point errors can cause false negatives or positives [S2, S3, unofficial]. Using an epsilon in the separation check ($min_1 > max_2 + \epsilon$) helps mitigate this [S2, S3, unofficial].
- **Degenerate Triangles:** If a triangle has zero area (collinear vertices), its normal is zero [G]. The algorithm assumes non-degenerate triangles; degenerate inputs may require pre-validation [G].

## Subtopics

### Bounding Box Pre-check
An early-out optimization where the axis-aligned bounding boxes (AABBs) of the two triangles are tested for intersection [S2, unofficial]. If the AABBs do not intersect, the triangles do not intersect [S2, unofficial]. This is an $O(1)$ check that can save the more expensive SAT test [S2, unofficial].

### Triangle-Triangle Overlap Area
An extension of the intersection test that computes the area of intersection between two triangles, useful for rendering and physics [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ per pair of triangles [S2, unofficial]. The algorithm tests at most 11 axes (2 face normals + 9 edge-edge cross products) for non-co-planar triangles, and up to 6 axes for co-planar triangles [S2, unofficial].
- **Memory Complexity:** $O(1)$ [G]. Only a few temporary vectors and scalars are required [G].
- **Measured Performance:** The SAT method is generally fast and robust for single pair tests [S2, unofficial].
- **Strengths:** No division by zero issues; handles all relative orientations; robust with epsilon tuning [G].
- **Limitations:** Does not provide the intersection curve or area; co-planar cases require special handling; less efficient than bounding volume hierarchies for large scenes [G].
- **When to Use:** When testing a small number of triangle pairs, or as a leaf-node test in a spatial partitioning structure [G].
- **When Not to Use:** For large-scale scene intersection, use a BVH or grid-based approach [G]. For ray-triangle intersection, use [Ray-Triangle Intersection](11312-ray-triangle-intersection.md) [G].

## Background
The Triangle-Triangle Intersection Test using SAT is a standard technique in computational geometry and computer graphics [G]. It is implemented in various open-source dual contouring and mesh processing libraries, including work by Tao Ju [S1, S2, S3, unofficial]. The method relies on the Separating Axis Theorem, which states that two convex objects are disjoint if and only if there exists a plane (axis) onto which their projections do not overlap [G]. For triangles, the potential separating axes are the face normals and the cross products of their edges [S2, unofficial].

## Sources
[S1] aewallin/dualcontouring, https://github.com/aewallin/dualcontouring, fetched 2026-10-26 (unofficial)
[S2] jbayardo/tao-ju-dual-contouring, https://github.com/jbayardo/tao-ju-dual-contouring, fetched 2026-10-26 (unofficial)
[S3] ChessQian/DualContouring, https://github.com/ChessQian/DualContouring, fetched 2026-10-26 (unofficial)
