# Octree-based Hierarchy

## Overview
Octree-based Hierarchy is a family of hierarchical decomposition methods for volume rendering that organizes data into an octree structure to enable adaptive, multi-resolution rendering [S1, unofficial]. It sets itself apart from sibling methods like [Multi-Resolution and Level of Detail (Pyramid, LOD, Downsampling)](12128-multi-resolution-and-level-of-detail-pyramid-lod-downsamplin.md) by using a 3D spatial subdivision (octree) rather than a 2D image-space pyramid, allowing for precise culling of homogeneous 3D regions based on stored error metrics [S1, unofficial].

## Shared principles
The methods in this family share the mechanism of hierarchical decomposition and error-driven or distance-driven primitive selection, building on [Splatting](23-splatting.md) [G].

1. **Hierarchical Decomposition:** The input data (volume, point cloud, or Gaussian set) is organized into an octree structure where each node represents a spatial region [S1, unofficial]. Each node stores aggregate properties derived from its children [G].
2. **Error Metric Computation:** For each internal node, a local error metric $E$ is computed to quantify the approximation cost of representing the node's children with the node's aggregate properties [G].
3. **Adaptive Traversal and Selection:** During rendering, the hierarchy is traversed from the root to the leaves [G]. A node is selected as a leaf (and thus rendered as a single primitive) if its error $E$ is below a user-defined threshold $\epsilon$ [G]. If the error exceeds the threshold, the algorithm recurses to the children [G].
4. **Primitive Aggregation:** When a node is selected, its children are merged into a single representative primitive [G]. The primitive's attributes are computed as weighted averages or sums of the children's attributes [G].
5. **Multi-Resolution Rendering:** The final image is composed of primitives from various levels of the hierarchy, allowing for efficient culling of distant or homogeneous regions while maintaining fidelity in foreground or complex areas [S1, unofficial].

## Subtopics

### [Octree-based Splatting](8067-octree-based-splatting.md)
An object-order interactive volume rendering technique that approximates volume contributions using an octree hierarchy and geometric splats. It sets itself apart from standard splatting by adapting the resolution of rendered primitives based on a user-defined error tolerance, trading visual fidelity for speed. Choose it when real-time interaction on dense volumes is required.

### [Splatting-Based AMR Volume Rendering](10507-splatting-based-amr-volume-rendering.md)
A hierarchical object-order rendering technique designed for Adaptive Mesh Refinement (AMR) data. It sets itself apart from sibling methods by explicitly handling the conversion of irregular AMR block structures into a k-d-tree framework for domain subdivision, augmented with octrees to manage isovalue ranges. Choose it for efficiently rendering non-uniform, multi-resolution scalar fields.

### [Octree Splatting](13649-octree-splatting.md)
A hierarchical object-order volume rendering technique that renders non-leaf octree nodes as single primitives. It sets itself apart from standard splatting by actively utilizing internal tree nodes as renderable kernels rather than restricting them to occlusion culling, thereby reducing draw calls and traversal overhead. Choose it for accelerating the visualization of dense volumetric data.

### [TopoGS](17246-topogs.md)
A topology-aware anchor feature aggregation framework designed to address feature isolation in octree-based 3D Gaussian Splatting. It sets itself apart from sibling methods by explicitly modeling the octree's topological structure during feature aggregation rather than relying solely on spatial proximity or error metrics for primitive selection. Choose it when optimizing cross-level information flow and reducing redundancy in 3DGS representations.

### Hierarchical Splat Culling
Splats are culled prior to sorting and rendering using a custom octree to optimize performance [S1, unofficial].

## Background
The technique is utilized in real-time viewers and web-based splat rendering to optimize performance by culling splats prior to sorting and rendering using a custom octree [S1, unofficial]. It supports progressive loading of `.ply` and `.splat` files, though converting scenes to the optimized `.ksplat` format results in the fastest loading times since the format matches the internal format for splat data [S1, unofficial]. The rendering pipeline also incorporates WASM splat sort implemented in C++ using WASM SIMD instructions, alongside partially GPU accelerated splat sort using transform feedback to pre-calculate splat distances [S1, unofficial].

## Sources
[S1] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
