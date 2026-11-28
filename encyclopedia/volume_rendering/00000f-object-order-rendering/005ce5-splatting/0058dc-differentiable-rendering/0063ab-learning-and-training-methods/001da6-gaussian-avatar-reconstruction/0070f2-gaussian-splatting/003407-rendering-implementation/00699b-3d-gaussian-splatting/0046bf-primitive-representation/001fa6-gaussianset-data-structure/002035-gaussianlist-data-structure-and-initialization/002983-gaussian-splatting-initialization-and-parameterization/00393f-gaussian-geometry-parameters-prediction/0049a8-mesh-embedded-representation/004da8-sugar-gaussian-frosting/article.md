# SuGaR / Gaussian Frosting

Also called: SuGaR, Gaussian Frosting

## Overview
SuGaR (Surface-Aligned Gaussian Splatting) and Gaussian Frosting are methods that extract a canonical triangle mesh from an optimized 3D Gaussian Splatting (3DGS) scene to enable explicit mesh editing while preserving high-fidelity rendering [S1, unofficial][S2]. SuGaR aligns Gaussians to a surface and extracts a base mesh via Poisson reconstruction, while Gaussian Frosting binds Gaussians to this mesh surface for hybrid rendering, introducing an adaptive Gaussian layer with variable thickness to enhance detail and volumetric effects [S1, unofficial][S2]. This approach combines the editing capabilities of explicit meshes with the rendering quality of differentiable Gaussian splatting [S2].

## How it works
**Inputs:** Optimized 3D Gaussian Splatting scene (means $\mu$, covariances $\Sigma$, colors, opacities); optional user-defined mesh deformations [S1, unofficial][S2].
**Outputs:** Canonical triangle mesh $\mathcal{M}$; Gaussian parameters parameterized relative to $\mathcal{M}$; rendered images [S1, unofficial][S2].

**Data Structures:**
1.  **Canonical Mesh $\mathcal{M}$:** Defined by vertices $V$, normals $n$, and faces $F$ [G].
2.  **Gaussian Embeddings:** Each Gaussian $i$ is defined by an embedding tuple $E_i = \{k, u, v, d\}$, where $k$ is the triangle index, $(u, v)$ are barycentric coordinates, and $d$ is the normal displacement [Parent](12559-mesh-embedded-representation.md).
3.  **Adaptive Layer:** Gaussian Frosting introduces a parameterization to constrain Gaussians within a layer of variable thickness around the base mesh [S2].

**Steps:**
1.  **Base Mesh Extraction:** Extract a base mesh $\mathcal{M}$ from the optimized 3DGS representation. SuGaR achieves this by aligning Gaussians to a surface and performing Poisson reconstruction [S1, unofficial].
2.  **Parameterization:** Bind each Gaussian to the extracted mesh using the embedding tuple $E_i$ defined in the parent article [Parent](12559-mesh-embedded-representation.md). This constrains the Gaussians to the surface geometry [G].
3.  **Adaptive Layer Introduction (Gaussian Frosting):** Introduce an adaptive Gaussian layer with variable thickness around the base mesh to enhance fine details and volumetric effects [S2]. The Gaussians are parameterized to remain within this layer [S2].
4.  **Deformation and Rendering:** When the base mesh is manipulated (e.g., via skeletal animation or direct editing), the Gaussians are automatically adjusted [S2]. The posed position $\mu_{pose}$ is computed by deforming the canonical position $\mu_{cano} + d \cdot n_{cano}$ using the mesh deformation function $\text{Deform}(\cdot)$ [S2]. The rotation and scale are derived from mesh attributes to maintain coherence [S2].
5.  **Cross-Triangle Handling:** If optimization or deformation causes barycentric coordinates to move outside the current triangle ($u < 0$, $v < 0$, or $u+v > 1$), the embedding must be re-expressed in the neighboring triangle to maintain differentiability [S2].

**Formulas:**
*   Canonical Position: $\mu_{cano} = u V_1 + v V_2 + (1 - u - v) V_3$ [S2]
*   Canonical Normal: $n_{cano} = u n_1 + v n_2 + (1 - u - v) n_3$ [S2]
*   Posed Position: $\mu_{pose} = \text{Deform}(\mu_{cano} + d \cdot n_{cano})$ [S2]
*   Posed Rotation: $q_{pose} = \delta q * \bar{q}$, where $\delta q = u q_1 + v q_2 + (1 - u - v) q_3$ [S2]
*   Posed Scale: $s_{pose} = \left( \frac{A_{pose}}{A_{cano}} \right) \bar{s}$ [S2]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_posed_gaussian(gaussian, mesh, pose_transform):
    """
    Computes the posed position, rotation, and scale for a single Gaussian.
    
    Args:
        gaussian: Dict with keys 'k', 'u', 'v', 'd', 'q_canon', 's_canon'.
        mesh: Dict with 'faces' (list of dicts with 'vertices', 'vertex_normals', 'vertex_quaternions').
        pose_transform: Function that applies deformation to mesh vertices.
        
    Returns:
        mu_pose: Posed mean position.
        q_pose: Posed rotation quaternion.
        s_pose: Posed scale vector.
    """
    k = gaussian['k']
    u = gaussian['u']
    v = gaussian['v']
    d = gaussian['d']
    
    face = mesh.faces[k]
    V1, V2, V3 = face['vertices']
    n1, n2, n3 = face['vertex_normals']
    
    # Canonical position and normal
    mu_cano = u * V1 + v * V2 + (1 - u - v) * V3
    n_cano = u * n1 + v * n2 + (1 - u - v) * n3
    
    # Posed position
    displaced_pos = mu_cano + d * n_cano
    mu_pose = pose_transform(displaced_pos)
    
    # Posed rotation
    q1, q2, q3 = face['vertex_quaternions']
    delta_q = u * q1 + v * q2 + (1 - u - v) * q3
    q_pose = delta_q * gaussian['q_canon']
    
    # Posed scale
    V1_pose, V2_pose, V3_pose = pose_transform(V1), pose_transform(V2), pose_transform(V3)
    A_pose = 0.5 * np.linalg.norm(np.cross(V2_pose - V1_pose, V3_pose - V1_pose))
    A_cano = 0.5 * np.linalg.norm(np.cross(V2 - V1, V3 - V1))
    s_pose = (A_pose / A_cano) * gaussian['s_canon']
    
    return mu_pose, q_pose, s_pose
```

**Parameters:**
*   Poisson reconstruction depth/resolution: Not given in the sources.
*   Adaptive layer thickness: Not given in the sources.

**Edge Cases:**
*   **Boundary Crossing:** Failure to handle barycentric coordinate boundaries causes Gaussians to stick to boundaries and pile up [S2].
*   **Mesh Degeneracy:** If the canonical triangle area $A_{cano}$ is zero or near-zero, the scale computation $s_{pose}$ becomes numerically unstable [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Enables real-time deformation when combined with mesh-based Gaussian splatting representation [S2].
*   **Strengths:** Combines mesh editing ability with 3DGS rendering quality [S2]. Enhances detail and volumetric effects via adaptive layer [S2].
*   **Limitations:** Relies on the quality of the extracted base mesh [S2].

## Background
SuGaR was introduced to address the lack of explicit geometry in 3D Gaussian Splatting, allowing for mesh extraction and editing [S1, unofficial]. Gaussian Frosting extends this by introducing an adaptive layer for better detail preservation [S2]. These methods are used in applications requiring both high-fidelity rendering and explicit mesh manipulation, such as character animation and scene editing [S2].

## Sources
[S1] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
