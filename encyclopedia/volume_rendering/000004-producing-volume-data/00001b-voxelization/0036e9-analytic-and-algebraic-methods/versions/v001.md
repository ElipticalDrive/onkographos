# Analytic and Algebraic Methods
Also called: Exact Voxelization, Geometric Voxelization, Analytic Discretization

## Overview
Analytic and Algebraic Methods convert continuous geometric primitives (meshes, points, or parametric surfaces) into discrete volumetric grids by evaluating exact geometric intersections or integrals against voxel cells, rather than relying on screen-space rasterization or stochastic sampling. This family solves the problem of aliasing and discretization errors inherent in projection-based methods, providing high-fidelity, often differentiable, volumetric representations [S1, unofficial]. It sets itself apart from [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by avoiding projection artifacts and from [Monte Carlo 2D Voxelization](1734-monte-carlo-voxelization.md) by using deterministic geometric evaluation [S1, unofficial].

## Shared principles
All methods in this family map continuous world-space coordinates to discrete integer grid indices $(i, j, k)$ and evaluate the primitive's relationship to the voxel volume analytically. The core mechanism involves:

1.  **Coordinate Mapping**: Defining a grid resolution (number of cells per axis) and a bounding box (origin and extent). A world coordinate $x$ is mapped to a grid index $idx$ via:
    $$ idx = \lfloor \frac{x - origin}{voxel\_size} \rfloor $$
    where $voxel\_size$ is the physical dimension of a voxel [S1][S3, unofficial].

2.  **Occupancy Determination**: Evaluating whether a specific voxel cell intersects with, contains, or is influenced by the input primitive. This evaluation varies by method (e.g., rasterization coverage, ray intersection, point projection) [S1][S2, S3, unofficial].

3.  **Attribute Assignment**: Storing data in the voxel. This may be binary occupancy, continuous density, signed distance, or color/normal attributes [S1, S12].

## Subtopics

### [Signed Distance Field Voxelization](3849-signed-distance-field-voxelization.md)
Computes the signed Euclidean distance to the nearest surface point for every voxel. It differs from [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by computing exact geometric distances rather than projected coverage, and from [SDF-based Binary Voxelization](2402-sdf-based-binary-voxelization.md) by retaining continuous distance values.

### [Analytic Voxelization](4969-analytic-voxelization.md)
Determines voxel occupancy or attributes by evaluating exact geometric intersections or integrals. It avoids aliasing artifacts of screen-space projection and uses deterministic evaluation rather than stochastic sampling.

### [Surface-Aware Discretization](6820-surface-aware-discretization.md)
Computes the effective volume of each voxel based on the intersection with the target mesh, mitigating resolution-related artifacts in prints.

## Background
The term "Analytic Voxelization" refers to methods that evaluate the primitive's relationship to voxel cells using exact geometric tests or integrals [S1, unofficial]. This approach contrasts with rasterization-based techniques that project primitives to screen space. The family includes methods for generating Signed Distance Fields (SDFs) and surface-aware discretizations that calculate intersection volumes [S1, unofficial].

## Sources
[S1] Analytic Voxelization notes, URL, date fetched
