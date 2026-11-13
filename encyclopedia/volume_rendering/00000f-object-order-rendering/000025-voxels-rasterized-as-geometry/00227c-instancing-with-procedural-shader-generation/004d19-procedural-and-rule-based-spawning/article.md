# Procedural and Rule-Based Spawning


## Overview
Procedural and Rule-Based Spawning is a family of techniques for populating voxel terrains with large numbers of objects (e.g., grass, rocks, trees) by evaluating spawn rules against terrain geometry rather than manual placement [S1]. It distinguishes itself from static instancing by integrating tightly with the voxel terrain's Level of Detail (LOD) system and Signed Distance Field (SDF) surface snapping, ensuring instances align with uneven terrain geometry automatically [S1].

## Shared principles
The family operates by evaluating spawn rules defined in a `VoxelInstanceLibrary` against eligible terrain chunks. The core mechanism involves:

1.  **Chunk Eligibility**: The system iterates through loaded voxel chunks. For each item in the library, it checks if `chunk.lod_level == item.lod_index` [S1].
2.  **Candidate Generation**: The item's `generator` evaluates conditions (e.g., noise thresholds) at candidate positions within eligible chunks [Parent].
3.  **Surface Snapping (Optional)**:
    *   If `snap_to_generator_sdf_enabled` is true, the system queries the `VoxelGenerator` for SDF values at the candidate position and nearby samples [S1].
    *   It interpolates between samples to find the zero-crossing (surface) [S1].
    *   *Pitfall*: This is an interpolation approximation, not exact ray-casting. It may fail or produce errors if the terrain is edited after generation, as the generator's cached data may not reflect recent changes [S1].
4.  **Normal Offset**: The instance position is displaced along the interpolated surface normal by `offset_along_normal` [S1].
5.  **Alignment**:
    *   If `up_mode` is 'Sphere mode', instances are rotated to align with the local origin of the terrain (suitable for planets) [S1].
    *   Otherwise, they align with a fixed world up vector [S1].
6.  **Instance Creation**: Instances are created as either `MultiMesh` (high-performance, static) or `Scene` (dynamic, scriptable) types [S1].
7.  **Persistence**: If `persistent` is true and a `VoxelStream` is active, the instance's ID and position are saved. Editing the terrain marks these instances as modified, preventing regeneration and saving their new positions [S1].

## Subtopics

### [Voxel Instancing System](12566-voxel-instancing-system.md)
The Voxel Instancing System is a procedural spawning mechanism designed for natural environments, such as grass, rocks, trees, and semi-random foliage. It solves the problem of efficiently populating large voxel terrains with thousands of objects by leveraging GPU-accelerated `MultiMesh` rendering for high-performance bulk placement, while also supporting CPU-side `Scene` instances for complex, interactive entities. It distinguishes itself from general geometry instancing by integrating tightly with the voxel terrain's Level of Detail (LOD) system and Signed Distance Field (SDF) surface snapping, ensuring instances align with uneven terrain geometry without requiring manual placement. See [Voxel Instancing System](12566-voxel-instancing-system.md).

### [Block LOD Instancing](12572-block-lod-instancing.md)
Block LOD Instancing is a procedural spawning method that determines instance eligibility and placement precision based on the voxel terrain's Level of Detail (LOD) hierarchy. It solves the performance-precision trade-off in large-scale terrain rendering by spawning distant, large-scale objects on low-resolution (high-index) chunks and local, detailed objects on high-resolution (low-index) chunks. It differs from standard instancing by tightly coupling the spawn range to the terrain's LOD structure, allowing for faster generation at distance at the cost of geometric precision near sharp terrain features. See [Block LOD Instancing](12572-block-lod-instancing.md).

#### Procedural Object Instancing
**Definition**: Items spawn based on procedural rules via a built-in generator, rather than manual painting. The generator is accessed via the 'generator' property of `VoxelInstanceLibraryItem` [S1].
**Pitfalls**:
*   **Editor Editing**: Manual editing of instances in the editor is not supported. Only procedural generation defines spawn locations [S1].
*   **Script API**: Custom instance generator API is recent and may change; it is not available to scripts at time of writing [S1].
*   **Streaming Events**: `VoxelInstancer` registers to parent block events to know when to spawn. This is not available to the script API of `VoxelLodTerrain` [S1].

## Background
The Voxel Instancing System is part of the Voxel Tools suite for the Godot engine. It leverages Godot's `MultiMeshInstance` for performance and integrates with `VoxelGeneratorGraph` for SDF queries. The system was designed to handle the specific constraints of voxel terrain rendering, where geometry changes dynamically and LODs vary by distance. The persistent instance feature relies on `VoxelStreamSQLite` for data storage.

## Sources
[S1] Instancing - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/instancing/, fetched 2026-10-27
