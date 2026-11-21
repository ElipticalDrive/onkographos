# Tetrahedral representation and traversal

Also called: Cell projection using tetrahedra, Projected Tetrahedra family

## Overview
This family renders unstructured tetrahedral volume data by projecting 3D tetrahedral cells onto the 2D image plane, decomposing them into partially transparent 2D polygons, and compositing them in back-to-front order [S1, S2, unofficial][S4, S6, S7][S8, S9, S10, unofficial][S11][S12, unofficial][S14, S16][S17, S18, unofficial]. It solves the problem of rendering irregular grids by leveraging hardware-accelerated rasterization for the projection and interpolation steps, avoiding the sampling artifacts of grid-based methods and the computational cost of per-ray traversal for every pixel [S2, unofficial][S4, S6, S7][S12, unofficial][S14, S16]. It differs from [Tetrahedral Volume Rendering](820-tetrahedral-volume-rendering.md) by using explicit geometric projection rather than ray casting, and from [Parallel Cell Projection](875-parallel-cell-projection.md) by focusing specifically on the tetrahedral decomposition and its associated sorting/projection challenges.

## Shared principles
The tetrahedral representation and traversal family shares the following mechanism, terms, and math:

1.  **Primitive Representation:** The volume is decomposed into tetrahedral cells. For regular cubic grids, each cube is subdivided into 5 or 6 tetrahedra to ensure planar faces and consistent topology, resolving "problem cells" where rays might intersect a face twice in curvilinear grids [S2, S8, S13, S19, unofficial][S1, unofficial].
2.  **Projection and Subdivision:** The 3D vertices of each tetrahedron are projected to 2D screen space. The projected edges divide the image plane into polygons, where each polygon lies within the projection of a single front-facing face and a single back-facing face of the tetrahedron [S1, S2, unofficial][S4]. For general cells, an incremental method builds this subdivision by adding projected edges one by one [S1, S2, unofficial].
3.  **Scalar Interpolation:** Scalar values $s$ at any point within a tetrahedron are interpolated linearly from the vertex values $s_i$ using barycentric coordinates or linear interpolation along edges [S1, unofficial][S2, unofficial][S3]. The scalar field is assumed to vary linearly across the cell, and transfer functions mapping scalar to color/extinction are assumed linear [S1, S2, unofficial][S4].
4.  **Opacity Calculation:** The integrated opacity $\alpha$ for a segment of length $l$ with average differential opacity $\tau_{avg}$ is:
    $$ \alpha = 1 - \exp(-\tau_{avg} l) $$
    [S1, unofficial][S4].
5.  **Visibility and Sorting:** Correct compositing requires back-to-front ordering. This is achieved via global visibility sorts (e.g., sorting by centroid depth) or image-space merging of ray segments [S1, unofficial][S4, S7, S11][S12, unofficial][S14][S17, unofficial].
6.  **Compositing:** Fragments or ray segments are composited using the Porter-Duff over operation in back-to-front order [S1, unofficial][S2, unofficial][S6, S7][S9, S10, unofficial][S11][S12, unofficial][S14, S16][S17, S18, unofficial].

## Subtopics

### [Projected Tetrahedra](877-projected-tetrahedra.md)
Projected Tetrahedra (PT) is the foundational technique of this family. It projects volumetric primitives, specifically tetrahedra, onto the 2D image plane to leverage hardware-accelerated rasterization. It solves the problem of rendering unstructured tetrahedral grids by decomposing each 3D cell into a set of partially transparent 2D triangles, which are then composited in back-to-front order. Unlike [Tetrahedral Volume Rendering](820-tetrahedral-volume-rendering.md), which often relies on ray casting, PT uses explicit geometric projection and scan-conversion. It differs from [GPU-based Projected Tetrahedra](7484-gpu-based-projected-tetrahedra.md) in that the original PT algorithm performs sorting and projection in software, whereas later variants offload these steps to the GPU.

### [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md)
The Marching Tetrahedrons Algorithm (MTA) is an isosurface extraction method that decomposes volumetric data cells into tetrahedra to resolve topological ambiguities inherent in cube-based methods like Marching Cubes. Unlike its siblings, MTA guarantees topological consistency by ensuring that the intersection of an isosurface with a tetrahedron is always a simple polygon, eliminating the need for complex case tables required by cubic decompositions.

### [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md)
Parallelized Accurate Sorting for Projected Tetrahedra is a view-dependent sorting algorithm designed to resolve the parallelization bottlenecks inherent in standard Projected Tetrahedra rendering. It solves the dependency and sequence limitations of neighboring tetrahedra by using logical partitioning to divide the dataset into blocks along the viewing direction, enabling the parallel sorting of unobstructed layers. It sets itself apart from its siblings, such as [Parallel Projected Tetrahedra with KD-Tree Partitioning](8767-parallel-projected-tetrahedra-with-kd-tree-partitioning.md), by relying on a logical partitioning scheme that peels tetrahedral subsets into layers rather than spatial tree traversal, and from [Sorting-free Pre-integrated Projected Tetrahedra](7491-sorting-free-pre-integrated-projected-tetrahedra.md) by maintaining explicit topological sorting to ensure accurate compositing order.

### [Tetrahedral Strips and Fans](1389-tetrahedral-strips-and-fans.md)
Tetrahedral Strips and Fans are graphics primitives designed to address the bandwidth bottleneck between main memory and the graphics processor when rendering unstructured tetrahedral meshes. They reduce the number of vertices transmitted per tetrahedron, achieving rates approaching 1.2 vertices per tetrahedron, similar to how triangle strips optimize triangle meshes. Unlike static triangle strips, these primitives are necessary because tetrahedral decomposition is view-dependent, preventing the use of static display lists when changing viewpoints. They enable performance improvements of 72 to 85 times over triangle-based hardware approaches, making interactive volumetric rendering feasible.

### [Adaptive Delaunay Update](4923-adaptive-delaunay-update.md)
Adaptive Delaunay Update is a strategy for maintaining the topological validity of a tetrahedral mesh during optimization-based volume rendering, specifically within frameworks like Radiance Meshes. It addresses the problem of mesh degeneracy caused by vertex movement during training by periodically recomputing the Delaunay triangulation to restore the empty-circumsphere property. It sets itself apart from incremental update methods by trading continuous topological accuracy for significant GPU acceleration speed, relying on batched recomputation rather than real-time edge flipping.

### [Grid-based Traversal](7083-grid-based-traversal.md)
Grid-based Traversal is a method for rendering irregular volume data (tetrahedral meshes) at interactive rates by partitioning the volume into a hierarchical grid structure to quickly determine candidate tetrahedra. It sets itself apart from its siblings by avoiding explicit connectivity information between tetrahedra, instead relying on spatial hashing into a grid to manage the large number of primitives. Unlike [Projected Tetrahedra](877-projected-tetrahedra.md), which projects primitives to screen space, this method operates in object space using ray-tetrahedron intersection tests within grid cells.

### [Sorting-free Pre-integrated Projected Tetrahedra](7491-sorting-free-pre-integrated-projected-tetrahedra.md)
Sorting-free Pre-integrated Projected Tetrahedra (PBVR) is a volume rendering technique for irregular volume datasets that eliminates the need for explicit visibility sorting of tetrahedral cells. It solves the scalability issues of conventional object-order methods, which require global sorting or large k-buffers, and the low image quality (visible footprints) of particle-based methods when the viewpoint is inside the volume. Unlike [Projected Tetrahedra](877-projected-tetrahedra.md), which requires back-to-front sorting, PBVR treats rendering as a stochastic process where fragments are rendered with a probability equal to their opacity, achieving semi-transparency without alpha-blending or sorting.

### [Parallel Projected Tetrahedra with KD-Tree Partitioning](8767-parallel-projected-tetrahedra-with-kd-tree-partitioning.md)
This method enables parallel volume rendering of large unstructured tetrahedral grids by partitioning the data using a kd-tree and applying a traditional sort-last compositing algorithm. It renders unstructured grids natively without sampling into a regular grid, avoiding the oversampling/undersampling issues and parallel distribution complexities associated with grid-based sampling approaches. Unlike other parallel rendering approaches that require continuous cell transfer, this method requires no further cell transfer once distributed; only image data is communicated between processes for final composition.

### [Tetrahedral Mesh Connectivity Encoding](8957-tetrahedral-mesh-connectivity-encoding.md)
Tetrahedral Mesh Connectivity Encoding, specifically the Basic Cut-Border Machine (CBM), is a region-growing compression scheme that encodes the connectivity of a tetrahedral mesh by traversing it and generating a stream of opcodes. It solves the problem of efficiently storing and transmitting mesh topology by describing the position of new vertices relative to a growing boundary (the cut-border). Unlike sibling methods that may rely on global sorting or explicit neighbor pointers, CBM uses a local, incremental construction process where the connectivity is fully determined by 11 distinct opcodes. This method is particularly suited for multi-resolution meshes where segments are encoded sequentially, requiring specific handling for shared vertices across segment boundaries.

### Screen Space Area Calculation
- **Definition:** Calculates the projected area of a tetrahedron in normalized screen space to determine refinement criteria or culling [S1, unofficial]. Uses cross products to compute wedge areas, which are twice the triangle area, then divides by four [S1, unofficial].
- **Steps:**
    1. Project tetrahedron vertices $T, L, R, B$ to screen space using camera projection: $T \leftarrow \text{camera.projectPoint}(T)$, $L \leftarrow \text{camera.projectPoint}(L)$, $R \leftarrow \text{camera.projectPoint}(R)$, $B \leftarrow \text{camera.projectPoint}(B)$ [S1, unofficial].
    2. Calculate area sum using cross products of vertex vectors: $\text{area} \leftarrow |(L-T) \times (R-T)| + |(R-T) \times (B-T)|$ [S1, unofficial].
    3. Add remaining terms: $\text{area} \leftarrow \text{area} + |(B-T) \times (L-T)| + |(L-B) \times (R-B)|$ [S1, unofficial].
    4. Return final area divided by 4: $\text{return area} / 4$ [S1, unofficial].
- **Pitfall:** Triangles cover the total area twice in projection, and cross products yield wedge areas (twice triangle area), necessitating division by 4 [S1, unofficial].

### Early Tetrahedron Termination
- **Definition:** A technique that reduces the amount of processed tetrahedra by leveraging a summed-area table for accumulating and comparing opacity of individual tetrahedra on the rendering phase, after they have been sorted into unobstructed layers [S2, unofficial].
- **Performance:** Gain of more than 10% in rendering performance for datasets with more than a million tetrahedra [S2, unofficial].

### Point Representation via Tetrahedra
- **Definition:** Representing data points as small tetrahedra to fit them into the triangular mesh framework for rendering [S3].
- **Parameter:** `size`: specifies the size of the point tetrahedra relative to the data ranges [S3].

### LinPrim Tetrahedron Representation
- **Definition:** A scene representation using transparent tetrahedra bounded by triangular faces with homogeneous density [S4].
- **Steps:**
    1. Define four basis vectors describing directions from the center on which corners are located, equally spaced to represent regular tetrahedra [S4].
    2. Optimize the distance between the center and each corner independently [S4].
- **Parameter:** Tetrahedra memory requirement: 12 floats per primitive (omitting color), due to absence of symmetry in the formulation [S4].
- **Pitfall:** Unlike octahedra, the optimized position (center) can differ from the geometric center of the primitive due to lack of symmetry [S4].

### Delaunay Triangulation for Tetrahedral Representation
- **Steps:**
    1. Obtain a tetrahedral representation of the Voronoi sites via Delaunay triangulation using gDel3D to accelerate computation on the GPU [S5].
    2. Recompute the triangulation from scratch at every iteration [S5].

## Background
The concept of projecting tetrahedra for volume rendering was introduced by Kindlmann et al. in 1998 [S1, S2, unofficial][S4, S6, S7][S8, S9, S10, unofficial][S11][S12, unofficial][S14, S16][S17, S18, unofficial]. It was developed to address the limitations of ray casting on unstructured grids, offering a way to utilize standard graphics hardware for interpolation and compositing [S2, unofficial][S4, S6, S7][S12, unofficial][S14, S16]. The method has since evolved to include parallel sorting strategies [S1, unofficial][S2, unofficial], bandwidth-optimized primitives [S1, S2, unofficial], and stochastic rendering variants [S1, unofficial].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
[S2] (PDF) Image-space visibility ordering for cell projection volume ..., https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
[S3] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S4] LinPrim: Linear Primitives for Differentiable Volumetric Rendering, https://arxiv.org/abs/2501.16312, fetched 2026-10-26
[S5] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
