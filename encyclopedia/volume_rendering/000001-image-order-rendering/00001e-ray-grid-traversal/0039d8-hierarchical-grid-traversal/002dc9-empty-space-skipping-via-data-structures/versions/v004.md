# Empty Space Skipping via Data Structures

Also called: Acceleration Structures for Volume Rendering, Active Region Culling

## Overview
Empty Space Skipping via Data Structures is a family of volume rendering optimization techniques that reduce computational cost by identifying and bypassing inactive (empty) regions of the volume data. It solves the problem of wasting samples on voxels that do not contribute to the final image (e.g., zero density or below an isovalue threshold). Unlike image-order methods like [Space Leaping](5863-space-leaping.md) which rely on ray coherence, or [Distance Map Acceleration](972-distance-map-acceleration.md) which uses metric distances, this family relies on explicit secondary data structures (grids, trees, or maps) to define the spatial extent of active data. The specific implementation varies significantly: some methods use object-order geometry culling [865-object-order-empty-space-skipping.md], others use hardware depth tests [1537-ray-casting-with-depth-alpha-test-for-empty-space-skipping.md], and others use hierarchical tree traversal [1911-semi-transparent-empty-space-skipping.md].

## Shared principles
All methods in this family share the core mechanism of using a secondary data structure to represent the spatial distribution of active volume data, allowing rays to skip over inactive regions without sampling the primary volume texture.

1.  **Acceleration Structure:** A data structure (e.g., occupancy grid, octree, BVH, hash map) is constructed to represent the volume. Each element in this structure corresponds to a region of the volume and stores metadata indicating whether that region contains "active" data (e.g., density above a threshold, non-zero opacity) [S1][S2, unofficial].
2.  **Active Region Definition:** A region is defined as active if it contributes to the final image. Common criteria include:
    *   Maximum voxel value exceeding an isovalue threshold $\tau$ [S1].
    *   Presence of non-zero opacity after transfer function application [S1][S2, unofficial].
    *   Density values above a specific threshold $\tau_1$ [S1].
3.  **Ray Query and Skip:** During rendering, for each ray, the algorithm queries the acceleration structure to find the entry and exit points of active regions.
    *   If a ray segment lies entirely within an inactive region indicated by the structure, the ray is advanced to the next active region boundary without sampling the volume [S1][S2, unofficial].
    *   If a ray segment intersects an active region, standard ray marching or integration is performed within that segment [S1][S2, unofficial].
4.  **Coordinate Mapping:** Efficient mapping between ray positions in world/voxel space and indices in the acceleration structure is required. For regular grids, this is often a simple division by block size: $u_{\text{occ}} = u_{\text{vol}} / B$ [S1].

## Subtopics

### [Object-Order Empty Space Skipping](865-object-order-empty-space-skipping.md)
Object-Order Empty Space Skipping is a GPU-based volume rendering optimization that reduces the number of volume texture samples by skipping inactive regions using a secondary data structure of block min-max values. It differs from image-order methods (like [Space Leaping](5863-space-leaping.md)) by computing ray entry and exit bounds via rasterized bounding geometry of active blocks, allowing the fragment shader to skip large inactive segments without per-sample overhead. It builds on [Empty Space Skipping via Data Structures](5619-empty-space-skipping-via-data-structures.md) by using a regular grid subdivision where metadata drives the geometric culling.

### [Ray Casting with Depth/Alpha Test for Empty Space Skipping](1537-ray-casting-with-depth-alpha-test-for-empty-space-skipping.md)
This method accelerates ray-based volume rendering by using OpenGL’s alpha test and depth test to determine the near and far intersection points of rays with the volume, allowing samples to be taken only between these points. It sets itself apart from sibling methods like [Space Leaping](5863-space-leaping.md) and [Occupancy Grid and Map-Based Empty Space Skipping](3852-occupancy-grid-and-map-based-empty-space-skipping.md) by performing the empty space detection entirely on the GPU using hardware z-occlusion culling, without requiring CPU-side preprocessing or hierarchical data structures like octrees.

### [Semi-transparent Empty-Space Skipping](1911-semi-transparent-empty-space-skipping.md)
Semi-transparent Empty-Space Skipping is a rendering technique for transparent or semi-transparent volumes that combines hierarchical acceleration structures with analytical integration to skip inactive regions. Unlike opaque volume rendering which stops at the first hit, this method allows rays to traverse through semi-transparent media, evaluating the optical contribution of the traversed segment analytically based on its length and density. It distinguishes itself by using lower bounds in bounding volume hierarchies (BVH) for sphere tracing inside objects, enabling efficient empty-space skipping for both constant and varying density materials.

### [Dynamic MLP Maps with Empty Space Skipping](3036-dynamic-mlp-maps-with-empty-space-skipping.md)
Dynamic MLP Maps is a rendering technique for volumetric videos that combines multi-level hash tables with 2D CNN-generated MLP maps and uses an occupancy volume for empty space skipping to achieve real-time performance. It differs from its siblings by using a hybrid representation where spatial features are stored in hash tables and network parameters are predicted by a CNN decoder onto orthogonal planes, allowing for efficient evaluation of multiple small MLPs. Unlike [Object-Order Empty Space Skipping](865-object-order-empty-space-skipping.md) which operates on geometry, or [Space Leaping](5863-space-leaping.md) which uses fixed grid strides, this method dynamically skips empty space using a pre-computed per-frame occupancy volume.

### [Occupancy Grid and Map-Based Empty Space Skipping](3852-occupancy-grid-and-map-based-empty-space-skipping.md)
This family of methods accelerates volumetric ray casting by using a secondary, lower-resolution grid (occupancy map) to identify inactive regions of the volume, allowing rays to skip sampling points in empty space. It differs from hierarchical methods like [Space Leaping](5863-space-leaping.md) by using a flat, regular grid structure, and from [Distance Map Acceleration](972-distance-map-acceleration.md) by relying on binary occupancy flags rather than distance-to-nearest-occupied-voxel metrics.

### [Space Leaping](5863-space-leaping.md)
Space Leaping is a ray-coherence-based acceleration technique for volume rendering that skips empty regions by leveraging the spatial correlation between adjacent rays. Unlike hierarchical data structures (e.g., octrees) that require complex traversal logic and significant memory overhead, this method uses a lightweight buffer to record the depth of the first hit for each pixel, allowing neighboring rays to leap over empty space based on that information. It is distinct from sibling methods like [Occupancy Grid and Map-Based Empty Space Skipping](3852-occupancy-grid-and-map-based-empty-space-skipping.md) because it avoids per-voxel metadata storage, relying instead on image-plane coherence. It is primarily suited for applications where the camera is close to the object surface, such as virtual endoscopy, where large volumes of empty space exist in front of the camera.

### Empty Space Skipping with HistoPyramid
- [S1] A technique achieving extremely fine-grained empty space skipping by marching rays only through active volume cells, located efficiently with a HistoPyramid texture and rendered as point primitives.
- [S1] 1. Locate active volume cells using a HistoPyramid texture. 2. Render active cells as point primitives to create rays close to the isosurface. 3. Accelerate by exploiting early-depth testing: compare fragment's interpolated z-coordinate with depth buffer value; discard if occluded. 4. Rasterize primitives in front-to-back order to increase discarded fragments, achieved via CPU-based sorting of volume slices.
- [S1] Considered a hybrid object-order and image-order approach, meaning it is dependent on rendering resolution.

### Hybrid kd-tree/Uniform Grid
- [S2, unofficial] A hybrid data structure combining a kd-tree structure at the root level and a uniform grid at the leaf-node level used for empty-space skipping.
- [S2, unofficial] During rendering, the kd-tree is traversed on the CPU to generate a sorted list of non-empty leaf nodes that are then iterated over on the GPU.

### ISSUE Algorithm
- [S3] Divide span space points into one of the L x L lattice elements.
- [S3] Assign each lattice element to a category based on its position relative to the isovalue v (see Figure 3 cases).
- [S3] Trivially include cells in region 2.
- [S3] For cells in region 3, require a test against their maximum value.
- [S3] For cells in region 4, require a test against their minimum value.
- [S3] For cells in region 5, require a full min-max search of its cells.
- [S3] Create search structures only at the lattice elements along the diagonal region 5.
- [S3] Coalesce lattice elements at other regions into sorted linear arrays.
- [S3] Build a kd-tree for each lattice element along the y = x line, since only these elements may fall into region 5 classification.
- [S3] Lattice resolution L: User-defined resolution for dividing span space points.

## Background
The concept of skipping empty space is fundamental to efficient volume rendering. Early approaches like Space Leaping [S2, unofficial] relied on image coherence. The use of explicit data structures for acceleration grew with the need to handle larger datasets and more complex transfer functions. Liu et al. [LCD09] introduced HistoPyramid-based skipping [S1], while Jiang et al. [JRZ*16] proposed IsoBAS [S1]. Zellmann et al. [ZML19] explored hybrid kd-tree/uniform grids [S2, unofficial]. The ISSUE algorithm by Shen et al. [14] [S3] used lattice subdivision for isosurface extraction, which shares the philosophy of spatial culling. Modern approaches like Dynamic MLP Maps [S1] apply these principles to neural radiance fields (NeRF) and volumetric video.

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S3] A Case Study of Isosurface Extraction Algorithm Performance, https://doi.org/10.1007/978-3-7091-6783-0_26, fetched 2026-10-26
