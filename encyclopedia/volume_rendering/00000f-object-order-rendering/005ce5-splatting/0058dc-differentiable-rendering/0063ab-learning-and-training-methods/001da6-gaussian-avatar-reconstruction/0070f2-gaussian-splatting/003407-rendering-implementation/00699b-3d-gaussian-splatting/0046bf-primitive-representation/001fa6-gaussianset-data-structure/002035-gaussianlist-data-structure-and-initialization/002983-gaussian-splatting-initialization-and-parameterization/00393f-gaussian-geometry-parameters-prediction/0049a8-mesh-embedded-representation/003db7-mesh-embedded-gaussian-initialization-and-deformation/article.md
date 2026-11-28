# Mesh-Embedded Gaussian Initialization and Deformation

Also called: Gaussian Embedding, Triangle Embedding

## Overview
This method initializes and deforms 3D Gaussians by anchoring them to a canonical triangle mesh $\mathcal{M}$ using barycentric coordinates and normal displacement [S1, unofficial]. It solves the problem of maintaining geometric coherence during mesh deformation by tracking Gaussians through mesh topology rather than direct vertex positions [S1, unofficial]. It builds on [Mesh-embedded representation](12559-mesh-embedded-representation.md).

## How it works
**Inputs and Outputs:**
*   **Input:** Canonical triangle mesh $\mathcal{M}$ with vertices $V$, per-vertex normals $n$, and faces $F$; deformation function $\text{Deform}(\cdot)$; number of Gaussians $N$ (10,000) [S1, unofficial].
*   **Output:** Set of $N$ Gaussians, each defined by embedding tuple $E_i = \{k, u, v, d\}$ and derived attributes (position, rotation, scale) [S1, unofficial].

**Data Structures:**
Each Gaussian $i$ stores an embedding tuple $E_i$:
*   $k \in F$: Index of the triangle in the canonical mesh.
*   $(u, v)$: Barycentric coordinates within triangle $k$, constrained such that $u \ge 0, v \ge 0, u+v \le 1$.
*   $d$: Scalar normal displacement along the interpolated surface normal [S1, unofficial].

**Canonical Position and Normal:**
The canonical mean $\mu_{cano}$ and normal $n_{cano}$ are computed via Phong interpolation on the canonical mesh [S1, unofficial]:
$$
\mu_{cano} = \mathcal{V}(k, u, v) = u V_1 + v V_2 + (1 - u - v) V_3
$$
$$
n_{cano} = \mathcal{N}(k, u, v) = u n_1 + v n_2 + (1 - u - v) n_3
$$
where $V_j$ and $n_j$ are the vertices and per-vertex normals of triangle $k$ [S1, unofficial].

**Posed State Computation:**
When the mesh is deformed, the posed position $\mu_{pose}$ is derived by applying the mesh deformation function $\text{Deform}(\cdot)$ to the displaced canonical position [S1, unofficial]:
$$
\mu_{pose} = \text{Deform}(\mu_{cano} + d \cdot n_{cano})
$$
The posed rotation $q_{pose}$ and scale $s_{pose}$ are derived from mesh attributes to maintain coherence [S1, unofficial]:
*   **Rotation:** Interpolate per-vertex quaternions $q_1, q_2, q_3$ using barycentric weights to get $\delta q$, then combine with canonical rotation $\bar{q}$:
    $$
    q_{pose} = \delta q * \bar{q}, \quad \delta q = u q_1 + v q_2 + (1 - u - v) q_3
    $$
*   **Scale:** Scale is proportional to the area change of the embedded triangle relative to the canonical state:
    $$
    s_{pose} = \left( \frac{A_{pose}}{A_{cano}} \right) \bar{s}
    $$
    where $A_{pose}$ and $A_{cano}$ are the areas of the triangle in the posed and canonical states, respectively [S1, unofficial].

**Initialization:**
1.  Randomly select 10,000 pairs of triangle indices $k$ and barycentric coordinates $(u, v)$ on the canonical mesh [S1, unofficial].
2.  Initialize all displacements $d$ to zero [S1, unofficial].
3.  Set canonical rotation $\bar{q}$ and scale $\bar{s}$ for each Gaussian [G].
4.  Spherical harmonics for color representation are turned off for simplicity [S1, unofficial].

**Cross-Triangle Handling:**
Barycentric coordinates are differentiable only within a single triangle [S1, unofficial]. If optimization moves an embedding point across a boundary ($u < 0$, $v < 0$, or $u+v > 1$), the embedding must be re-expressed in the neighboring triangle to maintain differentiability [S1, unofficial]. Failure to handle this causes Gaussians to stick to boundaries and pile up [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def quat_mul(q1, q2):
    """Hamilton product of two quaternions [w, x, y, z]."""
    w1, x1, y1, z1 = q1
    w2, x2, y2, z2 = q2
    return np.array([
        w1*w2 - x1*x2 - y1*y2 - z1*z2,
        w1*x2 + x1*w2 + y1*z2 - z1*y2,
        w1*y2 - x1*z2 + y1*w2 + z1*x2,
        w1*z2 + x1*y2 - y1*x2 + z1*w2
    ])

def quat_from_basis(tangent, bitangent, normal):
    """Convert orthonormal basis vectors to a quaternion [G]."""
    # Standard rotation matrix to quaternion conversion
    R = np.column_stack((tangent, bitangent, normal))
    trace = np.trace(R)
    if trace > 0:
        s = 0.5 / np.sqrt(trace + 1.0)
        w = 0.25 / s
        x = (R[2,1] - R[1,2]) * s
        y = (R[0,2] - R[2,0]) * s
        z = (R[1,0] - R[0,1]) * s
    else:
        i, j, k = np.argmax([R[0,0], R[1,1], R[2,2]])
        qs = [0, 0, 0, 0]
        qs[i] = 1.0 - R[i,i]
        qs[j] = R[j,i] + R[i,j]
        qs[k] = R[k,i] + R[i,k]
        qs[0] = R[j,k] - R[k,j]
        s = 0.5 / np.sqrt(qs[i] + 0.5)
        qs = np.array(qs) * s
    return qs

def triangle_area(v1, v2, v3):
    """Compute triangle area from vertices."""
    return 0.5 * np.linalg.norm(np.cross(v2 - v1, v3 - v1))

def initialize_gaussians(mesh, num_gaussians=10000):
    """Initialize Gaussians on a canonical mesh."""
    gaussians = []
    for _ in range(num_gaussians):
        k = np.random.randint(len(mesh['faces']))
        face = mesh['faces'][k]
        
        # Uniform barycentric sampling
        r1, r2 = np.random.rand(2)
        if r1 + r2 > 1:
            r1 = 1 - r1
            r2 = 1 - r2
        u, v = r1, r2
        
        gaussians.append({
            'k': k, 'u': u, 'v': v, 'd': 0.0,
            'q_canon': np.array([1.0, 0.0, 0.0, 0.0]),
            's_canon': 1.0
        })
    return gaussians

def compute_posed_state(gaussian, mesh, deformation_function):
    """Compute posed position, rotation, and scale for a Gaussian."""
    k, u, v, d = gaussian['k'], gaussian['u'], gaussian['v'], gaussian['d']
    face = mesh['faces'][k]
    
    V1, V2, V3 = face['vertices']
    n1, n2, n3 = face['vertex_normals']
    
    # Canonical position and normal
    mu_cano = u * V1 + v * V2 + (1 - u - v) * V3
    n_cano = u * n1 + v * n2 + (1 - u - v) * n3
    
    # Posed position
    mu_pose = deformation_function(mu_cano + d * n_cano)
    
    # Posed rotation
    q1, q2, q3 = face['vertex_quaternions']
    delta_q = u * q1 + v * q2 + (1 - u - v) * q3
    q_pose = quat_mul(delta_q, gaussian['q_canon'])
    
    # Posed scale
    A_cano = triangle_area(V1, V2, V3)
    V1_p, V2_p, V3_p = deformation_function(V1), deformation_function(V2), deformation_function(V3)
    A_pose = triangle_area(V1_p, V2_p, V3_p)
    s_pose = (A_pose / A_cano) * gaussian['s_canon']
    
    return {'position': mu_pose, 'rotation': q_pose, 'scale': s_pose}
```

## Subtopics

### [Walking on Triangle](9700-walking-on-triangle.md)
Walking on Triangle is a constraint mechanism that binds 3D Gaussians to the surface of a triangle mesh, ensuring they remain within the boundaries of their corresponding triangles during optimization and deformation. It solves the problem of Gaussians drifting off-mesh or piling up at triangle edges when the underlying mesh is manipulated, animated, or deformed. Unlike methods that rely heavily on precise mesh accuracy, it defines a local triangle space to ensure robust results even with inaccurate meshes.

### Gao et al. Mesh-based Deformation
A method that adapts mesh-based deformation to 3DGS by harnessing the priors of explicit representation [S2, unofficial]. It uses surface properties like normals of the mesh, and gradients generated by explicit deformation methods [S2, unofficial]. It learns the face split to optimize the parameters and number of Gaussians, which provides adequate topological information to 3DGS and improves the quality of both reconstruction and geometry editing results [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Initialization is $O(N)$ where $N$ is 10,000 [S1, unofficial]. Posed state computation is $O(1)$ per Gaussian per frame [G].
*   **Memory Complexity:** $O(N)$ to store the embedding tuples and derived attributes for each Gaussian [G].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with the number of Gaussians [G].
*   **Strengths:** Maintains geometric coherence during deformation [S1, unofficial].
*   **Limitations:** Requires careful handling of cross-triangle boundaries to avoid piling up [S1, unofficial].
*   **When to use:** When deforming a mesh-based representation with Gaussians [G].
*   **When not to use:** When the mesh deformation is unknown or non-differentiable [G].

## Background
Originated in SplattingAvatar [S1, unofficial]. The method builds on the concept of mesh-embedded representations [Parent].

## Sources
[S1] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
