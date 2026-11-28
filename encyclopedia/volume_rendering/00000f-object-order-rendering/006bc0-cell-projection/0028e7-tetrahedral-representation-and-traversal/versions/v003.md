# Tetrahedral representation and traversal

Also called: Projected Tetrahedra, Tetrahedral Volume Rendering, Marching Tetrahedrons, Adaptive Delaunay Update, GPU-based Projected Tetrahedra, Texture-Encoded Isosurfacing

## Overview
Tetrahedral representation and traversal decomposes volumetric data into tetrahedral primitives to enable rendering or surface extraction via geometric operations rather than ray-marching on regular grids. It solves the problem of accurately rendering unstructured, non-convex, or disconnected meshes with high fidelity [S1, S2][S3, unofficial]. It is distinguished from [Cell projection](26-cell-projection.md) siblings by its specific reliance on tetrahedral topology for either direct volume compositing (via projection) or iso-surface extraction (via subdivision), and from [Ray casting](52-ray-casting.md) by treating cells as explicit geometric primitives that are either rasterized or traversed [S1, S2][S3, unofficial].

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
Projected Tetrahedra (PT) is a cell-projection volume rendering technique for unstructured tetrahedral meshes that projects volumetric primitives onto the image plane and composites them in visibility order. It solves the problem of rendering non-uniform grids by approximating volume cells with partially transparent polygons that map efficiently to graphics hardware. Unlike ray casting, which operates in image space, PT maintains an object-order nature but requires global visibility sorting of tetrahedra to ensure correct back-to-front compositing. It sets itself apart from [Tetrahedral Volume Rendering](820-tetrahedral-volume-rendering.md) by decomposing cells into screen-space triangles rather than casting rays through cells, and from [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md) by supporting continuous scalar fields with transfer functions rather than discrete isosurfaces.

### [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md)
The Marching Tetrahedrons Algorithm (MTA) is an iso-surface extraction method that converts a scalar field defined on a regular grid into a triangular mesh by subdividing each cubic voxel into tetrahedra and processing each tetrahedron independently [G]. It solves the problem of extracting explicit polygonal surfaces from volumetric data [G]. Unlike its sibling [Tetrahedral Volume Rendering](820-tetrahedral-volume-rendering.md), which focuses on direct volume rendering via ray casting, MTA generates an explicit mesh [G]. It differs from [Projected Tetrahedra](877-projected-tetrahedra.md) by constructing a connected triangle mesh rather than projecting volumetric primitives to the screen [G]. The algorithm unfortunately has directional ambiguities because it subdivides a cube with tetrahedrons, which cannot be done symmetrically.

### [Adaptive Delaunay Update](4923-adaptive-delaunay-update.md)
Adaptive Delaunay Update is a strategy for maintaining the topological validity of a tetrahedral mesh during optimization-based volumetric reconstruction, specifically within the context of Radiance Meshes. It addresses the problem of computational cost versus geometric accuracy when vertex positions change during training. Unlike methods that recompute the Delaunay triangulation from scratch at every iteration, this method updates the topology only periodically, balancing GPU acceleration speed with rendering accuracy. It builds on [Tetrahedral representation and traversal](4396-tetrahedral-representation-and-traversal.md) by ensuring the underlying mesh remains a valid Delaunay triangulation despite vertex perturbations.

### [GPU-based Projected Tetrahedra](7484-gpu-based-projected-tetrahedra.md)
GPU-based Projected Tetrahedra is a direct volume rendering method for unstructured tetrahedral data that leverages programmable graphics hardware to achieve interactive frame rates. It extends the classic Projected Tetrahedra algorithm by offloading the geometric classification and decomposition of volume cells to the GPU via vertex shaders, allowing tetrahedral elements to be volume scan-converted within the GPU without pre-processing or additional data structures. Unlike traditional ray-casting approaches which require explicit per-pixel ray-tetrahedron intersection tests, this method treats each tetrahedron as a distinct primitive and rasterizes them using hardware operations, achieving competitive performance with a substantially smaller memory footprint.

### [Texture-Encoded Isosurfacing](7682-texture-encoded-isosurfacing.md)
Texture-Encoded Isosurfacing is a GPU-based method for extracting isosurfaces from unstructured grids by leveraging 2D texturing hardware to compute intersection points. It builds on [Tetrahedral representation and traversal](4396-tetrahedral-representation-and-traversal.md) by decomposing cells into smaller tetrahedra and projecting their front and back faces to exploit geometric coincidence. Unlike standard projected tetrahedra methods that render triangles, this approach renders fewer faces per tetrahedron by using the tetrahedral decomposition directly, mapping the scalar field values to texture coordinates to determine isosurface intersections via texture sampling.

#### Point Representation via Tetrahedra
- **Definition:** Representing data points as small tetrahedra to fit them into the triangular mesh framework for rendering [S1].
- **Parameter:** `size`: specifies the size of the point tetrahedra relative to the data ranges [S1].

#### LinPrim Tetrahedron Representation
- **Definition:** A scene representation using transparent tetrahedra bounded by triangular faces with homogeneous density [S2].
- **Step:** Define four basis vectors describing directions from the center on which corners are located, equally spaced to represent regular tetrahedra. Optimize the distance between the center and each corner independently [S2].
- **Parameter:** Tetrahedra memory requirement: 12 floats per primitive (omitting color), due to absence of symmetry in the formulation [S2].
- **Pitfall:** Unlike octahedra, the optimized position (center) can differ from the geometric center of the primitive due to lack of symmetry [S2].

#### Grid-based Traversal
- **Definition:** A method using a hierarchical data structure for rectilinear grids to quickly determine tetrahedra possibly hit by traversed grid cells, enabling interactive rendering of irregular volume data [S3, unofficial].
- **Step:** Attach all tetrahedra intersecting a given grid cell to that cell. Use minimum and maximum values to cull regions outside of the current iso-value [S3, unofficial].
- **Number:** Rendered a bioelectric field of over one million tetrahedra at interactive rates (512x512 viewport) with 16 processors or more [S3, unofficial].

## Background
The concept of decomposing volumetric data into tetrahedra for rendering and extraction has roots in the late 1990s and early 2000s. Parker et al. [PPL* 99] demonstrated one of the first methods capable of rendering irregular volume data at interactive rates on supercomputers by using a hierarchical data structure for rectilinear grids to determine tetrahedra possibly hit by traversed grid cells [S3, unofficial]. This approach allowed for the rendering of over one million tetrahedra representing bioelectric fields [S3, unofficial].

The Projected Tetrahedra (PT) algorithm, introduced by Wylie et al., became a standard for direct volume rendering on unstructured grids by projecting tetrahedra onto the screen and compositing them [S1][S4][S6, unofficial]. It solved the problem of rendering non-uniform grids by approximating volume cells with partially transparent polygons [S7, S8, unofficial].

The Marching Tetrahedrons Algorithm (MTA) was developed as an alternative to Marching Cubes to avoid the directional ambiguities inherent in cube subdivision [S1]. By subdividing cubes into tetrahedra, MTA provides a consistent topology for iso-surface extraction, although it introduces its own complexities regarding edge cases and precision [G].

In recent years, the family has expanded to include GPU-accelerated variants [7484-gpu-based-projected-tetrahedra.md] and differentiable rendering approaches [820-tetrahedral-volume-rendering.md], as well as methods for adaptive mesh updates during optimization [4923-adaptive-delaunay-update.md] and texture-encoded isosurfacing [7682-texture-encoded-isosurfacing.md].

## Sources
[S1] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S2] LinPrim: Linear Primitives for Differentiable Volumetric Rendering, https://arxiv.org/abs/2501.16312, fetched 2026-10-26
[S3] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
