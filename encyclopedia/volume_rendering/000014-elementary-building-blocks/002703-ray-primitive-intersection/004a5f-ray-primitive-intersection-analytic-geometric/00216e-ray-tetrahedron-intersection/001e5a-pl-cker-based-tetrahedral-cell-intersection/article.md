# Plücker-based Tetrahedral Cell Intersection

Also called: Plücker ray-tetrahedron intersection, Plücker coordinates ray-tetrahedron test, Ray-Tetrahedron Intersection using Plücker Coordinates, Ray-Tetrahedron Intersection with Plücker Coordinates (Optimized)

## Overview
This method determines the entry and exit points of a ray intersecting a tetrahedron by representing the ray and tetrahedron edges as Plücker coordinates, computing permuted inner products to classify faces, and deriving scaled barycentric coordinates directly from these products [S3, unofficial]. It sets itself apart from plane-intersection approaches by avoiding explicit plane calculations and divisions until the final normalization step, and from ray-triangle decomposition methods (like Moller-Trumbore) by leveraging shared edge computations and early termination logic specific to the tetrahedron's topology [S1][S3, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: Ray defined by origin $\mathbf{P}$ and direction $\mathbf{L}$ (infinite directed line); Tetrahedron defined by vertices $\mathbf{V}_0, \mathbf{V}_1, \mathbf{V}_2, \mathbf{V}_3$.
- **Output**: Entry face $F_{enter}$, exit face $F_{leave}$, entry point $\mathbf{P}_{enter}$, exit point $\mathbf{P}_{leave}$, and their respective barycentric coordinates.

**Data Structures**
- **Ray**: Defined by point $\mathbf{P}$ and vector $\mathbf{L}$.
- **Tetrahedron Vertices**: $\mathbf{V}_0, \mathbf{V}_1, \mathbf{V}_2, \mathbf{V}_3$.
- **Faces**: Indexed by the opposite vertex.
  - $F_3$: Vertices $\mathbf{V}_0, \mathbf{V}_1, \mathbf{V}_2$.
  - $F_2$: Vertices $\mathbf{V}_1, \mathbf{V}_0, \mathbf{V}_3$.
  - $F_1$: Vertices $\mathbf{V}_2, \mathbf{V}_3, \mathbf{V}_0$.
  - $F_0$: Vertices $\mathbf{V}_3, \mathbf{V}_2, \mathbf{V}_1$.
- **Face Vertices**: For face $F_i$, vertices are denoted $\mathbf{V}_{i0}, \mathbf{V}_{i1}, \mathbf{V}_{i2}$. Example: For $F_2$, $\mathbf{V}_{20}=\mathbf{V}_1, \mathbf{V}_{21}=\mathbf{V}_0, \mathbf{V}_{22}=\mathbf{V}_3$.
- **Edges**: Edges of face $F_i$ are subscripted by the index of the opposite vertex.
  - $e_{i0}$: Edge $\mathbf{V}_{i1}\mathbf{V}_{i2}$.
  - $e_{i1}$: Edge $\mathbf{V}_{i2}\mathbf{V}_{i0}$.
  - $e_{i2}$: Edge $\mathbf{V}_{i0}\mathbf{V}_{i1}$.
- **Orientation**: The tetrahedron is assumed oriented such that the outward-pointing normal of face $F_i$ is directed away from vertex $\mathbf{V}_i$ [S3, unofficial].

**Plücker Coordinates**
A directed line is represented by a six-vector $\boldsymbol{\pi} = \{\mathbf{p} : \mathbf{q}\}$, where $\mathbf{p}$ is the direction vector and $\mathbf{q} = \mathbf{p} \times \mathbf{o}$ is the moment vector, with $\mathbf{o}$ being any point on the line [S1].
- **Ray Plücker**: $\boldsymbol{\pi}_r = \{\mathbf{L} : \mathbf{L} \times \mathbf{P}\} = \{\mathbf{p}_r : \mathbf{q}_r\}$ [S1].
- **Edge Plücker**: For an edge from $\mathbf{A}$ to $\mathbf{B}$, $\boldsymbol{\pi}_{AB} = \{\mathbf{B}-\mathbf{A} : (\mathbf{B}-\mathbf{A}) \times \mathbf{A}\}$ [S1].

**Permuted Inner Product**
The relative orientation of two lines $r$ and $s$ is determined by the permuted inner product:
$$ \boldsymbol{\pi}_r \odot \boldsymbol{\pi}_s = \mathbf{p}_r \cdot \mathbf{q}_s + \mathbf{q}_r \cdot \mathbf{p}_s $$
- If $\boldsymbol{\pi}_r \odot \boldsymbol{\pi}_s > 0$, line $s$ passes counterclockwise around line $r$.
- If $\boldsymbol{\pi}_r \odot \boldsymbol{\pi}_s < 0$, line $s$ passes clockwise around line $r$.
- If $\boldsymbol{\pi}_r \odot \boldsymbol{\pi}_s = 0$, lines intersect or are parallel [S3, unofficial].
- Note: $\boldsymbol{\pi}_r \odot \boldsymbol{\pi}_{AB} = -(\boldsymbol{\pi}_r \odot \boldsymbol{\pi}_{BA})$ [S3, unofficial].

**Algorithm Steps**
1.  **Initialize**: Set $F_{enter} = \text{nil}$, $F_{leave} = \text{nil}$.
2.  **Loop Faces**: Iterate $i$ from 3 down to 0 (faces $F_3, F_2, F_1, F_0$).
3.  **Compute Signs**: For face $F_i$, compute signs $\sigma_{i0}, \sigma_{i1}, \sigma_{i2}$ for its three edges $e_{i0}, e_{i1}, e_{i2}$:
    $$ \sigma_{ij} = \text{sign}(\boldsymbol{\pi}_r \odot \boldsymbol{\pi}_{e_{ij}}) $$
    Where $\text{sign}(x)$ is $1$ if $x>0$, $0$ if $x=0$, and $-1$ if $x<0$ [S3, unofficial].
4.  **Optimization (Early Exit in Face Test)**:
    - Compute $\sigma_{i0}$ and $\sigma_{i1}$.
    - If $\sigma_{i0} \neq \sigma_{i1}$ and neither is zero, the face is not intersected (signs disagree). Skip computing $\sigma_{i2}$ and proceed to next face.
    - Otherwise, compute $\sigma_{i2}$ if not already determined.
5.  **Determine Face Sign**:
    - If $\sigma_{i0} \neq 0$, let $\sigma_i = \sigma_{i0}$.
    - Else if $\sigma_{i1} \neq 0$, let $\sigma_i = \sigma_{i1}$.
    - Else $\sigma_i = \sigma_{i2}$.
6.  **Classify Face**:
    - If $\sigma_i \neq 0$ and $(\sigma_{i2} == \sigma_i \text{ or } \sigma_{i2} == 0)$:
        - If $\sigma_i > 0$, set $F_{enter} = F_i$.
        - If $\sigma_i < 0$, set $F_{leave} = F_i$.
7.  **Post-First-Face Optimization**:
    - If one face (e.g., $F_{enter}$) is found, edges shared with it have conforming signs and need not be tested again.
    - For remaining faces, if $F_{leave}$ is not found, all edge signs must satisfy $\sigma_{ij} \le 0$. If $F_{enter}$ is not found, all edge signs must satisfy $\sigma_{ij} \ge 0$.
8.  **Two Faces Left Optimization**:
    - If $F_{leave}$ must be determined between $F_1(\mathbf{V}_2\mathbf{V}_3\mathbf{V}_0)$ and $F_0(\mathbf{V}_3\mathbf{V}_2\mathbf{V}_1)$, check the common edge $\mathbf{V}_2\mathbf{V}_3$.
    - If $\boldsymbol{\pi}_r \odot \boldsymbol{\pi}_{\mathbf{V}_2\mathbf{V}_3} < 0$, then $F_{leave} = F_1$.
    - Otherwise, $F_{leave} = F_0$ (unless all $\sigma_{0j}$ are zero, in which case $F_{leave} = F_1$) [S3, unofficial].
9.  **Termination**: If both $F_{enter}$ and $F_{leave}$ are determined, terminate.
10. **Barycentric Coordinates**:
    - Compute scaled barycentric coordinates for the intersection points on the faces:
      $$ w_i = \boldsymbol{\pi}_r \odot \boldsymbol{\pi}_{e_i} $$
      $$ u_i = \frac{w_i}{\sum_{k=0}^{2} w_k} $$
      (Note: The sum is over the three edges of the specific face, not all four faces) [S1][S3, unofficial].
    - Compute Cartesian coordinates of intersection point $\mathbf{P}_k$:
      $$ \mathbf{P}_k = u_0 \mathbf{V}_0 + u_1 \mathbf{V}_1 + u_2 \mathbf{V}_2 $$
    - Solve for parametric distance $t_k$ using a coordinate of $\mathbf{P}_k$ where $\mathbf{L}$ is non-zero:
      $$ \mathbf{P}_k = \mathbf{P} + t_k \mathbf{L} $$

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def plucker_ray_tetrahedron_intersection(ray_origin, ray_dir, tet_vertices):
    """
    ray_origin: tuple (x, y, z)
    ray_dir: tuple (x, y, z)
    tet_vertices: list of 4 tuples [(x0,y0,z0), (x1,y1,z1), (x2,y2,z2), (x3,y3,z3)]
    Returns: (F_enter, F_leave, P_enter, P_leave) or None if no intersection
    """
    # Plücker coordinates for ray
    p_r = ray_dir
    q_r = cross(ray_dir, ray_origin)
    
    # Precompute Plücker coords for all edges if needed, or compute on fly
    # Edge notation: e_ij is edge from V_ij1 to V_ij2 for face i
    # Face indices: 3, 2, 1, 0
    # Face 3: V0, V1, V2 -> edges: V1->V2, V2->V0, V0->V1
    # Face 2: V1, V0, V3 -> edges: V0->V3, V3->V1, V1->V0
    # Face 1: V2, V3, V0 -> edges: V3->V0, V0->V2, V2->V3
    # Face 0: V3, V2, V1 -> edges: V2->V1, V1->V3, V3->V2
    
    F_enter = None
    F_leave = None
    
    # Face vertex mapping: Fi vertices are Vi0, Vi1, Vi2
    # F3: V0, V1, V2
    # F2: V1, V0, V3
    # F1: V2, V3, V0
    # F0: V3, V2, V1
    face_vertices = [
        [tet_vertices[0], tet_vertices[1], tet_vertices[2]], # F3
        [tet_vertices[1], tet_vertices[0], tet_vertices[3]], # F2
        [tet_vertices[2], tet_vertices[3], tet_vertices[0]], # F1
        [tet_vertices[3], tet_vertices[2], tet_vertices[1]]  # F0
    ]
    
    for i in [3, 2, 1, 0]:
        if F_enter is not None and F_leave is not None:
            break
            
        verts = face_vertices[i]
        # Edges for face i: e_i0 (V_i1->V_i2), e_i1 (V_i2->V_i0), e_i2 (V_i0->V_i1)
        # Note: The paper defines edges e_i0, e_i1, e_i2 corresponding to vertices V_i1, V_i2, V_i0 in order
        # e_i0: V_i1 -> V_i2
        # e_i1: V_i2 -> V_i0
        # e_i2: V_i0 -> V_i1
        
        v0 = verts[0] # V_i0
        v1 = verts[1] # V_i1
        v2 = verts[2] # V_i2
        
        # Edge 0: v1 -> v2
        dir0 = sub(v2, v1)
        q0 = cross(dir0, v1)
        pi0 = dot(p_r, q0) + dot(q_r, dir0)
        sigma0 = sign(pi0)
        
        # Edge 1: v2 -> v0
        dir1 = sub(v0, v2)
        q1 = cross(dir1, v2)
        pi1 = dot(p_r, q1) + dot(q_r, dir1)
        sigma1 = sign(pi1)
        
        # Check if sigma0 and sigma1 agree or are zero
        if sigma0 != sigma1 and sigma0 != 0 and sigma1 != 0:
            continue # Face not intersected
            
        # Edge 2: v0 -> v1
        dir2 = sub(v1, v0)
        q2 = cross(dir2, v0)
        pi2 = dot(p_r, q2) + dot(q_r, dir2)
        sigma2 = sign(pi2)
        
        # Determine face sign sigma_i
        if sigma0 != 0:
            sigma_i = sigma0
        elif sigma1 != 0:
            sigma_i = sigma1
        else:
            sigma_i = sigma2
            
        # Classify face
        if sigma_i != 0 and (sigma2 == sigma_i or sigma2 == 0):
            if sigma_i > 0:
                F_enter = i
            else:
                F_leave = i
                
    if F_enter is None or F_leave is None:
        return None
        
    # Compute barycentric coordinates for F_enter
    # Re-calculate or store signs for F_enter
    # ... (Implementation detail: store w_i values during sign computation)
    
    # For simplicity, assume we recompute or stored w_i
    # w_i = pi_r . pi_ei
    # u_i = w_i / sum(w)
    
    # Compute intersection points
    # P_enter = u0*V0 + u1*V1 + u2*V2 for F_enter
    # P_leave = u0*V0 + u1*V1 + u2*V2 for F_leave
    
    # Solve for t
    # P = P_origin + t * L
    
    return (F_enter, F_leave, P_enter, P_leave)

def sign(x):
    if x > 0: return 1
    elif x < 0: return -1
    else: return 0

def sub(a, b):
    return (a[0]-b[0], a[1]-b[1], a[2]-b[2])

def cross(a, b):
    return (a[1]*b[2]-a[2]*b[1], a[2]*b[0]-a[0]*b[2], a[0]*b[1]-a[1]*b[0])

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]
```

**Edge Cases and Pitfalls**
- **Sliding Interfaces**: Interior faces must be shared by exactly two tetrahedra. Sliding interfaces (tetrahedra sharing only part of a face) are not allowed for the connectivity optimization to work [S1].
- **Numerical Ambiguity**: Ambiguity at shared edges due to numerical issues can introduce holes or overlaps. An additional Plücker test can resolve this [S2, unofficial].
- **Edge Exit**: If the ray leaves the last tetrahedron through an edge, the next tetrahedron cannot be determined directly via connectivity [S3, unofficial].
- **Zero Signs**: Any two (but not all three) signs $\sigma_{ij}$ may be zero. The logic for determining the face sign $\sigma_i$ must handle cases where $\sigma_{i0}$ and $\sigma_{i1}$ are both zero, falling back to $\sigma_{i2}$ [S3, unofficial].
- **Edge Orientation**: The sign of the permuted inner product changes if the edge direction is reversed. This must be handled when reusing computations for shared edges [S3, unofficial].

## Subtopics

### [Hexahedral Mesh Traversal (Plücker Space)](1889-hexahedral-mesh-traversal-pl-cker-space.md)
A traversal method for hexahedral meshes that uses Plücker coordinates to determine the exit face of a ray intersecting a hexahedral cell, requiring face decomposition into triangles for interpolation. It sets itself apart from its sibling [Hexahedral Mesh Traversal (Bilinear Patch-Extension)](1904-hexahedral-mesh-traversal-bilinear-patch-extension.md) by relying on triangular decomposition for interpolation, which introduces parametric discontinuities along the diagonal but allows unambiguous face determination.

## Operating characteristics
- **Time Complexity**: The optimized algorithm reduces the average number of Plücker tests to 2.67 per tetrahedron, compared to 3-9 tests in the naive approach [S1].
- **Memory Complexity**: Requires 16 additional bytes per tetrahedron for connectivity information [S1]. Pre-computing Plücker coordinates doubles memory requirements, so on-the-fly conversion is preferred [S1].
- **Performance**: Achieves 15.5 million tetrahedra per second on a Dual-core Opteron with 2GHz [S1]. Represents a 55% average increase in raw performance compared to the naive approach [S1].
- **Strengths**: Directly provides scaled barycentric coordinates from Plücker tests, avoiding division until the final normalization step [S1]. Outperforms Haines' algorithm and Moller/Trumbore-based algorithms in all tested cases, including those with precomputed quantities [S3, unofficial].
- **Limitations**: Cannot apply parametric distance restrictions (e.g., max ray length) early; these must be applied as post-processing [S3, unofficial]. Haines' algorithm performs comparatively better when few ray-tetrahedron pairs intersect because it can sometimes decide there is no intersection without examining all four faces [S3, unofficial].
- **When to Use**: When high-performance ray-tetrahedron intersection is required, especially in the context of tetrahedral mesh traversal where connectivity can be exploited.
- **When Not to Use**: When the mesh has sliding interfaces or when early parametric distance culling is critical and the overhead of post-processing is unacceptable.

## Background
The method was introduced by Platis and Theoharis in 2003 [S3, unofficial]. It builds on the properties of Plücker coordinates for representing directed lines in 3D space [S1]. The approach extends Garrity's grid-based traversal method [Gar90] and Wald's kd-tree implementation for initial tetrahedron search [Wal04]. It is used in ray tracing of triangular meshes (cited in [AC97] and [SF01] as equivalent 4x4 determinant method) and tetrahedral meshes [S3, unofficial].

## Sources
[S1] Fast Ray Traversal of Tetrahedral and Hexahedral Meshes for Direct Volume Rendering, https://doi.org/10.2312/vissym/eurovis06/235-242, fetched 2026-10-24
[S2] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S3] Fast Ray–Tetrahedron Intersection using Plücker ..., https://users.uop.gr/~nplatis%20/files/PlatisTheoharisRayTetra.pdf, fetched 2026-10-29 (unofficial)
[S4] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
