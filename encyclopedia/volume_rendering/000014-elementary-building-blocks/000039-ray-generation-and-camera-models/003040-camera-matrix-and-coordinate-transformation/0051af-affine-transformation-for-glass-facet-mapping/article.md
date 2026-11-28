# Affine Transformation for Glass Facet Mapping

Also called: Mesh to Source Panel Frame Transformation, Mesh to Source Panel Mapping, Multi-Piece Object Transformation and Intersection, Rigid Transformation for Glass Facets, Transformation T

## Overview
This method models complex, multi-piece glass objects (such as ancient stained glass windows) by mapping UV coordinates from a mesh face to a specific 3D section of a source panel [S1]. Unlike standard texture mapping, it maps the full physical properties of the source panel—including geometric displacement, bubble distribution, and varying index of refraction—onto a 3D facet using a rigid affine transformation [S1]. It solves the problem of intersecting rays with composite glass objects where each facet corresponds to a distinct region of a physical glass panel, allowing for accurate caustic estimation and transmission lighting in a unified local space [S1].

## How it works
**Inputs:**
*   A ray in world space $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ [G].
*   A mesh representing the composite glass object [S1].
*   For each mesh face: UV coordinates $(u, v)$ at the intersection point, and the face tangent frame basis vectors $\{\mathbf{e}_x, \mathbf{e}_y, \mathbf{e}_z\}$ (tangent, bitangent, normal) [S1].
*   Source panel parameters: Origin $O$, basis vectors $\{\mathbf{e}'_x, \mathbf{e}'_y, \mathbf{e}'_z\}$, and dimensions $L$ (horizontal) and $H$ (vertical) [S1].

**Outputs:**
*   The intersection point $p_0$ on the source panel in its local frame [S1].
*   The transformed intersection point $p$ in the mesh frame (or world space) [S1].

**Data Structures:**
*   **Transformation Matrix $T$**: A $4 \times 4$ affine transformation matrix [S1].
    *   Layout:
        $$
        T = \begin{bmatrix}
        \mathbf{e}'_x & \mathbf{e}'_y & \mathbf{e}'_z & O + \mathbf{k} \\
        0 & 0 & 0 & 1
        \end{bmatrix} \begin{bmatrix}
        \mathbf{e}_x & \mathbf{e}_y & \mathbf{e}_z & \mathbf{p} \\
        0 & 0 & 0 & 1
        \end{bmatrix}^{-1}
        $$
    *   The matrix maps the face tangent frame $R=\{p|\mathbf{e}_x, \mathbf{e}_y, \mathbf{e}_z\}$ to the source panel frame $R_0=\{O|\mathbf{e}'_x, \mathbf{e}'_y, \mathbf{e}'_z\}$ and translates by $\mathbf{k}$ [S1].
    *   For orthonormal bases, the inverse of the basis submatrix equals its transpose [G].

**Steps:**
1.  **Mesh Intersection**: Intersect the ray with the mesh to find the intersection point $p$ and the face's tangent, bitangent, and normal vectors $\{\mathbf{e}_x, \mathbf{e}_y, \mathbf{e}_z\}$ at $p$ [S1]. Retrieve the UV coordinates $(u, v)$ at $p$ [S1].
2.  **Construct Translation Vector $\mathbf{k}$**: Calculate the translation offset based on UV coordinates and panel dimensions:
    $$ \mathbf{k} = u L \mathbf{e}'_x + v H \mathbf{e}'_y $$
    Where $L$ and $H$ are the horizontal and vertical dimensions of the source panel [S1].
3.  **Construct Transformation Matrix $T$**: Build the rigid transformation that maps the local mesh frame $R$ to the source panel frame $R_0$ [S1].
    *   Let $M_R$ be the matrix with columns $\mathbf{e}_x, \mathbf{e}_y, \mathbf{e}_z$ and translation $\mathbf{p}$ [S1].
    *   Let $M_{R0}$ be the matrix with columns $\mathbf{e}'_x, \mathbf{e}'_y, \mathbf{e}'_z$ and translation $O + \mathbf{k}$ [S1].
    *   Compute $T = M_{R0} M_R^{-1}$ [S1].
4.  **Transform Ray**: Transform the ray origin $\mathbf{o}$ and direction $\mathbf{d}$ from world space to the source panel frame $R_0$ using $T$ [S1].
    *   $\mathbf{o}' = T \cdot [\mathbf{o}, 1]^T$
    *   $\mathbf{d}' = T \cdot [\mathbf{d}, 0]^T$
5.  **Intersect Source Panel**: Ray march along the transformed ray $\mathbf{r}'(t) = \mathbf{o}' + t\mathbf{d}'$ to find the intersection point $p_0$ with the source panel in its local space [S1].
6.  **Transform Back**: Transform $p_0$ back to the mesh frame $R$ (or world space) using $T^{-1}$ [S1].
    *   $p_{final} = T^{-1} \cdot [p_0, 1]^T$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transform_glass_facet(ray_origin, ray_direction, mesh_face, source_panel):
    # mesh_face: dict with keys 'p', 'u', 'v', 'ex', 'ey', 'ez'
    # source_panel: dict with keys 'O', 'ex_p', 'ey_p', 'ez_p', 'L', 'H'
    p = mesh_face['p']
    u, v = mesh_face['u'], mesh_face['v']
    ex, ey, ez = mesh_face['ex'], mesh_face['ey'], mesh_face['ez']
    O = source_panel['O']
    ex_p, ey_p, ez_p = source_panel['ex_p'], source_panel['ey_p'], source_panel['ez_p']
    L, H = source_panel['L'], source_panel['H']

    # 1. Compute translation vector k
    k = [u * L * ex_p[i] + v * H * ey_p[i] for i in range(3)]

    # 2. Compute rotation matrix R (3x3) mapping mesh basis to panel basis
    # R = [ex_p, ey_p, ez_p] * [ex, ey, ez]^T
    R = [[0.0]*3 for _ in range(3)]
    for i in range(3):
        for j in range(3):
            R[i][j] = ex_p[i]*ex[j] + ey_p[i]*ey[j] + ez_p[i]*ez[j]

    # 3. Compute translation vector t for T
    # t = O + k - R @ p
    t = [O[i] + k[i] - sum(R[i][j]*p[j] for j in range(3)) for i in range(3)]

    # 4. Construct 4x4 matrix T
    T = [row[:] for row in R]
    T[0].append(t[0]); T[1].append(t[1]); T[2].append(t[2])
    T.append([0.0, 0.0, 0.0, 1.0])

    # 5. Transform ray origin (w=1) and direction (w=0)
    o_prime = [sum(T[i][j] * (ray_origin[j] if j < 3 else 1.0) for j in range(4)) for i in range(3)]
    d_prime = [sum(T[i][j] * (ray_direction[j] if j < 3 else 0.0) for j in range(4)) for i in range(3)]

    # 6. Intersect with source panel plane z=0 in R0 frame
    denom = d_prime[2]
    if abs(denom) < 1e-9:
        return None
    t_int = -o_prime[2] / denom
    p0_local = [o_prime[i] + t_int * d_prime[i] for i in range(3)]

    # 7. Transform back to world space
    # Compute T inverse (rigid body inverse: R^T, -R^T @ t)
    R_T = [[R[j][i] for j in range(3)] for i in range(3)]
    t_inv = [-sum(R_T[i][j]*t[j] for j in range(3)) for i in range(3)]
    T_inv = [row[:] for row in R_T]
    T_inv[0].append(t_inv[0]); T_inv[1].append(t_inv[1]); T_inv[2].append(t_inv[2])
    T_inv.append([0.0, 0.0, 0.0, 1.0])

    p_world = [sum(T_inv[i][j] * (p0_local[j] if j < 3 else 1.0) for j in range(4)) for i in range(3)]
    return p_world
```

**Parameters:**
*   $u, v$: UV coordinates at the intersection point $p$ [S1].
*   $L, H$: Horizontal and vertical dimensions of the source panel [S1].
*   $\mathbf{e}_x, \mathbf{e}_y, \mathbf{e}_z$: Tangent, bitangent, and normal vectors of the mesh face [S1].
*   $\mathbf{e}'_x, \mathbf{e}'_y, \mathbf{e}'_z$: Basis vectors of the source panel frame [S1].

**Edge Cases and Pitfalls:**
*   **Coplanarity**: The method requires that all vertices of a given facet remain coplanar to represent arbitrary 3D shapes [S1]. Non-coplanar facets will result in incorrect mapping [S1].
*   **Precision**: Transforming rays back and forth between frames can introduce floating-point errors [G].
*   **Basis Orthogonality**: The basis vectors $\{\mathbf{e}_x, \mathbf{e}_y, \mathbf{e}_z\}$ must be orthonormal for the inverse to be simply the transpose [G]. If not, the full matrix inverse must be computed [G].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per facet intersection, dominated by matrix multiplication and ray-plane intersection [G].
*   **Memory Complexity**: $O(1)$ auxiliary memory per facet for storing the transformation matrix [G].
*   **Strengths**: Allows accurate modeling of complex, multi-piece glass objects with varying physical properties (bubbles, RIF) mapped from a single source panel [S1]. Enables unbiased iterative caustic estimation [S1].
*   **Limitations**: Requires coplanar facets [S1]. Limited to rigid transformations; cannot model flexible or deforming glass [S1].
*   **When to use**: When rendering ancient or stained glass windows composed of multiple flat glass fragments from a source panel [S1].
*   **When not to use**: For non-rigid glass, or when the glass panel is not flat (curved panels require different mapping) [G].

## Background
This method was introduced in the context of modeling old glass, specifically ancient windows composed of smaller fragments assembled with lead [S1]. It addresses the difficulty of making large flat glass sheets and exploits the variable depth of glass for artistic effects like color gradients [S1]. The technique is described in "Parabolic Sphere Tracing of Signed Distance Fields for Old Glass Modelling and Rendering" [S1].

## Sources
[S1] Parabolic Sphere Tracing of Signed Distance Fields for Old Glass Modelling and Rendering, https://doi.org/10.1109/tvcg.2025.3613853, fetched 2026-11-05
