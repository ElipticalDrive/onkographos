# Semantic and Structural Extensions

## Overview
This family extends 3D Gaussian Splatting by attaching high-dimensional feature vectors to Gaussian primitives or by modeling animatable human avatars from multi-view data [S1]. It differs from standard rendering by accumulating these feature vectors via differentiable alpha-blending rather than just color, and from sibling methods by rendering feature maps directly through differentiable alpha-compositing [8130-semantic-and-language-embeddings.md]. For structural extensions, it solves dynamic appearance and geometry reconstruction by maintaining a canonical space representation where Gaussians are deformed via differentiable functions driven by pose parameters before rasterization [8209-human-and-articulated-avatars.md].

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

### [Semantic and Language Embeddings](8130-semantic-and-language-embeddings.md)
This method extends 3D Gaussian Splatting by attaching high-dimensional semantic or language feature vectors to each primitive, enabling simultaneous photometric and semantic rasterization within a unified differentiable alpha-compositing pipeline [8130-semantic-and-language-embeddings.md]. It solves the problem of decoupled geometry and semantics by hard-coding alignment between visual boundaries and semantic boundaries, eliminating boundary bleeding [8130-semantic-and-language-embeddings.md]. Unlike sibling methods that may rely on external segmentation heads or separate optimization streams, this approach renders feature maps directly through differentiable alpha-blending, allowing for real-time open-vocabulary semantic segmentation and CLIP-based comparison [8130-semantic-and-language-embeddings.md].

### [Human and Articulated Avatars](8209-human-and-articulated-avatars.md)
This method extends 3D Gaussian Splatting to reconstruct and render animatable human avatars from multi-view or single-view inputs [8209-human-and-articulated-avatars.md]. It solves the problem of dynamic geometry and appearance modeling by maintaining a canonical space representation where Gaussians are deformed via differentiable functions driven by pose parameters before rasterization [8209-human-and-articulated-avatars.md]. It sets itself apart from static reconstruction methods by enabling novel view synthesis of dynamic subjects through learned deformation fields, and from mesh-based methods by leveraging the explicit, differentiable nature of 3D Gaussians for high-fidelity texture and geometry [8209-human-and-articulated-avatars.md].

### 3D Gaussian Splatting on Mesh Surfaces
GVA proposes a surface-guided Gaussians re-initialization strategy to make the trained Gaussians better fit the input monocular video, addressing the unbalanced aggregation of Gaussians caused by densification and splitting operations [S1]. HAHA attaches Gaussians onto the surface of a mesh and combines rendered results from a textured human body mesh and Gaussians together to reduce the number of Gaussians [S1].

## Background
MonoGaussianAvatar first applies 3D Gaussian Splatting to dynamic head reconstruction using canonical space modeling and deformation prediction [S1]. PSAvatar introduces the explicit FLAME face model to initialize Gaussians, which can capture high-fidelity facial geometry and complicated volumetric objects [S1]. GaussianHead uses a tri-plane representation and motion fields to simulate geometrically changing heads in continuous movements [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
