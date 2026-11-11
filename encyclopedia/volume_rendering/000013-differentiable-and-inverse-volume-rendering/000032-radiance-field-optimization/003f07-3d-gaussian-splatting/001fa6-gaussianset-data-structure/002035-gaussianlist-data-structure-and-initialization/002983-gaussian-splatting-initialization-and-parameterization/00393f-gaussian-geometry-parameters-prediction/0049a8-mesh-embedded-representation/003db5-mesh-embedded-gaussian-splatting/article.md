# Mesh-Embedded Gaussian Splatting

Also called: GaMeS, Gaussian Mesh Splatting, Gaussian-Mesh association

## Overview
Mesh-Embedded Gaussian Splatting is a hybrid 3D representation that defines Gaussian geometry (position, rotation, scale) as a function of mesh attributes rather than independent learnable vectors [S2, unofficial]. It solves the problem of disentangling motion from appearance in dynamic scenes, such as human avatars, by allowing the underlying mesh to control low-frequency deformation while Gaussians capture high-frequency details [S2, unofficial]. Unlike sibling methods that rely on MLP-based linear blend skinning (LBS) fields for motion, this approach controls Gaussian rotation and translation directly by the mesh, enabling compatibility with skeletal animation, blend shapes, and mesh editing [S2, unofficial]. It builds on [Mesh-embedded representation](12559-mesh-embedded-representation.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A canonical triangle mesh (e.g., SMPL-X, FLAME) with vertices $V_i$ and per-vertex normals $n_i$; monocular video frames of the subject in various poses.
*   **Output:** A set of 3D Gaussians parameterized by embedding coordinates on the mesh, optimized for photorealistic rendering of the animated avatar.

**Data Structures:**
Each Gaussian $i$ is defined by the following parameters:
1.  **Embedding $E_i$:** A tuple $(k, u, v, d)$.
    *   $k$: The index of the triangle on the canonical mesh.
    *   $(u, v)$: Barycentric coordinates within triangle $k$.
    *   $d$: Normal displacement along the interpolated normal.
2.  **Canonical Properties (Pose-Invariant):**
    *   $\bar{q}$: Canonical rotation (quaternion).
    *   $\bar{s}$: Canonical scale.
    *   $o$: Opacity.
    *   $c$: Color (SH coefficients).
3.  **Posed Properties (Derived):**
    *   $\mu$: Final position.
    *   $q_{pose}$: Final rotation.
    *   $s_{pose}$: Final scale.

**Formulas:**
1.  **Canonical Position and Normal (Phong Surface):**
    For a triangle $k$ with vertices $V_1, V_2, V_3$ and normals $n_1, n_2, n_3$:
    $$
    P(k, u, v) = u V_1 + v V_2 + (1 - u - v) V_3
    $$
    $$
    n(k, u, v) = u n_1 + v n_2 + (1 - u - v) n_3
    $$
    The canonical mean position $\mu_{cano}$ includes displacement $d$:
    $$
    \mu_{cano} = P(k, u, v) + d \cdot n(k, u, v)
    $$

2.  **Posed Transformation:**
    When the mesh deforms to a posed state, the Gaussian parameters are transformed:
    *   **Rotation:** Interpolated from per-vertex quaternions $q_i$ using barycentric weights:
        $$
        \delta q = u q_1 + v q_2 + (1 - u - v) q_3
        $$
        The final rotation is:
        $$
        q_{pose} = \delta q * \bar{q}
        $$
    *   **Scale:** Derived from the area change of the embedded triangle relative to the canonical state:
        $$
        s_{pose} = \left( \frac{A_{pose}}{A_{cano}} \right) \bar{s}
        $$
    *   **Position:** The canonical position is transformed by the mesh deformation (rigid or non-rigid) to obtain the final position $\mu_{posed}$.

**Algorithm Steps:**
1.  **Initialization:** Initialize Gaussians on the canonical mesh. Assign each Gaussian a random triangle $k$, barycentric coordinates $(u, v)$, and displacement $d$. Initialize canonical properties $\bar{q}, \bar{s}, o, c$.
2.  **Lifted Optimization:** During training, optimize both Gaussian properties and embedding parameters $(k, u, v, d)$ simultaneously.
    *   Compute the rendered image from the current Gaussians.
    *   Calculate loss against ground truth images.
    *   Backpropagate gradients to update $\bar{q}, \bar{s}, o, c$ and $(u, v, d)$.
3.  **Cross-Triangle Handling:**
    *   If an update to $(u, v)$ causes the embedding point to cross a triangle boundary, the embedding must be re-expressed in the neighboring triangle to maintain differentiability [S2, unofficial].
    *   This is handled by "walking" the embedding point on the mesh: detect the boundary crossing, identify the adjacent triangle, and re-parameterize the barycentric coordinates $(u', v')$ and triangle index $k'$ such that the physical position remains consistent.
4.  **Adaptation of Cloning/Splitting:** Adapt the standard 3D Gaussian Splatting clone and split schemes to operate on the mesh-embedded representation, ensuring that new Gaussians are also initialized with valid mesh embeddings [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_posed_gaussian(gaussian, mesh_canonical, mesh_posed):
    """
    Computes the posed parameters of a Gaussian given its embedding and mesh states.
    gaussian: dict with keys 'k', 'u', 'v', 'd', 'q_cano', 's_cano', 'o', 'c'
    mesh_canonical: object with methods get_vertex(k, i), get_normal(k, i)
    mesh_posed: object with methods get_vertex(k, i), get_normal(k, i), get_triangle_area(k)
    """
    k = gaussian['k']
    u = gaussian['u']
    v = gaussian['v']
    d = gaussian['d']
    
    # 1. Get canonical mesh data
    V1_c = mesh_canonical.get_vertex(k, 0)
    V2_c = mesh_canonical.get_vertex(k, 1)
    V3_c = mesh_canonical.get_vertex(k, 2)
    n1_c = mesh_canonical.get_normal(k, 0)
    n2_c = mesh_canonical.get_normal(k, 1)
    n3_c = mesh_canonical.get_normal(k, 2)
    
    # 2. Compute canonical position and normal (Phong surface)
    P_c = u * V1_c + v * V2_c + (1 - u - v) * V3_c
    n_c = u * n1_c + v * n2_c + (1 - u - v) * n3_c
    mu_cano = P_c + d * n_c
    
    # 3. Get posed mesh data
    V1_p = mesh_posed.get_vertex(k, 0)
    V2_p = mesh_posed.get_vertex(k, 1)
    V3_p = mesh_posed.get_vertex(k, 2)
    q1_p = mesh_posed.get_vertex_quaternion(k, 0) # Assuming per-vertex quaternions for rotation
    q2_p = mesh_posed.get_vertex_quaternion(k, 1)
    q3_p = mesh_posed.get_vertex_quaternion(k, 2)
    
    # 4. Compute posed rotation (barycentric interpolation of vertex quaternions)
    delta_q = u * q1_p + v * q2_p + (1 - u - v) * q3_p
    q_pose = delta_q * gaussian['q_cano'] # Quaternion multiplication
    
    # 5. Compute posed scale (area ratio)
    A_cano = mesh_canonical.get_triangle_area(k)
    A_posed = mesh_posed.get_triangle_area(k)
    s_pose = (A_posed / A_cano) * gaussian['s_cano']
    
    # 6. Compute posed position
    # Note: The exact transformation of mu_cano to mu_posed depends on the mesh deformation type.
    # For skeletal animation, this involves skinning. For general mesh editing, it's the new vertex positions.
    # Here we assume the mesh_posed provides the deformed surface point directly or via skinning.
    # A common approach is to apply the same deformation that moved the triangle vertices to the canonical point.
    # However, SplattingAvatar explicitly controls rotation/translation by mesh.
    # The position mu_posed is typically derived by applying the mesh deformation to mu_cano.
    # If the mesh is just vertices, we might need to interpolate the deformation field.
    # For simplicity, assuming mesh_posed provides the deformed position for the embedding point:
    P_p = u * V1_p + v * V2_p + (1 - u - v) * V3_p
    # The displacement d is usually kept in the local normal frame or transformed.
    # SplattingAvatar states: "control the rotation and translation of the Gaussians directly by mesh"
    # This implies the Gaussian's center follows the mesh surface.
    # We assume the displacement d is applied in the posed normal frame.
    n_p = u * mesh_posed.get_normal(k, 0) + v * mesh_posed.get_normal(k, 1) + (1 - u - v) * mesh_posed.get_normal(k, 2)
    mu_posed = P_p + d * n_p
    
    return {
        'mu': mu_posed,
        'q': q_pose,
        's': s_pose,
        'o': gaussian['o'],
        'c': gaussian['c']
    }

def handle_cross_triangle_update(gaussian, mesh, gradient_u, gradient_v):
    """
    Handles the case where barycentric update moves the embedding across a triangle boundary.
    """
    k = gaussian['k']
    u = gaussian['u']
    v = gaussian['v']
    
    # Check if (u, v) is still valid in triangle k
    # Valid if u >= 0, v >= 0, u + v <= 1
    if u < 0 or v < 0 or u + v > 1:
        # Find neighboring triangle
        neighbor_k = mesh.get_neighbor(k, u, v)
        
        # Re-express barycentric coordinates in the new triangle
        # This requires mapping the physical position to the new triangle's barycentric system
        # The physical position P is invariant.
        # P = u * V1 + v * V2 + (1-u-v) * V3
        # We need u', v' such that P = u' * V1' + v' * V2' + (1-u'-v') * V3'
        # This is a linear system solve.
        
        V1 = mesh.get_vertex(k, 0)
        V2 = mesh.get_vertex(k, 1)
        V3 = mesh.get_vertex(k, 2)
        P = u * V1 + v * V2 + (1 - u - v) * V3
        
        V1_n = mesh.get_vertex(neighbor_k, 0)
        V2_n = mesh.get_vertex(neighbor_k, 1)
        V3_n = mesh.get_vertex(neighbor_k, 2)
        
        # Solve for u', v'
        # P = u' * V1_n + v' * V2_n + (1 - u' - v') * V3_n
        # P - V3_n = u' * (V1_n - V3_n) + v' * (V2_n - V3_n)
        # Let E1 = V1_n - V3_n, E2 = V2_n - V3_n, D = P - V3_n
        # D = u' * E1 + v' * E2
        # This is a 2D linear system (assuming planar triangle)
        
        E1 = V1_n - V3_n
        E2 = V2_n - V3_n
        D = P - V3_n
        
        # Solve using cross products (in 3D, project to 2D or use normal)
        # u' = cross(D, E2) . normal / cross(E1, E2) . normal
        # v' = cross(E1, D) . normal / cross(E1, E2) . normal
        normal = cross(E1, E2)
        u_new = dot(cross(D, E2), normal) / dot(cross(E1, E2), normal)
        v_new = dot(cross(E1, D), normal) / dot(cross(E1, E2), normal)
        
        gaussian['k'] = neighbor_k
        gaussian['u'] = u_new
        gaussian['v'] = v_new
        
    return gaussian
```

**Parameters:**
*   **Learning Rate:** Not given in the sources.
*   **Mesh Type:** SMPL-X or FLAME for human avatars [S2, unofficial].
*   **Optimization Method:** Stochastic Gradient Descent (SGD) or Adam [G].

**Edge Cases and Pitfalls:**
*   **Cross-Triangle Updates:** Barycentric coordinates are only differentiable within a triangle. If an update moves the embedding across a boundary, the gradient flow breaks unless the embedding is re-expressed in the neighboring triangle [S2, unofficial].
*   **Mesh Accuracy:** Errors in segmentation or remeshing can produce artifacts, as the method relies on accurate semantic labels or mesh proxies [S1].
*   **Degenerate Triangles:** Triangles with zero area can cause division by zero in scale computation.

## Subtopics

### GaussianMesh
A method that defines Gaussians over an explicit mesh and ties the two via joint refinement: GS rendering guides adaptive face and Gaussian splitting, while mesh constraints regularize Gaussian shapes and support large, interactive deformations driven by mesh editing [S1]. It is chosen when interactive mesh editing is required.

## Operating characteristics
*   **Time Complexity:** Real-time rendering is achieved. Over 300 FPS on an NVIDIA RTX 3090 GPU and 30 FPS on an iPhone 13 [S2, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Explicit motion control by the mesh, compatibility with skeletal animation and blend shapes, high editability, real-time rendering capability [S2, unofficial].
*   **Limitations:** Relies on accurate semantic labels or mesh proxies; errors can produce artifacts; editing attributes requires careful regularization [S1].
*   **When to use:** When explicit mesh-based control of Gaussian deformation is needed, especially for human avatars.
*   **When not to use:** When the scene does not have a clear mesh structure or when high-frequency details are not best captured by Gaussians.

## Background
SplattingAvatar, a prominent implementation of Mesh-Embedded Gaussian Splatting, was introduced by Zhijing Shao et al. in 2024 [S2, unofficial]. It builds on the 3D Gaussian Splatting technique [23] and Phong surface shading [38]. The method was developed to create photorealistic human avatars from monocular videos, disentangling motion and appearance. It has been ported to Unity for real-time rendering [S2, unofficial].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
