# Tetrahedral representation and traversal

Also called: Projected tetrahedra, cell projection, tetrahedral volume rendering

## Overview
Tetrahedral representation and traversal is a family of object-order volume rendering techniques that decompose volumetric data into tetrahedral cells to resolve topological ambiguities and enable hardware-accelerated rasterization or efficient ray traversal [S1, S2, unofficial][S3][S4, S6, S7][S8, S9, S10, unofficial][S11][S12, unofficial][S14, S16][S17, S18, unofficial]. It solves the problem of rendering unstructured or curvilinear grids by projecting 3D tetrahedra into 2D screen space (or traversing them in object space) and compositing the results in back-to-front order [S1, S2, unofficial][S4, S6][S8, S9, S10, unofficial][S11][S12, unofficial][S14, S16][S17, S18, unofficial]. Unlike ray-casting methods that sample scalar fields directly, this family relies on explicit geometric projection and subdivision [S2, unofficial][S4, S6, S7][S12, unofficial][S14, S16].

## Shared principles
The cell projection family shares the following mechanism, terms, and math:

1.  **Primitive Representation:** The volume is decomposed into convex polyhedral cells (e.g., tetrahedra, hexahedra). For regular grids (cubes or hexahedra), cells are often subdivided into tetrahedra (typically 5 or 6) to ensure planar faces and consistent topology [S2, S8, S13, S19, unofficial][S1, unofficial]. This decomposition resolves "problem cells" where rays might intersect a face twice in curvilinear grids [S4].
2.  **Projection and Subdivision:** The 3D vertices of each cell are projected to 2D screen space. The projected edges of the cell divide the image plane into polygons, where each polygon lies within the projection of a single front-facing face and a single back-facing face of the cell [S1, S2, unofficial][S4]. For general cells, an incremental method builds this subdivision by adding projected edges one by one [S1, S2, unofficial]. For curvilinear grids, standard topologies are identified to use lookup tables for subdivision [S1, S2, unofficial].
3.  **Scalar Interpolation:** Scalar values $s$ at any point within a tetrahedron are interpolated linearly from the vertex values $s_i$ using barycentric coordinates or linear interpolation along edges [S1, unofficial][S2, unofficial][S3]. The scalar field is assumed to vary linearly across the cell, and transfer functions mapping scalar to color/extinction are assumed linear [S1, S2, unofficial][S4].
4.  **Opacity Calculation:** The integrated opacity $\alpha$ for a segment of length $l$ with average differential opacity $\tau_{avg}$ is:
    $$ \alpha = 1 - \exp(-\tau_{avg} l) $$
    [S1, unofficial][S4].
5.  **Visibility and Sorting:** Correct compositing requires back-to-front ordering. This is achieved via global visibility sorts (e.g., sorting by centroid depth) or image-space merging of ray segments [S1, unofficial][S4, S7, S11][S12, unofficial][S14][S17, unofficial].
6.  **Compositing:** Fragments or ray segments are composited using the Porter-Duff over operation in back-to-front order [S1, unofficial][S2, unofficial][S6, S7][S9, S10, unofficial][S11][S12, unofficial][S14, S16][S17, S18, unofficial].

## Subtopics

### [Projected Tetrahedra](877-projected-tetrahedra.md)
Projected Tetrahedra (PT) is an object-order volume rendering technique that projects volumetric primitives, specifically tetrahedra, onto the 2D image plane to leverage hardware-accelerated rasterization. It solves the problem of rendering unstructured tetrahedral grids by decomposing each 3D cell into a set of partially transparent 2D triangles, which are then composited in back-to-front order. Unlike [Tetrahedral Volume Rendering](820-tetrahedral-volume-rendering.md), which often relies on ray casting, PT uses explicit geometric projection and scan-conversion. It differs from [GPU-based Projected Tetrahedra](7484-gpu-based-projected-tetrahedra.md) in that the original PT algorithm performs sorting and projection in software, whereas later variants offload these steps to the GPU.

### [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md)
The Marching Tetrahedrons Algorithm (MTA) is an isosurface extraction method that decomposes volumetric data cells into tetrahedra to resolve topological ambiguities inherent in cube-based methods like Marching Cubes. Unlike its siblings, MTA guarantees topological consistency by ensuring that the intersection of an isosurface with a tetrahedron is always a simple polygon, eliminating the need for complex case tables required by cubic decompositions.

### [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md)
Parallelized Accurate Sorting for Projected Tetrahedra is a view-dependent sorting algorithm designed to resolve the parallelization bottlenecks inherent in standard Projected Tetrahedra rendering. It solves the dependency and sequence limitations of neighboring tetrahedra by using logical partitioning to divide the dataset into blocks along the viewing direction, enabling the parallel sorting of unobstructed layers. Unlike global visibility sorts, this method enhances the parallelization of topological sorting, significantly reducing sorting time and improving rendering performance for large datasets.

### [Tetrahedral Strips and Fans](1389-tetrahedral-strips-and-fans.md)
Tetrahedral Strips and Fans are graphics primitives designed to address the bandwidth bottleneck between main memory and the graphics processor when rendering unstructured tetrahedral meshes. They reduce the number of vertices transmitted per tetrahedron, achieving rates approaching 1.2 vertices per tetrahedron, similar to how triangle strips optimize triangle meshes. Unlike static triangle strips, these primitives are necessary because tetrahedral decomposition is view-dependent, preventing the use of static display lists when changing viewpoints. They enable performance improvements of 72 to 85 times over triangle-based hardware approaches, making interactive volumetric rendering feasible.

### [Adaptive Delaunay Update](4923-adaptive-delaunay-update.md)
Adaptive Delaunay Update is a strategy for maintaining the topological validity of a tetrahedral mesh during optimization-based volume rendering, specifically within frameworks like Radiance Meshes. It addresses the problem of mesh degeneracy caused by vertex movement during training by periodically recomputing the Delaunay triangulation to restore the empty-circumsphere property. It sets itself apart from incremental update methods by trading continuous topological accuracy for significant GPU acceleration speed, relying on batched recomputation rather than real-time edge flipping.

### [Grid-based Traversal](7083-grid-based-traversal.md)
Grid-based Traversal is a method for rendering irregular volume data (tetrahedral meshes) at interactive rates by partitioning the volume into a hierarchical grid structure to quickly determine candidate tetrahedra. It sets itself apart from its siblings by avoiding explicit connectivity information between tetrahedra, instead relying on spatial hashing into a grid to manage the large number of primitives. Unlike [Projected Tetrahedra](877-projected-tetrahedra.md), which projects primitives to screen space, this method operates in object space using ray-tetrahedron intersection tests within grid cells.

### [Sorting-free Pre-integrated Projected Tetrahedra](7491-sorting-free-pre-integrated-projected-tetrahedra.md)
Sorting-free Pre-integrated Projected Tetrahedra (PBVR) is a volume rendering technique for irregular datasets that eliminates the need for explicit back-to-front visibility sorting. It solves the scalability bottlenecks of conventional projected tetrahedra and the visible footprint artifacts of particle-based methods by treating rendering as a stochastic process where fragments are rendered with a probability equal to their opacity. Unlike [Projected Tetrahedra](877-projected-tetrahedra.md), which requires sorting, and [Particle-based Volume Rendering](877-projected-tetrahedra.md) (PBVR in the context of Anderson et al.), which suffers from low quality at close range, PBVR achieves semi-transparency through repeated stochastic sampling without alpha-blending.

### [Parallel Projected Tetrahedra with KD-Tree Partitioning](8767-parallel-projected-tetrahedra-with-kd-tree-partitioning.md)
This method enables parallel volume rendering of large unstructured tetrahedral grids by partitioning the data using a kd-tree and applying a traditional sort-last compositing algorithm. It renders unstructured grids natively without sampling into a regular grid, avoiding the oversampling/undersampling issues and parallel distribution complexities associated with grid-based sampling approaches. Unlike other parallel rendering approaches that require continuous cell transfer, this method requires no further cell transfer once distributed; only image data is communicated between processes for final composition.

### [Tetrahedral Mesh Connectivity Encoding](8957-tetrahedral-mesh-connectivity-encoding.md)
Tetrahedral Mesh Connectivity Encoding, specifically the Basic Cut-Border Machine (CBM), is a region-growing compression scheme that encodes the connectivity of a tetrahedral mesh by growing a region bounded by a cut-border. It solves the problem of efficiently storing and transmitting the topological structure of tetrahedral volumes for rendering and processing, ensuring unique identification of vertices across multi-resolution segments. It sets itself apart from sibling methods like [Projected Tetrahedra](877-projected-tetrahedra.md) by explicitly encoding the traversal order and adjacency via opcodes rather than relying on implicit grid structures or global sorting, and differs from [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md) by focusing on connectivity compression rather than isosurface extraction.

### Screen Space Area Calculation
- **Definition:** Calculates the projected area of a tetrahedron in normalized screen space to determine refinement criteria or culling [S1, unofficial]. Uses cross products to compute wedge areas, which are twice the triangle area, then divides by four [S1, unofficial].
- **Step:** Project tetrahedron vertices $T, L, R, B$ to screen space using camera projection: $T \leftarrow \text{camera.projectPoint}(T)$, $L \leftarrow \text{camera.projectPoint}(L)$, $R \leftarrow \text{camera.projectPoint}(R)$, $B \leftarrow \text{camera.projectPoint}(B)$ [S1, unofficial]. Calculate area sum using cross products of vertex vectors: $\text{area} \leftarrow |(L-T) \times (R-T)| + |(R-T) \times (B-T)|$. Add remaining terms: $\text{area} \leftarrow \text{area} + |(B-T) \times (L-T)| + |(L-B) \times (R-B)|$ [S1, unofficial]. Return final area divided by 4: $\text{return area} / 4$ [S1, unofficial].
- **Pitfall:** Triangles cover the total area twice in projection, and cross products yield wedge areas (twice triangle area), necessitating division by 4 [S1, unofficial].

### Early Tetrahedron Termination
- **Definition:** A technique that reduces the amount of processed tetrahedra by leveraging a summed-area table for accumulating and comparing opacity of individual tetrahedra on the rendering phase, after they have been sorted into unobstructed layers [S2, unofficial].
- **Number:** Gain of more than 10% in rendering performance for datasets with more than a million tetrahedra [S2, unofficial].

### Point Representation via Tetrahedra
- **Definition:** Representing data points as small tetrahedra to fit them into the triangular mesh framework for rendering [S3].
- **Parameter:** `size`: specifies the size of the point tetrahedra relative to the data ranges [S3].

### LinPrim Tetrahedron Representation
- **Definition:** A scene representation using transparent tetrahedra bounded by triangular faces with homogeneous density [S4].
- **Step:** Define four basis vectors describing directions from the center on which corners are located, equally spaced to represent regular tetrahedra [S4]. Optimize the distance between the center and each corner independently [S4].
- **Parameter:** Tetrahedra memory requirement: 12 floats per primitive (omitting color), due to absence of symmetry in the formulation [S4].
- **Pitfall:** Unlike octahedra, the optimized position (center) can differ from the geometric center of the primitive due to lack of symmetry [S4].

### Delaunay Triangulation for Tetrahedral Representation
- **Step:** Obtain a tetrahedral representation of the Voronoi sites via Delaunay triangulation using gDel3D [5] to accelerate computation on the GPU [S5]. Recompute the triangulation from scratch at every iteration [S5].

## Background
The tetrahedral representation for volume rendering originated from the need to handle unstructured grids and curvilinear volumes where standard hexahedral cell projection suffers from non-planar faces and ambiguous ray intersections [S4]. The decomposition of cubes into tetrahedra (typically 5 or 6) was established to resolve these topological issues [S2, S8, S13, S19, unofficial][S1, unofficial]. The Projected Tetrahedra method was introduced to leverage graphics hardware acceleration by rasterizing projected tetrahedra [S1, S2, unofficial][S3][S4, S6, S7][S8, S9, S10, unofficial][S11][S12, unofficial][S14, S16][S17, S18, unofficial]. Subsequent developments focused on parallelization [S1, unofficial][S2, unofficial], bandwidth optimization via strips and fans [S1, S2, unofficial], and integration with differentiable rendering frameworks [S1, unofficial][S4, S5].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
[S2] (PDF) Image-space visibility ordering for cell projection volume ..., https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
[S3] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S4] LinPrim: Linear Primitives for Differentiable Volumetric Rendering, https://arxiv.org/abs/2501.16312, fetched 2026-10-26
[S5] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
