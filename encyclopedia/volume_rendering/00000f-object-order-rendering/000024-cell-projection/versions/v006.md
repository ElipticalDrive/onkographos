# Cell projection
Also called: Projected Tetrahedra, Polyhedron Projection

## Overview
Cell projection is a direct volume rendering technique that decomposes volumetric data into geometric primitives (typically convex polyhedra or tetrahedra), projects these cells onto the 2D image plane, and composites the resulting polygons in back-to-front order [S1][S4, book]. It solves the problem of rendering unstructured, curvilinear, or irregular grids without resampling, offering higher accuracy than slice-based methods by treating the volume as a set of discrete geometric objects [S4, book]. It sets itself apart from [Splatting](23-splatting.md) by using exact geometric projection rather than approximate splats, and from [Texture slicing](25-texture-slicing.md) by avoiding the interpolation artifacts and data resampling required by slice-based approaches [S4, book].

## Shared principles
The cell projection family shares the following mechanism, terms, and math:

1.  **Primitive Representation:** The volume is decomposed into convex polyhedral cells. For regular grids (cubes), cells are often subdivided into tetrahedra (typically 5 or 6) to ensure planar faces and consistent topology [S2, S8, S13, S19, unofficial][S1]. Non-planar faces in curvilinear grids are subdivided into tetrahedra with planar faces to resolve "problem cells" where rays might intersect a face twice [S4, book].
2.  **Projection and Subdivision:** The 3D vertices of each cell are projected to 2D screen space. The projected edges of a single cell divide the image plane into polygons. Each polygon lies within the projection of a single front-facing face and a single back-facing face of the cell [S1][S4, book]. For general cells, this subdivision is built incrementally by adding projected edges one by one, starting with one infinite region [S1, S2].
3.  **Segment Length ($l$):** For each polygon, the length $l$ of the viewing ray segment within the cell is calculated.
    *   At "thin" vertices (where the segment length is zero), $l=0$ [S1, S2].
    *   At "thick" vertices, $l$ is calculated geometrically by finding the segment endpoints on the front and back faces that project to the vertex [S1, S2].
    *   For orthogonal projection, $l$ varies linearly across the polygon and can be interpolated by hardware [S1, S2].
4.  **Opacity Calculation:** The integrated opacity $\alpha$ for a segment of length $l$ with average differential opacity $\tau_{avg}$ is:
    $$ \alpha = 1 - \exp(-\tau_{avg} l) $$
    [S1][S4, book].
5.  **Visibility Sorting:** Correct compositing requires back-to-front ordering of cells. This is achieved via global visibility sorts (e.g., topological sorts for convex meshes) or image-space merging of ray segments [S1][S4, S7, S11][S12, unofficial][S14][S17, unofficial].
6.  **Compositing:** Polygons or ray segments are composited using the Porter-Duff over operation in back-to-front order [S1][S2][S6, S7][S9, S10, unofficial][S11][S12, unofficial][S14, S16][S17, S18, unofficial].

## Subtopics

### [Parallel Cell Projection](875-parallel-cell-projection.md)
Parallel Cell Projection distributes the computationally expensive polygon generation and compositing phases of cell projection across multiple processors or threads to accelerate volume rendering on unstructured or curvilinear grids. It solves the scalability bottleneck of single-threaded cell projection by parallelizing the projection of cells into screen-space polygons and the subsequent merging of ray segments. It sets itself apart from [Topological Sort for Cell Projection](999-topological-sort-for-cell-projection.md) by focusing on distributed-memory or multi-threaded execution of the projection and composition steps rather than solely on object-space sorting algorithms, and from [GPU-based Cell Projection for Structured Data](1864-gpu-based-cell-projection-for-structured-data.md) by often relying on software-based polygon generation and CPU-side compositing in its classic implementations.

### [Cell Projection with Texture Mapping for Opacity](993-cell-projection-with-texture-mapping-for-opacity.md)
This method accelerates the cell projection volume rendering technique by using hardware texture mapping to compute the exponential transparency term $\exp(-\int \tau ds)$ per pixel via table lookup, thereby avoiding per-pixel exponential calculations while preventing Mach bands caused by linear interpolation of transparency. It sets itself apart from the base [Cell projection](26-cell-projection.md) method by replacing the naive linear interpolation of opacity with a 1D or 2D texture lookup that correctly models the non-linear attenuation of light through the volume.

### [Topological Sort for Cell Projection](999-topological-sort-for-cell-projection.md)
Topological Sort for Cell Projection (MPVO) is an object-space algorithm that computes a correct back-to-front visibility ordering of convex polyhedral cells in a convex mesh by constructing a directed adjacency graph based on face-sharing topology and viewpoint orientation. It sets itself apart from its siblings, such as [XMPVO](1363-xmpvo.md) and [SXMPVO](944-exact-visibility-ordering-for-cell-projection-volume-renderi.md), by relying exclusively on local topological relations across shared faces rather than global geometric ray-shooting queries or image-space rasterization, provided the mesh is convex and acyclic.

### [GPU-based Cell Projection for Structured Data](1864-gpu-based-cell-projection-for-structured-data.md)
This method is an interactive volume rendering technique for large, regular (structured) volumetric data sets on GPUs. It solves the performance bottleneck of general Projected Tetrahedra (PT) by eliminating redundant projection and visibility sorting computations through the exploitation of data regularity and orthographic projection. Unlike general PT, which requires expensive cell sorting, this method achieves constant-time traversal order and reuses projection parameters for all voxels, dedicating over 60% of processing time to the rendering pipeline itself. It consumes less memory than Hardware-Based Ray Casting (HARC) while achieving high rendering speeds.

### [Cell Projection with Scan-line Conversion](1968-cell-projection-with-scan-line-conversion.md)
Cell Projection with Scan-line Conversion is a software-based object-order rendering algorithm for direct volume rendering on irregular, unstructured, and multiple grids. It solves the problem of efficiently rasterizing volumetric primitives by leveraging spatial coherence between adjacent pixels and scan lines, avoiding the computational overhead of per-pixel ray casting or the artifacts of hardware-accelerated Gouraud shading. Unlike its sibling [Parallel Cell Projection](875-parallel-cell-projection.md), which may distribute work differently, this method explicitly uses Y-bucket sorting and scan-line generation to process cells completely before moving to the next, progressively displaying the view from front-most to distant pixels. It sets itself apart from [Cell Projection with Texture Mapping for Opacity](993-cell-projection-with-texture-mapping-for-opacity.md) by performing scan conversion and compositing in software, thereby eliminating the need for special graphics hardware and associated artifacts.

### [Cell-Projection Rendering](4279-cell-projection-rendering.md)
Cell-Projection Rendering is a parallel volume rendering technique for unstructured data that projects mesh cells into image space, generating ray segments that are merged in image space. Unlike ray-casting, it does not require pre-sorting cells in depth order, offering greater flexibility in data distribution and higher accuracy than slice- or vertex-projected methods. It is specifically designed for distributed-memory systems where computational load and communication overhead must be balanced through asynchronous task management.

### [Tetrahedral representation and traversal](4396-tetrahedral-representation-and-traversal.md)
Tetrahedral representation and traversal decomposes volumetric data into tetrahedral cells to leverage their geometric simplicity for rendering, isosurface extraction, or spatial acceleration. Unlike hexahedral methods that may suffer from topological ambiguities or require complex connectivity handling, tetrahedra provide a consistent, convex primitive that simplifies intersection tests and sorting. This family encompasses explicit geometric projection for volume rendering [877-projected-tetrahedra.md], topological isosurface extraction [940-marching-tetrahedrons-algorithm.md], and spatial traversal structures for ray tracing [7083-grid-based-traversal.md].

### [3D Cube Projection and Rasterization](6678-3d-cube-projection-and-rasterization.md)
A software rendering technique that transforms the eight corners of a 3D voxel cube using sequential translation and rotation matrices, applies perspective projection to the vertices of each face, and rasterizes visible faces as 2D polygons. It differs from hardware rasterization by performing all depth testing, culling, and pixel writing in software, and from standard raycasting by explicitly drawing polygonal faces rather than casting rays per pixel.

### [Incoherent Projection](7541-incoherent-projection.md)
Incoherent projection is a cell projection method for general convex hexahedra that renders the 2D projection of each cell as an arrangement of polygons by analyzing each cell individually in screen space, thereby avoiding case analysis for the high number of projection topologies. It builds on [Cell projection](26-cell-projection.md) by extending the coherent projection technique for rectilinear grids to irregular hexahedra, where different cells may fall into different nondegenerate or degenerate projection topologies. Unlike coherent projection, which assumes all cells share the same topology, incoherent projection handles the complexity of general grids by processing each cell's projection geometry independently.

### Pre-integrated classification for cell projection
A method to determine color and opacity contribution by applying pre-integrated classification, using scalar values on front and back faces and thickness as texture coordinates for a 3D texture map storing volume ray integral values [S5]. It uses the scalar value on the front face $s_f$, the scalar value on the back face $s_b$, and the thickness $l$ resulting from scan converting the tetrahedron as texture coordinates for a 3D texture map storing the values of the volume ray integral in dependency of $l$, $s_f$, and $s_b$ [S5].

## Background
Cell projection was developed at Lawrence Livermore National Laboratory by Max, Williams, Silva, and Cook [S1, S2]. It is one of the four main paradigms of volume rendering, alongside raycasting, splatting, and shear-warp [S3, unofficial]. The method was designed to handle curvilinear and unstructured grids directly, avoiding the resampling required by grid-based methods [S1][S4, book]. It has been parallelized for distributed-memory systems and optimized for GPU execution on structured data [S1][S4, book].

## Sources
[S1] Volume rendering for curvilinear and unstructured grids, https://doi.org/10.1109/cgi.2003.1214468, fetched 2026-10-24
[S2] Volume Rendering for Curvilinear and Unstructured Grids, https://www.sci.utah.edu/~csilva/papers/cgi2003.pdf, fetched 2026-10-27
[S3] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S4] Cell projection of meshes with non-planar faces, https://digital.library.unt.edu/ark:/67531/metadc1408947/?q=projected tetrahedra, fetched 2026-10-28 (book)
[S5] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
