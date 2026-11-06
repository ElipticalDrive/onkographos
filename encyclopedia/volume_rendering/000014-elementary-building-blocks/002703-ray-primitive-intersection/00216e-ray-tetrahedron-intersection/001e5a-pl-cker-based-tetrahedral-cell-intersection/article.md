# Plücker-based Tetrahedral Cell Intersection

Also called: Plücker coordinates ray-tetrahedron test, Plücker ray-tetrahedron intersection, Plücker space traversal, Plücker test traversal, Plücker-based Tetrahedral Mesh Traversal, Ray-Tetrahedron Intersection using Plücker Coordinates, Ray-Tetrahedron Intersection with Plücker Coordinates (Optimized), Tetrahedral Plücker traversal

## Overview
This method traverses tetrahedral meshes by using Plücker coordinates to determine ray-triangle intersections and identify exit faces, avoiding explicit plane intersection calculations [S1]. It sets itself apart from its siblings [Point-in-Tetrahedron Test via Barycentric Coordinates](2591-point-in-tetrahedron-test-via-barycentric-coordinates.md) and [Tetrahedron Face Extraction](3742-tetrahedron-face-extraction.md) by directly providing scaled barycentric coordinates from Plücker tests, thereby avoiding division until the final normalization step [S1]. It outperforms Moller/Trumbore-based algorithms in all tested cases [S2, unofficial].

## How it works
**Inputs:** A ray defined by origin $P$ and direction $L$ (infinite directed line), and a tetrahedron defined by vertices $V_0, V_1, V_2, V_3$. The tetrahedron is assumed oriented so that the outward-pointing normal of face $F_i$ is directed away from vertex $V_i$ [S2, unofficial].
**Outputs:** Entry face $F_{enter}$, exit face $F_{leave}$, intersection points $P_{enter}, P_{leave}$, their barycentric coordinates, and parametric distances $t_{enter}, t_{leave}$ [S2, unofficial].

**Data Structures:**
*   **Ray:** Defined by point $P$ and direction $L$ [S2, unofficial].
*   **Tetrahedron Vertices:** $V_0, V_1, V_2, V_3$ [S2, unofficial].
*   **Faces:** Indexed by the opposite vertex: $F_3(V_0 V_1 V_2)$, $F_2(V_1 V_0 V_3)$, $F_1(V_2 V_3 V_0)$, $F_0(V_3 V_2 V_1)$ [S2, unofficial].
*   **Face Vertices:** Vertices of face $F_i$ are denoted $V_{i0}, V_{i1}, V_{i2}$. Example: For $F_2$, $V_{20}=V_1, V_{21}=V_0, V_{22}=V_3$ [S2, unofficial].
*   **Edges:** Edges of face $F_i$ are subscripted by the index of the opposite vertex: $e_{i0}(V_{i1}V_{i2})$, $e_{i1}(V_{i2}V_{i0})$, $e_{i2}(V_{i0}V_{i1})$ [S2, unofficial].
*   **Connectivity:** Neighbor references require 16 additional bytes per tetrahedron [S1].

**Plücker Coordinates:**
A directed line is represented by a six-vector $\pi = \{p : q\} = \{p : p \times o\}$, where $o$ is a point on the line and $p$ is the direction vector [S1].
For a ray $r(t) = P + tL$, the Plücker coordinates are $\pi_r = \{L : L \times P\} = \{p_r : q_r\}$ [S1].
For an edge from vertex $A$ to $B$, the Plücker coordinates are $\pi_{AB} = \{B-A : (B-A) \times A\}$ [S2, unofficial].

**Permuted Inner Product:**
The relative orientation of two lines $r$ and $s$ is given by:
$$ \pi_r \odot \pi_s = p_r \cdot q_s + q_r \cdot p_s $$
*   $> 0$: $s$ goes counterclockwise around $r$ [S2, unofficial].
*   $< 0$: $s$ goes clockwise around $r$ [S2, unofficial].
*   $= 0$: $s$ intersects or is parallel to $r$ [S2, unofficial].
Note: $\pi_r \odot \pi_{AB} = -(\pi_r \odot \pi_{BA})$ [S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Set $F_{enter} = \text{nil}$, $F_{leave} = \text{nil}$ [S2, unofficial].
2.  **Face Loop:** Iterate $i$ from 3 down to 0 (faces $F_3, F_2, F_1, F_0$) [S2, unofficial].
3.  **Sign Computation:** For face $F_i$, compute signs $\sigma_{i0}, \sigma_{i1}, \sigma_{i2}$ for its three edges $e_{i0}, e_{i1}, e_{i2}$:
    $$ \sigma_{j}^{i} = \text{sign}(\pi_r \odot \pi_{e_{ij}}) $$
    where $\text{sign}(x)$ is $1$ if $x>0$, $0$ if $x=0$, $-1$ if $x<0$ [S2, unofficial].
4.  **Optimization (Early Exit in Face Test):**
    *   Compute $\sigma_{i0}$ and $\sigma_{i1}$ [S2, unofficial].
    *   If $\sigma_{i0}$ and $\sigma_{i1}$ agree (both non-zero and equal) or either is zero, compute $\sigma_{i2}$ [S2, unofficial].
    *   If $\sigma_{i0}$ and $\sigma_{i1}$ disagree (one positive, one negative), the face is not intersected; skip to next face [S2, G, unofficial].
5.  **Face Sign Determination:** Determine the face sign $\sigma_i$:
    *   If $\sigma_{i0} \neq 0$, $\sigma_i = \sigma_{i0}$ [S2, unofficial].
    *   Else if $\sigma_{i1} \neq 0$, $\sigma_i = \sigma_{i1}$ [S2, unofficial].
    *   Else $\sigma_i = \sigma_{i2}$ [S2, unofficial].
6.  **Face Classification:**
    *   If $\sigma_i \neq 0$ and $(\sigma_{i2} == \sigma_i \text{ or } \sigma_{i2} == 0)$:
        *   If $F_{enter}$ is nil and all signs are $\ge 0$, set $F_{enter} = F_i$ [S2, unofficial].
        *   Else if $F_{leave}$ is nil and all signs are $\le 0$, set $F_{leave} = F_i$ [S2, unofficial].
7.  **Post-First-Face Optimization:**
    *   If one face (e.g., $F_{enter}$) is found, edges shared with it have conforming signs and need not be tested again [S2, unofficial].
    *   Remaining edges must satisfy $\sigma_{ij} \le 0$ if $F_{leave}$ is not found, or $\sigma_{ij} \ge 0$ if $F_{enter}$ is not found [S2, unofficial].
8.  **Two Faces Left Optimization:**
    *   If $F_{leave}$ must be determined between $F_1(V_2 V_3 V_0)$ and $F_0(V_3 V_2 V_1)$, check the common edge $V_2 V_3$ [S2, unofficial].
    *   If $\pi_r \odot \pi_{V_2 V_3} < 0$, then $F_{leave} = F_1$ [S2, unofficial].
    *   Otherwise $F_{leave} = F_0$ (unless all $\sigma_{0j}$ are zero, in which case $F_{leave} = F_1$) [S2, unofficial].
9.  **Termination:** If $F_{enter}$ and $F_{leave}$ are both determined, terminate [S2, unofficial].
10. **Barycentric Coordinates:** For the entry and exit faces, compute scaled barycentric coordinates $w_i$ using Plücker values:
    $$ w_i = \pi_r \odot \pi_{e_i} $$
    Normalize to get barycentric coordinates $u_i$:
    $$ u_i = \frac{w_i}{\sum_{k=0}^{3} w_k} $$
    Note: The sum is over the three edges of the specific face [S1].
11. **Intersection Points:** Compute Cartesian coordinates of intersection point $P_k$:
    $$ P_k = u_0^k V_0 + u_1^k V_1 + u_2^k V_2 $$
    where $u^k$ are the barycentric coordinates for the $k$-th intersection point ($k=enter, leave$) [S2, unofficial].
12. **Parametric Distance:** Solve for $t_k$ using a coordinate of $P_k$ where $L$ is non-zero:
    $$ P_k = P + t_k L $$
    [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def plucker_ray_tet_intersection(ray_origin, ray_dir, tet_vertices):
    # ray_origin: (x, y, z)
    # ray_dir: (x, y, z)
    # tet_vertices: [V0, V1, V2, V3], each V is (x, y, z)
    
    def cross(a, b):
        return (a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0])
    
    def dot(a, b):
        return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]
    
    def sign(x):
        if x > 0: return 1
        if x < 0: return -1
        return 0

    def permuted_inner_product(p_r, q_r, p_e, q_e):
        return dot(p_r, q_e) + dot(q_r, p_e)

    p_r = ray_dir
    q_r = cross(ray_dir, ray_origin)
    
    V = tet_vertices
    def edge_plucker(A, B):
        d = (B[0]-A[0], B[1]-A[1], B[2]-A[2])
        m = cross(d, A)
        return d, m

    pi_01 = edge_plucker(V[0], V[1])
    pi_10 = edge_plucker(V[1], V[0])
    pi_12 = edge_plucker(V[1], V[2])
    pi_21 = edge_plucker(V[2], V[1])
    pi_20 = edge_plucker(V[2], V[0])
    pi_02 = edge_plucker(V[0], V[2])
    pi_03 = edge_plucker(V[0], V[3])
    pi_30 = edge_plucker(V[3], V[0])
    pi_13 = edge_plucker(V[1], V[3])
    pi_31 = edge_plucker(V[3], V[1])
    pi_23 = edge_plucker(V[2], V[3])
    pi_32 = edge_plucker(V[3], V[2])

    faces = [
        (3, [pi_01, pi_12, pi_20]),
        (2, [pi_10, pi_03, pi_31]),
        (1, [pi_23, pi_30, pi_02]),
        (0, [pi_32, pi_21, pi_13])
    ]
    
    F_enter = None
    F_leave = None
    enter_edges = None
    leave_edges = None
    
    for idx, edges in faces:
        pi_e0 = edges[0]
        pi_e1 = edges[1]
        pi_e2 = edges[2]
        
        prod0 = permuted_inner_product(p_r, q_r, pi_e0[0], pi_e0[1])
        prod1 = permuted_inner_product(p_r, q_r, pi_e1[0], pi_e1[1])
        
        sign0 = sign(prod0)
        sign1 = sign(prod1)
        
        if sign0 == sign1 or sign0 == 0 or sign1 == 0:
            prod2 = permuted_inner_product(p_r, q_r, pi_e2[0], pi_e2[1])
            sign2 = sign(prod2)
        else:
            continue
            
        if sign0 != 0:
            sigma = sign0
        elif sign1 != 0:
            sigma = sign1
        else:
            sigma = sign2
            
        if sigma != 0 and (sign2 == sigma or sign2 == 0):
            if F_enter is None and sign0 >= 0 and sign1 >= 0 and sign2 >= 0:
                F_enter = idx
                enter_edges = edges
            elif F_leave is None and sign0 <= 0 and sign1 <= 0 and sign2 <= 0:
                F_leave = idx
                leave_edges = edges
                
        if F_enter is not None and F_leave is not None:
            break
            
    if F_enter is None or F_leave is None:
        return None
        
    # Compute barycentric coordinates and intersection points
    def compute_intersection(edges, is_enter):
        w = []
        for pe in edges:
            w.append(permuted_inner_product(p_r, q_r, pe[0], pe[1]))
        w_sum = sum(w)
        if w_sum == 0:
            return None, None
        u = [wi / w_sum for wi in w]
        P_k = (u[0]*V[0][0] + u[1]*V[1][0] + u[2]*V[2][0],
               u[0]*V[0][1] + u[1]*V[1][1] + u[2]*V[2][1],
               u[0]*V[0][2] + u[1]*V[1][2] + u[2]*V[2][2])
        # Solve for t_k using a non-zero component of ray_dir
        for k in range(3):
            if ray_dir[k] != 0:
                t_k = (P_k[k] - ray_origin[k]) / ray_dir[k]
                return P_k, t_k
        return P_k, None

    P_enter, t_enter = compute_intersection(enter_edges, True)
    P_leave, t_leave = compute_intersection(leave_edges, False)
    
    return F_enter, F_leave, P_enter, P_leave, t_enter, t_leave
```

**Parameters:**
*   **Memory overhead:** 16 bytes per tetrahedron for connectivity [S1].
*   **Precomputed quantities:** $\pi_r \odot \pi_{ij}$ and $\sigma_{ij}$ for all $i, j$ are computed once per ray in spatial data structure scenarios [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Sliding Interfaces:** Interior faces must be shared by exactly two tetrahedra; sliding interfaces (sharing only part of a face) are not allowed [S1].
*   **Edge Exit:** If the ray leaves the last tetrahedron through an edge, the next tetrahedron cannot be determined directly via connectivity [S2, unofficial].
*   **Zero Signs:** Any two (but not all three) signs $\sigma_{ij}$ may be zero. The logic for determining the face sign $\sigma_i$ must handle cases where $\sigma_{i0}$ and $\sigma_{i1}$ are both zero, falling back to $\sigma_{i2}$ [S2, unofficial].
*   **Edge Orientation:** The sign of the permuted inner product changes if the edge direction is reversed ($\pi_r \odot \pi_{V_0 V_1} = -(\pi_r \odot \pi_{V_1 V_0})$) [S2, unofficial].
*   **Default Leave Face:** If all three $\sigma_{0j}$ are zero during leave face determination, $F_{leave}$ defaults to $F_1$ [S2, unofficial].

## Subtopics

### [Hexahedral Mesh Traversal (Plücker Space)](1889-hexahedral-mesh-traversal-pl-cker-space.md)
A traversal method for hexahedral meshes that uses Plücker coordinates to determine the exit face of a ray intersecting a hexahedral cell, requiring face decomposition into triangles for interpolation. It sets itself apart from its sibling [Hexahedral Mesh Traversal (Bilinear Patch-Extension)](1904-hexahedral-mesh-traversal-bilinear-patch-extension.md) by relying on triangular decomposition for interpolation, which introduces parametric discontinuities along the diagonal but allows unambiguous face determination.

### Tetrahedral Volume Data Ray Tracing
Building and traversing an implicit bounding volume hierarchy (BVH) on top of a tetrahedral mesh to isosurface unstructured simplicial volume data [S3, unofficial]. It employs a specialized intersection algorithm inspired by the marching tetrahedra extraction method for the local implicit surfaces of first-order finite elements, performs a packet-polygon intersection test, and uses coherent BVH traversal and SSE enhancements for intersection [S3, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   The optimized algorithm reduces the average number of Plücker tests to 2.67 per tetrahedron [S1].
*   Memory overhead is 16 bytes per tetrahedron for connectivity [S1].
*   Pre-computing Plücker coordinates doubles memory requirements, so on-the-fly conversion is preferred [S1].

**Measured Performance:**
*   Baseline performance: 15.5 million tetrahedra per second on a Dual-core Opteron with 2GHz (both CPUs used) [S1].
*   Performance gain: 55% average increase in raw performance compared to the naive approach of testing all three exit faces independently [S1].
*   Tests on an Intel Celeron PC at 900MHz showed the optimized Plücker algorithm outperforms Haines' algorithm and Moller/Trumbore-based algorithms in all tested cases [S2, unofficial].
*   Haines' algorithm performs comparatively better when few ray-tetrahedron pairs intersect because it can sometimes decide there is no intersection without examining all four faces [S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Directly provides scaled barycentric coordinates, avoiding division until normalization [S1]. Robust and efficient, requiring few floating point operations and no division [S2, unofficial].
*   **Limitations:** Cannot apply parametric distance restrictions (e.g., max ray length) early; these must be applied as post-processing [S2, unofficial].

**When to Use:**
*   Use for ray tracing of tetrahedral meshes where performance and barycentric coordinate computation are critical [S2, unofficial].
*   Avoid if early rejection of non-intersecting rays is the primary concern and few intersections are expected (Haines' algorithm may be better) [S2, unofficial].

## Background
**Origin:**
The method is based on the ray-tetrahedron intersection algorithm introduced by Platis et al. [PT03] [S1]. Plücker coordinates were used to specify directed lines in three-dimensional space [Eri97]. The approach extends and accelerates an approach suggested by Garrity [Gar90], who used a grid as an acceleration structure and connectivity information to traverse cells incrementally by calculating plane intersections [S1].

**History and Development:**
The Plücker-based ray-tetrahedron intersection algorithm was presented by Nikos Platis and Theoharis Theoharis [S2, unofficial]. It was shown to outperform Haines' algorithm and Moller/Trumbore-based algorithms [S2, unofficial]. The method is used in ray tracing of triangular meshes (cited in [AC97] and [SF01] as the equivalent 4x4 determinant method) and tetrahedral meshes [S2, unofficial].

**Usage:**
Used in ray tracing of triangular meshes and tetrahedral meshes [S2, unofficial]. It is also used in the traversal of tetrahedral meshes for direct volume rendering [S1].

## Sources
[S1] Fast Ray Traversal of Tetrahedral and Hexahedral Meshes for Direct Volume Rendering, https://doi.org/10.2312/vissym/eurovis06/235-242, fetched 2026-10-24
[S2] Fast Ray–Tetrahedron Intersection using Plücker ..., https://users.uop.gr/~nplatis%20/files/PlatisTheoharisRayTetra.pdf, fetched 2026-10-29 (unofficial)
[S3] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
