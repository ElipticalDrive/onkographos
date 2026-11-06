# Adaptive Mesh Refinement (AMR) Data Structures

Also called: AMR Data Structures, AMR Hierarchies

## Overview
Adaptive Mesh Refinement (AMR) data structures organize volumetric data with non-uniform resolution to efficiently store and render sparse or high-gradient fields. This family solves the problem of managing hierarchical grids where refinement levels vary spatially, enabling efficient traversal and rendering without processing uniform high-resolution data everywhere. It sets itself apart from sibling methods like [Tetrahedral Regularization](834-tetrahedral-regularization.md) and [Bezier Tetrahedra Evaluation](1422-bezier-tetrahedra-evaluation.md) by focusing on the hierarchical organization of the data (typically box-based or tetrahedral forests) rather than the specific geometric interpolation or basis function evaluation techniques used during the rendering pass.

## Shared principles
All AMR data structures in this family share the principle of hierarchical decomposition. The domain is partitioned into a coarse base mesh (or root level), which is recursively subdivided into finer elements (children) based on error metrics or feature density. Key shared mechanisms include:

1.  **Hierarchical Indexing:** Elements are organized in trees (e.g., octrees, quadtree-like structures for tetrahedra) where parent nodes represent coarser volumes and children represent refined sub-volumes.
2.  **Level-of-Detail (LOD) Management:** The structure maintains explicit relationships between parent and child elements, allowing traversal algorithms to skip empty or low-detail regions (empty space skipping) and focus on high-detail areas.
3.  **Boundary Handling:** A critical challenge is maintaining continuity and correct neighbor relationships across level boundaries. Structures must define how data is interpolated or sampled when a ray crosses from a coarse element into a refined child or a neighboring block at a different level.
4.  **Scalar Aggregation:** To support efficient culling and shading, parent elements often store aggregated scalar statistics (min/max values) of their children, enabling early-out tests during rendering.

## Subtopics

### [TET-AMR Data Structure](1094-tet-amr-data-structure.md)
A forest of tetrahedral refinement trees designed to store volumetric data for elements across all refinement levels. It explicitly stores direct face neighbors and child counts within the element data to facilitate fast traversal and neighbor lookups without runtime index resolution. It differs from general AMR structures by using a tetrahedral basis and storing neighbor information directly, enabling the construction of a Bounding Volume Hierarchy (BVH) over the coarse elements to efficiently identify which refinement trees to sample. Choose this when rendering unstructured tetrahedral AMR data where neighbor connectivity is critical for traversal.

### [Scene Geometry and Scalar Range Initialization](1679-scene-geometry-and-scalar-range-initialization.md)
Constructs the scene geometry and scalar ranges for volume rendering by aggregating AMR box data, computing global bounding boxes via MPI, applying normalization scaling, and determining scalar value ranges including optional logarithmic scaling. It differs from sibling methods by explicitly handling the aggregation of distributed AMR level data into a unified scene representation with normalized coordinates and robust scalar range detection. Choose this for the initialization phase of distributed AMR rendering pipelines to ensure consistent coordinate systems and scalar mappings.

### [Exact AMR Rendering](4399-exact-amr-rendering.md)
Renders AMR gas cells as piecewise-constant cubic cells rather than SPH particles. Each cell is a cube of side 'smooth', volume mass/rho, with edges aligned to the original simulation axes. It sets itself apart by treating AMR data as a set of discrete, axis-aligned cubes, simplifying the intersection logic compared to complex tetrahedral or irregular meshes. Choose this for gas dynamics simulations where the underlying data is naturally structured as cubic cells.

### AMR Basis Reconstruction Filter
#### definition
A method to provide a continuous interpolant across AMR level boundaries using basis functions, allowing smooth sample reconstruction without costly cell location kernels [S1].
#### pitfall
The non-truncated support of the basis function results in larger support overlap between cells from neighboring blocks [S1].
#### comparison
Compared to Leaf et al. [21] and Ljung et al. [24], the basis function's support is not truncated at the first sample on the neighboring side, leading to a smoother interpolant but larger support overlap between cells from neighboring blocks [S1].

### AMR with 3DTM
#### definition
A technique rendering Adaptive Mesh Refinement (AMR) data using 3DTM, partitioning the volume into axis-aligned bricks of the same level of resolution [S2].
#### step
Render AMR data using 3DTM. Partition the volume into axis-aligned bricks of the same level of resolution. Extend the AMR tree-based 3DTM to render large and sparse data sets [S2].

## Background
The concept of Adaptive Mesh Refinement (AMR) originated in computational fluid dynamics and other scientific computing fields to efficiently resolve multi-scale phenomena. In visualization, early approaches relied on uniform grids or simple hierarchical box structures. The integration of AMR with ray tracing and volume rendering has evolved to handle the complexity of hierarchical data traversal. Key developments include the use of sparse octrees for traversal [S1] and the extension of tree-based techniques like 3DTM to handle large, sparse AMR datasets [S2]. The challenge of maintaining continuity across level boundaries has led to various interpolation strategies, including basis function reconstruction and unstructured element stitching [S1].

## Sources
[S1] PDF Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
