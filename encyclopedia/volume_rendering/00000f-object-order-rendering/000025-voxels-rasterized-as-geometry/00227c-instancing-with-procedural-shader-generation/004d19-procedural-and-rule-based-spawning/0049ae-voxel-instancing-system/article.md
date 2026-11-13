# Voxel Instancing System

Also called: VoxelInstancer

## Overview
The Voxel Instancing System is a procedural spawning mechanism designed for natural environments, such as grass, rocks, trees, and semi-random foliage [S1]. It solves the problem of efficiently populating large voxel terrains with thousands of objects by leveraging GPU-accelerated `MultiMesh` rendering for high-performance bulk placement, while also supporting CPU-side `Scene` instances for complex, interactive entities [S1]. It distinguishes itself from general geometry instancing by integrating tightly with the voxel terrain's Level of Detail (LOD) system and Signed Distance Field (SDF) surface snapping, ensuring instances align with uneven terrain geometry without requiring manual placement [S1].

## How it works
The system operates by evaluating spawn rules defined in a `VoxelInstanceLibrary` against eligible terrain chunks. It supports two primary instance types: `VoxelInstanceLibraryMultiMeshItem` for high-performance, static geometry and `VoxelInstanceLibrarySceneItem` for dynamic, scriptable entities [S1].

### Inputs and Outputs
*   **Inputs**:
    *   `VoxelInstancer` node: The controller managing the library and generation state.
    *   `VoxelInstanceLibrary`: Resource containing spawn rules (`VoxelInstanceLibraryItem`).
    *   `VoxelGenerator`: Provides terrain geometry and SDF data. Note: SDF snapping requires a generator supporting series generation (arbitrary floating-point queries), such as `VoxelGeneratorGraph` [S1].
    *   `VoxelStream`: Optional resource for persisting instance positions (e.g., `VoxelStreamSQLite`) [S1].
*   **Outputs**:
    *   Rendered instances positioned on the terrain surface.
    *   Persistent data saved to the stream if `persistent` is enabled.

### Data Structures
The core data structure is the `VoxelInstanceLibrary`, assigned to the `VoxelInstancer.library` property [S1]. It contains a list of `VoxelInstanceLibraryItem` objects.

**VoxelInstanceLibraryItem**
*   `lod_index` (int): Maps the instance to a specific terrain LOD chunk resolution. Range depends on terrain LOD configuration. Index 0 corresponds to high-resolution/close-range chunks; higher indices correspond to larger, distant chunks [S1].
*   `offset_along_normal` (float): Displacement distance along the surface normal after snapping to prevent z-fighting or sinking [S1].
*   `snap_to_generator_sdf_enabled` (bool): Toggle for SDF-based surface snapping [S1].
*   `persistent` (bool): If true, instance positions are saved to the `VoxelStream` and retained across terrain edits [S1].
*   `generator` (VoxelInstanceGenerator): Procedural rules defining where instances spawn (e.g., noise thresholds) [S1].

**VoxelInstanceLibraryMultiMeshItem** (inherits from `VoxelInstanceLibraryItem`)
*   `mesh` (Mesh): LOD0 mesh.
*   `lod1` (Mesh): LOD1 mesh.
*   `lod2` (Mesh): LOD2 mesh.
*   `lod3` (Mesh): LOD3 mesh.
*   `mesh_lod_settings` (Dictionary/Struct): Contains ratios (typically 0-1) relative to the terrain LOD view distance, defining switch distances between LODs [S1].
*   `hide_beyond_max_distance` (bool): Hides instances beyond the last LOD's range, though they are still generated [S1].

**VoxelInstanceLibrarySceneItem** (inherits from `VoxelInstanceLibraryItem`)
*   `scene` (PackedScene): The scene resource to instantiate. The root must be `Node3D` or a derivative [S1].
*   *Note*: Scene instances are significantly slower and prone to freezes/crashes if too many complex instances are spawned [S1].

### Setup Methods
Instances can be configured via the inspector or by converting existing scenes:
1.  **Inspector**: Manually set mesh properties and generator parameters [S1].
2.  **Runtime Conversion**: Assign a `scene` property to a `VoxelInstanceLibraryMultiMeshItem`. The scene is converted to a MultiMesh at runtime and linked, updating if the source scene changes [S1].
3.  **Editor Conversion**: Use the "Setup from scene" button. This converts the scene in the editor, copying embedded assets into the resource file. It is not linked to the original scene [S1].

**Scene Structure for Conversion**:
The source scene must follow this hierarchy for proper LOD and collision extraction:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```text
- PhysicsBody (StaticBody, RigidBody...)
    - MeshInstance_LOD0
    - MeshInstance_LOD1
    - MeshInstance_LOD2
    - MeshInstance_LOD3
    - CollisionShape1
    - CollisionShape2
```
Materials must be set via `material_override` on `MeshInstance` nodes or directly on the mesh resource. Surface material properties on the `MeshInstance` node are not supported [S1].

### Algorithm Steps
1.  **Chunk Eligibility**: The system iterates through loaded voxel chunks. For each `VoxelInstanceLibraryItem`, it checks if `chunk.lod_level == item.lod_index` [S1].
2.  **Candidate Generation**: The item's `generator` evaluates conditions (e.g., noise thresholds) at candidate positions within eligible chunks [S1].
3.  **Surface Snapping (Optional)**:
    *   If `snap_to_generator_sdf_enabled` is true, the system queries the `VoxelGenerator` for SDF values at the candidate position and nearby samples [S1].
    *   It interpolates between samples to find the zero-crossing (surface) [S1].
    *   *Pitfall*: This is an interpolation approximation, not exact ray-casting. It may fail or produce errors if the terrain is edited after generation, as the generator's cached data may not reflect recent changes [S1].
4.  **Normal Offset**: The instance position is displaced along the interpolated surface normal by `offset_along_normal` [S1].
5.  **Alignment**:
    *   If `up_mode` is 'Sphere mode', instances are rotated to align with the local origin of the terrain (suitable for planets) [S1].
    *   Otherwise, they align with a fixed world up vector [S1].
6.  **Instance Creation**:
    *   For `MultiMeshItem`: A `MultiMeshInstance` is created. The appropriate LOD mesh is selected based on distance ratios defined in `mesh_lod_settings` [S1].
    *   For `SceneItem`: A `Node3D` instance of the scene is created as a child of the `VoxelInstancer` [S1].
7.  **Persistence**: If `persistent` is true and a `VoxelStream` is active, the instance's ID and position are saved. Editing the terrain marks these instances as modified, preventing regeneration and saving their new positions [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_terrain_chunks(voxel_instancer, chunk_list):
    library = voxel_instancer.library
    generator = voxel_instancer.generator
    stream = voxel_instancer.stream
    up_mode = voxel_instancer.up_mode  # 'Sphere' or 'World'
    
    for chunk in chunk_list:
        for item in library.items:
            # Step 1: LOD Eligibility
            if chunk.lod_level != item.lod_index:
                continue
            
            # Step 2: Candidate Generation
            candidates = item.generator.evaluate(chunk)
            
            for candidate in candidates:
                position = candidate.position
                normal = candidate.normal
                
                # Step 3: Surface Snapping
                if item.snap_to_generator_sdf_enabled:
                    # Requires VoxelGeneratorGraph for series generation
                    sdf_samples = generator.query_sdf_series(position, radius=chunk.resolution)
                    if len(sdf_samples) >= 2:
                        # Interpolate to find zero-crossing
                        surface_pos, surface_normal = interpolate_sdf_zero_crossing(sdf_samples)
                        position = surface_pos
                        normal = surface_normal
                    else:
                        # Fallback if insufficient samples
                        pass
                
                # Step 4: Normal Offset
                position += normal * item.offset_along_normal
                
                # Step 5: Alignment
                if up_mode == 'Sphere':
                    # Align to local origin of terrain
                    rotation = align_to_sphere_normal(position, terrain_origin)
                else:
                    rotation = align_to_world_up(normal)
                
                # Step 6: Instance Creation
                if isinstance(item, VoxelInstanceLibraryMultiMeshItem):
                    # Select LOD based on distance ratios
                    lod_mesh = select_lod_mesh(item.meshes, item.lod_settings, position)
                    create_multimesh_instance(lod_mesh, position, rotation)
                elif isinstance(item, VoxelInstanceLibrarySceneItem):
                    create_scene_instance(item.scene, position, rotation)
                
                # Step 7: Persistence
                if item.persistent and stream:
                    stream.save_instance(item.id, position, rotation)
```

### Edge Cases and Pitfalls
*   **Cyclic Reference Crash**: Loading a `VoxelInstanceLibrary` from a scene that is also included in the library causes a cyclic reference crash in Godot [S1].
*   **Performance Issue**: Changing `MultiMesh.mesh` to a different LOD at runtime causes performance issues; a fix is pending a PR merge into Godot [S1].
*   **SDF Limitations**: SDF snapping does not account for terrain edits made after generation. Persistent instances are recommended to avoid regeneration errors in edited areas [S1].
*   **Far-Field Burying**: Instances may appear buried or floating far away when low-resolution meshes are active, as they are moved closer to the high-resolution surface approximation [S1].
*   **Generator Support**: SDF snapping only works with generators supporting series generation (e.g., `VoxelGeneratorGraph`). Standard voxel chunk generators do not support arbitrary floating-point queries [S1].
*   **Stream Support**: Only `VoxelStreamSQLite` supports saving persistent instances at the time of writing [S1].

## Operating characteristics
*   **Performance**: `MultiMesh` instances are extremely fast and efficient, suitable for thousands of objects. `Scene` instances are significantly slower and can cause freezes or crashes if too many complex instances are added [S1].
*   **Memory**: `MultiMesh` LODs reduce vertex count over distance. Using `Visibility Ranges` instead of MultiMesh LODs requires duplicating data and increasing scene objects to process, which is less efficient [S1].
*   **Scaling**: The system scales with the terrain's LOD system. Higher `lod_index` values spawn on larger chunks, covering a larger range but with lower detail [S1].
*   **Strengths**: High performance for natural spawning; tight integration with terrain LOD and SDF; supports persistence for edited terrains.
*   **Limitations**: Not suited for complex man-made structures; SDF snapping is an approximation; scene instances are resource-heavy; limited generator support for SDF.

## Background
The Voxel Instancing System is part of the Voxel Tools plugin for the Godot Engine. It was developed to address the lack of efficient procedural instancing in Godot 3, leveraging Godot 4's automatic mesh LOD capabilities while providing finer control over mesh and material selection per LOD [S1]. It is primarily used in voxel-based games for natural terrain decoration [S1].

## Sources
[S1] Instancing - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/instancing/, fetched 2026-10-27
