# Analytic Voxelization


## Overview
Analytic Voxelization is a family of methods that map continuous world-space coordinates to discrete integer grid indices and evaluate the relationship between input primitives and voxel volumes using exact geometric or algebraic operations rather than rasterization or stochastic sampling [S1, S3, unofficial]. It sets itself apart from [Monte Carlo 2D Voxelization](1734-monte-carlo-2d-voxelization.md) by providing deterministic, noise-free results through exact intersection tests, and from [Differentiable Voxelization (MC/CF)](1742-differentiable-voxelization-mc-cf.md) by focusing on conservative moment integration and binary occupancy via clipping rather than closed-form gradients of rasterization coverage.

## Shared principles
All methods in this family map continuous world-space coordinates to discrete integer grid indices $(i, j, k)$ and evaluate the primitive's relationship to the voxel volume analytically. The core mechanism involves:

1.  **Coordinate Mapping**: Defining a grid resolution (number of cells per axis) and a bounding box (origin and extent). A world coordinate $x$ is mapped to a grid index $idx$ via:
    $$ idx = \lfloor \frac{x - origin}{voxel\_size} \rfloor $$
    where $voxel\_size$ is the physical dimension of a voxel [S1, unofficial][S3, unofficial].

2.  **Occupancy Determination**: Evaluating whether a specific voxel cell intersects with, contains, or is influenced by the input primitive. This evaluation varies by method (e.g., rasterization coverage, ray intersection, point projection) [S1, unofficial][S2, S3, unofficial].

3.  **Attribute Assignment**: Storing data in the voxel. This may be binary occupancy, continuous density, signed distance, or color/normal attributes [S1, S12].

## Subtopics

### [Monte Carlo 2D Voxelization](1734-monte-carlo-2d-voxelization.md)
Uses stochastic sampling of the generalized winding number for 2D meshes, allowing adaptive sampling and gradient propagation, unlike the deterministic nature of this family.

### [Differentiable Voxelization (MC/CF)](1742-differentiable-voxelization-mc-cf.md)
Provides closed-form analytic gradients for rasterization (MC) or convolution filtering (CF) of 2D/3D primitives, focusing on differentiability rather than conservative physical conservation.

### [Analytic Moment Voxelization](3307-analytic-moment-voxelization.md)
Computes analytic moments of a polynomial density field over polyhedra using simplicial decomposition and geometric clipping, storing higher-order data than simple occupancy.

### [Analytic Tetrahedron Voxelization](3724-analytic-tetrahedron-voxelization.md)
Maps tetrahedral meshes to Cartesian grids via exact point-in-tetrahedron tests or conservative clipping, preserving physical conservation properties (mass, moments) for unstructured volumetric data.

### Precomputed Obstacle Voxelization
- **Definition**: A technique where voxelization for obstacles with simple analytic transformations is precomputed, and the inverse transformation is applied during texture access [S1, unofficial].
- **Steps**:
    1. Precompute the voxelization of the obstacle mesh [S1, unofficial].
    2. When accessing inside-outside and obstacle velocity textures, multiply the texture coordinates by the inverse of the transformation matrix [S1, unofficial].
- **Pitfall**: This method only applies to simple analytic transformations (e.g., rigid translation and rotation), not complex skinning operations [S1, unofficial].

## Background
The concept of precomputing voxelization for obstacles undergoing simple analytic transformations (like rigid body motion) was introduced in the context of real-time fluid simulation to reduce per-frame computational cost [S1, unofficial]. By simplifying obstacle models and leveraging inverse transformations during texture lookup, the method allows for efficient voxelization at every frame without significant performance penalties [S1, unofficial].

## Sources
[S1] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
