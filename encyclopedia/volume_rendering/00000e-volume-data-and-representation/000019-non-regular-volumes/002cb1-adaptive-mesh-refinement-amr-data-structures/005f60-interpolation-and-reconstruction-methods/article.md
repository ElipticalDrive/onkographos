# Interpolation and reconstruction methods


## Overview
This family defines how scalar values are estimated at arbitrary query points within unstructured, hierarchically refined AMR datasets for volume rendering [G]. It solves the problem of mapping discrete, non-uniform cell data to continuous ray samples while maintaining visual continuity across level boundaries [G]. Unlike sibling methods that rely on virtual cells, surface propagation, or specific tree structures (e.g., [Adaptive Octree Construction](830-adaptive-octree-construction-with-2-1-balance-and-surface-pr.md), [TET-AMR](1094-tet-amr-data-structure.md)), this family focuses on the mathematical basis functions and temporal consistency required to stitch multi-level data seamlessly [G].

## Shared principles
All methods in this family inherit the mechanisms and mathematical foundations defined in [Adaptive Mesh Refinement (AMR) Data Structures](5344-adaptive-mesh-refinement-amr-data-structures.md), including unstructured topology, adaptive resolution, basis function evaluation, ray-element intersection, and continuity requirements [G].

## Subtopics

### [AMR Basis Reconstruction Filter](5581-amr-basis-reconstruction-filter.md)
The AMR Basis Reconstruction Filter is a volume rendering interpolation strategy for Berger-Colella (BC) AMR data that computes a continuous scalar field by blending contributions from all AMR levels simultaneously. It remedies stitching artifacts at resolution boundaries without subdividing coarse cells into stitch elements or relying on non-rectilinear grid interpolation. Unlike the "Finest" method, which produces high-frequency transition artifacts at coarse cell boundaries, the Basis method yields smooth, continuous images by using a weighted, renormalized sum of hat-shaped basis functions centered at dual cell corners. Choose this method when rendering requires strict visual continuity across AMR level boundaries without introducing topological complexity.

### [Time-varying AMR Interpolation Framework](17558-time-varying-amr-interpolation-framework.md)
The Time-varying AMR Interpolation Framework is a method for visualizing adaptive mesh refinement (AMR) data that changes over time. It solves the problem of inconsistent refinement configurations between consecutive time steps, where finer patches are often updated more frequently than coarse ones. Unlike static AMR interpolation, this framework ensures temporal consistency by synchronizing the refinement topology before computing intermediate values, allowing for smooth visualization of features that split, merge, or appear/disappear. Choose this method when visualizing dynamic simulations with asynchronous refinement updates.

### [Feature-Aware Trivariate Spline Reconstruction](19909-feature-aware-trivariate-spline-reconstruction.md)
Feature-Aware Trivariate Spline Reconstruction transforms discrete volumetric data into a continuous, tensor-product spline representation with a regular structure, solving storage and rendering costs while enabling physical analysis. It sets itself apart from standard AMR basis filters by explicitly constructing a feature-aligned frame field and a globally smooth parameterization to remesh the data into regular volumetric patches, rather than relying on hierarchical basis functions on irregular cells. Unlike time-varying frameworks, it focuses on static geometric and density approximation using trivariate T-splines. Choose this method when high-fidelity static reconstruction and physical analysis of complex density fields are required.

### Nearest-Neighbor Lookup
The simplest reconstruction strategy for AMR data, returning the scalar value of the leaf cell containing the query point [S1, unofficial]. It is fast and simple but not continuous, even within same-level regions [S1, unofficial].

## Background
The foundational concepts of AMR interpolation stem from the need to render unstructured grids efficiently on CPUs [G]. Early strategies focused on basic lookup and trilinear interpolation on logical grids to address continuity limitations [S1, unofficial]. Subsequent research expanded these techniques to handle multi-level blending, temporal synchronization, and feature-aware parameterization for advanced visualization pipelines [G].

## Sources
[S1] CPU volume rendering of adaptive mesh refinement data, https://www.researchgate.net/publication/321321949_CPU_volume_rendering_of_adaptive_mesh_refinement_data, fetched 2026-10-26 (unofficial)
