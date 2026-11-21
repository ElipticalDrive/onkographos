# Mesh-Embedded Gaussian Initialization and Deformation

Also called: Gaussian Embedding, Triangle Embedding

## Overview
This method initializes and deforms 3D Gaussians by anchoring them to a canonical triangle mesh $\mathcal{M}$ using barycentric coordinates and normal displacement [S1, unofficial]. It solves the problem of maintaining geometric coherence and differentiability during mesh deformation (e.g., animation) by tracking Gaussians through mesh topology rather than direct vertex positions [S1, unofficial]. Unlike methods that rely on precise mesh accuracy, it defines a local triangle space to ensure robust results even with inaccurate meshes [S1, unofficial]. It builds on [Mesh-embedded representation](12559-mesh-embedded-representation.md).

## How it works
**Inputs and Outputs:**
*   **Input:** Canonical triangle mesh $\mathcal{M}$ with vertices $V$, per-vertex normals $n$, and faces $F$; deformation function $\text{Deform}(\cdot)$; number of Gaussians $N$ (typically 10,000).
*   **Output:** Set of $N$ Gaussians, each defined by embedding tuple $E_i = \{k, u, v, d\}$ and derived attributes (position, rotation, scale).

**Data Structures:**
Each Gaussian $i$ stores an embedding tuple $E_i$:
*   $k \in F$: Index of the triangle in the canonical mesh.
*   $(u, v)$: Barycentric coordinates within triangle $k$, such that $u \ge 0, v \ge 0, u+v \le 1$.
*   $d$: Normal displacement scalar along the interpolated surface normal.

**Canonical Position and Normal:**
The canonical mean $\mu_{cano}$ and normal $n_{cano}$ are computed via Phong interpolation on the canonical mesh [Parent, S1, unofficial]:
$$
\mu_{cano} = \mathcal{V}(k, u, v) = u V_1 + v V_2 + (1 - u - v) V_3
$$
$$
n_{cano} = \mathcal{N}(k, u, v) = u n_1 + v n_2 + (1 - u - v) n_3
$$
where $V_j$ and $n_j$ are the vertices and per-vertex normals of triangle $k$ [Parent, S1, unofficial].

**Posed State Computation:**
When the mesh is deformed, the posed position $\mu_{pose}$ is derived by applying the mesh deformation function $\text{Deform}(\cdot)$ to the displaced canonical position [Parent, S1, unofficial]:
$$
\mu_{pose} = \text{Deform}(\mu_{cano} + d \cdot n_{cano})
$$
The posed rotation $q_{pose}$ and scale $s_{pose}$ are derived from mesh attributes to maintain coherence [Parent, S1, unofficial]:
*   **Rotation:** Interpolate per-vertex quaternions $q_1, q_2, q_3$ using barycentric weights to get $\delta q$, then combine with canonical rotation $\bar{q}$:
    $$
    q_{pose} = \delta q * \bar{q}, \quad \delta q = u q_1 + v q_2 + (1 - u - v) q_3
    $$
*   **Scale:** Scale is proportional to the area change of the embedded triangle relative to the canonical state:
    $$
    s_{pose} = \left( \frac{A_{pose}}{A_{cano}} \right) \bar{s}
    $$
    where $A_{pose}$ and $A_{cano}$ are the areas of the triangle in the posed and canonical states, respectively [Parent, S1, unofficial].

**Initialization:**
1.  Randomly select $N$ pairs of triangle indices $k$ and barycentric coordinates $(u, v)$ on the canonical mesh [S1, unofficial].
2.  Initialize all displacements $d$ to zero [S1, unofficial].
3.  Set canonical rotation $\bar{q}$ and scale $\bar{s}$ for each Gaussian [S1, unofficial].
4.  [G] Color representation: Spherical harmonics are turned off for simplicity [S1, unofficial].

**Cross-Triangle Handling:**
Barycentric coordinates are only differentiable within a single triangle. If optimization moves an embedding point across a boundary ($u < 0$, $v < 0$, or $u+v > 1$), the embedding must be re-expressed in the neighboring triangle to maintain differentiability [Parent, S1, unofficial]. Failure to handle this causes Gaussians to stick to boundaries and pile up [Parent, S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_gaussians(mesh, num_gaussians=10000):
    """
    Initialize Gaussians on a canonical mesh.
    mesh: object with vertices V, normals n, faces F, and method get_triangle_area(k)
    """
    gaussians = []
    for i in range(num_gaussians):
        # Randomly select a triangle
        k = random_int(0, len(mesh.F) - 1)
        triangle = mesh.F[k]
        
        # Generate random barycentric coordinates
        r1 = random_float(0, 1)
        r2 = random_float(0, 1)
        if r1 + r2 > 1:
            r1 = 1 - r1
            r2 = 1 - r2
        u = r1
        v = r2
        
        # Initialize displacement to zero
        d = 0.0
        
        # Initialize canonical rotation and scale [G]
        q_canon = identity_quaternion()
        s_canon = 1.0
        
        gaussians.append({
            'k': k,
            'u': u,
            'v': v,
            'd': d,
            'q_canon': q_canon,
            's_canon': s_canon
        })
    return gaussians

def compute_posed_state(gaussian, mesh, deformation_function):
    """
    Compute the posed position, rotation, and scale for a Gaussian.
    gaussian: dict with keys k, u, v, d, q_canon, s_canon
    mesh: object with vertices V, normals n, faces F, and method get_triangle_area(k)
    deformation_function: function that takes a point and returns its deformed position
    """
    k = gaussian['k']
    u = gaussian['u']
    v = gaussian['v']
    d = gaussian['d']
    
    # Get triangle vertices and normals
    v1, v2, v3 = mesh.V[mesh.F[k][0]], mesh.V[mesh.F[k][1]], mesh.V[mesh.F[k][2]]
    n1, n2, n3 = mesh.n[mesh.F[k][0]], mesh.n[mesh.F[k][1]], mesh.n[mesh.F[k][2]]
    
    # Compute canonical position and normal
    mu_cano = u * v1 + v * v2 + (1 - u - v) * v3
    n_cano = u * n1 + v * n2 + (1 - u - v) * n3
    
    # Compute posed position
    displaced_pos = mu_cano + d * n_cano
    mu_pose = deformation_function(displaced_pos)
    
    # Compute posed rotation
    # Get per-vertex quaternions for the triangle [G]
    q1, q2, q3 = get_vertex_quaternions(mesh, k)
    delta_q = u * q1 + v * q2 + (1 - u - v) * q3
    q_pose = delta_q * gaussian['q_canon']
    
    # Compute posed scale
    a_cano = mesh.get_triangle_area(k)
    # Compute posed area by deforming vertices and calculating area [G]
    v1_pose = deformation_function(v1)
    v2_pose = deformation_function(v2)
    v3_pose = deformation_function(v3)
    a_pose = compute_triangle_area(v1_pose, v2_pose, v3_pose)
    s_pose = (a_pose / a_cano) * gaussian['s_canon']
    
    return {
        'position': mu_pose,
        'rotation': q_pose,
        'scale': s_pose
    }

def get_vertex_quaternions(mesh, k):
    """
    Get per-vertex quaternions for a triangle.
    This is a placeholder for the specific method of computing quaternions from tangent/bitangent/normal.
    [G] In practice, this involves computing rotation matrices from tangent, bitangent, and normal vectors
    for each vertex and converting to quaternions.
    """
    # Placeholder implementation
    v1, v2, v3 = mesh.V[mesh.F[k][0]], mesh.V[mesh.F[k][1]], mesh.V[mesh.F[k][2]]
    n1, n2, n3 = mesh.n[mesh.F[k][0]], mesh.n[mesh.F[k][1]], mesh.n[mesh.F[k][2]]
    
    # Compute tangent and bitangent for each vertex [G]
    # This is a simplified placeholder; actual implementation requires proper tangent space computation
    q1 = identity_quaternion()
    q2 = identity_quaternion()
    q3 = identity_quaternion()
    
    return q1, q2, q3

def compute_triangle_area(v1, v2, v3):
    """
    Compute the area of a triangle given its vertices.
    """
    edge1 = v2 - v1
    edge2 = v3 - v1
    cross_product = cross(edge1, edge2)
    return 0.5 * length(cross_product)
```

## Subtopics

### [Walking on Triangle](9700-walking-on-triangle.md)
Walking on Triangle is a constraint mechanism that binds 3D Gaussians to the surface of a triangle mesh, ensuring they remain within the boundaries of their corresponding triangles during optimization and deformation. It solves the problem of Gaussians drifting off-mesh or piling up at triangle edges when the underlying mesh is manipulated, animated, or deformed. Unlike methods that rely heavily on precise mesh accuracy, it defines a local triangle space to ensure robust results even with inaccurate meshes.

## Operating characteristics
*   **Time Complexity:** Initialization is $O(N)$ where $N$ is the number of Gaussians. Posed state computation is $O(1)$ per Gaussian per frame, assuming constant-time mesh deformation and area calculation.
*   **Memory Complexity:** $O(N)$ to store the embedding tuples and derived attributes for each Gaussian.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with the number of Gaussians. Performance depends on the efficiency of the mesh deformation function and area calculation.
*   **Strengths:** Maintains geometric coherence during deformation; robust to mesh inaccuracies; differentiable within triangle boundaries.
*   **Limitations:** Requires careful handling of cross-triangle boundaries to avoid piling up; assumes the mesh deformation is known and differentiable.
*   **When to use:** When deforming a mesh-based representation with Gaussians; when robustness to mesh inaccuracies is required.
*   **When not to use:** When the mesh deformation is unknown or non-differentiable; when high-frequency details require direct vertex manipulation.

## Background
Originated in SplattingAvatar [S1, unofficial]. The method builds on the concept of mesh-embedded representations [Parent].

## Sources
[S1] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
