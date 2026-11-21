# Mesh-embedded representation

Also called: Mesh-embedded Gaussian Splatting, Gaussian Mesh Splatting, SuGaR, Gaussian Frosting, Mesh to DRK Conversion, Gaussian Extraction Algorithm, GauS

## Overview
Mesh-embedded representations integrate explicit triangle mesh topology with 3D Gaussian Splatting (3DGS) primitives to enable coherent deformation, editing, and surface-aligned rendering [S1, unofficial]. Unlike standard 3DGS, which treats Gaussians as independent entities in free space, this family binds Gaussian parameters (position, rotation, scale) to mesh geometry via barycentric coordinates or vertex parameterization, ensuring that mesh edits or animations propagate deterministically to the radiance field [S1, unofficial]. This approach solves the problem of Gaussian drift and incoherent motion transfer during animation, while providing a bridge between implicit neural/radiance representations and explicit editable mesh assets [S1, unofficial].

## Shared principles
The core mechanism relies on parameterizing Gaussian attributes relative to a canonical triangle mesh $\mathcal{M}$ with vertices $V$, normals $n$, and faces $F$.

**Embedding Definition:**
Each Gaussian $i$ is defined by an embedding tuple $E_i = \{k, u, v, d\}$ :
*   $k$: The index of the triangle $k \in F$ in the canonical mesh.
*   $(u, v)$: Barycentric coordinates within triangle $k$, such that $u \ge 0, v \ge 0, u+v \le 1$.
*   $d$: Normal displacement along the interpolated surface normal.

**Canonical Position and Normal:**
The canonical mean $\mu_{cano}$ and normal $n_{cano}$ are computed via Phong interpolation on the canonical mesh [Parent, S2, unofficial]:
$$
\mu_{cano} = \mathcal{V}(k, u, v) = u V_1 + v V_2 + (1 - u - v) V_3
$$
$$
n_{cano} = \mathcal{N}(k, u, v) = u n_1 + v n_2 + (1 - u - v) n_3
$$
where $V_j$ and $n_j$ are the vertices and per-vertex normals of triangle $k$ [Parent].

**Posed State Computation:**
When the mesh is deformed (e.g., via skeletal animation or blend shapes), the posed position $\mu_{pose}$ is derived by applying the mesh deformation function $\text{Deform}(\cdot)$ to the displaced canonical position [Parent]:
$$
\mu_{pose} = \text{Deform}(\mu_{cano} + d \cdot n_{cano})
$$
The posed rotation $q_{pose}$ and scale $s_{pose}$ are derived from mesh attributes to maintain coherence [Parent]:
*   **Rotation:** Interpolate per-vertex quaternions $q_1, q_2, q_3$ using barycentric weights to get $\delta q$, then combine with canonical rotation $\bar{q}$:
    $$
    q_{pose} = \delta q * \bar{q}, \quad \delta q = u q_1 + v q_2 + (1 - u - v) q_3
    $$
*   **Scale:** Scale is proportional to the area change of the embedded triangle relative to the canonical state:
    $$
    s_{pose} = \left( \frac{A_{pose}}{A_{cano}} \right) \bar{s}
    $$
    where $A_{pose}$ and $A_{cano}$ are the areas of the triangle in the posed and canonical states, respectively [Parent].

**Cross-Triangle Handling:**
Barycentric coordinates are only differentiable within a single triangle. If optimization moves an embedding point across a boundary ($u < 0$, $v < 0$, or $u+v > 1$), the embedding must be re-expressed in the neighboring triangle to maintain differentiability [Parent, S2, unofficial]. Failure to handle this causes Gaussians to stick to boundaries and pile up [Parent].

## Subtopics

### [Mesh-Embedded Gaussian Splatting](9553-mesh-embedded-gaussian-splatting.md)
A hybrid representation that defines 3D Gaussians over an explicit triangle mesh (e.g., SMPL-X, FLAME) to enable large-scale, interactive deformations. It solves the problem of coherent motion transfer by tying Gaussian parameters to mesh geometry via joint refinement, where GS rendering guides adaptive splitting and mesh constraints regularize Gaussian shapes. It sets itself apart from siblings by utilizing a reconstructed explicit mesh for initialization and employing specific face-split and normal-guidance strategies to handle large deformations without relying on implicit MLP-based skinning fields. Unlike methods using MLP-based linear blend skinning (LBS), it controls Gaussian rotation and translation directly by the mesh, enabling compatibility with skeletal animation and mesh editing.

### [Mesh-Embedded Gaussian Initialization and Deformation](9559-mesh-embedded-gaussian-initialization-and-deformation.md)
This method anchors 3D Gaussians to a triangle mesh using barycentric coordinates and normal displacement, enabling deformation tracking through mesh topology rather than direct vertex positions. It solves the problem of Gaussians drifting off-mesh or piling up at triangle edges during optimization and animation by defining a local triangle space. Unlike methods relying on precise mesh accuracy, it ensures robust results even with inaccurate meshes by binding Gaussians to the surface.

### [GaMeS (Gaussian Mesh Splatting)](12542-games-gaussian-mesh-splatting.md)
GaMeS is a hybrid representation that integrates 3DGS with mesh topology by parameterizing each Gaussian component using the vertices of a mesh face. It solves the problem of conditioning and editing Gaussian splats by ensuring that means, rotations, and scales automatically adapt when the underlying mesh geometry is modified. Unlike siblings such as SuGaR, GaMeS avoids expensive pre-processing stages or SDF-based regularization, directly embedding Gaussians onto the mesh surface or a "Triangle Soup" structure.

### [SuGaR / Gaussian Frosting](13542-sugar-gaussian-frosting.md)
SuGaR (Surface-Aligned Gaussian Splatting) and Gaussian Frosting bridge the gap between implicit Gaussian representations and explicit mesh geometry. SuGaR aligns optimized 3D Gaussians to a surface and extracts a base mesh via Poisson reconstruction. Gaussian Frosting builds on this by binding Gaussians to the extracted mesh surface to enable hybrid rendering, combining the editing capabilities of meshes with the high-fidelity rendering quality of 3DGS. It introduces an adaptive Gaussian layer with variable thickness to enhance detail and volumetric effects.

### [Mesh to DRK Conversion](13700-mesh-to-drk-conversion.md)
Mesh to DRK Conversion is a training-free procedure that converts traditional triangle and polygon mesh faces into Deformable Radial Kernel (DRK) primitives for seamless integration into radiance field scenes. It differs from siblings like [Mesh-Embedded Gaussian Splatting](9553-mesh-embedded-gaussian-splatting.md) and [SuGaR / Gaussian Frosting](13542-sugar-gaussian-frosting.md), which typically require optimization or learning-based initialization, by providing a direct, deterministic mapping that eliminates the need for training data rendering or model optimization.

### [Gaussian Extraction Algorithm](15577-gaussian-extraction-algorithm.md)
The Gaussian Extraction Algorithm extracts discrete 3D Gaussian primitives from continuous Gaussian Splatting Functions (GauPF) at arbitrary densities. It solves the problem of converting a continuous, neural-network-parameterized probability field into a finite set of splattable Gaussians, analogous to how Marching Cubes extracts meshes from Signed Distance Functions. It is distinguished from siblings by its use of an octree-guided progressive sampling strategy to locate high-probability regions, followed by gradient-based optimization of proxy points, rather than fixed-grid voxelization or direct mesh embedding.

### GauS (Gaussian Splatting Surface Reconstruction)
**Definition:** GauS is a render-then-fuse approach for high-fidelity mesh reconstruction from 3DGS inputs without fine-tuning. It uses volumetric fusion to convert 3D Gaussians into textured triangle meshes. It is a plug-and-play module compatible with various GS-based methods [S1, unofficial].
**Comparison:** Unlike NeRF-based re-training or further fine-tuning required for explicit surface modeling in other methods, GauS converts optimized Gaussians directly into textured meshes serving as a versatile plug-and-play solution [S1, unofficial].

## Background
The concept of embedding Gaussian primitives onto explicit geometry emerged to address the limitations of free-space 3DGS in handling dynamic scenes and editable assets. Early works like [Mesh-Embedded Gaussian Splatting](9553-mesh-embedded-gaussian-splatting.md) and [Mesh-Embedded Gaussian Initialization and Deformation](9559-mesh-embedded-gaussian-initialization-and-deformation.md) introduced barycentric parameterization to tie Gaussians to canonical meshes (e.g., SMPL-X) for animation [S1, unofficial]. Subsequent developments like [GaMeS](12542-games-gaussian-mesh-splatting.md) and [SuGaR / Gaussian Frosting](13542-sugar-gaussian-frosting.md) refined this by aligning Gaussians to reconstructed surfaces or extracting meshes from optimized Gaussians, enabling hybrid rendering and editing [S1, unofficial]. The family also includes training-free conversions like [Mesh to DRK Conversion](13700-mesh-to-drk-conversion.md) and neural-to-discrete extraction methods like the [Gaussian Extraction Algorithm](15577-gaussian-extraction-algorithm.md), expanding the applicability of mesh-embedded concepts to broader radiance field representations [S1, unofficial].

## Sources
[S1] GauStudio: A Modular Framework for 3D Gaussian Splatting and Beyond, https://doi.org/10.48550/arXiv.2403.19632, fetched 2026-10-30 (unofficial)
