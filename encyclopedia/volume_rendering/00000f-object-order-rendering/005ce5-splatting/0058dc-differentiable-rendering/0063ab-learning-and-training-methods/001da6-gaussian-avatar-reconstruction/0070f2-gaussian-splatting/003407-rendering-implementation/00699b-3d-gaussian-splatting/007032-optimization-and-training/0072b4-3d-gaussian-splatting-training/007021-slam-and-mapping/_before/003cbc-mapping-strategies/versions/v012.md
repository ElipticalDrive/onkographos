# Mapping Strategies

Also called: Pixel mapping strategies, Scene alignment strategies

## Overview
Mapping strategies define the correspondence mechanism between pixels or features of two distinct scenes (e.g., a source and a target scene) to enable alignment, fusion, or editing [S1]. They solve the problem of spatial misalignment that arises when direct pixel-to-pixel correspondence is invalid due to viewpoint changes, object deformation, or semantic differences [S1]. Unlike sibling methods such as [CG-SLAM](9196-cg-slam.md) which rely on geometric feature matching, or [NeRF Representation](12096-neural-radiance-field-nerf-representation.md) which optimizes continuous fields, these strategies explicitly define how visual information is transferred or matched between discrete scene representations [S1].

## How it works
The method consists of three primary strategies for establishing correspondence, followed by a resampling technique to enforce geometric consistency [S1].

**Inputs:** Two scenes (Source and Target), represented as images or radiance fields.
**Outputs:** A mapping function or set of correspondences between pixels/features of the two scenes.

### 1. Identity Mapping
*   **Definition:** Assumes a direct 1:1 pixel correspondence between the source and target images [S1].
*   **Mechanism:** Maps pixel $(u, v)$ in the source directly to $(u, v)$ in the target.
*   **Limitation:** Lacks spatial alignment accuracy; fails when scenes differ in viewpoint or scale [S1].

### 2. Semantic Correspondence-Based Mapping
*   **Definition:** Uses deep learning features to establish dense semantic matching [S1].
*   **Mechanism:**
    1.  Extract dense semantic features from both scenes using Vision Transformers (ViT) [Dosovitskiy 2020] and DINO [Amir et al. 2021].
    2.  Compute similarity metrics (e.g., cosine similarity) between feature vectors of pixels in the source and target.
    3.  Establish correspondences based on feature proximity [S1].
*   **Advantage:** Improves spatial matching over identity mapping by leveraging semantic context [S1].

### 3. Co-Segmentation-Based Mapping
*   **Definition:** Refines alignment by segmenting images into semantically meaningful parts [S1].
*   **Mechanism:**
    1.  Use DINO’s clustering capabilities to segment both scenes into coherent regions [S1].
    2.  Establish correspondences at the region level rather than pixel level.
    3.  Map pixels within corresponding semantic segments [S1].
*   **Advantage:** Yields finer correspondence and maintains semantic boundaries, handling intra-class variations better than dense feature matching [S1].

### Resampling for Geometry-Color Consistency
To align geometry and color after establishing correspondence, a density-based resampling strategy is applied along corresponding rays [S1].
1.  **Problem:** Initial uniform sampling fails due to misalignment between scenes [S1].
2.  **Method:** Identify density peaks along each ray in the source scene.
3.  **Adjustment:** Adjust sampling points in the target scene to align with these density peaks [S1].
4.  **Result:** Ensures improved color-geometry consistency and realistic fusion of visual attributes [S1].

**Parameters:**
*   Sampling density along rays: Not given in the sources.
*   Threshold for density peaks: Not given in the sources.

**Edge Cases:**
*   **Misalignment:** Identity mapping fails completely under viewpoint changes [S1].
*   **Semantic Ambiguity:** Co-segmentation may fail if DINO clustering produces inconsistent regions across scenes [S1].

## Subtopics

### [iMAP](11979-imap.md)
iMAP is the first NeRF-based dense online SLAM model that jointly optimizes camera pose and an implicit scene representation. It solves the problem of simultaneous localization and mapping by treating the scene as a continuous radiance field rather than a discrete mesh or point cloud. It sets itself apart from its siblings by being the foundational method that introduced the iterative two-step approach of tracking and mapping to NeRF-based SLAM, upon which NICE-SLAM and subsequent methods build.

### [RNR-Map Construction](16526-rnr-map-construction.md)
RNR-Map Construction embeds visual information from an RGB-D image into a 2D grid map (RNR-Map) using a pretrained encoder and inverse camera projection. It differs from sibling methods like [iMAP](11979-imap.md) by operating directly on latent codes to bypass the computationally expensive differentiable rendering step required for alignment, enabling high-frequency mapping.

### Joint Mapping and Pose Update
A mapping subroutine that fuses tracked RGB-D frames into the existing scene map by jointly optimizing scene geometry, camera poses, and feature embeddings using a sliding window of keyframes [S2].

**Steps:**
1.  Randomly select $N_{kf}$ keyframes for each frame to form an optimization window [S2].
2.  For each frame in the optimization window, randomly sample $N_m$ rays and transform them into the world coordinate system with estimated frame poses [S2].
3.  Sample points within sparse voxels, render pixels, and calculate loss functions using the method from Section 4.1 of [S2] [S2].

**Parameters:**
*   $N_{kf}$: Number of keyframes randomly selected for the optimization window [S2].
*   $N_m$: Number of rays randomly sampled for each frame in the optimization window [S2].

## Operating characteristics
*   **Identity Mapping:** $O(1)$ per pixel; extremely fast but inaccurate for non-aligned scenes [S1].
*   **Semantic Mapping:** High computational cost due to ViT/DINO inference; robust to viewpoint changes [S1].
*   **Co-Segmentation:** Moderate to high cost due to clustering; robust to semantic variations [S1].
*   **Resampling:** Adds overhead proportional to ray length and sampling density; improves fusion quality [S1].
*   **Joint Mapping:** Depends on window size $N_{kf}$ and ray count $N_m$; prone to network forgetting if not jointly optimized [S2].

## Background
Mapping strategies in the context of 3D editing and SLAM emerged with the need to transfer visual attributes between scenes or align incremental maps. Generalized TensoRF introduced these strategies for view-consistent 3D editing [S1]. Vox-Fusion adapted similar concepts for dense tracking and mapping using voxel-based neural implicit representations [S2].

## Sources
[S1] Generalized TensoRF: Efficient Multi-Scene Radiance Fields and View-Consistent 3D Editing, https://doi.org/10.1145/3722564.3728385, fetched 2026-10-29
[S2] Vox-Fusion: Dense Tracking and Mapping with Voxel-based Neural Implicit Representation, https://doi.org/10.1109/ismar55827.2022.00066, fetched 2026-10-26
