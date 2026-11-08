# Tetrahedral representation and traversal

Also called: Projected Tetrahedra family, Tetrahedral Volume Rendering primitives

## Overview
This family encompasses methods that represent volumetric data as a collection of tetrahedral primitives, leveraging their geometric properties for rendering, isosurface extraction, or spatial traversal. Unlike voxel-based methods, tetrahedral representations handle unstructured grids natively. The core distinction among siblings is the rendering pipeline: some project primitives to 2D for rasterization ([Projected Tetrahedra](877-projected-tetrahedra.md)), others extract surfaces via topological decomposition ([Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md)), and others use spatial structures for efficient ray traversal ([Grid-based Traversal](7083-grid-based-traversal.md)).

## Shared principles
The family shares the fundamental mechanism of treating volumetric data as a set of geometric primitives that are transformed and rasterized [S1, unofficial][S3][S4, book].

1.  **Primitive Representation:** The volume is decomposed into primitives. These may be explicit geometric shapes (cubes, cells, splats) or implicit functions sampled via texture hardware [S1, unofficial][S3][S4, book].
2.  **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp) [S2][S3][S4, book].
3.  **Rasterization:** The transformed primitives are projected onto the 2D image plane. This step leverages hardware-accelerated scan-conversion, depth testing, and blending [S1, unofficial][S2][S3][S4, book].
4.  **Compositing:** Contributions are accumulated to form the final image. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly [S1, unofficial][S2][S3, S7, unofficial].
5.  **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity [S1, unofficial][S2][S3].

## Subtopics

### [Projected Tetrahedra](877-projected-tetrahedra.md)
Project Tetrahedra (PT) is an object-order volume rendering technique that projects volumetric primitives, specifically tetrahedra, onto the 2D image plane to leverage hardware-accelerated rasterization. It solves the problem of rendering unstructured tetrahedral grids by decomposing each 3D cell into a set of partially transparent 2D triangles, which are then composited in back-to-front order. Unlike [Tetrahedral Volume Rendering](820-tetrahedral-volume-rendering.md), which often relies on ray casting, PT uses explicit geometric projection and scan-conversion. It differs from [GPU-based Projected Tetrahedra](7484-gpu-based-projected-tetrahedra.md) in that the original PT algorithm performs sorting and projection in software, whereas later variants offload these steps to the GPU.

### [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md)
The Marching Tetrahedrons Algorithm (MTA) is an isosurface extraction method that decomposes volumetric data cells into tetrahedra to resolve topological ambiguities inherent in cube-based methods like Marching Cubes. Unlike its siblings, MTA guarantees topological consistency by ensuring that the intersection of an isosurface with a tetrahedron is always a simple polygon, eliminating the need for complex case tables required by cubic decompositions.

### [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md)
Parallelized Accurate Sorting for Projected Tetrahedra is a view-dependent sorting algorithm for tetrahedral datasets that uses logical partitioning to enable parallel processing, solving the dependency and sequence limitations of neighboring tetrahedra in parallel performance. It sets itself apart from standard [Projected Tetrahedra](877-projected-tetrahedra.md) by replacing global topological sorting with a block-based logical partitioning scheme that allows for the parallel peeling of unobstructed layers, thereby significantly reducing sorting time and improving rendering performance.

### [Tetrahedral Strips and Fans](1389-tetrahedral-strips-and-fans.md)
Tetrahedral Strips and Fans are graphics primitives designed to address the bandwidth bottleneck between main memory and the graphics processor when rendering unstructured tetrahedral meshes. They reduce the number of vertices transmitted per tetrahedron, achieving rates approaching 1.2 vertices per tetrahedron, similar to how triangle strips optimize triangle meshes. Unlike static triangle strips, these primitives are necessary because tetrahedral decomposition is view-dependent, preventing the use of static display lists when changing viewpoints. They enable performance improvements of 72 to 85 times over triangle-based hardware approaches, making interactive volumetric rendering feasible.

### [Adaptive Delaunay Update](4923-adaptive-delaunay-update.md)
Adaptive Delaunay Update is a method for maintaining the topological validity of a Delaunay tetrahedralization during optimization processes, such as those used in radiance field reconstruction. It solves the problem of computational cost associated with full Delaunay re-triangulation by selectively updating vertex positions and topology only when necessary. It sets itself apart from full re-computation approaches by decoupling the frequency of topological updates from the optimization steps, balancing GPU acceleration speed with geometric accuracy.

### [Grid-based Traversal](7083-grid-based-traversal.md)
Grid-based Traversal is a method for rendering irregular volume data (tetrahedral meshes) at interactive rates by partitioning the volume into a hierarchical grid structure to quickly determine candidate tetrahedra. It differs from sibling methods like [Projected Tetrahedra](877-projected-tetrahedra.md) by avoiding explicit geometric projection and sorting of primitives; instead, it relies on spatial hashing via a grid to filter tetrahedra for ray intersection tests.

### [Parallel Projected Tetrahedra with KD-Tree Partitioning](8767-parallel-projected-tetrahedra-with-kd-tree-partitioning.md)
This method enables parallel rendering of unstructured tetrahedral grids by partitioning the data using a kd-tree to facilitate a sort-last compositing algorithm. Unlike approaches that sample the volume into a regular grid—which require large numbers of samples, parallel distribution of samples, and suffer from oversampling or undersampling in varying cell sizes—this method renders the unstructured grid natively. It sets itself apart from other parallel rendering approaches by requiring no further cell transfer once cells are distributed; only image data is communicated between processes.

### [Tetrahedral Mesh Connectivity Encoding](8957-tetrahedral-mesh-connectivity-encoding.md)
The Cut-Border Machine (CBM) is a region-growing compression scheme for tetrahedral meshes that encodes connectivity by growing a region bounded by a cut-border. It solves the problem of efficiently storing and transmitting the topological structure of tetrahedral meshes, which is required for object-order volume rendering methods like [Projected Tetrahedra](877-projected-tetrahedra.md) and [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md). Unlike sibling methods that may rely on explicit geometric sorting or fixed grid structures, CBM encodes the mesh topology purely through a stream of opcodes describing the position of new vertices relative to a growing boundary. This allows for compact storage and reconstruction of the mesh connectivity without preserving the original vertex ordering, though it requires a specific traversal order for efficiency.

### Screen Space Area Calculation
- **Definition:** Calculates the projected area of a tetrahedron in normalized screen space to determine refinement criteria or culling. Uses cross products to compute wedge areas, which are twice the triangle area, then divides by four [S1, unofficial].
- **Step:** Project tetrahedron vertices $T, L, R, B$ to screen space using camera projection: $T \leftarrow \text{camera.projectPoint}(T)$, $L \leftarrow \text{camera.projectPoint}(L)$, $R \leftarrow \text{camera.projectPoint}(R)$, $B \leftarrow \text{camera.projectPoint}(B)$. Calculate area sum using cross products of vertex vectors: $\text{area} \leftarrow |(L-T)\times(R-T)| + |(R-T)\times(B-T)|$. Add remaining terms: $\text{area} \leftarrow \text{area} + |(B-T)\times(L-T)| + |(L-B)\times(R-B)|$. Return final area divided by 4: $\text{return area} / 4$ [S1, unofficial].
- **Pitfall:** Triangles cover the total area twice in projection, and cross products yield wedge areas (twice triangle area), necessitating division by 4 [S1, unofficial].

### Point Representation via Tetrahedra
- **Definition:** Representing data points as small tetrahedra to fit them into the triangular mesh framework for rendering [S2].
- **Parameter:** `size`: specifies the size of the point tetrahedra relative to the data ranges [S2].

### Delaunay Triangulation for Tetrahedral Representation
- **Step:** Obtain a tetrahedral representation of the Voronoi sites via Delaunay triangulation using gDel3D to accelerate computation on the GPU [S3]. Recompute the triangulation from scratch at every iteration [S3].

## Background
The use of tetrahedra in volume rendering originated with the need to handle unstructured grids, which are common in scientific simulations (e.g., finite element analysis). The Projected Tetrahedra method was introduced to allow the use of standard graphics hardware for unstructured data by projecting tetrahedra into 2D triangles [S1, unofficial][S2, S3][S4, S6, S7]. The Marching Tetrahedrons Algorithm was developed as a robust alternative to Marching Cubes, resolving topological ambiguities by decomposing cubes into tetrahedra [S1, unofficial]. Subsequent research focused on optimizing performance through parallelization [S1, unofficial][S2], efficient data transmission via strips and fans [S1, unofficial][S2], and adaptive updates for dynamic data [S1, unofficial].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
[S2] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S3] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
