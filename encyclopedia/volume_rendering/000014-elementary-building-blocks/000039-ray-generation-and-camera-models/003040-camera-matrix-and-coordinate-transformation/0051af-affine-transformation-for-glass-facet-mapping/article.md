# Affine Transformation for Glass Facet Mapping

Also called: Mesh to Source Panel Frame Transformation, Mesh to Source Panel Mapping, Multi-Piece Object Transformation and Intersection, Rigid Transformation for Glass Facets, Transformation T

## Overview
This method models complex, multi-piece glass objects (such as ancient stained glass windows) by mapping UV coordinates from a mesh face to a specific 3D section of a source panel. Unlike standard texture mapping, it maps the full physical properties of the source panel—including geometric displacement, bubble distribution, and varying index of refraction—onto a 3D facet using a rigid affine transformation [S1]. It solves the problem of intersecting rays with composite glass objects where each facet corresponds to a distinct region of a physical glass panel, allowing for accurate caustic estimation and transmission lighting in a unified local space [S1]. It differs from [NDC Ray Transformation](905-ndc-ray-transformation.md) by operating on local facet frames rather than global camera coordinates, and from [Arbitrary Source-Detector Geometry Projector](2975-arbitrary-source-detector-geometry-projector.md) by using a rigid body transformation specific to glass panel geometry rather than general projection matrices.

## How it works
**Inputs:**
*   A ray in world space $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$.
*   A mesh representing the composite glass object.
*   For each mesh face: UV coordinates $(u, v)$ at the intersection point, and the face tangent frame basis vectors $\{\mathbf{e}_x, \mathbf{e}_y, \mathbf{e}_z\}$ (tangent, bitangent, normal).
*   Source panel parameters: Origin $O$, basis vectors $\{\mathbf{e}'_x, \mathbf{e}'_y, \mathbf{e}'_z\}$, and dimensions $L$ (horizontal) and $H$ (vertical).

**Outputs:**
*   The intersection point $p_0$ on the source panel in its local frame.
*   The transformed intersection point $p$ in the mesh frame (or world space).

**Data Structures:**
*   **Transformation Matrix $T$**: A $4 \times 4$ affine transformation matrix.
    *   Layout:
        $$
        T = \begin{bmatrix}
        \mathbf{e}_x & \mathbf{e}_y & \mathbf{e}_z & \mathbf{p} \\
        0 & 0 & 0 & 1
        \end{bmatrix} \times \begin{bmatrix}
        \mathbf{e}'_x & \mathbf{e}'_y & \mathbf{e}'_z & \mathbf{k} \\
        0 & 0 & 0 & 1
        \end{bmatrix}^{-1} \quad \text{(Note: Source notation is ambiguous, see Formula section)}
        $$
    *   Correction based on source description: The source defines $T$ as mapping frame $R$ to $R_0$. The explicit matrix construction provided in the source is garbled. However, the text states $T$ maps the face tangent frame $R=\{p|\mathbf{e}_x, \mathbf{e}_y, \mathbf{e}_z\}$ to the source panel frame $R_0=\{O|\mathbf{e}'_x, \mathbf{e}'_y, \mathbf{e}'_z\}$ and translates by $\mathbf{k}$.
    *   Standard Rigid Body Construction:
        $$
        T = \begin{bmatrix}
        \mathbf{e}'_x & \mathbf{e}'_y & \mathbf{e}'_z & \mathbf{k} \\
        0 & 0 & 0 & 1
        \end{bmatrix} \begin{bmatrix}
        \mathbf{e}_x & \mathbf{e}_y & \mathbf{e}_z & \mathbf{p} \\
        0 & 0 & 0 & 1
        \end{bmatrix}^{-1}
        $$
        Where the inverse of the basis matrix is its transpose (since basis vectors are orthonormal).

**Steps:**
1.  **Mesh Intersection**: Intersect the ray with the mesh to find the intersection point $p$ and the face's tangent, bitangent, and normal vectors $\{\mathbf{e}_x, \mathbf{e}_y, \mathbf{e}_z\}$ at $p$. Retrieve the UV coordinates $(u, v)$ at $p$.
2.  **Construct Translation Vector $\mathbf{k}$**: Calculate the translation offset based on UV coordinates and panel dimensions:
    $$ \mathbf{k} = u L \mathbf{e}'_x + v H \mathbf{e}'_y $$
    Where $L$ and $H$ are the horizontal and vertical dimensions of the source panel.
3.  **Construct Transformation Matrix $T$**: Build the rigid transformation that maps the local mesh frame $R$ to the source panel frame $R_0$.
    *   Let $M_R$ be the matrix with columns $\mathbf{e}_x, \mathbf{e}_y, \mathbf{e}_z$ and translation $\mathbf{p}$.
    *   Let $M_{R0}$ be the matrix with columns $\mathbf{e}'_x, \mathbf{e}'_y, \mathbf{e}'_z$ and translation $\mathbf{k}$ (relative to $O$, or absolute if $O$ is origin).
    *   The source notation implies $T$ combines basis transformation and translation. Given the garbled matrix in [S1], the standard interpretation for mapping frame $R$ to $R_0$ is:
        $$ T = M_{R0} M_R^{-1} $$
4.  **Transform Ray**: Transform the ray origin $\mathbf{o}$ and direction $\mathbf{d}$ from world space to the source panel frame $R_0$ using $T$.
    *   $\mathbf{o}' = T \cdot [\mathbf{o}, 1]^T$
    *   $\mathbf{d}' = T \cdot [\mathbf{d}, 0]^T$
5.  **Intersect Source Panel**: Ray march along the transformed ray $\mathbf{r}'(t) = \mathbf{o}' + t\mathbf{d}'$ to find the intersection point $p_0$ with the source panel in its local space.
6.  **Transform Back**: Transform $p_0$ back to the mesh frame $R$ (or world space) using $T^{-1}$.
    *   $p_{final} = T^{-1} \cdot [p_0, 1]^T$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transform_glass_facet(ray_origin, ray_direction, mesh_face, source_panel):
    """
    ray_origin: Vector3
    ray_direction: Vector3
    mesh_face: Object containing intersection_point p, uv u,v, 
               tangent ex, bitangent ey, normal ez
    source_panel: Object containing origin O, basis ex', ey', ez', 
                  width L, height H
    """
    # 1. Get mesh frame data
    p = mesh_face.intersection_point
    u, v = mesh_face.uv
    ex, ey, ez = mesh_face.tangent, mesh_face.bitangent, mesh_face.normal
    
    # 2. Get source panel frame data
    O = source_panel.origin
    ex_prime, ey_prime, ez_prime = source_panel.basis_x, source_panel.basis_y, source_panel.basis_z
    L, H = source_panel.width, source_panel.height
    
    # 3. Calculate translation vector k
    # k = u * L * ex' + v * H * ey'
    k = Vector3(0, 0, 0)
    k += u * L * ex_prime
    k += v * H * ey_prime
    
    # 4. Construct Transformation Matrix T
    # T maps frame R (mesh) to frame R0 (source panel)
    # R basis: [ex, ey, ez], translation: p
    # R0 basis: [ex', ey', ez'], translation: O + k (assuming k is offset from O)
    
    # Construct matrix M_R (World -> Mesh Local)
    # Inverse of M_R transforms World to Mesh Local
    # M_R = [ex ey ez p; 0 0 0 1]
    # M_R_inv = [ex' ey' ez' -dot(ex,p); ...] (Rigid body inverse)
    
    # Construct matrix M_R0 (World -> Source Panel Local)
    # M_R0 = [ex' ey' ez' O+k; 0 0 0 1]
    
    # T = M_R0 * M_R_inv
    # This transforms a point in World Space to Source Panel Local Space
    
    # Let's define the rotation part
    R_mesh = Matrix3(ex, ey, ez)
    R_panel = Matrix3(ex_prime, ey_prime, ez_prime)
    
    # Rotation from Mesh to Panel
    R_trans = R_panel * R_mesh.transpose()
    
    # Translation from Mesh to Panel
    # Point p in mesh local is (0,0,0) relative to p? No, p is the origin of the local frame R.
    # The frame R is defined as {p | ex, ey, ez}.
    # The frame R0 is defined as {O | ex', ey', ez'}.
    # The transformation maps R to R0.
    # A point q in R is q = p + u*ex + v*ey + w*ez.
    # We want to map this to q' in R0.
    # The source says T maps R to R0 and translates by k.
    # k = u*L*ex' + v*H*ey'. This k seems to be the offset of the specific UV patch.
    
    # Actually, the source says: "T maps the face tangent frame R ... to the source panel frame R0 ... and translate it by k".
    # This implies T is the transformation from World to Panel Local.
    
    # Let's compute T explicitly as a 4x4 matrix
    # T = [ R_trans  |  T_vec ]
    #     [ 0 0 0    |  1    ]
    
    # T_vec = O + k - R_trans * p
    T_vec = O + k - R_trans * p
    
    T = Matrix4x4(R_trans, T_vec)
    
    # 5. Transform Ray
    # Origin
    o_prime = T.transform_point(ray_origin)
    # Direction
    d_prime = T.transform_vector(ray_direction)
    
    # 6. Intersect Source Panel
    # Assume source panel is a plane at z=0 in its local frame, or a box.
    # For simplicity, assume intersection with plane z=0 in R0 frame.
    # Ray: o' + t*d'
    # Plane normal ez', point O+k (or O depending on definition).
    # Let's assume the panel is at distance 0 from origin O in local Z.
    
    # Intersection with plane z=0 in local frame
    denom = d_prime.z
    if abs(denom) < epsilon:
        return None # Parallel
        
    t = -o_prime.z / denom
    p0_local = o_prime + t * d_prime
    
    # 7. Transform back to World
    T_inv = T.inverse()
    p_world = T_inv.transform_point(p0_local)
    
    return p_world
```

**Parameters:**
*   $u, v$: UV coordinates at the intersection point $p$.
*   $L, H$: Horizontal and vertical dimensions of the source panel.
*   $\mathbf{e}_x, \mathbf{e}_y, \mathbf{e}_z$: Tangent, bitangent, and normal vectors of the mesh face.
*   $\mathbf{e}'_x, \mathbf{e}'_y, \mathbf{e}'_z$: Basis vectors of the source panel frame.

**Edge Cases and Pitfalls:**
*   **Coplanarity**: The method requires that all vertices of a given facet remain coplanar to represent arbitrary 3D shapes [S1]. Non-coplanar facets will result in incorrect mapping.
*   **Precision**: Transforming rays back and forth between frames can introduce floating-point errors. Use double precision if available.
*   **Basis Orthogonality**: The basis vectors $\{\mathbf{e}_x, \mathbf{e}_y, \mathbf{e}_z\}$ must be orthonormal for the inverse to be simply the transpose. If not, the full matrix inverse must be computed.

## Operating characteristics
*   **Time Complexity**: $O(1)$ per facet intersection, dominated by matrix multiplication and ray-plane intersection.
*   **Memory Complexity**: $O(1)$ auxiliary memory per facet for storing the transformation matrix.
*   **Strengths**: Allows accurate modeling of complex, multi-piece glass objects with varying physical properties (bubbles, RIF) mapped from a single source panel. Enables unbiased iterative caustic estimation [S1].
*   **Limitations**: Requires coplanar facets. Limited to rigid transformations; cannot model flexible or deforming glass.
*   **When to use**: When rendering ancient or stained glass windows composed of multiple flat glass fragments from a source panel.
*   **When not to use**: For non-rigid glass, or when the glass panel is not flat (curved panels require different mapping).

## Background
This method was introduced in the context of modeling old glass, specifically ancient windows composed of smaller fragments assembled with lead [S1]. It addresses the difficulty of making large flat glass sheets and exploits the variable depth of glass for artistic effects like color gradients [S1]. The technique is described in "Parabolic Sphere Tracing of Signed Distance Fields for Old Glass Modelling and Rendering" [S1].

## Sources
[S1] Parabolic Sphere Tracing of Signed Distance Fields for Old Glass Modelling and Rendering, https://doi.org/10.1109/tvcg.2025.3613853, fetched 2026-11-05
