# Interpolation and reconstruction methods

Also called: AMR reconstruction, AMR interpolation

## Overview
This family defines how scalar values are estimated at arbitrary query points within unstructured, hierarchically refined AMR datasets for volume rendering. It solves the problem of mapping discrete, non-uniform cell data to continuous ray samples while maintaining visual continuity across level boundaries. Unlike sibling methods that rely on virtual cells, surface propagation, or specific tree structures (e.g., [Adaptive Octree Construction](830-adaptive-octree-construction-with-2-1-balance-and-surface-pr.md), [TET-AMR](1094-tet-amr-data-structure.md)), this family focuses on the mathematical basis functions and temporal consistency required to stitch multi-level data seamlessly.

## Shared principles
All methods in this family share the following mechanisms and mathematical foundations:
1. **Unstructured Topology:** Data is stored in elements (tetrahedra, hexahedra, cells) defined by arbitrary vertex positions, rather than a regular lattice. [G]
2. **Adaptive Resolution:** The domain is partitioned hierarchically or locally refined based on error metrics, gradient magnitudes, or feature density. [G]
3. **Basis Function Evaluation:** Scalar values at query points are computed using basis functions defined over the elements. [G]
4. **Ray-Element Intersection:** Rendering requires determining the entry and exit points of rays through element shapes. [G]
5. **Continuity and Regularization:** In differentiable rendering contexts, maintaining mesh validity (non-inverted elements) is critical. [G]

## Subtopics

### [AMR Basis Reconstruction Filter](5581-amr-basis-reconstruction-filter.md)
The AMR Basis Reconstruction Filter is a volume rendering interpolation strategy for Berger-Colella (BC) Adaptive Mesh Refinement (AMR) data that computes a continuous, smooth sample value by performing a weighted and renormalized sum of hat-shaped basis functions centered at leaf cell corners. It solves the "stitching" problem at AMR level boundaries by interpolating across all contributing AMR levels simultaneously, thereby avoiding the high-frequency transition artifacts and ghosting associated with methods that rely on virtual cells, inner nodes, or non-rectilinear subdivision. Unlike sibling methods that subdivide coarse cells into stitch elements or truncate basis support, this approach ensures continuity without introducing topological complexity, though it sacrifices strict interpolation properties. See [AMR Basis Reconstruction Filter](5581-amr-basis-reconstruction-filter.md) for implementation details.

### [Time-varying AMR Interpolation Framework](17558-time-varying-amr-interpolation-framework.md)
The Time-varying AMR Interpolation Framework is a method for visualizing time-varying Adaptive Mesh Refinement (AMR) data that addresses the inconsistency of refinement configurations between subsequent time steps. It solves the problem where finer AMR patches update more frequently than coarse patches by ensuring both time steps share an identical refinement configuration through interpolation, thereby enabling consistent feature tracking and rendering. It differs from static AMR interpolation by explicitly handling temporal evolution and remote data access for interpolated values. See [Time-varying AMR Interpolation Framework](17558-time-varying-amr-interpolation-framework.md) for implementation details.

### Nearest-Neighbor Lookup
The simplest reconstruction strategy for AMR data, returning the scalar value of the leaf cell containing the query point. It is fast and simple but not continuous, even within same-level regions [S1, unofficial].

## Background
The foundational concepts of AMR interpolation stem from the need to render unstructured grids efficiently on CPUs. The "AMR Basis Reconstruction Filter" and related strategies were detailed in research focusing on CPU-based volume rendering of AMR data, specifically addressing the limitations of nearest-neighbor and finest-level interpolation in maintaining continuity across refinement boundaries [S1, unofficial]. The time-varying framework addresses the specific challenges of dynamic simulations where refinement masks change over time, requiring synchronization of spatial data structures across temporal steps [S1, unofficial].

## Sources
[S1] CPU volume rendering of adaptive mesh refinement data, https://www.researchgate.net/publication/321321949_CPU_volume_rendering_of_adaptive_mesh_refinement_data, fetched 2026-10-26 (unofficial)
[S2] Not given in the sources.
