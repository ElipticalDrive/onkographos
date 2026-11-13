# Mapping Strategies

Also called: Pixel mapping strategies, Scene alignment strategies

## Overview
Mapping strategies define the correspondence mechanism used to align pixels or features between two distinct 3D scenes or radiance fields, typically for tasks such as multi-scene editing, fusion, or transfer [S1]. Unlike standard SLAM mapping which aligns a current frame to a static map, these strategies address the alignment of two complete scene representations [S1]. They are distinguished from sibling methods like [Semantic-aware mapping](12053-semantic-aware-mapping.md) by their explicit focus on cross-scene pixel-to-pixel or feature-to-feature correspondence rather than intra-scene consistency [S1].

## How it works
The method operates by establishing a mapping function $M$ that associates elements (pixels, features, or segments) from a source scene $S_{src}$ to a target scene $S_{tgt}$ [S1].

**Inputs:**
*   Source scene representation (e.g., radiance field, point cloud, or image set).
*   Target scene representation.
*   Camera poses for both scenes (if view-dependent).

**Outputs:**
*   A correspondence map linking source elements to target elements.
*   Aligned scene parameters for downstream tasks (e.g., resampling, editing) [S1].

**Data Structures:**
*   Feature vectors: Dense semantic features extracted via Vision Transformers (ViT) or DINO [S1].
*   Segmentation masks: Co-segmentation labels derived from clustering [S1].

**Steps:**
1.  **Feature Extraction:** Extract dense semantic features from the source and target scenes using a pre-trained model (e.g., DINO) [S1].
2.  **Correspondence Computation:** Compute similarity between source and target features to establish initial matches [S1].
3.  **Strategy Selection:** Apply one of the following three strategies based on required alignment accuracy and semantic fidelity [S1]:
    *   **Identity Mapping:** Assumes direct pixel correspondence based on camera projection or index. This is the baseline but lacks spatial alignment accuracy if scenes are not perfectly registered [S1].
    *   **Semantic Correspondence-Based Mapping:** Uses the dense semantic features from Step 1 to find nearest-neighbor matches in the feature space. This improves spatial alignment by matching semantically similar regions rather than just geometrically projected pixels [S1].
    *   **Co-segmentation-Based Mapping:** Refines the alignment by first segmenting both images into semantically meaningful parts using DINO’s clustering capabilities. Correspondence is then established at the segment level, ensuring that semantic boundaries are preserved and alignment is consistent within object categories [S1].
4.  **Resampling (Optional):** If geometry and color alignment are required, use density-based resampling along corresponding rays to align key points, adjusting sampling positions to match density peaks in both scenes [S1].

**Formulas:**
*   **Semantic Similarity:** For a pixel $p$ in source and $q$ in target, the match is determined by minimizing the distance between their feature vectors $f(p)$ and $f(q)$:
    $$ q^* = \arg \min_q \| f(p) - f(q) \|_2 $$
    where $f(\cdot)$ is the feature extractor (e.g., DINO) [S1].
*   **Co-segmentation:** Images are partitioned into $K$ segments $\{S_1, ..., S_K\}$ such that semantic homogeneity is maximized within segments [S1].

**Parameters:**
*   Feature extractor: DINO or ViT [S1].
*   Clustering algorithm: Used for co-segmentation (specific algorithm not given in sources) [S1].

**Edge Cases:**
*   **Misalignment:** Identity mapping fails significantly if scenes are not pre-aligned [S1].
*   **Semantic Ambiguity:** If objects lack distinct semantic features, semantic correspondence may fail; co-segmentation helps by enforcing structural boundaries [S1].

## Subtopics

### [iMAP](11979-imap.md)
iMAP is the first NeRF-based dense online SLAM model that jointly optimizes camera pose and an implicit scene representation. It solves the problem of simultaneous localization and mapping by treating the scene as a continuous radiance field rather than a discrete mesh or point cloud. It sets itself apart from its siblings by being the foundational method that introduced the iterative two-step approach of tracking and mapping to NeRF-based SLAM, upon which NICE-SLAM and subsequent methods build.

## Operating characteristics
*   **Identity Mapping:** Fastest computation but lowest accuracy for misaligned scenes [S1].
*   **Semantic Correspondence:** Higher accuracy than identity mapping; requires feature extraction overhead [S1].
*   **Co-segmentation:** Highest semantic fidelity and boundary preservation; highest computational cost due to clustering [S1].
*   **Scaling:** Performance depends on the resolution of the feature maps and the complexity of the clustering algorithm [S1].

## Background
These strategies were introduced in the context of multi-scene radiance fields and view-consistent 3D editing, where aligning geometry and color between different scenes is critical [S1]. The progression from identity to semantic to co-segmentation mapping reflects a move towards preserving structural fidelity and semantic consistency during scene manipulation [S1].

## Sources
[S1] Generalized TensoRF: Efficient Multi-Scene Radiance Fields and View-Consistent 3D Editing, https://doi.org/10.1145/3722564.3728385, fetched 2026-10-29
