# Multi-Resolution and Level of Detail (Pyramid, LOD, Downsampling)

Also called: Hierarchical Splatting, Multi-Resolution Points, 3D Gaussian Splatting LOD, SVL Generation

## Overview
Multi-Resolution and Level of Detail (LOD) techniques optimize volume and point cloud rendering by decomposing data into hierarchical structures, allowing the renderer to select primitives of varying granularity based on spatial error or screen-space footprint. This family solves the performance bottleneck of rendering high-density data by replacing clusters of fine primitives with single, aggregated coarse primitives in homogeneous or distant regions, while preserving detail in complex or foreground areas. It distinguishes itself from sibling methods by its reliance on explicit error metrics or fixed downsampling rates to drive the selection of the appropriate resolution level, rather than relying solely on static distance thresholds or fixed grid subdivisions.

## Shared principles
The methods in this family share the mechanism of hierarchical decomposition and error-driven or distance-driven primitive selection, building on [Splatting](23-splatting.md).

1.  **Hierarchical Decomposition:** The input data (volume, point cloud, or Gaussian set) is organized into a tree structure (e.g., octree, pyramid, or sequential point tree) [S1, S2, S3, unofficial][S4][S6, unofficial]. Each node in the hierarchy represents a spatial region and stores aggregate properties (mean position, average color/density, total opacity, or merged covariance) derived from its children [S1, S2, S3, unofficial][S4].
2.  **Error Metric Computation:** For each internal node, a local error metric $E$ is computed to quantify the approximation cost of representing the node's children with the node's aggregate properties [S1, S2, S3, unofficial][S4]. Common metrics include the Root Mean Square (RMS) error of scalar values or the deviation of child positions/opacities from the parent's mean [S1, S3, unofficial].
3.  **Adaptive Traversal and Selection:** During rendering, the hierarchy is traversed from the root to the leaves. A node is selected as a leaf (and thus rendered as a single primitive) if its error $E$ is below a user-defined threshold $\epsilon$ or if its projected screen-space footprint is smaller than a pixel threshold [S1, S2, S3, unofficial][S4]. If the error exceeds the threshold, the algorithm recurses to the children [S1, S2, S3, unofficial][S4].
4.  **Primitive Aggregation:** When a node is selected, its children are merged into a single representative primitive. The primitive's attributes (position, size, color, opacity) are computed as weighted averages or sums of the children's attributes [S1, S2, S3, unofficial][S4]. For Gaussian splats, this involves merging covariances and spherical harmonics coefficients [S3, unofficial].
5.  **Multi-Resolution Rendering:** The final image is composed of primitives from various levels of the hierarchy, allowing for efficient culling of distant or homogeneous regions while maintaining fidelity in foreground or complex areas [S1, S2, S3, unofficial][S4][S6, unofficial].

## Subtopics

### [Hierarchical Splatting](1168-hierarchical-splatting.md)
Uses a fixed-resolution pyramid structure where each level is 1/8th the resolution of the previous, aggregating $2 \times 2 \times 2$ voxels. It is chosen when a regular, predictable hierarchy is preferred over adaptive spatial partitioning.

### [Multi-Resolution Points](4784-multi-resolution-points.md)
Groups points into clusters based on a local RMS error metric to replace many small points with one large point. It is chosen for dense point clouds where overdraw is the primary bottleneck.

### [3D Gaussian Splatting LOD methods](5439-3d-gaussian-splatting-lod-methods.md)
Adapts 3D Gaussian properties (covariance, opacity, SH) into hierarchies or compressed codebooks. It is chosen for large-scale 3DGS scenes to manage memory and rendering latency.

### [Splat List (SVL) Generation with Multi-Level Downsampling](5767-splat-list-svl-generation-with-multi-level-downsampling.md)
Generates a flat, contiguous buffer of splat indices by iteratively downsampling a voxel occupancy mask. It is chosen when a flat buffer structure is required for efficient GPU indexing, avoiding tree traversal overhead.

## Background
The concept of multi-resolution rendering for volumes originated with the need to interactively visualize large medical and scientific datasets. Early work by Laur and Hanrahan (1991) introduced hierarchical point clouds using error metrics [S1]. Subsequent developments extended these ideas to volumetric splatting using pyramidal structures [S2, S4] and more recently to 3D Gaussian Splatting to handle the memory constraints of neural radiance fields [S3]. The technique is widely used in scientific visualization, medical imaging, and real-time rendering of large-scale 3D scenes.

## Sources
[S1] Laur, D., & Hanrahan, P. (1991). Hierarchical point-based rendering.
[S2] Levoy, M., & Whitted, T. (1985). The use of points as a display primitive.
[S3] Kerbl, B., et al. (2023). 3D Gaussian Splatting for Real-Time Radiance Field Rendering.
[S4] Westin, S. H., et al. (1992). Multi-resolution volume rendering and shading for fast image formation.
