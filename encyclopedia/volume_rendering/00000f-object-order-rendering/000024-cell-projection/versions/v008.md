# Cell projection

Also called: Projected Tetrahedra, Polyhedron Projection

## Overview
Cell projection is a direct volume rendering technique that treats volumetric data as a set of geometric primitives (convex polyhedral cells) which are projected onto the 2D image plane, rasterized, and composited in back-to-front order [S1][S4, book]. It solves the problem of rendering unstructured, curvilinear, or irregular grids where ray-casting is computationally expensive or difficult to parallelize, by leveraging standard graphics hardware for scan-conversion [S3, unofficial][S4, book]. It sets itself apart from [Splatting](23-splatting.md) by using exact geometric primitives rather than approximate splats, and from [Texture slicing](25-texture-slicing.md) by projecting the volume elements themselves rather than slicing the volume with planes [S4, book].

## Shared principles
The cell projection family shares the following mechanism, terms, and math:

1. **Primitive Representation:** The volume is decomposed into convex polyhedral cells (e.g., tetrahedra, hexahedra) [S1][S4, book]. For grids with non-planar faces, cells are subdivided into tetrahedra to ensure planar faces and consistent topology [S1][S4, book]. This decomposition resolves "problem cells" where rays might intersect a face twice in curvilinear grids [S4, book].
2. **Projection and Subdivision:** The 3D vertices of each cell are projected to 2D screen space [S4, book]. The projected edges of the cell divide the image plane into polygons, where each polygon lies within the projection of a single front-facing face and a single back-facing face of the cell [S1][S2][S4, book]. For general cells, an incremental method builds this subdivision by adding projected edges one by one [S1][S2]. For curvilinear grids, standard topologies are identified to use lookup tables for subdivision [S1][S2].
3. **Scalar Interpolation:** Scalar values $s$ at any point within a tetrahedron are interpolated linearly from the vertex values $s_i$ using barycentric coordinates or linear interpolation along edges [S1][S2]. The scalar field is assumed to vary linearly across the cell, and transfer functions mapping scalar to color/extinction are assumed linear [S1][S2][S4, book].
4. **Opacity Calculation:** The integrated opacity $\alpha$ for a segment of length $l$ with average differential opacity $\tau_{avg}$ is:
$$ \alpha = 1 - \exp(-\tau_{avg} l) $$
[S1][S4, book].
5. **Visibility and Sorting:** Correct compositing requires back-to-front ordering [S1]. This is achieved via global visibility sorts or image-space merging of ray segments [S1][S5, unofficial].
6. **Compositing:** Fragments or ray segments are composited in back-to-front order using an exponential opacity model [S1][S2].

## Subtopics

### [Cell Projection with Texture Mapping for Opacity](993-cell-projection-with-texture-mapping-for-opacity.md)
Accelerates rendering by using hardware texture mapping to compute the exponential transparency term per pixel via table lookup, avoiding per-pixel exponential calculations while preventing Mach bands. Sets itself apart by replacing linear interpolation of integrated opacity with a 2D texture lookup that correctly models non-linear exponential decay.

### [Topological Sort for Cell Projection](999-topological-sort-for-cell-projection.md)
An object-space algorithm that computes a correct back-to-front visibility ordering of convex polyhedral cells by constructing a directed adjacency graph based on face-sharing topology and viewpoint orientation. Sets itself apart by relying on local ordering relations between cells sharing a common face to define a partial order, converted to a total order via topological sort.

### [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md)
A view-dependent sorting algorithm for tetrahedral datasets that uses logical partitioning to enable parallel processing, solving dependency and sequence limitations of neighboring tetrahedra. Sets itself apart by replacing global topological sorting with a block-based logical partitioning scheme that allows independent parallel sorting of unobstructed layers.

### [Two-Pass GPU Approach](1339-two-pass-gpu-approach.md)
An efficient ray-casting algorithm for tetrahedral meshes implemented on consumer graphics hardware using fragment programs, pre-integration, and multiple rendering passes. Sets itself apart by offloading geometric projection and classification logic to the GPU in a strict two-stage pipeline to minimize CPU-GPU data transfer.

### [GPU-based Cell Projection for Structured Data](1864-gpu-based-cell-projection-for-structured-data.md)
An interactive volume rendering technique for large, regular volumetric data sets on GPUs. Sets itself apart from general Projected Tetrahedra by eliminating redundant projection and visibility sorting computations through the exploitation of data regularity and orthographic projection, achieving constant-time traversal order.

### [Cell Projection with Scan-line Conversion](1968-cell-projection-with-scan-line-conversion.md)
A software-based object-order volume rendering algorithm that projects volumetric primitives into screen space and rasterizes them using scan-line conversion to leverage spatial coherence. Sets itself apart by processing cells completely before moving to the next, progressively displaying the view from front-most to distant pixels without relying on expensive graphics hardware.

### [Tetrahedral representation and traversal](4396-tetrahedral-representation-and-traversal.md)
Covers methods that decompose volumetric data into tetrahedral primitives to enable object-order rendering or precise isosurface extraction. Sets itself apart by resolving topological ambiguities and non-planar face issues inherent in hexahedral or cubic grid representations, ensuring all cells have planar faces and consistent connectivity.

### [Exact AMR Rendering](4399-exact-amr-rendering.md)
Treats Adaptive Mesh Refinement (AMR) gas cells as piecewise-constant geometric cubes rather than smoothing them as SPH kernels. Sets itself apart by providing exact pixel-averaged cross-sections for slices, projections, and perspective images, ensuring mass conservation to round-off.

### [Point-based Projected Tetrahedra](5527-point-based-projected-tetrahedra.md)
A technique for rendering large, unstructured tetrahedral meshes by approximating each tetrahedron with a single point primitive. Sets itself apart by decoupling footprint generation and reshaping from compositing, using hardware shaders to reshape point footprints and composite fragments without processing connectivity information.

### [3D Cube Projection and Rasterization](6678-3d-cube-projection-and-rasterization.md)
A software rendering technique that transforms the eight corners of a 3D voxel cube using sequential translation and rotation matrices, applies perspective projection to the vertices of each face, and rasterizes visible faces as 2D polygons. Sets itself apart by performing all depth testing and pixel writing in software rather than relying on hardware rasterization pipelines.

### [Sorting-free Pre-integrated Projected Tetrahedra](7491-sorting-free-pre-integrated-projected-tetrahedra.md)
A sorting-free volume rendering technique for irregular volume datasets that employs projected tetrahedra with pre-integration. Sets itself apart by treating volume rendering as a stochastic process where sampling points occur along a viewing ray with probability equal to opacity, eliminating the need for visibility sorting of tetrahedral cells.

### [Incoherent Projection](7541-incoherent-projection.md)
A cell projection method for general convex hexahedra that renders the 2D projection of each cell as an arrangement of polygons by analyzing each cell individually in screen space. Sets itself apart from coherent projection by avoiding case analysis for the high number of projection topologies, handling irregular hexahedra where different cells fall into different topologies.

### [Cell-Projection Volume Rendering](12721-cell-projection-volume-rendering.md)
A distributed-memory object-order volume rendering algorithm for unstructured grids that projects mesh cells into image space to generate ray segments, which are then merged in image space. Sets itself apart from synchronous cell projection by allowing asynchronous communication and task management, avoiding the need for global depth-sorting of cells.

#### Screen Space Area Calculation
Calculates the projected area of a tetrahedron in normalized screen space to determine refinement criteria or culling [S6, unofficial]. Uses cross products to compute wedge areas, which are twice the triangle area, then divides by four [S6, unofficial]. Triangles cover the total area twice in projection, and cross products yield wedge areas, necessitating division by four [S6, unofficial].

#### Pre-integrated classification for cell projection
Determines color and opacity contribution by applying pre-integrated classification, using scalar values on front and back faces and thickness as texture coordinates for a 3D texture map storing volume ray integral values [S7]. The texture coordinates are the scalar value on the front face $s_f$, the scalar value on the back face $s_b$, and the thickness $l$ resulting from scan converting the tetrahedron [S7].

## Background
Cell projection was developed at Lawrence Livermore National Laboratory by Max, Williams, Silva, and Cook [S1][S2]. It is listed among the main paradigms of volume rendering, alongside raycasting, splatting, and shear-warp [S3, unofficial]. The method was designed to handle curvilinear and unstructured grids directly, avoiding the need for resampling [S1][S4, book]. Early implementations addressed the issue of non-planar faces in curvilinear grids by subdividing problem cells into tetrahedra with planar faces [S4, book]. The technique has been parallelized for distributed-memory systems and optimized for GPU execution on structured data [S1][S2][S4, book].

## Sources
[S1] Volume rendering for curvilinear and unstructured grids, https://doi.org/10.1109/cgi.2003.1214468, fetched 2026-10-24
[S2] Volume Rendering for Curvilinear and Unstructured Grids, https://www.sci.utah.edu/~csilva/papers/cgi2003.pdf, fetched 2026-10-27
[S3] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S4] Cell projection of meshes with non-planar faces, https://digital.library.unt.edu/ark:/67531/metadc1408947/?q=projected tetrahedra, fetched 2026-10-28 (book)
[S5] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29 (unofficial)
[S6] LinPrim: Linear Primitives for Differentiable Volumetric Rendering, https://arxiv.org/abs/2501.16312, fetched 2026-10-26 (unofficial)
