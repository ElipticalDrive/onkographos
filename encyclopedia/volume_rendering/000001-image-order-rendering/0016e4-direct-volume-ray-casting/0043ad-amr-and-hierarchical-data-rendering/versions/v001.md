# AMR and Hierarchical Data Rendering

Also called: Adaptive Mesh Refinement Volume Rendering, Hierarchical Volume Rendering

## Overview
This family of methods addresses the inefficiency of rendering non-uniform, multi-resolution datasets (such as Adaptive Mesh Refinement or AMR data) where uniform sampling wastes computation on coarse regions or loses detail in fine regions. It sets itself apart from standard direct volume ray casting by dynamically adapting the sampling resolution and traversal strategy to the local data density and hierarchical structure, rather than treating the volume as a static grid.

## Shared principles
The rendering pipeline retains the core loop of [Direct volume ray casting](19-direct-volume-ray-casting.md): ray generation, intersection, sampling, classification, and compositing. However, the **Sampling and Interpolation** stage is modified to support hierarchical data structures. Instead of accessing a single uniform grid, the algorithm must:
1.  Identify the active resolution level or cluster for a given spatial position $\mathbf{p}$.
2.  Retrieve or reconstruct the scalar value $V(\mathbf{p})$ from the appropriate data structure (e.g., wavelet coefficients, cluster nodes, or AMR patches).
3.  Adjust the sampling step size $\Delta t$ or the interpolation kernel based on the local resolution to maintain visual fidelity while minimizing sample count.

The **Compositing** stage remains physically based, using the Beer-Lambert law and front-to-back accumulation, but the opacity $\alpha_i$ and color $C_i$ are derived from the hierarchical representation.

## Subtopics

### [Multiresolution Volume Rendering](3541-multiresolution-volume-rendering.md)
A method that utilizes hierarchical data structures, such as wavelet or Laplacian pyramid decompositions, to perform adaptive integration and early termination along rays. It solves the computational cost problem by allowing the use of lower-resolution data for parts of the ray where accuracy requirements are lower or occlusion is high, thereby reducing the number of sampling steps required. It sets itself apart from standard direct volume ray casting by dynamically selecting resolution levels based on the ray's progress and occlusion state, and from sibling methods like Cluster Hierarchy-based Volume Rendering by relying on mathematical decomposition (wavelets/pyramids) rather than spatial clustering of voxels.

### [Cluster Hierarchy-based Volume Rendering](10972-cluster-hierarchy-based-volume-rendering.md)
This method renders multi-variate volume data by mapping hierarchical clusters of feature vectors to transfer function parameters (color, opacity, size) for direct volume rendering. It solves the problem of interactive exploration in high-dimensional spaces by decoupling the classification logic (handled via cluster tree and parallel coordinates widgets) from the rendering engine, allowing users to filter and modify data subsets in real-time. It differs from pre-classification approaches by performing classification post-sampling on the GPU to maintain interactive frame rates, and from static multiresolution methods by supporting dynamic, user-driven cluster modification.

### Texture-Based AMR Volume Rendering
A texture/slicing-based volume rendering technique for AMR data that resamples to node-centered data, subdivides into homogeneous resolution regions using a kD-tree, and optimizes texture packing [S1, unofficial]. The process involves resampling AMR data to a node-centered format, subdividing the data into homogeneous resolution regions using a kD-tree (minimizing the number of blocks using AMR grid placement algorithm information), and performing texture/slicing-based volume rendering with optimized texture packing [S1, unofficial]. The method adapts slice spacing and corrects opacity to handle varying resolution levels [S1, unofficial]. It was used in Virtual Director and CAVE for rendering 'The First Star of the Universe' astrophysical data set [S1, unofficial].

## Background
The visualization of Adaptive Mesh Refinement (AMR) data has been a significant challenge due to the irregular and multi-resolution nature of the grids. Early approaches focused on direct volume rendering of AMR data using techniques like homogenization and distributed rendering [S1, unofficial]. The "Current State of the Art in Adaptive Mesh Refinement Visualization" survey highlights several key developments:
*   **Texture-based AMR Volume Rendering:** Proposed by Kähler & Hege (2001/2002), this approach resamples data to node-centered format and uses kD-trees for subdivision [S1, unofficial].
*   **Splatting-based Volume Rendering:** Park et al. (2002) introduced kD-tree- and Octree-based domain subdivision for hierarchical splatting [S1, unofficial].
*   **Direct Volume Rendering of AMR Data:** Kreylos et al. (2002) utilized homogenization via kD-trees and distributed rendering using texture-based slicing with cost-range decomposition [S1, unofficial].
*   **Applications:** These techniques were applied to large-scale astrophysical datasets, such as 'The First Star of the Universe', which was aired on the Discovery Channel [S1, unofficial].

## Sources
[S1] Visualization Tools for Adaptive Mesh Refinement Data, https://www.vacet.org/publications/AMRVisJune2007.pdf, fetched 2026-10-27 (unofficial)
