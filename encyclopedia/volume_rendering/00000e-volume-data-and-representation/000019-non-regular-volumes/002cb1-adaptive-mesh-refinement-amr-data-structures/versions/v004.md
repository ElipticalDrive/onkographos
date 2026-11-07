# Adaptive Mesh Refinement (AMR) Data Structures

Also called: Adaptive Mesh Refinement, Hierarchical Grids

## Overview
Adaptive Mesh Refinement (AMR) data structures store volumetric data on hierarchically refined grids to efficiently represent multiscale phenomena without the memory overhead of uniform high-resolution grids [S1][S2, unofficial]. This family solves the problem of navigating and interpolating data across disparate resolution levels during volume rendering. It sets itself apart from unstructured tetrahedral meshes by maintaining logical grid structures (bricks or blocks) that allow for efficient neighbor lookups and structured interpolation, while differing from point-based methods by explicitly defining cell boundaries and refinement hierarchies [S2, unofficial].

## Shared principles
All methods in this family share the following mechanisms and mathematical foundations:
1. **Unstructured Topology:** Data is stored in elements (tetrahedra, hexahedra, cells) defined by arbitrary vertex positions, rather than a regular lattice. [G]
2. **Adaptive Resolution:** The domain is partitioned hierarchically or locally refined based on error metrics, gradient magnitudes, or feature density. [G]
3. **Basis Function Evaluation:** Scalar values at query points are computed using basis functions defined over the elements. [G]
4. **Ray-Element Intersection:** Rendering requires determining the entry and exit points of rays through element shapes. [G]
5. **Continuity and Regularization:** In differentiable rendering contexts, maintaining mesh validity (non-inverted elements) is critical. [G]

## Subtopics

### [TET-AMR Data Structure](1094-tet-amr-data-structure.md)
The TET-AMR data structure is a forest of tetrahedral refinement trees designed to store volumetric data for elements at all refinement levels, including child counts, direct face neighbors, and associated scalars. It solves the problem of efficiently identifying which refinement trees to sample during volume rendering by organizing the data to support fast point location queries and bounding volume hierarchy (BVH) construction over the coarse mesh. It sets itself apart from general AMR structures by explicitly storing direct face neighbors and child counts within the element data to avoid runtime index lookups, building on [Non-regular volumes](9-non-regular-volumes.md).

### [Scene Geometry and Scalar Range Initialization](1679-scene-geometry-and-scalar-range-initialization.md)
Constructs the scene geometry and scalar ranges for volume rendering by aggregating Adaptive Mesh Refinement (AMR) box data, computing global bounding boxes via MPI, applying normalization scaling, and determining scalar value ranges including optional logarithmic scaling. It differs from sibling methods by focusing on the initialization phase of the rendering pipeline, specifically establishing the spatial and data domains before reconstruction or ray marching begins.

### [Exact AMR Rendering](4399-exact-amr-rendering.md)
Exact AMR Rendering treats Adaptive Mesh Refinement (AMR) gas cells as piecewise-constant geometric cubes rather than smoothing them as SPH kernels. It solves the problem of biased, smoothed representations in standard SPH rendering by providing exact pixel-averaged cross-sections for slices, exact volume integrals for projections, and exact mass-conserving maps for HEALPix. It differs from its sibling [Approximate perspective projection of AMR cells](4400-approximate-perspective-projection-of-amr-cells.md) by avoiding orthographic approximations for perspective views, ensuring geometric fidelity where perspective distortion is significant. It builds on [Exact AMR Rendering](4399-exact-amr-rendering.md) by treating cells as piecewise-constant cubic volumes rather than particles.

### [AMR Basis Reconstruction Filter](5581-amr-basis-reconstruction-filter.md)
The AMR Basis Reconstruction Filter is a volume rendering interpolation strategy for block-structured Adaptive Mesh Refinement (AMR) data that computes a continuous, smooth scalar field by blending contributions from all AMR levels simultaneously. It remedies stitching artifacts at level boundaries without relying on non-rectilinear grid interpolation or cell subdivision, distinguishing it from methods that use stitch cells or simplicial elements. Unlike the "Finest" method, which produces high-frequency transition artifacts at coarse boundaries, the Basis method yields smoother, continuous images at the cost of higher computational expense.

### AMR with 3DTM
- **Definition:** A technique rendering Adaptive Mesh Refinement (AMR) data using 3DTM, partitioning the volume into axis-aligned bricks of the same level of resolution [S1].
- **Steps:** Render AMR data using 3DTM; partition the volume into axis-aligned bricks of the same level of resolution; extend the AMR tree-based 3DTM to render large and sparse data sets [S1].

### Subdivision for Cell-Centered AMR Boundaries
- **Definition:** A challenge in cell-centered AMR where stitching across boundaries requires unstructured mesh geometry because dual grids do not align across levels [S2, unofficial].
- **Comparison:** This approach (subdividing into tets/hexes) is contrasted with the paper's basis function method which avoids subdivision [S2, unofficial].

### Nearest-Neighbor Lookup
- **Definition:** The simplest reconstruction strategy for AMR data, returning the scalar value of the leaf cell containing the query point. It is fast and simple but not continuous, even within same-level regions [S2, unofficial].

## Background
The concept of multiresolution data representation using octrees for hierarchical volume "bricking" was introduced in early volume visualization research [S1]. Kähler and Hege extended this by rendering AMR data using 3DTM, partitioning the volume into axis-aligned bricks of uniform resolution and extending the tree-based structure for large, sparse datasets [S1]. The Berger-Colella AMR data structure, which underpins many modern block-structured AMR implementations, organizes data into bricks containing cell-centered values [S2, unofficial]. Recent developments focus on continuous reconstruction strategies, such as the AMR Basis Reconstruction Filter, which addresses stitching artifacts at level boundaries by blending contributions from all levels simultaneously, avoiding the need for unstructured mesh subdivision [S2, unofficial].

## Sources
[S1] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S2] CPU volume rendering of adaptive mesh refinement data, https://www.researchgate.net/publication/321321949_CPU_volume_rendering_of_adaptive_mesh_refinement_data, fetched 2026-10-26 (unofficial)
