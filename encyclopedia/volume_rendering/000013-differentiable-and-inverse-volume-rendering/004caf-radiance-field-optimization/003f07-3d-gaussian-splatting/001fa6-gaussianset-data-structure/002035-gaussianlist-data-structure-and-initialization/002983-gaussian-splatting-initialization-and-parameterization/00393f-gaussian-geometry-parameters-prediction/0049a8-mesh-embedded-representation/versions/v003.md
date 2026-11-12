# Mesh-embedded representation
Also called: Mesh-embedded Gaussian Splatting, Mesh-to-DRK Conversion, GaMeS
## Overview
Mesh-embedded representations hybridize explicit triangle meshes with 3D Gaussian Splatting to solve the problem of animating high-fidelity, detailed appearances and integrating traditional 3D assets into radiance field scenes [S1, unofficial][S2, unofficial]. Unlike sibling methods that rely on MLP-based linear blend skinning (LBS) fields or direct vertex optimization for deformation, this family controls Gaussian rotation, translation, and scale directly via mesh topology and barycentric coordinates, enabling compatibility with skeletal animation, blend shapes, and mesh editing [S2, unofficial][S1, unofficial]. It bridges the gap between millions of traditional 3D assets and fast, high-quality reconstruction representations, enabling unified rendering, VR digital world creation, and AR scene enhancement [S1, unofficial].

## Shared principles
All methods in this family parameterize 3D Gaussians using embedding coordinates on a canonical triangle mesh. Each Gaussian $i$ is defined by embedding parameters $E_i$ and Gaussian parameters.

**Embedding Parameters ($E_i$):**
*   $k$: Triangle index (integer).
*   $(u, v)$: Barycentric coordinates within triangle $k$ (floats, $u \ge 0, v \ge 0, u+v \le 1$).
*   $d$: Normal displacement along the interpolated normal (float, optional in some variants like GaMeS).

**Canonical Position and Normal:**
The canonical position $\mu_{cano}$ and normal $n_{cano}$ are computed via Phong interpolation [S2, unofficial]:
$$
\mu_{cano} = \mathcal{V}(k, u, v) = u V_1 + v V_2 + (1 - u - v) V_3
$$
$$
n_{cano} = \mathcal{N}(k, u, v) = u n_1 + v n_2 + (1 - u - v) n_3
$$
where $V_1, V_2, V_3$ are the vertices of triangle $k$, and $n_1, n_2, n_3$ are the per-vertex normals [S2, unofficial].

**Deformation:**
For a posed frame, the posed position $\mu_{pose}$ is obtained by applying mesh deformation to the canonical position and adding normal displacement:
$$
\mu_{pose} = \text{Deform}(\mu_{cano} + d \cdot n_{cano})
$$
where $\text{Deform}(\cdot)$ applies skeletal or blend-shape animation to the mesh vertices [S2, unofficial].

**Rotation:**
The posed rotation $q_{pose}$ is interpolated from per-vertex quaternions using barycentric weights and combined with the canonical rotation [S2, unofficial]:
$$
q_{pose} = \delta q * \bar{q}
$$
where $\delta q = u q_1 + v q_2 + (1 - u - v) q_3$ is the interpolated rotation from the mesh vertices, and $*$ denotes quaternion multiplication [S2, unofficial].

**Scale:**
The posed scale $s_{pose}$ is derived from the area change of the embedded triangle relative to the canonical state [S2, unofficial]:
$$
s_{pose} = \left( \frac{A_{pose}}{A_{cano}} \right) \bar{s}
$$
where $A_{pose}$ and $A_{cano}$ are the areas of the triangle in the posed and canonical states, respectively [S2, unofficial].

**Cross-Triangle Handling:**
Barycentric coordinates are only differentiable within a single triangle. If an optimization step moves the embedding point across a triangle boundary, the embedding must be re-expressed in the neighboring triangle to maintain differentiability [S2, unofficial]. Failure to handle this causes Gaussians to stick to boundaries and pile up [S2, unofficial].

## Subtopics

### [Mesh-Embedded Gaussian Splatting](9553-mesh-embedded-gaussian-splatting.md)
Mesh-Embedded Gaussian Splatting is a hybrid 3D representation that parameterizes 3D Gaussians using embedding coordinates on a canonical triangle mesh. It solves the problem of animating high-fidelity, detailed appearances by disentangling low-frequency motion (handled by the explicit mesh) from high-frequency geometry and appearance (handled by the Gaussians). Unlike sibling methods that rely on MLP-based linear blend skinning (LBS) fields for deformation, this method controls Gaussian rotation and translation directly via the mesh, enabling compatibility with skeletal animation, blend shapes, and mesh editing.

### [Mesh-Embedded Gaussian Initialization and Deformation](9559-mesh-embedded-gaussian-initialization-and-deformation.md)
This method initializes and deforms 3D Gaussians by anchoring them to a canonical triangle mesh via barycentric coordinates and normal displacement. It solves the problem of maintaining geometric consistency and differentiability during animation by tracking deformation through mesh topology rather than direct vertex positions. It differs from sibling methods like [Mesh-Embedded Gaussian Splatting](9553-mesh-embedded-gaussian-splatting.md) by explicitly defining the "walking on triangle" constraint mechanism to prevent Gaussians from crossing triangle boundaries during optimization.

### [GaMeS (Gaussian Mesh Splatting)](12542-games-gaussian-mesh-splatting.md)
GaMeS is a hybrid representation that integrates 3D Gaussian Splatting with mesh topology, allowing Gaussian components to be parameterized by the vertices of mesh faces. It solves the problem of conditioning and editing large numbers of independent Gaussians by tying their mean, rotation, and scale directly to the geometry of a canonical triangle mesh. Unlike sibling methods such as [Mesh-Embedded Gaussian Splatting](9553-mesh-embedded-gaussian-splatting.md) which may rely on expensive pre-processing or SDF regularization, GaMeS directly constructs Gaussians on mesh faces or a "Triangle Soup" derived from flat splats, enabling automatic adaptation of Gaussian parameters when the underlying mesh is deformed.

### [Mesh to DRK Conversion](13700-mesh-to-drk-conversion.md)
Mesh to DRK Conversion is a training-free procedure that converts traditional triangle and polygon mesh faces into Deformable Radial Kernel (DRK) primitives for seamless integration into radiance field scenes. Unlike existing splatting methods (2D-GS, 3D-GS, 3D-HGS, GES) which fail to integrate traditional assets seamlessly, this method provides a direct, training-free conversion pipeline. It bridges the gap between millions of traditional 3D assets and fast, high-quality reconstruction representations, enabling unified rendering, VR digital world creation, and AR scene enhancement.

### SuGaR / Gaussian Frosting
SuGaR aligns optimized Gaussians to a surface and extracts it via Poisson reconstruction, while Frosting binds Gaussians to a mesh surface for hybrid rendering [S1, unofficial].

## Background
Origin: SuGaR and Gaussian Frosting are mentioned in the context of mesh extraction from optimized Gaussians [S1, unofficial]. Mesh-embedded representations generally aim to combine the editability of meshes with the rendering speed of Gaussian Splatting.

## Sources
[S1] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] Mesh-Embedded Gaussian Splatting notes, unofficial [S2, unofficial]
