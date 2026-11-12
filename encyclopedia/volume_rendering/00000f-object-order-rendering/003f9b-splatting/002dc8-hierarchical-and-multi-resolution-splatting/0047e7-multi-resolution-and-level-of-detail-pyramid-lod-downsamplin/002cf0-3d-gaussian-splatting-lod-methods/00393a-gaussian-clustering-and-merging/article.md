# Gaussian clustering and merging

## Overview
This family of techniques reduces the rendering cost of 3D Gaussian Splatting by grouping Gaussians into clusters and replacing them with representative proxies or hierarchical structures for Level-of-Detail (LOD) selection [G]. It differs from sibling methods by focusing specifically on the merging primitive (how to compute the representative Gaussian) or the culling logic (how to select which merged clusters to render), rather than just storage optimization or hierarchical traversal alone [G].

## Shared principles
All methods in this family operate on a set of input Gaussians $\{G_i\}_{i=1}^N$, where each $G_i$ is defined by position $\mathbf{t}_i$, covariance matrix $\mathbf{\Sigma}_i$ (or rotation/scale), opacity $\alpha_i$, and color/amplitude coefficients $c_i$ [G].

The core mechanism involves two phases:
1. **Partitioning:** The $N$ Gaussians are grouped into clusters $C_j$ based on spatial proximity or attribute similarity [G].
2. **Merging/Reduction:** Each cluster $C_j$ is reduced to a single representative Gaussian $G_{merged}$ or a lower-resolution proxy [G]. The parameters of $G_{merged}$ ($\mathbf{t}_m, \mathbf{\Sigma}_m, \alpha_m, c_m$) are computed to approximate the visual contribution of the original cluster [G].

Common mathematical operations include:
* **Weighted Moments:** Computing the first moment (center of mass) and second moment (covariance) of the cluster to determine position and shape [G].
  $$ \mathbf{t}_{m} = \frac{1}{W} \sum_{i \in C} w_i \mathbf{t}_i $$
  $$ \mathbf{\Sigma}_{m} = \frac{1}{W} \sum_{i \in C} w_i (\mathbf{\Sigma}_i + \mathbf{t}_i \mathbf{t}_i^T) - \mathbf{t}_{m} \mathbf{t}_{m}^T $$
  where $W = \sum w_i$ and $w_i$ is typically opacity $\alpha_i$ or a volume-normalized weight [G].
* **Arithmetic Averaging:** Simple averaging of position, opacity, and color, often used in hierarchical LOD approaches [G].
* **L2 Optimization:** Minimizing the integrated squared error between the merged Gaussian and the sum of original Gaussians to optimize opacity and amplitude [G].

## Subtopics

### [K-wise moment-matched merging with L2-optimal amplitude](5440-k-wise-moment-matched-merging-with-l2-optimal-amplitude.md)
Reduces a cluster of $K$ Gaussians into a single representative splat by matching statistical moments and optimizing the resulting amplitude to minimize integrated squared error. It distinguishes itself from simple arithmetic averaging by explicitly solving for the optimal amplitude coefficient via L2 optimization, ensuring the merged splat best approximates the visual energy of the original group. Choose when visual fidelity during merging is prioritized over computational speed.

### [Hierarchical Gaussian Merging](8358-hierarchical-gaussian-merging.md)
Generates multiple levels of detail (LOD) for 3D Gaussian Splatting by iteratively reducing the number of Gaussians in each cluster. It solves the problem of rendering large Gaussian datasets efficiently by allowing the renderer to select a coarser representation based on distance or performance constraints. Unlike methods that optimize merged parameters via L2 error minimization or moment matching, this approach relies on simple arithmetic averaging of Gaussian properties to create simplified proxies. Choose for real-time LOD generation where computational efficiency is critical.

### [LOD Generation for 3DGS](8408-lod-generation-for-3dgs.md)
Reduces the number of Gaussians copied, stored, sorted, and rendered by merging Gaussians into clusters based on similar attributes, minimizing distant appearance impact without altering scene creation. It differs from virtual texturing approaches where mip levels are uniform-sized pages; instead, it reduces the Gaussian count with each LOD level, allowing multiple virtual pages to map to a single physical page. Choose when dynamic memory management and virtual texturing integration are required.

### [Cluster Culling (Nanite-style)](10402-cluster-culling-nanite-style.md)
A hierarchical culling and Level-of-Detail (LOD) selection mechanism for 3D Gaussian Splatting, designed to efficiently determine which clusters of Gaussians are visible and at what resolution they should be rendered. It solves the problem of rendering overhead in dense scenes by testing cluster bounding volumes against the view frustum and selecting the appropriate LOD based on screen-space error, outputting a compact list of visible clusters for subsequent rendering passes. It differs from sibling methods by focusing on the *selection* and *culling* phase rather than the geometric merging algorithm itself. Choose when frustum culling and screen-space error metrics are needed to optimize draw calls.

### Gaussian Cluster Builder
#### Definition
A builder class for constructing hierarchical cluster structures from Gaussian splat data, implementing Nanite-style spatial clustering [G].
#### Steps
- Sort splats by Morton code (Z-order curve) for spatial locality [G].
- Group into leaf clusters of N splats each [G].
- Recursively merge clusters into parent nodes [G].
- Calculate bounding volumes and error metrics [G].
#### Parameters
- SplatsPerCluster: Number of splats per leaf cluster [G].
- MaxChildrenPerCluster: Maximum children per parent cluster [G].
- bReorderSplats: Whether to reorder splat data for cluster locality [G].
- bGenerateLOD: Whether to generate LOD splats for parent clusters [G].
- LODReductionRatio: How many original splats per LOD splat (default: divide by 4) [G].
#### Layout
- FBuildSettings: int32 SplatsPerCluster, int32 MaxChildrenPerCluster, bool bReorderSplats, bool bGenerateLOD, int32 LODReductionRatio [G].

### Cluster Hierarchy Construction
#### Steps
- Calculate global bounding box of all splats [G].
- Build parent level by merging child clusters, specifying MaxChildrenPerCluster, ParentLODLevel, and NextClusterID [G].
#### Parameters
- SplatsPerCluster: Number of splats per leaf cluster. Default value: 128 [G].
- MaxChildrenPerCluster: Maximum number of children per parent cluster. Default value: 8 [G].
- bReorderSplats: Boolean flag to reorder splat data for cluster locality. Default value: true [G].
- bGenerateLOD: Boolean flag to generate LOD splats for parent clusters. Default value: true [G].
- LODReductionRatio: Ratio of original splats per LOD splat. Default value: 4 [G].
#### Layout
- FSplatSortEntry: Structure for sorting splats by Morton code. Fields: uint64 MortonCode, int32 OriginalIndex. Comparison operator: returns true if MortonCode < Other.MortonCode [G].

## Background
Not given in the sources.

## Sources
