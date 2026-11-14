# Modular, Hierarchical, and Latent Architectures

Also called: HyperNeRF, Strata-NeRF

## Overview
This family extends standard NeRF architectures to handle non-rigid deformation with topological changes and hierarchical scene structures, respectively. [HyperNeRF](13906-hypernerf.md) models non-rigidly deforming scenes by lifting the canonical 3D space into a higher-dimensional ambient space, allowing smooth topological variations without discontinuous deformation mappings. [Strata-NeRF](14015-strata-nerf.md) models stratified scenes (e.g., interior/exterior) by replacing continuous positional encoding inputs with discrete latent codes, preventing the "cloudy" artifacts that arise when training a single continuous field on hierarchical levels.

## Shared principles
The standard NeRF architecture [S6, unofficial] consists of two Multi-Layer Perceptrons (MLPs): a coarse network and a fine network. The coarse network takes positional encoded position $\gamma(\mathbf{x})$ and outputs a latent feature vector $\mathbf{h}$ and volume density $\sigma$. The fine network takes $\mathbf{h}$, positional encoded position $\gamma(\mathbf{x})$, and positional encoded view direction $\gamma(\mathbf{d})$ to output color $\mathbf{c}$.

$$
\gamma(\mathbf{x}, d_1, d_2) = [\mathbf{x}, \sin(2^{d_1}\pi\mathbf{x}), \cos(2^{d_1}\pi\mathbf{x}), \dots, \sin(2^{d_2-1}\pi\mathbf{x}), \cos(2^{d_2-1}\pi\mathbf{x})]
$$

Modifications generally fall into three categories:
1.  **Input Expansion:** Adding new conditioning vectors (e.g., appearance embeddings $\mathbf{z}_i$, fluid volume $v$, or previous stage RGB) to the input concatenation.
2.  **Output Expansion:** Adding auxiliary heads to predict values other than $\sigma$ and $\mathbf{c}$, such as semantic labels $y_{aux}$, object codes $o$, or area encoding probabilities $p$. These are rendered using the same transmittance weights $T_i$ and opacity $\alpha_i$ as color:
    $$
    \hat{Y}(r) = \sum_{i=1}^{N} T_i \alpha_i y_{aux}(x_i)
    $$
3.  **Representation Substitution:** Replacing the continuous MLP function with discrete structures (voxel grids, hash tables) or different implicit surfaces (SDFs).

## Subtopics

### [HyperNeRF](13906-hypernerf.md)
HyperNeRF models non-rigidly deforming scenes with topological variations (e.g., openings, closings) by extending the canonical 3D coordinate space into a higher-dimensional ambient space. Unlike continuous deformation fields which struggle with discontinuities, HyperNeRF represents each observation frame as a non-planar slice through this hyperdimensional template. It sets itself apart from sibling methods like Strata-NeRF by explicitly using higher-dimensional ambient coordinates and deformable slicing manifolds to handle topology, rather than stratified sampling or static hierarchical structures.

### [Strata-NeRF](14015-strata-nerf.md)
Strata-NeRF is a neural radiance field method designed to model "stratified scenes"—environments deconstructable into distinct hierarchical tiers (e.g., exterior and interior of a building). Unlike standard continuous NeRFs that produce cloudy artifacts and white haziness when trained jointly on multiple hierarchical levels, Strata-NeRF uses Vector Quantized (VQ) latent representations to explicitly capture sudden structural changes between levels. It achieves this by conditioning a single radiance field network on discrete latent codes generated from 3D positions, allowing for seamless switching between scene hierarchies without training independent models for each level.

#### Strata-NeRF Architecture
- [S1] D1: Disable second router in Latent Router. D2: Disable first router in Latent Router. D3: Remove Latent Router, directly concatenate generated embedding with positional encoding. D4: Replace VQ-VAE with VAE in Latent Generator.
- [S1] Using two parallel dense layers in the Latent Router outperforms individual dense layers. The full Latent Router architecture is superior to direct concatenation or VAE substitution.

## Background
HyperNeRF was introduced to address the limitations of continuous deformation fields (like Nerfies) in handling topological changes, where band-limited MLPs fail to model discontinuities [S2]. Strata-NeRF was developed to solve the issue of joint training artifacts in hierarchical scenes, where standard continuous NeRFs (like Mip-NeRF 360) produce unsatisfactory "cloudy" views when trained on multiple levels simultaneously [S1, S2].

## Sources
[S1] Strata-NeRF : Neural Radiance Fields for Stratified Scenes, https://arxiv.org/abs/2308.10337v1, fetched 2026-10-26
