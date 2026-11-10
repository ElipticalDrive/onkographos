# Tetrahedral representation and traversal

Also called: Projected Tetrahedra, Cell Projection with Tetrahedra

## Overview
This family covers methods that decompose volumetric data into tetrahedral primitives, project them into screen space (or traverse them in object space), and rasterize or ray-march the resulting geometry to render the volume. It solves the problem of efficiently rendering unstructured or irregular grids (e.g., CFD data) by leveraging hardware-accelerated rasterization or optimized spatial traversal, avoiding the sampling artifacts and performance bottlenecks of regular grid marching [S3][S5]. Unlike [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md), which extracts isosurfaces, this family renders the volume directly via translucent polygons or ray intersections [S11, unofficial].

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
Projected Tetrahedra (PT) is an object-order volume rendering technique that decomposes volumetric data into tetrahedral primitives, projects them onto the 2D image plane, and rasterizes the resulting polygons to accumulate color and opacity. It solves the problem of rendering unstructured or irregular grids (e.g., CFD data) efficiently by leveraging hardware-accelerated rasterization instead of explicit per-pixel ray marching. Unlike [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md), which extracts isosurfaces, PT renders the volume directly via translucent polygons. It differs from [Tetrahedral Strips and Fans](1389-tetrahedral-strips-and-fans.md) by focusing on the general classification and decomposition of arbitrary tetrahedra rather than optimized connectivity.

### [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md)
The Marching Tetrahedrons Algorithm (MTA) is an iso-surface extraction method that decomposes volumetric grid cells (typically cubes) into tetrahedra to resolve topological ambiguities inherent in cube-based methods like Marching Cubes. It sets itself apart by ensuring planar faces and consistent topology through tetrahedral subdivision, though it introduces directional ambiguities due to the non-symmetric nature of cube-to-tetrahedron decomposition.

### [Adaptive Delaunay Update](4923-adaptive-delaunay-update.md)
Adaptive Delaunay Update is a strategy for maintaining the topological validity of a tetrahedral mesh during optimization-based volume rendering, specifically within frameworks like Radiance Meshes. It addresses the problem of mesh degeneracy caused by vertex movement during training by periodically recomputing the Delaunay triangulation to restore the empty-circumsphere property. It sets itself apart from incremental update methods by trading continuous topological accuracy for significant GPU acceleration speed, relying on batched recomputation rather than real-time edge flipping.

### [Grid-based Traversal](7083-grid-based-traversal.md)
Grid-based Traversal is a method for rendering irregular volume data (tetrahedral meshes) at interactive rates by partitioning the volume into a hierarchical grid structure to quickly determine candidate tetrahedra. It sets itself apart from its siblings by avoiding explicit connectivity information between tetrahedra, instead relying on spatial hashing into a grid to manage the large number of primitives. Unlike [Projected Tetrahedra](877-projected-tetrahedra.md), which projects primitives to screen space, this method operates in object space using ray-tetrahedron intersection tests within grid cells.

### [Parallel Projected Tetrahedra with KD-Tree Partitioning](8767-parallel-projected-tetrahedra-with-kd-tree-partitioning.md)
This method enables parallel volume rendering of large unstructured tetrahedral grids by partitioning the data using a kd-tree and applying a traditional sort-last compositing algorithm. It renders unstructured grids natively without sampling into a regular grid, avoiding the oversampling/undersampling issues and parallel distribution complexities associated with grid-based sampling approaches. Unlike other parallel rendering approaches that require continuous cell transfer, this method requires no further cell transfer once distributed; only image data is communicated between processes for final composition.

### [Tetrahedral Mesh Connectivity Encoding](8957-tetrahedral-mesh-connectivity-encoding.md)
Tetrahedral Mesh Connectivity Encoding, specifically the Basic Cut-Border Machine (CBM), is a region-growing compression scheme that encodes the connectivity of a tetrahedral mesh by traversing it and generating a stream of opcodes. It solves the problem of efficiently storing and transmitting the topological structure of tetrahedral meshes, which is required for object-order volume rendering methods like Projected Tetrahedra. It sets itself apart from sibling methods like [Grid-based Traversal](7083-grid-based-traversal.md) by using a dynamic cut-border boundary and a specific set of 11 opcodes to describe vertex positions relative to that boundary, rather than relying on static grid indices or global sorting.

### [Delaunay Triangulation for Tetrahedral Representation](9905-delaunay-triangulation-for-tetrahedral-representation.md)
This method constructs a tetrahedral mesh from arbitrarily distributed 3D points by enforcing the Delaunay property: no point lies within the circumsphere of any tetrahedron in the mesh. It serves as a robust alternative to silhouette-based methods like Visual Hull Methods (VMC), which suffer from threshold sensitivity and discontinuous surfaces. Unlike its sibling [Local Convex Hull (LCH)](11862-local-convex-hull-lch.md), which uses convex hull algorithms for local surface construction, this method relies on global Delaunay triangulation to ensure topological consistency and connectivity. It is primarily used in surface reconstruction and volumetric tessellation where arbitrary point distributions must be converted into a valid tetrahedral representation.

### [TetraSDF](11632-tetrasdf.md)
TetraSDF is an analytic isosurface extraction framework for neural Signed Distance Functions (SDFs) that enables exact zero-level set extraction by preserving a global Continuous Piecewise Affine (CPWA) structure. Unlike sampling-based methods like Marching Cubes, which introduce discretization error, or plain ReLU MLPs, which lack high-frequency expressiveness, TetraSDF composes a ReLU MLP with a multi-resolution tetrahedral positional encoder. This composition ensures the entire network remains CPWA, allowing the zero-level set to be extracted analytically via edge subdivision without marching artifacts. It is distinguished by its use of barycentric interpolation to maintain affine continuity across tetrahedral cells and an analytic input preconditioner to mitigate directional bias from the encoder's grid geometry.

### Screen Space Area Calculation
- **Definition:** Calculates the projected area of a tetrahedron in normalized screen space to determine refinement criteria or culling. Uses cross products to compute wedge areas, which are twice the triangle area, then divides by four [S1, unofficial].
- **Step:** Project tetrahedron vertices $T, L, R, B$ to screen space using camera projection: $T \leftarrow \text{camera.projectPoint}(T)$, $L \leftarrow \text{camera.projectPoint}(L)$, $R \leftarrow \text{camera.projectPoint}(R)$, $B \leftarrow \text{camera.projectPoint}(B)$. Calculate area sum using cross products of vertex vectors: $\text{area} \leftarrow |(L-T)\times(R-T)| + |(R-T)\times(B-T)|$. Add remaining terms: $\text{area} \leftarrow \text{area} + |(B-T)\times(L-T)| + |(L-B)\times(R-B)|$. Return final area divided by 4: $\text{return area} / 4$ [S1, unofficial].
- **Pitfall:** Triangles cover the total area twice in projection, and cross products yield wedge areas (twice triangle area), necessitating division by 4 [S1, unofficial].

### Early Tetrahedron Termination
- **Definition:** A technique that reduces the amount of processed tetrahedra by leveraging a summed-area table for accumulating and comparing opacity of individual tetrahedra on the rendering phase, after they have been sorted into unobstructed layers [S2, unofficial].
- **Performance:** Gain of more than 10% in rendering performance for datasets with more than a million tetrahedra [S2, unofficial].

### Point Representation via Tetrahedra
- **Definition:** Representing data points as small tetrahedra to fit them into the triangular mesh framework for rendering [S3].
- **Parameter:** `size`: specifies the size of the point tetrahedra relative to the data ranges [S3].

### Neighbor Configuration Lookup Tables
- **Definition:** Lookup tables defining neighbor configurations for the tetrahedral grid, divided into face, edge, and vertex neighbors, used for analytic traversal [S4].
- **Formula:** Edge definitions: $e_0 = \{v_0, v_1\}, e_1 = \{v_0, v_2\}, e_2 = \{v_0, v_3\}, e_3 = \{v_1, v_2\}, e_4 = \{v_1, v_3\}, e_5 = \{v_2, v_3\}$ [S4].
- **Layout:** Tetrahedron $T$ has ordered vertices $(v_0, v_1, v_2, v_3)$. Face index $f \in \{0, 1, 2, 3\}$ refers to the face opposite vertex $v_f$. Edge index $e \in \{0, \dots, 5\}$ refers to specific vertex pairs. Face neighbor lookup table (Table A6): Input is tetra offset $t \in \{0, \dots, 5\}$ and local face index $f \in \{0, \dots, 3\}$. Output lists neighbor index $i \ge 1$, anchor displacement $\Delta_i = (\Delta_{x,i}, \Delta_{y,i}, \Delta_{z,i}) \in \mathbb{Z}^3$, and neighbor tetra offset $t_i$. Self configuration ($i=0$) is omitted. Edge neighbor lookup table (Table A7): Input is edge $(t, e)$. Output lists neighbor index $i \ge 1$, anchor displacement $\Delta_i$, and neighbor tetra offset $t_i$. Self configuration is omitted [S4].

## Background
The tetrahedral representation originated as a solution to the topological ambiguities of cube-based isosurface extraction (Marching Cubes) [S1, unofficial]. By decomposing hexahedral cells into tetrahedra, methods like Marching Tetrahedrons ensured planar faces and consistent topology [S1, unofficial]. In volume rendering, the Projected Tetrahedra method was introduced to handle unstructured grids efficiently by projecting these primitives to screen space, leveraging GPU rasterization [S2, unofficial][S3][S7]. Subsequent developments focused on parallelization [S1, unofficial], connectivity encoding [S1, unofficial], and integration with neural networks (TetraSDF) [S1, unofficial].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
[S2] Image-space visibility ordering for cell projection volume rendering of unstructured data, https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
[S3] Computing and Displaying Isosurfaces in R, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S4] TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, https://arxiv.org/abs/2511.16273v4, fetched 2026-10-26
