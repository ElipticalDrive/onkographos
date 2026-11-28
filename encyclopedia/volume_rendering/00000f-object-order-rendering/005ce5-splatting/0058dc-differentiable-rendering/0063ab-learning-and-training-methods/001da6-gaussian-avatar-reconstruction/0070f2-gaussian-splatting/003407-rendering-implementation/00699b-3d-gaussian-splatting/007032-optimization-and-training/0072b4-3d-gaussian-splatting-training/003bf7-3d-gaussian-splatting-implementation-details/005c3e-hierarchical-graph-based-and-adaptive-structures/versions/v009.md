# Hierarchical, Graph-Based, and Adaptive Structures

Also called: Adaptive Spatial Partitioning, Gaussian Mixture Trees, Elastic Inference, Scale-Adaptive Splatting, Surface-Aware Connectivity

## Overview
This family of methods extends standard 3D Gaussian Splatting (3DGS) by introducing structural organization—such as graphs, trees, grids, or hierarchical levels—to address limitations in scalability, memory efficiency, geometric consistency, and view-dependent aliasing. While standard 3DGS treats primitives as an unstructured flat list, these methods enforce spatial or semantic relationships to enable large-scale rendering, dynamic budget adaptation, and robust deformation. They differ from flat 3DGS by explicitly modeling connectivity or hierarchy, and from sibling methods like [Pruning and Density Control](16215-pruning-and-density-control.md) by using learned or structural constraints rather than simple heuristic removal.

## Shared principles
The core principle across this family is the replacement of the flat Gaussian list with a structured representation that imposes constraints on spatial proximity, feature similarity, or hierarchical containment.

1.  **Structural Connectivity:** Methods define adjacency not just by Euclidean distance, but by graph edges, tree parent-child relationships, or grid neighborhood indices. This connectivity is used for aggregation (e.g., context vectors in [Hierarchical Anchor Coupling](17260-hierarchical-anchor-coupling.md)), deformation (e.g., Laplacian operators in [Laplacian-Based Deformation](#laplacian-based-deformation)), or rendering selection (e.g., region blending in [TraGraph-GS](14454-tragraph-gs.md)).
2.  **Adaptive Selection/Transformation:** Instead of static primitives, these structures allow for dynamic adjustment. This includes selecting subsets of Gaussians based on a target ratio ([FlexGS](16110-flexgs.md)), merging nodes based on probabilistic similarity ([Gaussian Mixture Tree Construction](16717-gaussian-mixture-tree-construction.md)), or deforming primitives to match a specific sampling rate ([Mipmap-GS](16893-mipmap-gs.md)).
3.  **Surface Awareness:** Several methods explicitly align the structure with the underlying surface geometry rather than the 3D volume. This involves constructing graphs on surface manifolds ([Surface-Aware Splat Graph Construction](22007-surface-aware-splat-graph-construction.md)) or using geodesic distances for deformation ([Laplacian-Based Deformation](#laplacian-based-deformation)).

## Subtopics

### [TraGraph-GS](14454-tragraph-gs.md)
A rendering method for arbitrary large-scale scenes that uses graph-based adaptive spatial partitioning. It differs from flat 3DGS by partitioning the scene into connected subgraphs based on camera feature matching, training local representations independently, and using a progressive blending strategy for novel viewpoints to avoid boundary artifacts. It is chosen when rendering vast scales while maintaining real-time performance.

### [FlexGS](16110-flexgs.md)
An elastic inference method that enables a single trained model to adapt to varying computational constraints at inference time by selecting and transforming a subset of Gaussians based on a desired model size ratio, without requiring additional fine-tuning. It solves the problem of high GPU memory demand by jointly training a learnable selector and transformation field, unlike post-hoc compression methods that use fixed heuristics.

### [Gaussian Mixture Tree Construction](16717-gaussian-mixture-tree-construction.md)
A hierarchical clustering method that organizes 3D Gaussian primitives into a tree structure to identify distinct 3D plane instances. It solves spatial redundancy by grouping primitives into local clusters and recursively merging them based on probabilistic similarity (Bhattacharya distance) and plane descriptor similarity. Unlike sibling methods relying on fixed grids, it uses a bottom-up agglomerative approach.

### [Mipmap-GS](16893-mipmap-gs.md)
A test-time adaptation method that deforms pre-trained 3D Gaussians into scale-adaptive primitives to eliminate aliasing artifacts (zoom-in spikes and zoom-out thickness). Unlike sibling methods that apply heuristic smoothing filters, Mipmap-GS self-adjusts primitive properties using scale-specific mipmap-like pseudo ground-truth, effectively informing the Gaussians of the new sampling rate.

### [Hierarchical Anchor Coupling](17260-hierarchical-anchor-coupling.md)
A module that establishes cross-level gradient pathways in structured 3D Gaussian splatting by aggregating context from adjacent octree levels and broadcasting a residual feature adjustment to all visible anchors at a specific level. It solves feature isolation in anchor-based pipelines by operating at a per-level granularity, acting as an implicit regularizer against over-parameterization.

### [Self-Organizing Gaussian Grids](19022-self-organizing-gaussian-grids.md)
A compact 3D scene representation that organizes high-dimensional Gaussian parameters into a 2D grid with local homogeneity to reduce storage size. It exploits perceptual redundancies via unsupervised competitive learning (Self-Organizing Maps), allowing neighbors in all dimensions to be modeled effectively, unlike fixed-ordering methods such as Z-order curves.

### [Surface-Aware Splat Graph Construction](22007-surface-aware-splat-graph-construction.md)
Establishes connectivity exclusively along the surface manifold of a reconstructed scene, filtering out spurious neighbors that are close in Euclidean space but distant along the underlying geometry. It defines adjacency as the intersection of Gaussian kernel occupancy regions, specifically designed for surface-aligned representations like SpikingGS to enable proxy-free deformation.

### KoalaGS
*   **Definition:** A method that integrates surface-aware splat graphs into off-the-shelf point cloud Laplacian methods to enable proxy-free editing. It treats the collection of Gaussian kernel means as a point cloud and builds a Laplacian operator directly on them.
*   **Step:** Construct the Laplacian operator $\Delta$ on the set of Gaussian kernel means $\{\pi_i\}_{i=1}^N$. Integrate the surface-aware splat graph into the Laplacian computation. Use graph geodesic distance rather than raw graph connectivity for robustness against missing edges. Compute the cotangent Laplacian on a local triangulation using NonManifold [42]. Deform the means of kernels $\{\pi_i\}_{i=1}^N$ through a Laplacian-based deformation map $\phi_\Delta$ obtained by solving a system such as ARAP [1] or BBW [2] using the Laplacian operator $\Delta$.
*   **Pitfall:** Simply displacing the positions of the primitives leads to severe visual artifacts because the covariances of the Gaussian kernels are not adapted properly.

### Hierarchical 3D Gaussian BVH Construction
*   **Definition:** A top-down construction of an Axis-Aligned Bounding Box (AABB) Bounding Volume Hierarchy (BVH) over 3D Gaussian primitives to ensure spatial compactness of children nodes.
*   **Step:** Initialize the root node of the BVH as the AABB of all Gaussians, using 3x their stored size to capture extent. Recursively perform a binary median split on the current node to obtain child nodes. Project the means of all Gaussians in the group onto the longest axis of their bounding box. Partition the group by assigning each primitive based on the position of its projected mean with respect to the median of all projections. Compute intermediate node Gaussians from their respective children, starting from the leaves and recursively merging up the tree.

## Background
The need for hierarchical and graph-based structures in 3D Gaussian Splatting arises from the limitations of the original flat representation, particularly regarding scalability to large scenes, memory efficiency, and geometric consistency. Standard 3DGS treats primitives as an unstructured list, which can lead to redundancy, aliasing at different zoom levels, and difficulties in deformation.

The concept of using graphs for connectivity in Gaussian representations is explored in works like [Proxy-Free Gaussian Splats Deformation with Splat-Based Surface Estimation](https://doi.org/10.1109/3DV69130.2026.00155) [S1], which introduces surface-aware splat graphs to enable robust Laplacian-based deformation. This work highlights that standard kNN-based connectivity fails in non-manifold structures, necessitating surface-aware intersection checks.

Hierarchical structures, specifically BVHs, are detailed in [A Hierarchical 3D Gaussian Representation for Real-Time Rendering of Very Large Datasets](https://inria.hal.science/hal-04568786v1) [S2]. This approach addresses the rendering performance of very large datasets by organizing Gaussians into a tree structure, allowing for efficient culling and level-of-detail management. The construction involves recursive median splits based on projected means, ensuring spatial compactness.

These methods collectively represent a shift from "unstructured splatting" to "structured splatting," where the spatial or semantic relationships between primitives are explicitly modeled to improve performance, quality, and editability.

## Sources
[S1] Proxy-Free Gaussian Splats Deformation with Splat-Based Surface Estimation, https://doi.org/10.1109/3DV69130.2026.00155, fetched 2026-10-30
[S2] A Hierarchical 3D Gaussian Representation for Real-Time Rendering of Very Large Datasets, https://inria.hal.science/hal-04568786v1, fetched 2026-10-27
