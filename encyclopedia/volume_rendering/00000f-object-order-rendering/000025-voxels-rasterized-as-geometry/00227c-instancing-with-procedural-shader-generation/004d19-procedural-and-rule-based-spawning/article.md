# Procedural and Rule-Based Spawning


## Overview
Procedural and Rule-Based Spawning is a family of techniques for automatically generating and placing large numbers of 3D instances (such as vegetation, rocks, or debris) onto a surface or volume based on algorithmic rules rather than manual placement. It solves the scalability problem of hand-placing assets in large environments by decoupling instance logic from the rendering pipeline, allowing for dynamic, data-driven distribution. It distinguishes itself from sibling methods like [Surface-based Instancing with Noise Modulation](1582-surface-based-instancing-with-noise-modulation.md) by relying on explicit procedural generators and LOD-aware grid constraints rather than purely continuous noise fields, and from [3D Gaussian Splatting via Instancing](1586-3d-gaussian-splatting-via-instancing.md) by using discrete geometric primitives instead of point-sprites.

## Shared principles
The family shares a core workflow: define a spawnable item, define a procedural rule for its location, and execute placement during terrain or chunk updates.

1.  **Item Definition**: Each spawnable entity is defined by a configuration object (e.g., `VoxelInstanceLibraryItem`) containing:
    *   `lod_index`: An integer determining the terrain Level of Detail (LOD) at which the item spawns. Lower indices correspond to higher-resolution, closer chunks; higher indices correspond to lower-resolution, distant chunks.
    *   `offset_along_normal`: A float value adjusting the instance's position along the surface normal to prevent sinking or floating.
    *   `snap_to_generator_sdf_enabled`: A boolean toggle enabling Signed Distance Field (SDF) based surface snapping.
    *   `persistent`: A boolean indicating if the instance's position is saved to the voxel stream and persists across edits/load.
    *   `generator`: The procedural logic defining where instances spawn.

2.  **Procedural Generation**:
    *   Instances are placed where procedural conditions (density, noise thresholds) are met.
    *   The system uses a built-in generator by default, accessed via the `generator` property of the item configuration.
    *   Manual editing of instance positions in the editor is not supported; locations are defined solely by procedural rules.

3.  **Surface Snapping**:
    *   If `snap_to_generator_sdf_enabled` is true, the system queries the voxel generator for SDF data.
    *   *Requirement*: The generator must support series generation (querying arbitrary floating-point positions).
    *   *Logic*: The system samples the SDF at the instance's projected position. If at least 2 nearby samples are available, it interpolates to snap the instance closer to the surface. This is an approximation, not exact ray-casting.
    *   *Limitation*: Snapping does not work correctly if the terrain is modified; instances in edited areas will not regenerate unless set to `persistent`.

4.  **Persistence**:
    *   If `persistent` is true, the instance ID and position are saved to the voxel stream.
    *   If the terrain is edited, persistent instances in the modified block are saved rather than regenerated. Non-persistent instances are regenerated.
    *   *Constraint*: Only specific streams (e.g., `VoxelStreamSQLite`) support saving persistent instances.

5.  **Rendering**:
    *   Items are rendered either as GPU-instanced `MultiMesh` buffers (high performance) or as standard scene nodes (flexible but slower).
    *   `MultiMesh` items support LOD switching based on distance ratios relative to the terrain LOD view distance.

## Subtopics

### [Voxel Instancing System](12566-voxel-instancing-system.md)
The Voxel Instancing System is a method for procedurally spawning and managing large numbers of 3D objects on a voxel terrain surface. It solves the problem of rendering natural environments efficiently by integrating tightly with voxel terrain LOD systems and SDF-based surface snapping. It distinguishes itself from general GPU instancing by allowing instances to adhere to complex terrain geometry without manual placement.
*   **When to choose**: Use for dense, natural environments (grass, rocks) where performance is critical and terrain geometry is complex.
*   **Link**: [Voxel Instancing System](12566-voxel-instancing-system.md)

### [Block LOD Instancing](12572-block-lod-instancing.md)
Block LOD Instancing is a voxel terrain instancing strategy where the spatial resolution of the spawning grid is determined by the terrain's Level of Detail (LOD) index. It solves the performance bottleneck of generating high-frequency instances across large, detailed terrains by deferring precise placement to lower-resolution meshes at distance. It differs from [Voxel Instancing](9449-voxel-instancing.md) by explicitly tying instance density and precision to the terrain's LOD hierarchy rather than a fixed voxel grid, and from [Surface-based Instancing with Noise Modulation](1582-surface-based-instancing-with-noise-modulation.md) by relying on geometric mesh resolution rather than procedural noise for placement distribution.
*   **When to choose**: Use for large-scale terrains where distant instances can tolerate lower placement precision to save CPU/GPU cycles.
*   **Link**: [Block LOD Instancing](12572-block-lod-instancing.md)

### Procedural Object Instancing
Items spawn based on procedural rules via a built-in generator, rather than manual painting. The generator is accessed via the 'generator' property of VoxelInstanceLibraryItem.
*   **Pitfalls**:
    *   Editor Editing: Manual editing of instances in the editor is not supported. Only procedural generation defines spawn locations [S1].
    *   Script API: Custom instance generator API is recent and may change; it is not available to scripts at time of writing [S1].
    *   Streaming Events: VoxelInstancer registers to parent block events to know when to spawn. This is not available to the script API of VoxelLodTerrain [S1].

## Background
This family of techniques is primarily implemented in the Voxel Tools plugin for the Godot Engine. It addresses the need for dynamic, large-scale environment generation in voxel-based games. The system integrates with the `VoxelLodTerrain` node, leveraging its LOD hierarchy and SDF capabilities to optimize instance placement.

## Sources
[S1] Instancing - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/instancing/, fetched 2026-10-27
