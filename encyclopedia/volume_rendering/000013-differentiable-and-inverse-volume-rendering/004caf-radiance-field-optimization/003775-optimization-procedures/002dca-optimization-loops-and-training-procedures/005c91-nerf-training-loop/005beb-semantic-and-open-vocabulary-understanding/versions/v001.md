# Semantic and Open-Vocabulary Understanding

Also called: Semantic NeRF, Open-Vocabulary NeRF

## Overview
This family extends NeRF training to predict semantic features or segmentation logits alongside color and density, enabling object-level understanding and editing. It differs from standard NeRF training by incorporating external vision-language models (e.g., CLIP) or foundation models (e.g., SAM) to provide semantic supervision, addressing the challenge of view-inconsistent 2D labels through 3D-consistent refinement strategies like Cross-view Self-enhancement (CSE) and Region Semantic Ranking (RSR).

## Shared principles
All methods in this family augment the standard NeRF representation $\Gamma_\theta: (x, d) \to (c, \sigma)$ with a semantic branch.

1.  **Semantic Representation:** An additional MLP branch outputs a semantic vector $f_k \in \mathbb{R}^D$ for each sampled point $k$ along a ray [S1]. The feature dimension $D$ is typically 512 [S1].
2.  **Semantic Rendering:** Semantic features are volume-rendered similarly to color:
    $$ \hat{F}(r) = \sum_{k=1}^K T_k \alpha_k f_k $$
    where $T_k$ is transmittance and $\alpha_k$ is opacity [S1].
3.  **Logit Computation:** For open-vocabulary tasks, rendered semantic features $\hat{F}(r)$ are compared against text embeddings $t_c$ (from CLIP) to produce segmentation logits:
    $$ \hat{S}(r) = \cos\langle t_c, \hat{F}(r) \rangle $$
    [S1].
4.  **Supervision Sources:** Supervision comes from 2D vision-language models (CLIP) or segmentation models (SAM). Direct use of 2D CLIP maps is noisy and view-inconsistent, necessitating regularization (RSR) and self-enhancement (CSE) [S1].

## Subtopics

### [Cross-view Self-enhancement](13434-cross-view-self-enhancement.md)
Cross-view Self-enhancement (CSE) is a training strategy that replaces view-inconsistent 2D semantic labels with 3D-consistent pseudo-maps rendered from the model itself [13434-cross-view-self-enhancement.md]. It addresses the ambiguity where 2D vision-language models fail to maintain consistent object identities across diverse perspectives [13434-cross-view-self-enhancement.md]. Unlike methods relying solely on external supervision, CSE leverages the inherent 3D consistency of NeRFs to iteratively refine supervision [13434-cross-view-self-enhancement.md]. It operates by generating refined pseudo relevancy maps for training views (Iterative Pseudo Map Update) and unseen viewpoints (Novel Pseudo Map Synthesis), both regularized by Region Semantic Ranking (RSR) to ensure fine edges and integral objects [13434-cross-view-self-enhancement.md]. Choose this when view-inconsistency in 2D labels degrades multi-view semantic consistency.

### [OV-NeRF](13995-ov-nerf.md)
OV-NeRF is a method for accurate NeRF-based open-vocabulary 3D semantic understanding that leverages pre-trained vision and language foundation models (CLIP and SAM) [13995-ov-nerf.md]. It addresses limitations of prior methods (e.g., LERF, 3DOVS) that directly use noisy CLIP relevancy maps [13995-ov-nerf.md]. OV-NeRF introduces Region Semantic Ranking (RSR) regularization to rectify coarse boundaries in single views using SAM mask proposals, and Cross-view Self-enhancement (CSE) to enforce view-consistent supervision using refined 3D semantics [13995-ov-nerf.md]. Choose this for open-vocabulary segmentation requiring precise object boundaries and 3D consistency.

### ISRF
ISRF is a method that learns additional feature fields to realize self-supervised 3D segmentation [S1]. It is part of a series of works (alongside N3F, LERF) that extend NeRFs with feature fields for object extraction and deletion based on image patches or text [S1].

## Background
Semantic NeRFs emerged alongside the rise of vision-language models like CLIP and segmentation foundation models like SAM. Early methods like DFF used CLIP for feature distillation [S1]. Subsequent works like N3F, LERF, and ISRF learned additional feature fields for self-supervised 3D segmentation [S1]. The integration of SAM enabled prompt-based 3D segmentation (e.g., SA3D) [S1]. Recent surveys highlight these methods as key components of 3D editing pipelines [S1].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
