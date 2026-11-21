# Signed Distance and Two-Step Methods

Also called: SDF-based Marching Cubes, Narrow-Band Filtering, Two-Step Reconstruction

## Overview
This family extends standard isosurface extraction by leveraging the geometric properties of Signed Distance Fields (SDFs) or by optimizing the evaluation of implicit functions. It solves the problem of computational inefficiency and memory overhead when processing large-scale implicit data or high-resolution grids. It differs from its siblings by either filtering voxels based on distance to the surface (Narrow-Band) or by decoupling the expensive implicit function evaluation from the geometric extraction (Two-Step), whereas [Snap Marching Cubes](2181-snap-marching-cubes.md) focuses on topological correctness and [Marching Cubes Revisited](10126-marching-cubes-revisited.md) on table simplification.

## Shared principles
All methods in this family operate on a structured 3D grid and produce a triangle mesh, building on [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md). The core mechanism of determining edge intersections via a case index remains foundational. However, the computation of this index is optimized:
1.  **Voxel Pruning:** Voxels that cannot intersect the isosurface are identified and skipped before expensive vertex sampling or full-case processing occurs.
2.  **Implicit Function Optimization:** When the implicit function $f(x)$ is complex (e.g., RBF interpolation), its evaluation is approximated or accelerated to reduce the cost of determining vertex signs.
3.  **Coordinate Handling:** Methods explicitly manage the transformation between voxel indices, physical world coordinates, and grid origins, often using spacing parameters or explicit offset additions .

## Subtopics

### [SDF to PLY Conversion](2343-sdf-to-ply-conversion.md)
Converts a raw SDF grid to a PLY mesh. It handles the full pipeline including chunked evaluation for memory management and coordinate transformation. Choose this for exporting neural network outputs or analytical SDFs.

### [Marching Cubes with Narrow-Band Filtering](3826-marching-cubes-with-narrow-band-filtering.md)
Filters cubes based on the distance from the cube center to the isosurface. It reduces expensive geometric evaluations by skipping cubes far from the surface. Choose this for sparse SDFs or when the surface is localized.

### [Two-Step Surface Reconstruction Method](7364-two-step-surface-reconstruction-method.md)
Uses a coarse, low-accuracy evaluation to classify voxels, then performs high-accuracy evaluation only on boundary voxels. Choose this when the implicit function evaluation (e.g., via FMM) is the bottleneck.

## Background
The use of Signed Distance Fields for surface extraction has become prevalent with the rise of neural implicit representations . Standard Marching Cubes evaluates every voxel, which is inefficient for large domains. Narrow-band filtering exploits the property that an SDF's zero-crossing is confined to a band of width equal to the voxel diagonal . The Two-Step method originates from the need to accelerate RBF-based implicit reconstruction using Fast Multipole Methods (FMM) .

## Sources
[S1] Two-Step Surface Reconstruction Method notes, URL, 2023-10-27
[S2] SDF to PLY Conversion notes, URL, 2023-10-27
