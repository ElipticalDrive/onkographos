# Semantic and Language Embeddings

Also called: SAM-guided Gaussian Grouping

## Overview
This family extends 3D Gaussian Splatting by attaching high-dimensional semantic or language feature vectors to each primitive, enabling simultaneous photometric and semantic rasterization within a unified differentiable alpha-compositing pipeline [S2]. It solves the problem of decoupled geometry and semantics by hard-coding alignment between visual boundaries and semantic boundaries, eliminating boundary bleeding [S2]. Unlike sibling methods that may rely on external segmentation heads or separate optimization streams, this approach renders feature maps directly through differentiable alpha-blending, allowing for real-time open-vocabulary semantic segmentation and CLIP-based comparison [S1, unofficial].

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

### [3D Gaussian Splatting with Feature Vectors](8060-3d-gaussian-splatting-with-feature-vectors.md)
Extends 3DGS by attaching a 512-dimensional semantic embedding vector to each primitive, enabling simultaneous photometric and semantic rasterization. It eliminates boundary bleeding by ensuring geometric parameters dictate blending weights for both color and semantics. Choose when hard-coded alignment between visual and semantic boundaries is required without external segmentation heads.

### [Language Embedded 3D Gaussians](8103-language-embedded-3d-gaussians.md)
Encodes object-level embeddings for phrase grounding in addition to pixel-level features, enabling open-vocabulary semantic segmentation and zero-shot reasoning via CLIP alignment. Choose when full scene understanding and phrase grounding are needed.

### [LangSplat](8104-langsplat.md)
Distills vision-language features into a scene-specific latent space via an autoencoder to compress features, allowing efficient querying using natural language text embeddings. Choose when memory efficiency is critical and high-frequency detail preservation is secondary to language grounding.

### [3D Vision-Language GS](8105-3d-vision-language-gs.md)
Projects VLMs into the 3DGS framework to create a joint embedding space for 3D visual question answering and complex semantic queries. Choose when object-level embeddings and complex reasoning are required.

### [Text Embedding Generation with Directional Views](8239-text-embedding-generation-with-directional-views.md)
Generates semantic text embeddings by appending directional view descriptors (e.g., 'front', 'side') to prompts, conditioning generation on the camera's viewing angle. Choose when view-dependent semantic consistency is required.

### [LSeg Feature Encoding](12078-lseg-feature-encoding.md)
Distills knowledge from the LSeg vision-language model to provide pixel-level or patch-level semantic supervision. Choose when localized semantic editing and zero-shot labeling without explicit 3D annotation are needed.

### [VLM-Augmented Segmentation](15758-vlm-augmented-segmentation.md)
Compresses VLM embeddings into latent attributes per Gaussian using a scene-specific autoencoder, aggregating them via splatting and decoding for semantic alignment. Choose when high memory consumption of explicit high-dimensional modeling must be avoided while preserving volumetric consistency.

### CLIP-Based 3D Model Search
Searches a database of 3D models (3D-FUTURE) to find assets matching segmented objects in an input image using CLIP embeddings [S2].
*   Encode all renderings from the 3D-FUTURE database using CLIP into normalized 768-dimensional vectors [S2].
*   Crop a target image around each object's mask from the input image and encode it using CLIP [S2].
*   Select Nearest Neighbors in CLIP space using cosine distance to find the closest assets [S2].
*   Database: 3D-FUTURE dataset [Fu et al. 2021], containing 10,000+ artist-made models [S2].
*   Embedding dimension: 768 [S2].

## Background
The concept of assigning per-Gaussian instance IDs supervised by SAM (Segment Anything Model) or video tracking was introduced as a method for Gaussian Grouping [S1, unofficial]. The use of CLIP for zero-shot ranked classification of 3D geometries from databases like 3D-FUTURE was detailed in PSDR-Room, which encodes renderings into normalized 768-dimensional vectors to search for matching assets [S2].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] PSDR-Room: Single Photo to Scene using Differentiable Rendering, https://doi.org/10.1145/3610548.3618165, fetched 2026-11-08
