# Tetrahedral representation and traversal
Also called: Projected Tetrahedra family, Cell Projection via Tetrahedra

## Overview
This family renders volumetric data by decomposing the volume into tetrahedral primitives, projecting them onto the 2D image plane, and rasterizing them to accumulate color and opacity [S1, S2][S3, unofficial][S4, S6, S7][S8, S9, S10, unofficial][S11][S12, unofficial][S14, S16][S17, S18, unofficial]. It solves the problem of rendering unstructured tetrahedral grids with high accuracy and interactivity, avoiding the approximations of ray-marching on regular grids. It is distinguished from [Tetrahedral Volume Rendering](820-tetrahedral-volume-rendering.md) by its specific use of Basis Tensor (BT) volumes for isosurface selection and its two-pass GPU pipeline architecture, and from [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md) by computing isosurfaces per fragment rather than generating explicit polygonal cuts.

## Shared principles
The family shares the fundamental mechanism of treating volumetric data as a set of geometric primitives that are transformed and rasterized [S1][S3, unofficial][S4, book].

1.  **Primitive Representation:** The volume is decomposed into primitives. These may be explicit geometric shapes (cubes, cells, splats) or implicit functions sampled via texture hardware [S1][S3, unofficial][S4, book].
2.  **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp) [S2][S3, unofficial][S4, book].
3.  **Rasterization:** The transformed primitives are projected onto the 2D image plane. This step leverages hardware-accelerated scan-conversion, depth testing, and blending [S1, S2][S3, unofficial][S4, book].
4.  **Compositing:** Contributions are accumulated to form the final image. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly [S1, S2, S3, S7, book].
5.  **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity [S1, S2][S3, unofficial].

## Subtopics

### [Tetrahedral Volume Rendering](820-tetrahedral-volume-rendering.md)
Tetrahedral Volume Rendering is a cell-projection method that renders unstructured tetrahedral grids by projecting individual tetrahedra onto the screen and accumulating color and opacity in screen space. It solves the problem of rendering complex, non-convex, or disconnected meshes with high accuracy and interactivity, avoiding the approximations of ray-marching on regular grids. It is distinguished from [Projected Tetrahedra](877-projected-tetrahedra.md) by its specific use of Basis Tensor (BT) volumes for isosurface selection and its two-pass GPU pipeline architecture, and from [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md) by computing isosurfaces per fragment rather than generating explicit polygonal cuts.

### [Projected Tetrahedra](877-projected-tetrahedra.md)
Projected Tetrahedra (PT) is an object-order volume rendering technique that projects volumetric primitives, specifically tetrahedra, onto the 2D image plane to leverage hardware-accelerated rasterization. It solves the problem of rendering unstructured tetrahedral grids by decomposing each 3D cell into a set of partially transparent 2D triangles, which are then composited in back-to-front order. Unlike [Tetrahedral Volume Rendering](820-tetrahedral-volume-rendering.md), which often relies on ray casting, PT uses explicit geometric projection and scan-conversion. It differs from [GPU-based Projected Tetrahedra](7484-gpu-based-projected-tetrahedra.md) in that the original PT algorithm performs sorting and projection in software, whereas later variants offload these steps to the GPU.

### [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md)
The Marching Tetrahedrons Algorithm (MTA) is an iso-surface extraction method that converts a scalar field defined on a regular grid into a triangular mesh by subdividing each cubic voxel into tetrahedra and processing each tetrahedron independently [G]. It solves the problem of extracting explicit polygonal surfaces from volumetric data [G]. Unlike its sibling [Tetrahedral Volume Rendering](820-tetrahedral-volume-rendering.md), which focuses on direct volume rendering via ray casting, MTA generates an explicit mesh [G]. It differs from [Projected Tetrahedra](877-projected-tetrahedra.md) by constructing a connected triangle mesh rather than projecting volumetric primitives to the screen [G]. The algorithm unfortunately has directional ambiguities because it subdivides a cube with tetrahedrons, which cannot be done symmetrically.

### [Adaptive Delaunay Update](4923-adaptive-delaunay-update.md)
Adaptive Delaunay Update is a strategy for maintaining the topological validity of a tetrahedral mesh during optimization-based volumetric reconstruction, specifically within frameworks like Radiance Meshes. It addresses the problem of computational cost versus geometric accuracy when vertex positions change dynamically. Unlike methods that recompute the Delaunay triangulation from scratch at every iteration, this method updates the topology only periodically, balancing GPU acceleration speed with rendering accuracy.

### [GPU-based Projected Tetrahedra](7484-gpu-based-projected-tetrahedra.md)
GPU-based Projected Tetrahedra is a direct volume rendering method for unstructured tetrahedral data that achieves interactive rates by treating each tetrahedron as a distinct geometric primitive and rasterizing it using hardware operations. It differs from traditional ray casting by leveraging hardware rasterization and blending to compute pixel color, avoiding explicit per-pixel ray-tetrahedron intersection tests. It builds on [Projected Tetrahedra](877-projected-tetrahedra.md) by mapping the classification and decomposition logic into programmable vertex shaders, allowing tetrahedral elements to be volume scan-converted within the GPU without pre-processing or additional data structures.

### [Texture-Encoded Isosurfacing](7682-texture-encoded-isosurfacing.md)
Texture-Encoded Isosurfacing is a GPU-based method for extracting isosurfaces from unstructured tetrahedral grids by leveraging 2D texturing hardware to compute intersection points. It solves the problem of efficiently rasterizing implicit surfaces on unstructured data by replacing explicit geometric intersection calculations with texture lookups. It sets itself apart from sibling methods like [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md) by avoiding the generation of explicit triangle primitives for the surface; instead, it projects tetrahedra faces and uses a checkerboard texture to encode the scalar field values, allowing the GPU's texture sampling hardware to determine the isosurface location.

### [Parallel Projected Tetrahedra with KD-Tree Partitioning](8767-parallel-projected-tetrahedra-with-kd-tree-partitioning.md)
This method enables parallel rendering of large unstructured tetrahedral grids by partitioning the data using a kd-tree and applying a sort-last compositing algorithm. It renders unstructured grids natively without sampling into a regular grid, avoiding the oversampling and undersampling issues inherent in grid-based approaches. Unlike other parallel rendering approaches that require continuous cell transfer, this method requires no further cell transfer once distributed; only image data is communicated between processes.

### [Tetrahedral Mesh Connectivity Encoding](8957-tetrahedral-mesh-connectivity-encoding.md)
The Basic Cut-Border Machine (CBM) is a region-growing compression scheme that encodes the connectivity of a tetrahedral mesh by growing a region bounded by a cut-border. It solves the problem of efficiently storing and transmitting tetrahedral mesh topology by generating a stream of opcodes that describe the position of new vertices relative to the current boundary. Unlike sibling methods that may rely on global indexing or explicit adjacency lists, CBM determines connectivity locally during traversal, ensuring unique identification of vertices across multi-resolution segments through shared vertex indices.

### Point Representation via Tetrahedra
- **Definition:** Representing data points as small tetrahedra to fit them into the triangular mesh framework for rendering [S1].
- **Parameter:** `size`: specifies the size of the point tetrahedra relative to the data ranges [S1].

### LinPrim Tetrahedron Representation
- **Definition:** A scene representation using transparent tetrahedra bounded by triangular faces with homogeneous density [S2].
- **Step:** Define four basis vectors describing directions from the center on which corners are located, equally spaced to represent regular tetrahedra [S2]. Optimize the distance between the center and each corner independently [S2].
- **Parameter:** Tetrahedra memory requirement: 12 floats per primitive (omitting color), due to absence of symmetry in the formulation [S2].
- **Pitfall:** Unlike octahedra, the optimized position (center) can differ from the geometric center of the primitive due to lack of symmetry [S2].

### Grid-based Traversal
- **Definition:** A method using a hierarchical data structure for rectilinear grids to quickly determine tetrahedra possibly hit by traversed grid cells, enabling interactive rendering of irregular volume data [S3, unofficial].
- **Step:** Attach all tetrahedra intersecting a given grid cell to that cell [S3, unofficial]. Use minimum and maximum values to cull regions outside of the current iso-value [S3, unofficial].
- **Number:** Rendered a bioelectric field of over one million tetrahedra at interactive rates (512x512 viewport) with 16 processors or more [S3, unofficial].

## Background
Projected Tetrahedra was introduced by Parker et al. in 1999 as one of the first methods capable of rendering irregular volume data at interactive rates on a supercomputer [S3, unofficial]. The method decomposes the volume into tetrahedra and projects them onto the screen, leveraging hardware rasterization [S3, unofficial]. Early implementations used grid-based traversal to accelerate the identification of tetrahedra intersecting grid cells [S3, unofficial]. Later developments moved the projection and classification logic to the GPU, enabling real-time interactivity [S4, S7][S12, unofficial][S14, S16]. The method has been extended to parallel rendering using kd-tree partitioning [S1] and differentiable rendering [S1].

## Sources
[S1] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S2] LinPrim: Linear Primitives for Differentiable Volumetric Rendering, https://arxiv.org/abs/2501.16312, fetched 2026-10-26
[S3] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
