# Mesh-embedded representation
## Overview
Mesh-embedded representations define 3D Gaussian geometry (position, rotation, scale) as a function of attributes on an underlying triangle mesh, rather than as independent learnable vectors [S2, unofficial]. This approach solves the problem of disentangling motion from appearance in dynamic scenes, such as human avatars, by allowing the mesh to control low-frequency deformation while Gaussians capture high-frequency details [S2, unofficial]. Unlike sibling methods that rely on MLP-based linear blend skinning (LBS) fields for motion, this approach controls Gaussian rotation and translation directly by the mesh, enabling compatibility with skeletal animation, blend shapes, and mesh editing [S2, unofficial]. It builds on [Gaussian Geometry Parameters Prediction](8442-gaussian-geometry-parameters-prediction.md).
## Shared principles
All methods in this family parameterize the Gaussian mean $\mu$ and orientation $q$ using embedding coordinates on a canonical triangle mesh. The embedding $E_i$ for each Gaussian $i$ typically consists of a triangle index $k$, barycentric coordinates $(u, v)$, and a normal displacement $d$. The canonical position is computed via Phong interpolation of the mesh vertices and normals:
$$
\mu_{cano} = P(k, u, v) + d \cdot n(k, u, v)
$$
where $P(k, u, v)$ is the barycentric interpolation of the triangle vertices and $n(k, u, v)$ is the interpolated normal. During deformation to a posed state, the Gaussian parameters are transformed based on the mesh deformation. Rotation is often interpolated from per-vertex quaternions using barycentric weights:
$$
q_{pose} = \delta q * \bar{q}
$$
where $\delta q$ is the interpolated rotation from the mesh and $\bar{q}$ is the canonical rotation. Scale is typically derived from the area change of the embedded triangle relative to the canonical state:
$$
s_{pose} = \left( \frac{A_{pose}}{A_{cano}} \right) \bar{s}
$$
A critical implementation detail is handling cross-triangle updates. Since barycentric coordinates are only differentiable within a triangle, if an optimization step moves the embedding point across a triangle boundary, the embedding must be re-expressed in the neighboring triangle to maintain differentiability and gradient flow [S2, unofficial]. Failure to handle this causes Gaussians to stick to boundaries and pile up [S1, unofficial].
## Subtopics

### [Mesh-Embedded Gaussian Splatting](9553-mesh-embedded-gaussian-splatting.md)
Mesh-Embedded Gaussian Splatting is a hybrid 3D representation that defines Gaussian geometry (position, rotation, scale) as a function of mesh attributes rather than independent learnable vectors. It solves the problem of disentangling motion from appearance in dynamic scenes, such as human avatars, by allowing the underlying mesh to control low-frequency deformation while Gaussians capture high-frequency details. Unlike sibling methods that rely on MLP-based linear blend skinning (LBS) fields for motion, this approach controls Gaussian rotation and translation directly by the mesh, enabling compatibility with skeletal animation, blend shapes, and mesh editing. It builds on [Mesh-embedded representation](12559-mesh-embedded-representation.md).

### [Mesh-Embedded Gaussian Initialization and Deformation](9559-mesh-embedded-gaussian-initialization-and-deformation.md)
This method anchors 3D Gaussians to a triangle mesh via barycentric coordinates and a normal displacement, allowing deformation tracking through mesh topology rather than direct vertex positions. It differs from sibling methods like [Mesh-Embedded Gaussian Splatting](9553-mesh-embedded-gaussian-splatting.md) by explicitly parameterizing the Gaussian mean as a displacement from the mesh surface, and from [GaMeS](12542-games-gaussian-mesh-splatting.md) by using barycentric interpolation for rotation and area-based scaling rather than face-vertex convex combinations.

### [GaMeS (Gaussian Mesh Splatting)](12542-games-gaussian-mesh-splatting.md)
GaMeS is a hybrid 3D representation that integrates mesh and Gaussian Splatting (GS) by parameterizing each Gaussian component directly by the vertices of a mesh face. This approach allows the modification of Gaussian components (position, scale, and rotation) in a manner similar to classical mesh editing, ensuring that modifying the underlying mesh automatically updates the Gaussian representation. Unlike sibling methods such as SuGaR, GaMeS does not require expensive pre-processing stages or regularization terms with Signed Distance Fields (SDFs); instead, it directly parameterizes Gaussians on mesh faces. It supports two input scenarios: mesh-initialized input, where a provided mesh is used, and mesh-free input, where flat Gaussians are parameterized to create a "Triangle Soup" structure.

### SuGaR / Gaussian Frosting
Mesh extraction from optimized Gaussians. SuGaR: align Gaussians to a surface; extract via Poisson reconstruction. Frosting: bind Gaussians to mesh surface for hybrid render [S1, unofficial].

## Background
Origin: The concept of embedding Gaussians on meshes emerged to address the limitations of independent Gaussian optimization in dynamic scenes, particularly for human avatars where motion is structured by skeletal meshes. Key implementations include Mesh-Embedded Gaussian Splatting (often associated with SplattingAvatar) and GaMeS. The approach leverages the well-established topology of parametric human models like SMPL-X or FLAME to drive Gaussian deformation.

## Sources
[S1] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
