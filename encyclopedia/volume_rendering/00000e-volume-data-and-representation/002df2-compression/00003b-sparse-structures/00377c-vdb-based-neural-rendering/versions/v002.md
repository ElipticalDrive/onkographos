# VDB-based Neural Rendering

Also called: Hyan et al. VDB approach, Learning VDB from images

## Overview
VDB-based Neural Rendering is a method that directly learns a VDB (hierarchical data structure for sparse volumes) from a set of posed images for real-time rendering [S1]. It sets itself apart from traditional NeRF by replacing the dense MLP with a sparse hierarchical structure inspired by classical computer graphics techniques such as Bounding Volume Hierarchy (BVH) and Sparse Voxel Octree (SVO) to model scenes in a sparse hierarchical form [S1]. This approach achieves a 10× to 20× speed-up over traditional NeRF, with rendering quality comparable to Neural Volumes and Scene Representation Networks [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of posed images (camera poses and corresponding 2D views).
*   **Output:** A learned VDB structure representing the scene, enabling real-time rendering.

**Data Structures:**
The core data structure is a VDB (Volume Data Base), which is a hierarchical sparse volume representation [S1, S1]. The specific internal layout (e.g., block size, tree depth, node types) is not detailed in the provided sources, but it is described as a "hierarchical data structure for sparse volumes" [S1].

**Method Steps:**
1.  **Initialization:** Initialize a sparse hierarchical structure inspired by BVH and SVO [S1].
2.  **Learning:** Learn the VDB structure directly from the posed images [S1]. This involves optimizing the structure to represent the scene geometry and appearance.
3.  **Training Process:** The training process is complex because the sparse data structure must be periodically updated during training as surfaces emerge progressively [S1].
4.  **Rendering:** Use the learned VDB for real-time rendering [S1].

**Formulas:**
Not given in the sources.

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Training Complexity:** The method leads to a more complex training process because the sparse data structure must be periodically updated during training [S1].
*   **Trade-off:** There is an unavoidable trade-off between memory usage and precision arising from the voxel structure and the density grid [S1].

## Subtopics

## Operating characteristics
*   **Performance:** Achieves a 10× to 20× speed-up over traditional NeRF [S1].
*   **Quality:** Rendering quality is comparable to Neural Volumes and Scene Representation Networks [S1].
*   **Memory:** Introduces an unavoidable trade-off between memory usage and precision [S1].
*   **Training:** Requires a complex training process with periodic updates to the sparse data structure [S1].

## Background
The method was introduced by Hyan et al. [YLMM23] and is inspired by classical computer graphics techniques such as Bounding Volume Hierarchy (BVH) [RW80] and Sparse Voxel Octree (SVO) [LK10] [S1]. It aims to accelerate ray tracing by modeling scenes in a sparse hierarchical form [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
