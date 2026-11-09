# Mapping Strategies

Also called: Pixel mapping strategies, Scene alignment strategies

## Overview
Mapping strategies define the correspondence between pixels or features of two distinct scenes (e.g., a source and a target scene) to enable operations such as 3D editing, style transfer, or multi-scene fusion [S1]. Unlike sibling methods that focus on SLAM tracking or densification, these strategies address the problem of spatial alignment when transferring attributes (density, radiance, or geometry) between scenes that may differ in viewpoint, content, or structure [S1]. They are chosen when the goal is to map visual or geometric properties from one radiance field to another, rather than estimating camera poses or building a map from scratch.

## How it works
The method operates on pairs of images or feature maps representing two scenes. The core task is to establish a pixel-wise or region-wise mapping $\mathcal{M}: \text{Scene}_A \rightarrow \text{Scene}_B$.

**Inputs:**
*   Two sets of images or feature representations: $I_A$ (source) and $I_B$ (target).
*   Camera poses for $I_A$ and $I_B$ (optional, depending on strategy).
*   Pre-trained feature extractor (e.g., DINO, ViT).

**Outputs:**
*   A correspondence map or transformation field aligning $I_A$ to $I_B$.
*   Mapped attributes (e.g., density, color) for the target scene.

**Data Structures:**
*   Feature tensors: $F \in \mathbb{R}^{H \times W \times C}$, where $H, W$ are spatial dimensions and $C$ is the feature dimension (e.g., 768 for DINO).
*   Correspondence map: $M \in \mathbb{R}^{H \times W}$, storing indices or displacement vectors.

**Strategies:**

1.  **Identity Mapping:**
    *   Assumes direct pixel correspondence based on image coordinates.
    *   Formula: $M(u, v) = (u, v)$ for all $(u, v) \in \text{Image}_A$.
    *   Pitfall: Lacks spatial alignment accuracy if scenes differ in viewpoint or scale [S1].

2.  **Semantic Correspondence-based Mapping:**
    *   Uses Vision Transformers (ViT) and DINO to extract dense semantic features.
    *   Steps:
        1.  Extract features $F_A = \text{DINO}(I_A)$ and $F_B = \text{DINO}(I_B)$.
        2.  Compute similarity matrix $S \in \mathbb{R}^{H_A W_A \times H_B W_B}$ using cosine similarity:
            $$ S_{ij} = \frac{F_A[i] \cdot F_B[j]}{\|F_A[i]\| \|F_B[j]\|} $$
        3.  Find nearest neighbors in $S$ to establish correspondence.
    *   Advantage: Improves spatial matching by leveraging semantic content [S1].

3.  **Co-segmentation-based Mapping:**
    *   Refines alignment by segmenting images into semantically meaningful parts using DINO’s clustering.
    *   Steps:
        1.  Cluster features $F_A$ and $F_B$ using DINO’s self-attention or k-means to obtain segment masks $Seg_A$ and $Seg_B$.
        2.  Match segments based on semantic similarity.
        3.  Map pixels within corresponding segments.
    *   Advantage: Yields finer correspondence and maintains semantic boundaries [S1].

**Resampling (for geometry/color alignment):**
*   After mapping, density-based resampling aligns geometry along rays.
*   Identify density peaks along corresponding rays in both scenes.
*   Adjust sampling points to align these peaks, ensuring color-geometry consistency [S1].

**Parameters:**
*   Feature dimension $C$: Not given in the sources.
*   Clustering method: DINO clustering [S1].

**Edge Cases:**
*   Misalignment: Initial uniform sampling fails if scenes are not aligned; density-based resampling is required [S1].
*   Semantic ambiguity: Objects with similar appearance but different semantics may cause incorrect matches.

## Operating characteristics
*   **Time Complexity:** Depends on feature extraction (ViT/DINO) and similarity computation. Not given in the sources.
*   **Memory Complexity:** Requires storing feature tensors for both scenes. Not given in the sources.
*   **Strengths:** Semantic correspondence and co-segmentation provide robust alignment for complex scenes [S1].
*   **Limitations:** Identity mapping is inaccurate for non-aligned scenes [S1]. Computationally expensive due to feature extraction [S1].
*   **When to use:** When transferring attributes between scenes with semantic similarities but different viewpoints or structures [S1].
*   **When not to use:** When scenes are completely unrelated or lack semantic overlap.

## Background
Originated in the context of multi-scene radiance fields and 3D editing, specifically in the Generalized TensoRF framework [S1]. The strategies were developed to enable view-consistent 3D editing by aligning density and radiance components across scenes [S1].

## Sources
[S1] Generalized TensoRF: Efficient Multi-Scene Radiance Fields and View-Consistent 3D Editing, https://doi.org/10.1145/3722564.3728385, fetched 2026-10-29
