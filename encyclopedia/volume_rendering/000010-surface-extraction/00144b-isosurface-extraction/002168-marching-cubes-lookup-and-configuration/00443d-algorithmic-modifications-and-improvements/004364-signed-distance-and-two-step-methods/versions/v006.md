# Signed Distance and Two-Step Methods

Also called: SDF-based Marching Cubes, Narrow-Band Filtering, Two-Step Reconstruction

## Overview
This family extends standard isosurface extraction by leveraging Signed Distance Field (SDF) properties or optimizing implicit function evaluation [G]. It addresses computational inefficiency and memory overhead for large-scale implicit data [G]. It differs from siblings by filtering voxels via distance (Narrow-Band) or decoupling evaluation from extraction (Two-Step), while [Snap Marching Cubes](2181-snap-marching-cubes.md) targets topological correctness and [Marching Cubes Revisited](10126-marching-cubes-revisited.md) targets table simplification [G].

## Shared principles
All variants operate on a structured 3D grid and produce a triangle mesh, building on [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) [G]. The core mechanism of determining edge intersections via a case index remains foundational [G]. Optimizations include:
1. **Voxel Pruning:** Voxels that cannot intersect the isosurface are identified and skipped before expensive vertex sampling .
2. **Implicit Function Optimization:** When $f(x)$ is complex (e.g., RBF interpolation), evaluation is approximated or accelerated to reduce vertex sign determination cost .
3. **Coordinate Handling:** Methods explicitly manage transformations between voxel indices, world coordinates, and grid origins using spacing parameters or offset additions .

## Subtopics

### [SDF to PLY Conversion](2343-sdf-to-ply-conversion.md)
Converts a raw SDF grid to a PLY mesh. It handles the full pipeline including chunked evaluation for memory management and coordinate transformation. Choose this for exporting neural network outputs or analytical SDFs.

### [Marching Cubes with Narrow-Band Filtering](3826-marching-cubes-with-narrow-band-filtering.md)
Filters cubes based on the distance from the cube center to the isosurface. It reduces expensive geometric evaluations by skipping cubes far from the surface. Choose this for sparse SDFs or when the surface is localized.

### [Two-Step Surface Reconstruction Method](7364-two-step-surface-reconstruction-method.md)
Uses a coarse, low-accuracy evaluation to classify voxels, then performs high-accuracy evaluation only on boundary voxels. Choose this when the implicit function evaluation (e.g., via FMM) is the bottleneck.

## Background
The use of SDFs for surface extraction has become prevalent with neural implicit representations [G]. Standard Marching Cubes evaluates every voxel, which is inefficient for large domains [G]. Narrow-band filtering exploits the property that an SDF's zero-crossing is confined to a band of width equal to the voxel diagonal [G]. The Two-Step method originates from accelerating RBF-based implicit reconstruction using Fast Multipole Methods (FMM) .

## Sources
[S1] Two-Step Surface Reconstruction Method notes, URL, 2023-10-27
[S2] SDF to PLY Conversion notes, URL, 2023-10-27
