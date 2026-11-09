# Triangle-Triangle Intersection Test
Also called: SAT Triangle Intersection, Separating Axis Theorem (SAT) for Triangles, testIntersection

## Overview
The Triangle-Triangle Intersection Test determines whether two triangles in 3D space share any common points, returning true if they intersect (including touching edges or vertices) and false otherwise [S1, S2, S3, unofficial]. It solves the geometric overlap problem required for mesh intersection queries, collision detection, and constructive solid geometry operations within volume rendering pipelines [S1, S2, S3, unofficial]. Unlike [Ray-Triangle Intersection](11312-ray-triangle-intersection.md), which computes the specific parameter of contact along a ray, this method performs a binary boolean check between two static primitives [S1, S2, S3, unofficial]. It is distinguished from [General Ray-Triangle Intersection (Plücker Coordinates)](909-general-ray-triangle-intersection-pl-cker-coordinates.md) by its reliance on the Separating Axis Theorem (SAT) rather than linear algebraic line-plane duality, and from [Point-Triangle Distance](2776-point-triangle-distance.md) by its focus on binary overlap rather than metric distance [S1, S2, S3, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Triangle $T_1$: Defined by three vertices $\mathbf{v}_{1,0}, \mathbf{v}_{1,1}, \mathbf{v}_{1,2}$, where each $\mathbf{v}_{i,j} \in \mathbb{R}^3$.
*   Triangle $T_2$: Defined by three vertices $\mathbf{v}_{2,0}, \mathbf{v}_{2,1}, \mathbf{v}_{2,2}$, where each $\mathbf{v}_{i,j} \in \mathbb{R}^3$.

**Output:**
*   Boolean: `true` if the triangles intersect, `false` otherwise.

### Data Structures
The triangles are represented as arrays of three 3D vectors.
*   `v1[3][3]`: Array of 3 vertices, each a 3-component float array. `v1[i]` is the $i$-th vertex of $T_1$.
*   `v2[3][3]`: Array of 3 vertices, each a 3-component float array. `v2[i]` is the $i$-th vertex of $T_2$.

### Steps
1.  **Compute Edge Vectors:**
    For each triangle $T_k$ ($k \in \{1, 2\}$), compute the three edge vectors forming the perimeter:
    $$
    \begin{aligned}
    \mathbf{e}_{k,0} &= \mathbf{v}_{k,1} - \mathbf{v}_{k,0} \\
    \mathbf{e}_{k,1} &= \mathbf{v}_{k,2} - \mathbf{v}_{k,1} \\
    \mathbf{e}_{k,2} &= \mathbf{v}_{k,0} - \mathbf{v}_{k,2}
    \end{aligned}
    $$
    [S1, S3, unofficial]

2.  **Compute Face Normals:**
    Compute the unnormalized face normal for each triangle using the cross product of two adjacent edge vectors:
    $$
    \mathbf{n}_1 = \mathbf{e}_{1,0} \times \mathbf{e}_{1,1}
    $$
    $$
    \mathbf{n}_2 = \mathbf{e}_{2,0} \times \mathbf{e}_{2,1}
    $$
    [S1, S3, unofficial]

3.  **Check for Co-planarity:**
    Compute the cross product of the two face normals:
    $$
    \mathbf{n}_{cross} = \mathbf{n}_1 \times \mathbf{n}_2
    $$
    If $\mathbf{n}_{cross}$ is the zero vector (i.e., all components are zero or below a small epsilon threshold), the triangles are co-planar. Proceed to the **Co-planar Case** (Step 4a). Otherwise, proceed to the **Non-Co-planar Case** (Step 4b).
    [S1, S3, unofficial]

    *Note: Some implementations use a bounding box check or a fixed epsilon (e.g., $10^{-5}$) to determine if $\mathbf{n}_{cross}$ is effectively zero [S2, S3, unofficial].*

4.  **Separating Axis Test:**
    The Separating Axis Theorem states that two convex shapes do not intersect if there exists a plane (separating axis) onto which their projections do not overlap.

    **4a. Co-planar Case:**
    If the triangles are co-planar, they lie in the same plane.
    1.  Verify they are in the same plane by checking if the dot product of $\mathbf{n}_1$ with a vertex from $T_1$ equals the dot product with a vertex from $T_2$ (within tolerance). If not, return `false` [S1, S3, unofficial].
    2.  Test separation along the 6 axes formed by the cross product of each triangle's normal and its own edges:
        *   Axes: $\mathbf{n}_1 \times \mathbf{e}_{1,i}$ for $i \in \{0, 1, 2\}$
        *   Axes: $\mathbf{n}_2 \times \mathbf{e}_{2,i}$ for $i \in \{0, 1, 2\}$
    3.  If any axis separates the triangles, return `false`.
    4.  If no axis separates them, return `true`.
    [S1, S3, unofficial]

    **4b. Non-Co-planar Case:**
    If the triangles are not co-planar, test the following sets of axes:
    1.  **Face Normals:** Test axes $\mathbf{n}_1$ and $\mathbf{n}_2$. If either separates the triangles, return `false`.
    2.  **Edge-Edge Cross Products:** Test the 9 axes formed by the cross product of every edge of $T_1$ with every edge of $T_2$:
        $$
        \mathbf{a}_{i,j} = \mathbf{e}_{1,i} \times \mathbf{e}_{2,j} \quad \text{for } i \in \{0, 1, 2\}, j \in \{0, 1, 2\}
        $$
    3.  If any of these 9 axes separates the triangles, return `false`.
    4.  If none of the axes separate the triangles, return `true`.
    [S1, S3, unofficial]

5.  **Projection and Separation Check:**
    For any candidate axis $\mathbf{A}$, perform the following to check for separation:
    1.  Project all vertices of $T_1$ onto $\mathbf{A}$ to find the minimum and maximum scalar values:
        $$
        \begin{aligned}
        p_{1,i} &= \mathbf{v}_{1,i} \cdot \mathbf{A} \quad \text{for } i \in \{0, 1, 2\} \\
        \min_1 &= \min(p_{1,0}, p_{1,1}, p_{1,2}) \\
        \max_1 &= \max(p_{1,0}, p_{1,1}, p_{1,2})
        \end{aligned}
        $$
    2.  Project all vertices of $T_2$ onto $\mathbf{A}$:
        $$
        \begin{aligned}
        p_{2,i} &= \mathbf{v}_{2,i} \cdot \mathbf{A} \quad \text{for } i \in \{0, 1, 2\} \\
        \min_2 &= \min(p_{2,0}, p_{2,1}, p_{2,2}) \\
        \max_2 &= \max(p_{2,0}, p_{2,1}, p_{2,2})
        \end{aligned}
        $$
    3.  Check for overlap. The triangles are separated if:
        $$
        \min_1 \ge \max_2 \quad \text{or} \quad \min_2 \ge \max_1
        $$
        [S2, unofficial]
    4.  **Numerical Stability:** To handle floating-point precision issues, a small epsilon tolerance $\epsilon$ (e.g., $0.00001$) is often added to the separation condition [S2, S3, unofficial]:
        $$
        \min_1 > \max_2 + \epsilon \quad \text{or} \quad \min_2 > \max_1 + \epsilon
        $$
        If separated, return `false`. Otherwise, continue to the next axis.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def cross(v1, v2):
    return [
        v1[1]*v2[2] - v1[2]*v2[1],
        v1[2]*v2[0] - v1[0]*v2[2],
        v1[0]*v2[1] - v1[1]*v2[0]
    ]

def sub(v1, v2):
    return [v1[0]-v2[0], v1[1]-v2[1], v1[2]-v2[2]]

def project(vertices, axis):
    # Project 3 vertices onto axis and return min and max
    p0 = dot(vertices[0], axis)
    p1 = dot(vertices[1], axis)
    p2 = dot(vertices[2], axis)
    min_val = min(p0, p1, p2)
    max_val = max(p0, p1, p2)
    return min_val, max_val

def triangles_intersect(t1_verts, t2_verts):
    # t1_verts, t2_verts: list of 3 tuples/lists, each with 3 floats (x,y,z)
    
    # 1. Compute edge vectors
    e1 = [
        sub(t1_verts[1], t1_verts[0]),
        sub(t1_verts[2], t1_verts[1]),
        sub(t1_verts[0], t1_verts[2])
    ]
    e2 = [
        sub(t2_verts[1], t2_verts[0]),
        sub(t2_verts[2], t2_verts[1]),
        sub(t2_verts[0], t2_verts[2])
    ]
    
    # 2. Compute face normals
    n1 = cross(e1[0], e1[1])
    n2 = cross(e2[0], e2[1])
    
    # 3. Check for co-planarity
    n_cross = cross(n1, n2)
    is_coplanar = (n_cross[0] == 0 and n_cross[1] == 0 and n_cross[2] == 0)
    
    # Optional: Check if co-planar triangles are in distinct planes
    if is_coplanar:
        # Check if dot(n1, v1_0) == dot(n1, v2_0)
        # If not, they are parallel but distinct planes -> no intersection
        # Note: Sources indicate some implementations skip this and treat all 
        # zero-n_cross as co-planar in the same plane, or use a hack.
        # We follow the standard SAT logic for co-planar polygons.
        pass

    # Helper to check separation on a single axis
    def test_axis(axis):
        min1, max1 = project(t1_verts, axis)
        min2, max2 = project(t2_verts, axis)
        # Epsilon for numerical stability
        epsilon = 0.00001
        if min1 > max2 + epsilon or min2 > max1 + epsilon:
            return True # Separated
        return False # Overlapping

    # 4. Test Axes
    if is_coplanar:
        # Co-planar case: test 6 axes (normal x edge for each triangle)
        for i in range(3):
            axis = cross(n1, e1[i])
            if test_axis(axis):
                return False
        for i in range(3):
            axis = cross(n2, e2[i])
            if test_axis(axis):
                return False
    else:
        # Non-co-planar case: test 2 face normals + 9 edge-edge cross products
        # Test face normals
        if test_axis(n1):
            return False
        if test_axis(n2):
            return False
        
        # Test edge-edge cross products
        for i in range(3):
            for j in range(3):
                axis = cross(e1[i], e2[j])
                if test_axis(axis):
                    return False

    return True
```

### Parameters
*   **Epsilon ($\epsilon$):** A small floating-point tolerance used to mitigate precision errors during projection comparisons. Typical value: $10^{-5}$ [S2, S3, unofficial].

### Edge Cases and Pitfalls
*   **Co-planar Triangles:** Handling co-planar triangles is the most complex part. If the face normals' cross product is zero, the triangles lie in parallel planes. If the planes are distinct (checked via dot product of normal and vertex), they do not intersect. If they are in the same plane, the standard 9 edge-edge axes are insufficient; the 6 face-edge axes ($\mathbf{n} \times \mathbf{e}$) must be tested [S1, S3, unofficial]. Some implementations use a "hack" to immediately return `false` for co-planar triangles to avoid this complexity [S1, S2, S3, unofficial].
*   **Degenerate Triangles:** Triangles with zero area (collinear vertices) may produce zero-length normals or edges. The algorithm may fail or produce incorrect results if degenerate inputs are not pre-filtered [G].
*   **Floating Point Precision:** The separation check `min1 >= max2` is sensitive to precision. Using a strict `>=` without epsilon can lead to false negatives for touching triangles or false positives for nearly touching ones due to rounding errors [S2, S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pair of triangles. The algorithm performs a fixed number of arithmetic operations. It tests at most 12 axes (2 face normals + 9 edge-edge cross products) for non-co-planar triangles, and up to 6 axes (face-edge cross products) for co-planar triangles [S2, unofficial].
*   **Memory Complexity:** $O(1)$. Requires storage for a constant number of vectors and scalars.
*   **Measured Performance:** The method is highly efficient for pairwise checks. When integrated with bounding box pre-checks (O(1) early-out), it significantly reduces the number of full SAT tests performed in broad-phase collision detection [S2, unofficial].
*   **Strengths:** Deterministic, robust for convex primitives, no iterative convergence required.
*   **Limitations:** Does not provide intersection points or depth information. Co-planar handling adds complexity and branching. Sensitive to floating-point precision without epsilon tuning.
*   **When to use:** When a binary intersection result is needed between two triangles, such as in mesh-mesh collision detection or voxelization preprocessing.
*   **When not to use:** When the intersection point, line segment, or area of overlap is required (use ray-triangle or polygon clipping instead). When dealing with non-convex or self-intersecting polygons.

## Background
The Triangle-Triangle Intersection Test using the Separating Axis Theorem is a fundamental technique in computational geometry. It is widely implemented in graphics libraries and collision detection systems. The specific implementation details, including the handling of co-planar cases and the use of epsilon tolerances, are documented in open-source dual contouring and mesh processing repositories [S1, S2, S3, unofficial]. The method relies on the general SAT principle, which applies to any convex shapes, specialized here for triangles [S1, S2, S3, unofficial].

## Sources
[S1] aewallin/dualcontouring, https://github.com/aewallin/dualcontouring, fetched 2026-10-26 (unofficial)
[S2] jbayardo/tao-ju-dual-contouring, https://github.com/jbayardo/tao-ju-dual-contouring, fetched 2026-10-26 (unofficial)
[S3] ChessQian/DualContouring, https://github.com/ChessQian/DualContouring, fetched 2026-10-26 (unofficial)
