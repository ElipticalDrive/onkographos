# Semantic and Structural Extensions

## Overview
This family extends 3D Gaussian Splatting by attaching high-dimensional feature vectors (semantic, language, or latent) to Gaussian primitives or by modeling animatable human and articulated animal avatars from multi-view data [S1]. It differs from standard rendering by accumulating these feature vectors via differentiable alpha-blending rather than just color, and from sibling methods by leveraging 2D vision-language models to distill features without explicit 2D-to-3D lifting networks during inference [S1]. For structural extensions, it solves dynamic appearance and geometry reconstruction by maintaining a canonical 3D Gaussian field and applying differentiable deformation to map primitives to target poses, distinguished by its focus on geometric articulation rather than semantic feature accumulation.

## Shared principles
All methods in this family rely on the differentiable alpha-blending accumulation of feature vectors or colors during the rasterization step. For a pixel $p$, the rendered vector $\mathbf{V}(p)$ is computed by sorting overlapping Gaussians $\mathcal{O}_p$ front-to-back and applying:

$$
\mathbf{V}(p) = \sum_{i \in \mathcal{O}_p} \mathbf{v}_i \cdot \alpha_i \cdot \prod_{j=1}^{i-1} (1 - \alpha_j)
$$

where:
*   $\mathbf{v}_i$ is the vector (feature $\mathbf{f}_i$ or color $\mathbf{c}_i$) of the $i$-th Gaussian.
*   $\alpha_i$ is the opacity of the $i$-th Gaussian.
*   $\prod_{j=1}^{i-1} (1 - \alpha_j)$ is the transmittance from all preceding Gaussians $j < i$.
*   If $\mathcal{O}_p$ is empty, $\mathbf{V}(p) = \mathbf{0}$.

For structural extensions, all methods share a canonical space representation where 3D Gaussians are initialized in a reference pose (e.g., T-pose or neutral expression). Deformation is achieved by mapping these canonical Gaussians to a target pose defined by parameters (e.g., skeleton joints, expression coefficients). The rendering pipeline remains differentiable, allowing optimization of Gaussian attributes and deformation parameters via photometric loss.

The deformation function $D(\cdot)$ maps a canonical Gaussian $i$ with position $\mathbf{p}_i^{canonical}$ to a target position $\mathbf{p}_i^{target}$ based on pose parameters $\theta$:
$$
\mathbf{p}_i^{target} = D(\mathbf{p}_i^{canonical}, \theta)
$$
The rotation and scale of the Gaussian may also be transformed by the deformation function to maintain physical consistency [G].

## Subtopics

### [Semantic and Language Embeddings](8130-semantic-and-language-embeddings.md)
This method extends 3D Gaussian Splatting by attaching high-dimensional feature vectors (semantic, language, or latent) to Gaussian primitives, enabling real-time open-vocabulary semantic segmentation and phrase grounding. It differs from standard rendering by accumulating these feature vectors via differentiable alpha-blending [Parent] rather than just color, and from sibling methods by leveraging 2D vision-language models (like CLIP or LSeg) to distill or guide these features without explicit 2D-to-3D lifting networks during inference. The feature dimension $d$ varies by method (e.g., 512 for CLIP, 768 for LSeg/CLIP text, or compressed latent dimensions). The rendering pipeline produces a 2D feature map $\mathbf{F} \in \mathbb{R}^{H \times W \times d}$, which is subsequently compared against text embeddings or decoded to produce segmentation masks.

### [Human and Articulated Avatars](8209-human-and-articulated-avatars.md)
This method extends 3D Gaussian Splatting to reconstruct animatable human and articulated animal avatars from multi-view video or image data. It solves the problem of dynamic appearance and geometry reconstruction by maintaining a canonical 3D Gaussian field and applying differentiable deformation to map primitives to target poses. It is distinguished from [Semantic and Structural Extensions](17384-semantic-and-structural-extensions.md) by its focus on geometric articulation (skeleton/mesh-driven) rather than semantic feature accumulation, and from [Gaussian Splatting](52-gaussian-splatting.md) by its explicit optimization of deformation parameters alongside Gaussian attributes.

### 3D Gaussian Splatting on Mesh Surfaces
GVA [222] proposes a surface-guided Gaussians re-initialization strategy to make the trained Gaussians better fit the input monocular video, addressing the unbalanced aggregation of Gaussians caused by densification and splitting operations. HAHA [223] attaches Gaussians onto the surface of a mesh and combines rendered results from a textured human body mesh and Gaussians together to reduce the number of Gaussians [S1].

## Background
The semantic extension family leverages the differentiable nature of 3D Gaussian Splatting to support open-vocabulary queries, a capability not natively present in standard color-based rendering. The structural extension family addresses the limitations of static 3DGS in modeling dynamic humans, building on the canonical space modeling and deformation prediction techniques introduced for dynamic head reconstruction [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
