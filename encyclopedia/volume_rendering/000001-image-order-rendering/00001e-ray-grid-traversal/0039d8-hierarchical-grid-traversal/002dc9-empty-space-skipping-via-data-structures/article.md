# Empty Space Skipping via Data Structures

Also called: Acceleration Structure-Based Skipping, Active Region Culling

## Overview
Empty Space Skipping via Data Structures is a family of volume rendering optimization techniques that reduce computational cost by identifying and bypassing inactive (transparent or irrelevant) regions of a volume before or during ray sampling [S1][S2][S3]. It solves the problem of wasted samples in large, sparse, or complex volumes where significant portions of the ray path contribute nothing to the final image. What sets this family apart from sibling methods like [Space Leaping](5863-space-leaping.md) (which relies on ray coherence and image-plane buffers) or [Early Shader Termination](969-early-shader-termination.md) (which stops rays dynamically during compositing) is the use of explicit, precomputed or dynamically maintained data structures (e.g., min-max blocks, occupancy grids, BVHs, or hash maps) to define spatial bounds of active data [S1][S2][S3]. Unlike [Rasterizing Bounding Geometry for Empty Space Skipping](5141-rasterizing-bounding-geometry-for-empty-space-skipping.md), which uses polygonal approximations, these methods typically rely on scalar block statistics or hierarchical tree structures [S1].

## Shared principles
All methods in this family share the core mechanism of using a secondary data structure to represent the spatial distribution of active volume data, allowing rays to skip over inactive regions without sampling the primary volume texture [G].

1.  **Acceleration Structure:** A secondary data structure (e.g., occupancy grid, min-max block map, BVH, or hash table) is constructed to represent the volume. Each element in this structure corresponds to a block or region of volume voxels and stores metadata indicating occupancy or activity [G].
2.  **Active Region Definition:** A block or region is defined as "active" if it contributes to the final image. This is typically determined by density, opacity, or scalar value thresholds (e.g., $\max(\text{block}) > \tau$) [G].
3.  **Ray Query and Skip:** During rendering, the algorithm queries the acceleration structure to find entry and exit points of active regions. Rays advance past inactive blocks without sampling, and perform standard integration only within active segments [G].
4.  **Coordinate Mapping:** Efficient mapping between ray positions and grid indices is required, typically via division by block size: $u_{\text{occ}} = \lfloor u_{\text{vol}} / B \rfloor$ [G].

## Subtopics

### [Object-Order Empty Space Skipping](865-object-order-empty-space-skipping.md)
Object-Order Empty Space Skipping is a technique that reduces the number of volume samples by culling transparent or irrelevant regions before raycasting begins. It differs from its siblings by performing culling during the rasterization setup phase using precomputed block statistics, rather than during ray traversal or via bounding geometry rasterization. Unlike [Early Shader Termination](969-early-shader-termination.md), which stops rays dynamically during compositing, this method prevents rays from entering empty blocks entirely. Unlike [Rasterizing Bounding Geometry for Empty Space Skipping](5141-rasterizing-bounding-geometry-for-empty-space-skipping.md), it relies on scalar block bounds rather than polygonal approximations. It builds on [Empty Space Skipping via Data Structures](5619-empty-space-skipping-via-data-structures.md) by using a secondary data structure (min-max blocks) to represent active regions, allowing rays to skip inactive volumes.

### [Ray Casting with Depth/Alpha Test for Empty Space Skipping](1537-ray-casting-with-depth-alpha-test-for-empty-space-skipping.md)
This method accelerates ray-based volume rendering by using OpenGL’s alpha test and depth test to determine the near and far intersection points of rays with the volume, allowing samples to be taken only between these points. It sets itself apart from sibling methods like [Space Leaping](5863-space-leaping.md) and [Occupancy Grid and Map-Based Empty Space Skipping](3852-occupancy-grid-and-map-based-empty-space-skipping.md) by performing the empty space detection entirely on the GPU using hardware z-occlusion culling, without requiring CPU-side preprocessing or hierarchical data structures like octrees.

### [Semi-transparent Empty-Space Skipping](1911-semi-transparent-empty-space-skipping.md)
Semi-transparent Empty-Space Skipping is a rendering technique for transparent or semi-transparent volumes that combines hierarchical acceleration structures with analytical integration to skip inactive regions. Unlike opaque volume rendering which stops at the first hit, this method allows rays to traverse through semi-transparent media, evaluating the optical contribution of the traversed segment analytically based on its length and density. It distinguishes itself by using lower bounds in bounding volume hierarchies (BVH) for sphere tracing inside objects, enabling efficient empty-space skipping for both constant and varying density materials.

### [Dynamic MLP Maps with Empty Space Skipping](3036-dynamic-mlp-maps-with-empty-space-skipping.md)
Dynamic MLP Maps is a rendering technique for volumetric videos that combines multi-level hash tables with 2D CNN-generated MLP maps and uses an occupancy volume for empty space skipping to achieve real-time performance. It differs from its siblings by using a hybrid representation where spatial features are stored in hash tables and network parameters are predicted by a CNN decoder onto orthogonal planes, allowing for efficient evaluation of multiple small MLPs. Unlike [Object-Order Empty Space Skipping](865-object-order-empty-space-skipping.md) which operates on geometry, or [Space Leaping](5863-space-leaping.md) which uses fixed grid strides, this method dynamically skips empty space using a pre-computed per-frame occupancy volume.

### [Occupancy Grid and Map-Based Empty Space Skipping](3852-occupancy-grid-and-map-based-empty-space-skipping.md)
This family of methods accelerates volumetric ray casting by using a secondary, lower-resolution grid (occupancy map) to identify inactive regions of the volume, allowing rays to skip sampling points in empty space. It differs from hierarchical methods by using a flat, regular grid structure, and from distance-based methods by relying on binary occupancy flags rather than distance-to-nearest-occupied-voxel metrics [G].

### [Space Leaping](5863-space-leaping.md)
Space Leaping is a ray-coherence-based acceleration technique for volume rendering that skips empty regions by leveraging the spatial correlation between adjacent rays. Unlike hierarchical data structures (e.g., octrees) that require complex traversal logic and significant memory overhead, this method uses a lightweight image-plane leap buffer to record the sample count to the first non-empty voxel for each pixel, allowing neighboring rays to skip empty space based on that information. It cannot achieve as much empty space skipping as techniques using data structures for empty/non-empty voxel info because it only leaps one out of every 4 rays, getting 75% of possible acceleration. It is primarily suited for applications where the camera is close to the object surface, such as virtual endoscopy, where large volumes of empty space exist in front of the camera.

### [Hybrid kd-tree/Uniform Grid](6487-hybrid-kd-tree-uniform-grid.md)
This method accelerates volume rendering by combining a kd-tree at the root level with a uniform grid at the leaf-node level to perform empty-space skipping. It addresses the trade-off between the high construction cost of deep trees and the coarse skipping granularity of shallow trees by decoupling tree depth from skipping granularity, allowing for faster construction while maintaining efficiency in large datasets.

### Empty Space Skipping with HistoPyramid
- **Definition:** A technique achieving extremely fine-grained empty space skipping by marching rays only through active volume cells, located efficiently with a HistoPyramid texture and rendered as point primitives [S1].
- **Steps:**
    1.  Locate active volume cells using a HistoPyramid texture [S1].
    2.  Render active cells as point primitives to create rays close to the isosurface [S1].
    3.  Accelerate by exploiting early-depth testing: compare fragment's interpolated z-coordinate with depth buffer value; discard if occluded [S1].
    4.  Rasterize primitives in front-to-back order to increase discarded fragments, achieved via CPU-based sorting of volume slices [S1].
- **Comparison:** Considered a hybrid object-order and image-order approach, meaning it is dependent on rendering resolution [S1].

### ISSUE Algorithm
- **Steps:**
    1.  Divide span space points into one of the $L \times L$ lattice elements [S2].
    2.  Assign each lattice element to a category based on its position relative to the isovalue $v$ [S2].
    3.  Trivially include cells in region 2 [S2].
    4.  For cells in region 3, require a test against their maximum value [S2].
    5.  For cells in region 4, require a test against their minimum value [S2].
    6.  For cells in region 5, require a full min-max search of its cells [S2].
    7.  Create search structures only at the lattice elements along the diagonal region 5 [S2].
    8.  Coalesce lattice elements at other regions into sorted linear arrays [S2].
    9.  Build a kd-tree for each lattice element along the $y = x$ line, since only these elements may fall into region 5 classification [S2].
- **Parameters:** Lattice resolution $L$: User-defined resolution for dividing span space points [S2].

### Mixture Graph Mipmap Skipping
- **Steps:**
    1.  Propagate current transfer function values to the leafs of the Mixture Graph [S3].
    2.  Use the hierarchy for skipping longer segments [S3].
    3.  If the opacity value in parent voxels falls below a certain threshold, traverse the hierarchy upwards to establish larger blocks that can be skipped [S3].
- **Parameters:** Threshold: A low opacity threshold used to trigger skipping; typical value is $10^{-8}$ [S3].
- **Performance:** Average 3x speedup compared to the same implementation without mipmaps using a threshold of $10^{-8}$ [S3].

## Background
The concept of empty space skipping has evolved from simple ray coherence techniques to complex hierarchical data structures. Early methods like Space Leaping [S7] relied on image-plane buffers. Later, GPU-driven approaches introduced object-order skipping using min-max block structures [S1]. The introduction of HistoPyramids [S1] allowed for fine-grained, GPU-driven identification of active cells. More recent developments involve hybrid structures like the Hybrid kd-tree/Uniform Grid [S1] and dynamic representations for neural volumes like Dynamic MLP Maps [S1]. The Mixture Graph [S3] and ISSUE algorithm [S2] represent specialized approaches for histogram-based and isosurface extraction acceleration, respectively.

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S2] A Case Study of Isosurface Extraction Algorithm Performance, https://doi.org/10.1007/978-3-7091-6783-0_26, fetched 2026-10-26
[S3] The Mixture Graph-A Data Structure for Compressing, Rendering, and Querying Segmentation Histograms, https://doi.org/10.1109/tvcg.2020.3030451, fetched 2026-10-26
