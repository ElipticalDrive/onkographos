# Hierarchical Data Structures


## Overview
Hierarchical data structures accelerate volume rendering by organizing voxel data into a spatial subdivision hierarchy where internal nodes store aggregate information derived from their children or leaf voxels [Parent]. This structure enables the renderer to prune empty or inactive regions during ray traversal, significantly reducing the number of samples required to generate an image. Unlike flat grid traversal methods, hierarchical approaches adaptively skip large volumes of empty space, making them particularly effective for sparse or multi-resolution datasets [Parent].

## Shared principles
All methods in this family rely on a spatial subdivision hierarchy where internal nodes store aggregate information derived from their children or leaf voxels. The core mechanism is:
1.  **Aggregation:** Leaf nodes encode voxel properties. Internal nodes aggregate these properties (e.g., binary occupancy, min/max scalar ranges) from their children.
2.  **Pruning:** During ray traversal, if a node indicates an empty or inactive region (e.g., occupancy flag is 0, or min/max bounds are outside the transfer function's opacity range), the entire subtree is skipped.
3.  **Construction:** The hierarchy is typically constructed bottom-up. The depth is determined by data dimensions, often requiring padding to powers of two or specific size constraints to maintain regular subdivision [2560-hierarchical-grid-traversal].

## Subtopics

### [Hierarchical Spatial Enumeration Optimization](1663-hierarchical-spatial-enumeration-optimization.md)
Hierarchical Spatial Enumeration Optimization is a ray-tracing acceleration technique for volume rendering that encodes spatial coherence (empty regions) using a pyramid of binary volumes, also known as a complete octree representation. It solves the problem of efficiently skipping large empty regions in volumetric datasets during image-order rendering. Unlike geometric ray tracing which uses bounding volumes or spatial subdivision with lists of primitives, this method uses a hierarchical binary volume structure, which is more efficient for volume data because volume datasets have tens of millions of spatially ordered cells with single samples, whereas geometric scenes have hundreds of cells with many primitives. It differs from [Jiang et al. Acceleration Structure](1673-jiang-et-al-acceleration-structure.md) by using a pyramid of binary volumes/complete octree rather than a condensed representation.

### [Jiang et al. Acceleration Structure](1673-jiang-et-al-acceleration-structure.md)
The Jiang et al. acceleration structure [JRZ*16] is a hierarchical data structure designed to efficiently locate active cells for both direct and indirect volume visualization approaches. It solves the problem of redundant ray marching through inactive (empty or background) regions by pruning the search space. Unlike its sibling [Hierarchical Spatial Enumeration Optimization](1663-hierarchical-spatial-enumeration-optimization.md) which focuses on spatial enumeration, this method specifically targets hybrid object-order and image-order rendering, making it dependent on the rendering resolution.

### [Silhouette Image Space Acceleration Structure (Mip Pyramid)](3731-silhouette-image-space-acceleration-structure-mip-pyramid.md)
This method accelerates voxelization queries by providing conservative bounds on silhouette coverage using a mip pyramid over silhouette images. It solves the problem of efficiently determining if a voxel intersects with the visual hull surface without sampling every point within the voxel volume. It is distinguished from sibling methods by operating in image space rather than voxel space, leveraging the hierarchical z-buffer concept to prune voxels based on projected 2D bounds.

### [Flex-block tree](9716-flex-block-tree.md)
The flex-block tree is a hierarchical spatial subdivision structure derived from a kd-tree, designed primarily for load balancing in distributed volume rendering (both ray casting and ray tracing). Unlike traditional hierarchical structures that enforce strict axis-alternating splits, the flex-block partition allows flexible cut orientations to accommodate non-empty subvolume boundaries and render-node constraints. It serves as a dependency graph that determines the deterministic depth order of cells for image compositing in ray casting and accelerates ray traversal by indicating potential neighbor cells in ray tracing.

### Spatial Partitioning Tree for Unstructured Data
- [S1, unofficial] A preprocessing step that performs a parallel, synchronized partitioning of the volume data to produce a hierarchical spatial tree. This tree is used during rendering to optimize the compositing process, reduce runtime memory consumption, and speed up cell searching for zoom-in viewing.
- [S1, unofficial] Perform parallel, synchronized partitioning of the volume data.
- [S1, unofficial] Construct a hierarchical representation (spatial tree) of the data space.
- [S1, unofficial] Use the spatial tree during rendering to guide the compositing process.
- [S1, unofficial] Use the spatial tree to reduce runtime memory consumption.
- [S1, unofficial] Use the spatial tree to speed up cell searching for zoom-in viewing operations.

### HiGS
- [S2, unofficial] A hierarchical rendering architecture that achieves real-time performance through adaptive level selection and efficient GPU execution.

### Cell-tree Peeling
- [S3, unofficial] An efficient cell-processing schedule for the next frame determined using a ray dependency graph (the cell-tree). When a render-node becomes available, the cell with the longest queue is assigned to it.
- [S3, unofficial] The cell-tree represents ray dependencies between cells. Each node in the cell-tree has an associated cell ID, and more than one node may have the same cell ID.
- [S3, unofficial] The algorithm exploits frame-to-frame coherence by using the cell tree to find potential cache savings links. A schedule with any combination of non-conflicting link groups is a feasible one, and the optimal schedule for a single render-node includes a maximal group of cache savings links.
- [S3, unofficial] As long as a cell of data is in local memory, all rays on the corresponding queue are processed before it is replaced with another cell.
- [S3, unofficial] The tree is processed starting with the leaves by peeling the tree nodes with the same cell ID and adding it to the reverse schedule.
- [S3, unofficial] An interim tree is maintained by keeping a list of nodes that have not been included in the schedule.
- [S3, unofficial] Cache savings links are found by gathering leaf nodes which correspond to the same cell.
- [S3, unofficial] The cell-tree peeling algorithm performs well and has polynomial worst time.

## Background
The concept of hierarchical spatial subdivision for volume rendering originated in the mid-1990s as a means to handle the computational cost of ray marching through large, sparse datasets. Early work focused on octrees and kd-trees to skip empty space [Parent]. The specific formulation of "Hierarchical Spatial Enumeration" using binary volume pyramids was detailed in the early 2000s, addressing the inefficiencies of traditional geometric acceleration structures when applied to dense voxel grids [S1, unofficial]. The "Flex-block tree" and associated distributed rendering techniques were developed later to address load balancing in parallel volume rendering systems, particularly for massive datasets where static partitioning leads to severe load imbalances [S3, unofficial]. More recently, hierarchical structures have been adapted for novel rendering paradigms such as 3D Gaussian Splatting, where hierarchical architectures like HiGS enable real-time performance through adaptive level selection [S2, unofficial].

## Sources
[S1] DTIC ADA329328: A Scalable Parallel Cell-Projection Volume Rendering Algorithm for Three-Dimensional Unstructured Data, https://archive.org/details/DTIC_ADA329328, fetched 2026-10-26 (unofficial)
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S3] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
