# Mapping Strategies

Also called: Pixel mapping strategies, Scene alignment strategies

## Overview
Mapping strategies define the correspondence mechanism used to align pixels or features between two distinct scenes or views for the purpose of alignment, editing, or fusion. In the context of radiance-field optimization, these strategies solve the problem of establishing reliable spatial matching when direct geometric priors are insufficient or when transferring attributes between scenes. This method sets itself apart from sibling optimization techniques by focusing specifically on the *correspondence* phase rather than the gradient-based parameter update, serving as a prerequisite for resampling and fusion.

## How it works
The method operates by establishing a mapping function $M: \mathcal{I}_A \rightarrow \mathcal{I}_B$ between a source image/scene $A$ and a target image/scene $B$. The core input is a pair of images or radiance field representations; the output is a set of correspondences or a warped representation of $A$ in the coordinate system of $B$.

Three primary strategies are employed, ranging from simple geometric assumptions to semantic feature matching:

1.  **Identity Mapping**: This strategy assumes a direct pixel-to-pixel correspondence based on camera pose or grid alignment. It maps pixel $(u, v)$ in source $A$ directly to $(u, v)$ in target $B$.
    *   *Limitation*: It lacks spatial alignment accuracy if the scenes are not perfectly registered or if there are viewpoint changes.
2.  **Semantic Correspondence-based Mapping**: This strategy improves alignment by extracting dense semantic features. It utilizes Vision Transformers (ViT) [Dosovitskiy 2020] and DINO [Amir et al. 2021] to generate feature embeddings for each pixel. The mapping is determined by finding the nearest neighbor in feature space between pixels of $A$ and $B$.
    *   *Benefit*: It provides robust spatial matching that is invariant to appearance changes, leveraging semantic consistency.
3.  **Co-segmentation-based Mapping**: This strategy refines alignment by segmenting images into semantically meaningful parts. It uses DINO’s clustering capabilities to identify coherent regions. Correspondences are established at the region level or by enforcing consistency within segmented boundaries.
    *   *Benefit*: It yields finer correspondence and maintains semantic boundaries, preventing the blending of distinct objects.

**Parameters**:
*   Feature extractor: DINO or ViT [S1].
*   Clustering method: DINO-based clustering [S1].

**Edge Cases**:
*   Identity mapping fails under significant misalignment or viewpoint shifts [S1].
*   Semantic methods may struggle in textureless regions where feature descriptors are ambiguous [S1].

## Subtopics

### [iMAP](11979-imap.md)
iMAP is the first NeRF-based dense online SLAM model that jointly optimizes camera pose and an implicit scene representation. It solves the problem of simultaneous localization and mapping by treating the scene as a continuous radiance field rather than a discrete mesh or point cloud. It sets itself apart from its siblings by being the foundational method that introduced the iterative two-step approach of tracking and mapping to NeRF-based SLAM, upon which NICE-SLAM and subsequent methods build.

## Operating characteristics
*   **Identity Mapping**: $O(1)$ per pixel; fastest but least accurate. Suitable only for perfectly aligned static scenes.
*   **Semantic Correspondence**: High computational cost due to ViT/DINO inference. Robust to appearance changes but sensitive to semantic ambiguity.
*   **Co-segmentation**: Highest computational cost due to clustering. Provides the highest alignment fidelity for semantically rich scenes.
*   **Strengths**: Enables view-consistent 3D editing and multi-scene fusion by ensuring structural fidelity [S1].
*   **Limitations**: Identity mapping lacks spatial accuracy; semantic methods require significant memory for feature storage [S1].

## Background
The concept of mapping strategies in radiance fields emerged with the need to align and edit multi-scene representations. The specific strategies of identity, semantic, and co-segmentation mapping were introduced in the context of Generalized TensoRF to enable efficient multi-scene radiance fields and view-consistent 3D editing [S1]. The approach builds on the observation that simple geometric alignment is insufficient for complex scene editing, necessitating semantic-aware correspondence.

## Sources
[S1] Generalized TensoRF: Efficient Multi-Scene Radiance Fields and View-Consistent 3D Editing, https://doi.org/10.1145/3722564.3728385, fetched 2026-10-29
