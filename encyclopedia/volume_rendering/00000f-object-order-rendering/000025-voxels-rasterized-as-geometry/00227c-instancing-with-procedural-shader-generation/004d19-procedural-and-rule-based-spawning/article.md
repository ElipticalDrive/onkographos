# Procedural and Rule-Based Spawning

Also called: Procedural Object Instancing

## Overview
Procedural and Rule-Based Spawning is a family of techniques for automatically placing volumetric or geometric instances (such as foliage, rocks, or particles) onto a terrain surface based on algorithmic rules rather than manual placement. It solves the performance bottleneck of rendering thousands of individual objects by integrating instance generation with the terrain's spatial hierarchy. It distinguishes itself from sibling methods like [Surface-based Instancing with Noise Modulation](1582-surface-based-instancing-with-noise-modulation.md) by decoupling instance density from geometric resolution via LOD-indexed chunk selection, and from [3D Gaussian Splatting via Instancing](1586-3d-gaussian-splatting-via-instancing.md) by relying on explicit mesh or scene node instantiation rather than implicit splatting.

## Shared principles
The family shares a core mechanism of mapping procedural generation rules to specific spatial regions defined by the terrain's Level of Detail (LOD) hierarchy.

1.  **LOD-Indexed Spawning**: Instances are associated with a `lod_index` (integer). This index determines which terrain chunks are eligible for spawning.
    *   `lod_index = 0` corresponds to high-resolution, close-range chunks.
    *   Higher indices correspond to lower-resolution, distant chunks.
    *   The system iterates through active terrain chunks and selects only those where `chunk.lod_level == item.lod_index`.
2.  **Procedural Generation**: Spawn locations are determined by a `generator` object (or built-in logic) that evaluates conditions (e.g., noise thresholds, density) at candidate positions. Manual editing of positions is not supported; locations are defined solely by these rules.
3.  **Surface Snapping**: To prevent instances from floating or sinking, the system may query a Signed Distance Field (SDF) from the voxel generator.
    *   **Requirement**: The generator must support series generation (querying arbitrary floating-point positions).
    *   **Logic**: The system samples the SDF at the candidate position and nearby points. If at least 2 nearby samples are available, it interpolates to snap the instance closer to the surface. This is an interpolation approximation, not exact ray-casting.
    *   **Limitation**: Snapping does not work correctly if the terrain is modified; instances in edited areas will not regenerate unless set to `persistent`.
4.  **Normal Offset**: After snapping, the instance position is adjusted by `offset_along_normal` along the surface normal vector to compensate for mesh imprecision.
5.  **Persistence**: Instances can be marked as `persistent`. If enabled, their ID and position are saved to a `VoxelStream` (e.g., `VoxelStreamSQLite`). If the terrain is edited, persistent instances in modified blocks are retained; non-persistent instances are regenerated.

## Subtopics

### [Voxel Instancing System](12566-voxel-instancing-system.md)
The Voxel Instancing System is a procedural placement and rendering method designed for natural spawning of semi-random foliage, rocks, and trees within voxel terrains. It solves the performance bottleneck of rendering thousands of individual objects by batching them into GPU instanced `MultiMesh` buffers or managing them as standard scene nodes. It distinguishes itself from sibling methods like [Block LOD Instancing](12572-block-lod-instancing.md) by integrating tightly with the terrain's Level of Detail (LOD) system, allowing items to spawn on specific resolution chunks and switch mesh LODs based on distance ratios relative to the terrain's view distance.

### [Block LOD Instancing](12572-block-lod-instancing.md)
Block LOD Instancing is a procedural placement method for voxel terrains that determines instance spawn locations based on the terrain's Level of Detail (LOD) hierarchy rather than fixed world coordinates. It solves the performance bottleneck of generating high-density instances (e.g., grass, rocks) across large terrains by restricting detailed placement to high-resolution chunks and coarse placement to low-resolution chunks. It differs from standard voxel instancing by decoupling instance density from geometric resolution, allowing distant, low-LOD chunks to host simplified or sparse instances while maintaining visual continuity.

### Procedural Object Instancing
#### definition
Items spawn based on procedural rules via a built-in generator, rather than manual painting. The generator is accessed via the 'generator' property of `VoxelInstanceLibraryItem` [S1].

#### pitfall
Editor Editing: Manual editing of instances in the editor is not supported. Only procedural generation defines spawn locations [S1]. Script API: Custom instance generator API is recent and may change; it is not available to scripts at time of writing [S1]. Streaming Events: `VoxelInstancer` registers to parent block events to know when to spawn. This is not available to the script API of `VoxelLodTerrain` [S1].

## Background
The Voxel Instancing System is part of the Voxel Tools suite for the Godot Engine. It was developed to address the limitations of traditional instancing in voxel environments, where terrain geometry changes dynamically. The system integrates with `VoxelGenerator` and `VoxelStream` to provide persistent, LOD-aware procedural placement. The specific implementation details regarding `VoxelInstanceLibrary` and `MultiMesh` rendering are documented in the Voxel Tools documentation [S1].

## Sources
[S1] Instancing - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/instancing/, fetched 2026-10-27
