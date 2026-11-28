# VDB-based Neural Rendering

Also called: Hyan et al. VDB approach, Learning VDB from images

## Overview
VDB-based Neural Rendering is a method that directly learns a VDB (hierarchical data structure for sparse volumes) from a set of posed images for real-time rendering [S1]. It sets itself apart from traditional NeRF by replacing the dense grid with a sparse hierarchical structure inspired by classical computer graphics techniques such as Bounding Volume Hierarchy (BVH) and Sparse Voxel Octree (SVO) to model scenes in a sparse hierarchical form to accelerate ray tracing [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of posed images of the scene.
*   **Output:** A learned VDB structure containing density and color information, optimized for real-time rendering.

**Data Structures:**
The core data structure is a VDB (Volume Data Base), which is a hierarchical sparse volume representation [S1]. The specific internal layout (e.g., block size, tree depth, bit packing) is not given in the sources. The structure maintains a trade-off between memory usage and precision arising from the voxel structure and the density grid [S1].

**Steps:**
1.  **Initialization:** Initialize a sparse hierarchical structure inspired by BVH and SVO [S1].
2.  **Learning:** Optimize the VDB structure directly from the set of posed images [S1]. This involves updating the sparse data structure periodically during training because surfaces emerge only progressively [S1].
3.  **Rendering:** Use the learned VDB for real-time rendering [S1].

**Parameters:**
*   Typical speed-up: 10× to 20× over traditional NeRF [S1].
*   Rendering quality: Comparable to Neural Volumes and Scene Representation Networks [S1].

**Edge Cases and Pitfalls:**
*   The training process is more complex than standard NeRF because the sparse data structure must be periodically updated during training [S1].
*   There is an unavoidable trade-off between memory usage and precision [S1].

## Subtopics

### [Sparse Domain Decomposition with Expert Networks](8739-sparse-domain-decomposition-with-expert-networks.md)
A method to encode sparse volumetric data by dividing the domain into subdomains, each with dedicated gate functions and expert neural networks, allowing concurrent training and efficient storage.

## Operating characteristics
*   **Speed:** Achieves a 10× to 20× speed-up over traditional NeRF [S1].
*   **Quality:** Rendering quality is comparable to Neural Volumes [LSS∗ 19] and Scene Representation Networks [SZW19] [S1].
*   **Memory:** Introduces an unavoidable trade-off between memory usage and precision arising from the voxel structure and the density grid [S1].
*   **Training Complexity:** Leads to a more complex training process compared to dense grid methods [S1].

## Background
This method was introduced by Hyan et al. [YLMM23] and is inspired by classical computer graphics techniques such as the Bounding Volume Hierarchy (BVH) [RW80] and the Sparse Voxel Octree (SVO) [LK10] [S1]. The VDB structure itself is referenced as [KLM22] [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
