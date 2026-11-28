# Human and Articulated Avatars


## Overview
This family extends 3D Gaussian Splatting to reconstruct and render animatable human avatars from multi-view or single-view inputs [S1][S2, unofficial]. It solves the problem of dynamic geometry and appearance modeling by maintaining a canonical space representation where Gaussians are deformed via differentiable functions driven by pose parameters (e.g., skeleton joints, expression coefficients) before rasterization [G]. It sets itself apart from static reconstruction methods by enabling novel view synthesis of dynamic subjects through learned deformation fields, and from mesh-based methods by leveraging the explicit, differentiable nature of 3D Gaussians for high-fidelity texture and geometry [G].

## Shared principles
All methods in this family rely on the differentiable alpha-blending accumulation of feature vectors or colors during the rasterization step [G]. For a pixel $p$, the rendered vector $\mathbf{V}(p)$ is computed by sorting overlapping Gaussians $\mathcal{O}_p$ front-to-back and applying:

$$
\mathbf{V}(p) = \sum_{i \in \mathcal{O}_p} \mathbf{v}_i \cdot \alpha_i \cdot \prod_{j=1}^{i-1} (1 - \alpha_j)
$$

where:
*   $\mathbf{v}_i$ is the vector (feature $\mathbf{f}_i$ or color $\mathbf{c}_i$) of the $i$-th Gaussian [G].
*   $\alpha_i$ is the opacity of the $i$-th Gaussian [G].
*   $\prod_{j=1}^{i-1} (1 - \alpha_j)$ is the transmittance from all preceding Gaussians $j < i$ [G].
*   If $\mathcal{O}_p$ is empty, $\mathbf{V}(p) = \mathbf{0}$ [G].

For structural extensions, all methods share a canonical space representation where 3D Gaussians are initialized in a reference pose (e.g., T-pose or neutral expression) [G]. Deformation is achieved by mapping these canonical Gaussians to a target pose defined by parameters (e.g., skeleton joints, expression coefficients) [G]. The rendering pipeline remains differentiable, allowing optimization of Gaussian attributes and deformation parameters via photometric loss [G].

The deformation function $D(\cdot)$ maps a canonical Gaussian $i$ with position $\mathbf{p}_i^{\text{canonical}}$ to a target position $\mathbf{p}_i^{\text{target}}$ based on pose parameters $\theta$:
$$
\mathbf{p}_i^{\text{target}} = D(\mathbf{p}_i^{\text{canonical}}, \theta)
$$
The rotation and scale of the Gaussian may also be transformed by the deformation function to maintain physical consistency [G].

## Subtopics

### [GaussianHead](8051-gaussianhead.md)
GaussianHead reconstructs animatable head avatars by combining a tri-plane representation with motion fields to simulate geometrically changing heads during continuous movements. It renders rich textures, including skin and hair. It builds on the canonical space deformation principles of [Human and Articulated Avatars](8209-human-and-articulated-avatars.md).

### [GART](8093-gart.md)
GART extends human models to more articulated models based on the 3DGS representation. It builds on [Human and Articulated Avatars](8209-human-and-articulated-avatars.md) [G].

### [D3GA](8094-d3ga.md)
D3GA creates animatable human avatars using drivable 3D Gaussians and tetrahedral cages for full-body modeling. It addresses dynamic human reconstruction from multi-view videos by providing promising geometry and appearance modeling. It sets itself apart from head-only siblings like [GaussianHead](8051-gaussianhead.md), [GART](8093-gart.md), and [HeadGas](17293-headgas.md) by extending the drivable Gaussian representation to the full body.

### [HeadGas](17293-headgas.md)
HeadGaS is a method for real-time animatable 3D head reconstruction and animation that extends 3D Gaussian Splatting (3DGS) with a base of learnable latent features. It solves the problem of modeling dynamic facial expressions by allowing Gaussians to alter their opacity and color over time, achieving dynamics via over-representation (duplicates of dynamic face areas that become active one at a time) rather than explicit geometric deformation. It sets itself apart from siblings like [GaussianHead](8051-gaussianhead.md) and [GART](8093-gart.md) by blending latent features with 3DMM expression weights to predict appearance, avoiding the complexity of optimizing 3D motion for every Gaussian.

### 3D Gaussian Splatting for Human101
Push the speed of high-fidelity dynamic human creation to 100 seconds using a fixed-perspective camera [S1][S2, unofficial].

### SplattingAvatar / GoMAvatar
Embed Gaussians onto a canonical human body mesh [S2, unofficial]. Determine the position of a Gaussian by the barycenter and displacement along the normal direction [S2, unofficial].

### 3DPSHR
Combine hand geometry priors (MANO) with 3DGS [S2, unofficial]. Provide the first real-time hand reconstruction [S2, unofficial].

### MonoGaussianAvatar
Apply 3DGS to dynamic head reconstruction using canonical space modeling and deformation prediction [S2, unofficial].

### MANUS
Explore the interaction between hands and objects using 3DGS [S2, unofficial].

## Background
The concept of animatable avatars using 3D Gaussian Splatting emerged as an extension of static reconstruction methods to handle dynamic subjects [G]. Early works like GaussianAvatar incorporated optimizable tensors with dynamic appearance networks to capture dynamics [S1]. Human101 further optimized the training speed for high-fidelity dynamic human creation, achieving 100 seconds using a fixed-perspective camera [S1][S2, unofficial]. Subsequent methods like SplattingAvatar and GoMAvatar embedded Gaussians onto canonical human body meshes, determining positions via barycentric coordinates and normal displacements [S1][S2, unofficial]. For head modeling, MonoGaussianAvatar applied canonical space modeling and deformation prediction [S2, unofficial], while GaussianHead introduced tri-plane representations and motion fields for continuous movement simulation [S2, unofficial]. Extensions to other body parts include 3DPSHR for real-time hand reconstruction using MANO priors [S2, unofficial] and MANUS for hand-object interaction [S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
