# Signed Distance Field Voxelization

Also called: SDF Voxelization, Signed Distance Grid Generation

## Overview
Signed Distance Field (SDF) Voxelization maps a continuous parametric object $Q_c$ into a digital representation $D_p$ by computing the exact Euclidean distance from every voxel center to the object's surface, assigning a sign to indicate interior or exterior status [S1]. It differs from [Analytic Voxelization](10493-analytic-voxelization.md) by preserving precise metric distance values rather than fractional occupancy, and from [Surface-Aware Discretization](6820-surface-aware-discretization.md) by operating on the signed distance metric rather than surface area intersection.

## Shared principles
All methods in this family map a continuous parametric object $Q_c$ into a digital representation $D_p$ defined on a 3D grid. The core mechanism involves:
1.  **Parametric Definition**: The input is defined by analytical equations (e.g., implicit functions, parametric surfaces) rather than discrete samples [S1].
2.  **Exact Evaluation**: For each voxel, the method computes an exact scalar value based on the geometric relationship between the voxel volume and the object $Q_c$ [S1].
3.  **Membership/Distance Mapping**: The output is a scalar field where values represent either the signed Euclidean distance to the surface or the fractional volume of the voxel occupied by the object, typically normalized to $[0, 1]$ or a signed range [S1].

## Subtopics

### [Signed Distance Field (SDF) Generation from Scanned Surfaces](854-signed-distance-field-sdf-generation-from-scanned-surfaces.md)
This method converts a triangulated surface mesh (typically from 3D scanning) into a 3D Signed Distance Field (SDF) stored as a texture or grid. It solves the problem of representing explicit, potentially noisy or complex scanned geometry as an implicit function suitable for raymarching, isosurfacing, or volume rendering. Unlike [Slice-based SDF Voxelization](1736-slice-based-sdf-voxelization.md), which processes depth slices, this method operates directly on geometric primitives (triangles) to compute exact Euclidean distances. It differs from [Signed Distance Field from Mesh](1327-signed-distance-field-from-mesh.md) by explicitly handling the conversion to a bounded 3D texture domain with specific boundary shell logic.

### [SDF-based Binary Voxelization](2402-sdf-based-binary-voxelization.md)
SDF-based Binary Voxelization converts a continuous signed distance field (SDF) into a discrete binary volume representation, where each voxel is classified as either "inside" or "outside" the target geometry. It solves the problem of creating a topologically consistent binary mask from a continuous scalar field, serving as a foundational step for generating voxel-based meshes (e.g., Minecraft-style cubes) or preparing data for binary volume rendering. Unlike dense SDF generation which preserves distance magnitudes, this method discards geometric precision in favor of a clean, binary boundary definition. It builds on [Signed Distance Field Voxelization](3849-signed-distance-field-voxelization.md) by applying a thresholding operation to the resulting scalar field.

### [MeshIntegrator Integration](3630-meshintegrator-integration.md)
This method integrates a triangle mesh into a `VoxelBlockGrid` by computing a Truncated Signed Distance Field (TSDF) and an associated weight map. Unlike sibling methods that blend or accumulate data, this integrator overwrites voxels within the mesh's truncation band and leaves blocks outside the band untouched, requiring the grid to be cleared for isolated mesh conversion.

### [Dense Volume SDF Generation](4499-dense-volume-sdf-generation.md)
Dense Volume SDF Generation computes a Signed Distance Field (SDF) for a triangle mesh by iterating over every voxel in a defined 3D grid, calculating the Euclidean distance from the voxel center to the mesh surface, and determining the inside/outside status via ray intersection parity or winding numbers. It differs from sparse methods by storing a value for every voxel in the bounding box, providing full data access suitable for simulation, marching cubes extraction, or machine learning inputs. Unlike [SDF-based Binary Voxelization](2402-sdf-based-binary-voxelization.md), which only stores occupancy, this method retains precise distance metrics.

### [Level Set Water Representation](4635-level-set-water-representation.md)
The Level Set Water Representation tracks a liquid's free surface on a 3D grid by storing the signed distance from each cell center to the water-air interface. It distinguishes itself from sibling methods by requiring only a scalar value per cell, making it particularly well suited to GPU implementation. It builds on [Signed Distance Field Voxelization](3849-signed-distance-field-voxelization.md) by advecting the distance field with fluid velocity rather than computing it statically from geometry.

#### Function Volume SDF
**Definition**
A method that stores no data in memory, returning a function object that computes distance on each query, trading computational power for memory savings [S1].

**Steps**
1.  Load mesh and calculate `oneDimVoxelSize` using `mm.suggestVoxelSize(mesh, 1e7)` [S1].
2.  Calculate origin and dimensions using `mm.calcOriginAndDimensions(mesh.computeBoundingBox(), oneDimVoxelSize)` [S1].
3.  Set `dist.signMode` to `mm.SignDetectionMode.HoleWindingRule` [S1].
4.  If CUDA is available, create `FastWindingNumber` mesh and assign to `params.fwn` [S1].
5.  Call `mm.meshToDistanceFunctionVolume(mesh, params)` to generate `functionVolume` [S1].
6.  Call `mm.marchingCubes(functionVolume, mcParams)` to extract `offMesh` [S1].

**Parameters**
*   `voxelSize`: Vector3f diagonal size. Calculated via `mm.suggestVoxelSize(mesh, 1e7)` [S1].
*   `origin`: Vector3f defining the grid origin, calculated via `mm.calcOriginAndDimensions` [S1].
*   `dimensions`: Vector3i defining the grid dimensions, calculated via `mm.calcOriginAndDimensions` [S1].
*   `maxDistSq`: float squared maximum distance to calculate. Set to `(3 * oneDimVoxelSize)^2` [S1].
*   `nullOutsideMinMax`: boolean. If True, do not calculate values too far [S1].
*   `signMode`: enum for sign detection. `mm.SignDetectionMode.HoleWindingRule` is robust [S1].
*   `fwn`: Optional FastWindingNumber object (GPU/CUDA) for accelerated sign calculation [S1].

## Background
The concept of Signed Distance Fields for voxelization originates from the need to represent complex geometry efficiently for rendering and simulation. The specific implementation details regarding `MeshToDistanceVolumeParams`, `HoleWindingRule`, and the `meshlib` library are documented in the Mesh to SDF Library for Python & C++ [S1]. The library provides both dense volume generation and function-volume approaches, with the latter introduced to address memory constraints by computing distances on-demand [S1].

## Sources
[S1] Mesh to SDF (Signed Distance Field) Library for Python & C++, https://meshlib.io/feature/mesh-to-sdf/, fetched 2026-10-30
