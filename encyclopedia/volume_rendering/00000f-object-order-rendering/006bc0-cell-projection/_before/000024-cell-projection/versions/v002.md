# Cell projection
Also called: Projected Polyhedra, Projected Tetrahedra (when tetrahedra are the primitive)

## Overview
Cell projection is a direct volume rendering technique that projects the geometric boundaries of volumetric cells (polyhedra) onto the 2D image plane, decomposing the projection into polygons that are rasterized and composited in back-to-front order [S1][S4, book]. It solves the problem of rendering unstructured, curvilinear, or irregular grids without resampling, distinguishing itself from ray-casting by operating in object space and from splatting by using exact geometric projections of cell boundaries [S3, unofficial][S4, book]. Unlike texture slicing, it handles arbitrary cell shapes directly, though it requires a visibility sort to ensure correct transparency compositing [S1].

## Shared principles
The family shares the fundamental mechanism of treating volumetric data as a set of geometric primitives that are transformed and rasterized [S1][S3, unofficial][S4, book].

1.  **Primitive Representation:** The volume is decomposed into primitives. These may be explicit geometric shapes (cubes, cells, splats) or implicit functions sampled via texture hardware [S1][S3, unofficial][S4, book].
2.  **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp) [S2][S3, unofficial][S4, book].
3.  **Rasterization:** The transformed primitives are projected onto the 2D image plane. This step leverages hardware-accelerated scan-conversion, depth testing, and blending [S1, S2][S3, unofficial][S4, book].
4.  **Compositing:** Contributions are accumulated to form the final image. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly [S1, S2, S3, S7, book].
5.  **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity [S1, S2][S3, unofficial].

## Subtopics

### [Parallel Cell Projection](875-parallel-cell-projection.md)
A distributed-memory strategy that parallelizes the object-order cell projection method by decoupling projection and composition phases to hide communication overheads.

### [Cell Projection with Texture Mapping for Opacity](993-cell-projection-with-texture-mapping-for-opacity.md)
Accelerates rendering by using hardware texture mapping to compute the exponential transparency term per pixel, avoiding Mach bands caused by linear interpolation of opacity.

### [Topological Sort for Cell Projection](999-topological-sort-for-cell-projection.md)
An object-order visibility sorting algorithm (MPVO) that computes a back-to-front rendering order for cells in a convex mesh using topological connectivity and face orientation.

### [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md)
A view-dependent sorting algorithm that resolves parallelization bottlenecks in projected tetrahedra by employing logical partitioning into blocks along the viewing direction.

### [GPU-based Cell Projection for Structured Data](1864-gpu-based-cell-projection-for-structured-data.md)
A GPU-accelerated implementation of Projected Tetrahedra optimized for structured data by exploiting geometric regularity to eliminate redundant sorting computations.

### [Cell Projection with Scan-line Conversion](1968-cell-projection-with-scan-line-conversion.md)
A software-based method for irregular grids that uses Y-bucket sorting and scanline generation to leverage spatial coherence, avoiding expensive graphics hardware.

### [Cell-Projection Rendering](4279-cell-projection-rendering.md)
A parallel algorithm for unstructured data that projects mesh cells into image space to generate ray segments, which are then merged in image space, avoiding pre-sorting.

### [Tetrahedral representation and traversal](4396-tetrahedral-representation-and-traversal.md)
Methods that decompose volumetric data into tetrahedra to facilitate rendering, resolving "problem faces" (non-planar quadrilaterals) and handling unstructured grids.

### [Sorting-free Pre-integrated Projected Tetrahedra](7491-sorting-free-pre-integrated-projected-tetrahedra.md)
A stochastic volume rendering technique (PBVR) that eliminates visibility sorting by treating rendering as a probabilistic sampling process based on opacity.

### [Incoherent Projection](7541-incoherent-projection.md)
A method for general convex hexahedra that renders the 2D projection of each cell individually in screen space using a sweep-line technique, avoiding case analysis for projection topologies.

### Pre-integrated classification for cell projection
A method using scalar values on front and back faces and thickness as texture coordinates for a 3D texture map storing volume ray integral values [S5].

### CellFast
A software algorithm that moves computations for order-independent transparency and volume scan conversion from CPU to graphics hardware, eliminating the need for a software sorting pass [S6, unofficial].

## Background
Cell projection was developed at Lawrence Livermore National Laboratory by Peter Max, Claudio Silva, Richard Cook, and Peter Williams [S1, S2]. It is one of the four main paradigms of volume rendering, alongside ray-casting, splatting, and shear-warp [S3, unofficial]. The method was designed to handle curvilinear and unstructured grids directly, avoiding the need for resampling [S1][S4, book]. It addresses the issue of non-planar faces in such grids by subdividing problem cells into tetrahedra [S4, book].

## Sources

[S1] Volume rendering for curvilinear and unstructured grids, https://doi.org/10.1109/cgi.2003.1214468, fetched 2026-10-24
[S2] Volume Rendering for Curvilinear and Unstructured Grids, https://www.sci.utah.edu/~csilva/papers/cgi2003.pdf, fetched 2026-10-27
[S3] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S4] Cell projection of meshes with non-planar faces, https://digital.library.unt.edu/ark:/67531/metadc1408947/?q=projected tetrahedra, fetched 2026-10-28 (book)
[S5] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S6] (PDF) An Architecture For Interactive Tetrahedral Volume ..., https://www.researchgate.net/publication/221257851_An_Architecture_For_Interactive_Tetrahedral_Volume_Rendering, fetched 2026-11-01 (unofficial)

### Pre-integrated classification for cell projection
A method to determine color and opacity contribution by applying pre-integrated classification, using scalar values on front and back faces and thickness as texture coordinates for a 3D texture map storing volume ray integral values [S5]. The texture coordinates are the scalar value on the front face $s_f$, the scalar value on the back face $s_b$, and the thickness $l$ resulting from scan converting the tetrahedron [S5].

### CellFast
A software algorithm for interactive unstructured volume rendering that moves computations for order-independent transparency and volume scan conversion from CPU to graphics hardware, eliminating the need for a software sorting pass [S6]. The algorithm follows these steps: I. Preprocess dataset. For a new viewpoint: II. Visibility sort cells. For every cell in sorted back-to-front order: III. Test plane equations to determine class (1,2,3,4). IV. Project and split cell unique frontback faces. V. Compute color and opacity for thick vertex. VI. (Hardware) Scanconvert new triangles [S6]. In the original CellFast implementation on OpenGL accelerated desktops, Steps II-V (visibility sorting, plane equation testing, projection/splitting, and color/opacity computation) saturate the host CPU, leaving the graphics hardware running below capacity [S6]. In experiments with CellFast, an average of 3.4 triangles are written per tetrahedron during the splitting process [S6].
