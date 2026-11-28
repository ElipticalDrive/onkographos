# Signed Distance and Two-Step Methods


## Overview
This family adapts standard isosurface extraction to Signed Distance Functions (SDFs) and implicit fields defined by scattered data. It solves the problem of efficiently extracting the zero-level set from continuous implicit representations by combining spatial filtering or error-bounded approximation with standard lookup-based triangulation. It differs from siblings by prioritizing continuous implicit inputs over pre-rasterized discrete grids, using techniques like narrow-band filtering or two-step evaluation to avoid processing the entire volume [SDF to PLY Conversion](2343-sdf-to-ply-conversion.md), [Marching Cubes with Narrow-Band Filtering](3826-marching-cubes-with-narrow-band-filtering.md), [Two-Step Surface Reconstruction Method](7364-two-step-surface-reconstruction-method.md).

## Shared principles
The family relies on the parent's [Shared principles](2518-marching-cubes-lookup-and-configuration.md) regarding the `edgeTable` and `triangleTable`.

1. **Implicit Input**: The scalar field is not necessarily pre-rasterized into a dense grid. It may be a continuous function $S: \mathbb{R}^3 \to \mathbb{R}$ (SDF) or an RBF interpolation of scattered points [SDF to PLY Conversion](2343-sdf-to-ply-conversion.md), [Two-Step Surface Reconstruction Method](7364-two-step-surface-reconstruction-method.md).
2. **Zero-Level Set**: The target isosurface is typically defined by $S(x) = 0$ [SDF to PLY Conversion](2343-sdf-to-ply-conversion.md), [Marching Cubes with Narrow-Band Filtering](3826-marching-cubes-with-narrow-band-filtering.md).
3. **Voxel Classification**: Voxels are classified based on the signs of the scalar field at their 8 vertices [SDF to PLY Conversion](2343-sdf-to-ply-conversion.md), [Marching Cubes with Narrow-Band Filtering](3826-marching-cubes-with-narrow-band-filtering.md), [Two-Step Surface Reconstruction Method](7364-two-step-surface-reconstruction-method.md).
4. **Filtering/Acceleration**: To avoid evaluating the implicit function for every voxel in the domain, methods use:
   * **Narrow-Band Filtering**: Skipping voxels where the distance from the center to the surface exceeds the cube's geometric bounds [Marching Cubes with Narrow-Band Filtering](3826-marching-cubes-with-narrow-band-filtering.md).
   * **Two-Step Evaluation**: Using a low-cost, low-accuracy approximation with error bounds to screen voxels, then evaluating only boundary voxels with high accuracy [Two-Step Surface Reconstruction Method](7364-two-step-surface-reconstruction-method.md).

## Subtopics

### [SDF to PLY Conversion](2343-sdf-to-ply-conversion.md)
Converts an implicit SDF to a PLY mesh by sampling on a dense grid. It prioritizes memory management via chunked evaluation. Choose when a dense grid representation is acceptable and the input is a callable SDF.

### [Marching Cubes with Narrow-Band Filtering](3826-marching-cubes-with-narrow-band-filtering.md)
Restricts MC to voxels near the zero-level set. It solves the performance bottleneck of full-volume processing by skipping cubes too far from the surface. Choose for large domains with sparse surfaces.

### [Two-Step Surface Reconstruction Method](7364-two-step-surface-reconstruction-method.md)
Extracts surfaces from RBF-interpolated scattered data. It uses FMM approximation with error bounds to filter voxels, then exact evaluation for boundary voxels. Choose for scattered data inputs where dense grid sampling is inefficient or impossible.

## Background
Standard Marching Cubes operates on discrete voxel grids. The adaptation to Signed Distance Functions and implicit fields arose from the need to render smooth surfaces defined by continuous mathematical functions or scattered point clouds without explicit mesh structures [SDF to PLY Conversion](2343-sdf-to-ply-conversion.md), [Two-Step Surface Reconstruction Method](7364-two-step-surface-reconstruction-method.md). The Two-Step method specifically addresses the computational cost of evaluating complex implicit functions (like RBFs) for every voxel in a large domain [Two-Step Surface Reconstruction Method](7364-two-step-surface-reconstruction-method.md).

## Sources

[S1] Marching Cubes with Narrow-Band Filtering, https://github.com/unofficial-mc-notes/narrow-band, 2023-10-01
[S2] SDF to PLY Conversion, https://github.com/unofficial-mc-notes/sdf-ply, 2023-10-01
