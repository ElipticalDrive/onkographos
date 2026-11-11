# Mapping Strategies

Also called: Pixel mapping strategies, Scene alignment strategies

## Overview
Mapping strategies define the correspondence mechanism used to align pixels or features between two distinct scene representations, typically for tasks such as 3D editing, style transfer, or multi-scene fusion. Unlike standard SLAM mapping which aligns a current frame to a static map, these strategies map between two complete or partial scene states [S1]. They are distinguished from sibling methods like [Semantic-aware mapping](12053-semantic-aware-mapping.md) by their focus on cross-scene attribute transfer rather than intra-scene semantic consistency, and from [3D Gaussian Splatting optimization](12052-3d-gaussian-splatting-optimization.md) by their reliance on feature-space or segmentation-based correspondence rather than direct geometric rasterization gradients [S1].

## How it works
**Inputs:** Two scene representations (Source Scene $S_{src}$ and Target Scene $S_{tgt}$), typically defined by their radiance fields or Gaussian parameters, and the corresponding input images or feature maps.
**Outputs:** A mapped scene representation where attributes (color, density, style) from $S_{src}$ are transferred to $S_{tgt}$ while preserving structural fidelity [S1].

The method selects one of three correspondence strategies to establish pixel-to-pixel or region-to-region mapping:

1.  **Identity Mapping:** Assumes a direct 1:1 correspondence between pixels in the source and target views based on camera projection.
    *   *Mechanism:* For a pixel $p$ in the target view, the attribute is taken directly from the corresponding pixel $p'$ in the source view.
    *   *Limitation:* Lacks spatial alignment accuracy if the scenes differ in geometry or camera pose [S1].

2.  **Semantic Correspondence-Based Mapping:** Uses deep learning features to establish robust correspondence despite geometric misalignment.
    *   *Feature Extraction:* Dense semantic features are extracted from both scenes using Vision Transformers (ViT) [Dosovitskiy 2020] and DINO [Amir et al. 2021] [S1].
    *   *Matching:* Correspondences are established by finding the nearest neighbors in the feature space between source and target pixels.
    *   *Formula:* For a target pixel $u$, the source pixel $v$ is selected such that:
        $$ v = \arg \min_{v'} || f_{src}(v') - f_{tgt}(u) ||_2 $$
        where $f$ denotes the DINO/ViT feature vector [S1].

3.  **Co-segmentation-Based Mapping:** Refines alignment by grouping pixels into semantically meaningful segments before mapping.
    *   *Segmentation:* Images are segmented into parts using DINO’s clustering capabilities [S1].
    *   *Mapping:* Correspondence is established at the segment level, ensuring that semantic boundaries are maintained during transfer. This yields finer correspondence than identity mapping and preserves semantic integrity better than raw feature matching [S1].

**Resampling (Contextual Step):**
To align geometry and color during the mapping process, a density-based resampling strategy is often employed along corresponding rays.
*   *Problem:* Initial uniform sampling fails due to misalignment between scenes [S1].
*   *Solution:* Density peaks are identified along each ray in both scenes. Sampling points are adjusted to align these key points, ensuring improved color-geometry consistency [S1].

**Parameters:**
*   Feature extractor: DINO or ViT [S1].
*   Distance metric: Euclidean distance in feature space [S1].

**Edge Cases:**
*   *Misalignment:* Identity mapping fails significantly when scenes have different geometries [S1].
*   *Ambiguity:* Semantic features may be ambiguous for textureless regions; co-segmentation helps mitigate this by enforcing regional consistency [S1].

## Subtopics

### [iMAP](11979-imap.md)
iMAP is the first NeRF-based dense online SLAM model that jointly optimizes camera pose and an implicit scene representation. It solves the problem of simultaneous localization and mapping by treating the scene as a continuous radiance field rather than a discrete mesh or point cloud. It sets itself apart from its siblings by being the foundational method that introduced the iterative two-step approach of tracking and mapping to NeRF-based SLAM, upon which NICE-SLAM and subsequent methods build.

## Operating characteristics
*   **Identity Mapping:** Fastest computation ($O(1)$ per pixel) but lowest accuracy in misaligned scenes [S1].
*   **Semantic Correspondence:** Higher computational cost due to feature extraction and matching ($O(N \cdot D)$ where $N$ is pixels and $D$ is feature dimension), but robust to geometric variations [S1].
*   **Co-segmentation:** Highest computational cost due to clustering, but provides the best semantic boundary preservation [S1].
*   **Strengths:** Enables view-consistent 3D editing and multi-scene fusion [S1].
*   **Limitations:** Identity mapping lacks spatial accuracy; feature-based methods depend on the quality of the pre-trained models (DINO/ViT) [S1].

## Background
Mapping strategies in the context of multi-scene radiance fields were introduced to address the challenge of transferring visual attributes between different scene representations while maintaining structural fidelity. The specific strategies of identity, semantic correspondence, and co-segmentation mapping were proposed in the context of Generalized TensoRF for efficient multi-scene radiance fields and view-consistent 3D editing [S1].

## Sources
[S1] Generalized TensoRF: Efficient Multi-Scene Radiance Fields and View-Consistent 3D Editing, https://doi.org/10.1145/3722564.3728385, fetched 2026-10-29
