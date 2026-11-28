# Post-Voxelization Modifiers

Also called: Voxel Post-Processing, Voxel Refinement

## Overview
Post-Voxelization Modifiers are algorithms applied to a discrete voxel grid after the initial geometric conversion to correct artifacts, optimize storage, or adapt the data for specific rendering pipelines. Unlike the parent [Voxelization](12-voxelization.md) methods which map continuous sources to discrete values, this family modifies existing voxel attributes (density, material, opacity) without re-evaluating the source geometry. They are distinct from [Volumetric Attribute and Density Formation](6304-volumetric-attribute-and-density-formation.md) in that they operate on already-voxelized data rather than forming the initial scalar field, and from [Sparse and Hierarchical Structures](4391-sparse-and-hierarchical-structures.md) in that they focus on semantic correction rather than memory layout optimization.

## Shared principles
The family shares a common execution model:
1.  **Input**: A fully populated voxel grid (or sparse representation) with initial attributes (solid/empty, material ID, density).
2.  **Operation**: A local or global transformation rule applied to voxel values. This may involve neighborhood analysis (e.g., checking children in a hierarchy) or statistical aggregation.
3.  **Output**: A modified voxel grid where artifacts (such as transparency in thin geometry or excessive porosity) are reduced, or data is compressed/abstracted (e.g., LOD downsampling).
4.  **Timing**: Executed post-generation, allowing for iterative refinement or user-driven adjustments without re-running the expensive geometric intersection tests of the parent voxelization step.

## Subtopics

### Voxel Downsampling with Material Propagation
A hierarchical compression technique used in large-scale planetary voxel systems to reduce memory footprint while preserving visual fidelity across distance levels. It operates on a hierarchy of voxel grids (bricks) where each coarse voxel represents a 2x2x2 block of finer voxels.

*   **Mechanism**: A coarse voxel is classified as "solid" if at least 4 of its 8 children are solid [S1, unofficial]. The material of the coarse voxel is determined by the most common material among its solid children [S1, unofficial].
*   **Propagation**: Edits to the fine grid propagate immediately up all LOD levels, ensuring that features like craters remain visible from orbit without requiring separate mesh fallbacks [S1, unofficial].
*   **Usage**: Chosen for open-world or planetary rendering where memory budget is strict (e.g., ~2 GB on a 4090) and distance-based detail is required [S1, unofficial].
*   **Link**: See [Sparse and Hierarchical Structures](4391-sparse-and-hierarchical-structures.md) for the underlying data structure.

### Porosity via Spatial Pseudo-Random Hash Drop-out
A technique to introduce controlled void space into voxelized meshes, simulating porous materials or reducing voxel count for aesthetic/performant reasons.

*   **Mechanism**: Voxels are dropped (set to empty) based on a spatial pseudo-random hash function. The decision is deterministic for a given coordinate, ensuring consistency across updates.
*   **Parameter**: Controlled by a `porosity` float parameter (0.0 to 1.0), passed via CLI (`--porosity <FLOAT>`) or API (`MeshProcessor::voxelize`).
*   **Usage**: Chosen when simulating materials like foam, rock, or when reducing voxel density for performance without altering the mesh geometry.
*   **Link**: See [VoxelGrid from Mesh (Fill Method)](1013-voxelgrid-from-mesh-fill-method.md) for the base generation method.

### Opacify Modifier
A correction step for rendering thin or zero-thickness geometry (such as meshes) that would otherwise appear dashed, aliased, or partially transparent during ray marching or cone marching.

*   **Mechanism**: Scales the surface coverage of voxels grazed by thin geometry to read as solid. It effectively thickens the voxel representation of the surface.
*   **Parameter**: `Amount` (default 2.0f). Increasing this value prevents surface leakage but may cause inflation; decreasing it reduces inflation but may reveal transparency. It is not a substitute for sufficient voxel resolution [S3, unofficial].
*   **Usage**: Chosen when rendering wireframes, thin shells, or low-poly meshes where standard voxelization fails to produce a solid appearance.
*   **Link**: See [Voxel Volume Rendering](6488-voxel-volume-rendering.md) for the rendering context.

## Background
Post-voxelization processing emerged as voxelization moved from simple binary occupancy grids to complex, attribute-rich representations for real-time rendering and global illumination. Early systems required immediate re-voxelization to correct errors, which was computationally prohibitive. The introduction of hierarchical structures (like Sparse Voxel Octrees) enabled efficient downsampling [S1, unofficial]. The Opacify modifier is specific to real-time voxel GI systems (e.g., StrideVoxelGI) to handle the visual artifacts of rasterizing thin geometry into voxels [S3, unofficial]. Porosity features have been added to open-source voxelizers to support procedural material generation [S2, unofficial].

## Sources
[S1] Choose voxel data structure & LOD scheme #9, https://github.com/theHimanshuShekhar/bhayanakvoxel/issues/9, fetched 2026-10-28 (unofficial)
[S2] ✨ Add porosity feature to voxelization ✨ Add porosity feature to voxelization, https://github.com/lhemerly/voxelizer-rs/pull/175/commits/e0b536fd0e9a9c4f6abd9cc6f9942905193acd13, fetched 2026-10-28 (unofficial)
[S3] Nicogo1705/StrideVoxelGI, https://github.com/Nicogo1705/StrideVoxelGI, fetched 2026-10-26 (unofficial)
