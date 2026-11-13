# Voxel Instancing System

Also called: VoxelInstancer

## Overview
The Voxel Instancing System is a procedural spawning mechanism for voxel terrains, designed primarily for natural elements like grass, rocks, and trees [S1]. It solves the problem of efficiently rendering thousands of distinct objects by evaluating spawn rules against terrain chunks and placing instances either as high-performance `MultiMesh` batches or flexible `Scene` nodes [S1]. It distinguishes itself from sibling methods by its tight integration with the voxel terrain's LOD system and its ability to snap instances to procedural surface approximations using SDF interpolation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A `VoxelInstancer` node, a `VoxelInstanceLibrary` resource, a `VoxelGenerator` (typically `VoxelGeneratorGraph` for SDF snapping), and a `VoxelStream` (typically `VoxelStreamSQLite` for persistence) [S1].
*   **Output:** A set of spawned instances (`MultiMeshInstance3D` or `Scene` nodes) positioned on the voxel surface [S1].

**Data Structures:**
1.  **`VoxelInstanceLibrary`**: A resource assigned to `VoxelInstancer.library`. It contains a list of `VoxelInstanceLibraryItem` entries [S1].
2.  **`VoxelInstanceLibraryItem`**: Base item configuration.
    *   `lod_index`: Integer. Determines the terrain LOD level at which the item spawns (0 = high-res/close, higher = low-res/far) [S1].
    *   `offset_along_normal`: Float. Distance to displace the instance from the surface along the normal [S1].
    *   `snap_to_generator_sdf_enabled`: Boolean. Enables SDF-based surface snapping [S1].
    *   `persistent`: Boolean. If true, instance position/ID is saved to the stream [S1].
3.  **`VoxelInstanceLibraryMultiMeshItem`**: Inherits from `VoxelInstanceLibraryItem`.
    *   `mesh`: `Mesh`. LOD0 mesh [S1].
    *   `lod1`, `lod2`, `lod3`: `Mesh`. Subsequent LOD levels [S1].
    *   `mesh_lod_settings`: Object containing ratios (0-1) relative to terrain LOD view distance for switching between LODs [S1].
4.  **`VoxelInstanceLibrarySceneItem`**: Inherits from `VoxelInstanceLibraryItem`.
    *   `scene`: `PackedScene`. The scene to instantiate [S1].
    *   Root of the scene must be `Node3D` or a derivative [S1].

**Algorithm Steps:**
1.  **Chunk Eligibility**: Iterate through loaded voxel chunks. For each `VoxelInstanceLibraryItem`, check if `chunk.lod_level == item.lod_index` [Parent].
2.  **Candidate Generation**: The item's `generator` evaluates conditions (e.g., noise thresholds) at candidate positions within eligible chunks [Parent].
3.  **Surface Snapping (Optional)**:
    *   If `snap_to_generator_sdf_enabled` is true, query the `VoxelGenerator` for Signed Distance Field (SDF) values at the candidate position and nearby samples [Parent].
    *   Interpolate between samples to find the zero-crossing (surface) [Parent].
    *   *Pitfall*: This is an interpolation approximation, not exact ray-casting. It may fail or produce errors if the terrain is edited after generation, as the generator's cached data may not reflect recent changes [S1].
    *   *Constraint*: Requires a generator supporting series generation (querying arbitrary floating-point positions). At time of writing, only `VoxelGeneratorGraph` supports this [S1].
4.  **Normal Offset**: Displace the instance position along the interpolated surface normal by `offset_along_normal` [Parent].
5.  **Alignment**:
    *   If `up_mode` is 'Sphere mode', rotate instances to align with the local origin of the terrain (suitable for planets) [Parent].
    *   Otherwise, align with a fixed world up vector [Parent].
6.  **Instance Creation**:
    *   If `MultiMesh`: Create a `MultiMeshInstance3D`. Assign the appropriate LOD mesh based on distance and `mesh_lod_settings` ratios [S1].
    *   If `Scene`: Instantiate the `PackedScene` as a child of `VoxelInstancer` [S1].
7.  **Persistence**:
    *   If `persistent` is true and a `VoxelStream` is active, save the instance's ID and position [Parent].
    *   Editing the terrain marks these instances as modified, preventing regeneration and saving their new positions [Parent].
    *   *Constraint*: Only `VoxelStreamSQLite` supports saving persistent instances at time of writing [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_terrain_chunks(voxel_instancer, chunks):
    library = voxel_instancer.library
    generator = voxel_instancer.generator
    stream = voxel_instancer.stream
    up_mode = voxel_instancer.up_mode  # 'Sphere mode' or 'World Up'
    
    for chunk in chunks:
        for item in library.items:
            # Step 1: Chunk Eligibility
            if chunk.lod_level != item.lod_index:
                continue
            
            # Step 2: Candidate Generation
            candidates = item.generator.evaluate(chunk)
            
            for candidate in candidates:
                position = candidate.position
                
                # Step 3: Surface Snapping
                if item.snap_to_generator_sdf_enabled:
                    # Requires VoxelGeneratorGraph for arbitrary float queries
                    sdf_val = generator.query_sdf(position)
                    # Interpolate to find zero-crossing (approximate surface)
                    # [G] Implementation detail: typically involves sampling neighbors
                    # and linear interpolation to find t where sdf(t) = 0.
                    normal = generator.query_sdf_gradient(position)
                    position = position + normal * item.offset_along_normal
                else:
                    # [G] Default behavior: use chunk surface or center if no SDF
                    position = position + Vector3(0, item.offset_along_normal, 0)
                
                # Step 5: Alignment
                if up_mode == 'Sphere mode':
                    # Align with local terrain origin
                    rotation = align_to_sphere_normal(position)
                else:
                    rotation = align_to_world_up()
                
                # Step 6: Instance Creation
                if isinstance(item, VoxelInstanceLibraryMultiMeshItem):
                    mesh = select_lod_mesh(item, chunk, position)
                    # [G] Add to MultiMesh batch
                    add_multimesh_instance(position, rotation, mesh)
                elif isinstance(item, VoxelInstanceLibrarySceneItem):
                    scene_instance = item.scene.instantiate()
                    scene_instance.position = position
                    scene_instance.rotation = rotation
                    voxel_instancer.add_child(scene_instance)
                
                # Step 7: Persistence
                if item.persistent and stream:
                    stream.save_instance(item.id, position)

def select_lod_mesh(item, chunk, position):
    # [G] Logic to select mesh based on distance and item.mesh_lod_settings ratios
    # relative to chunk's view distance.
    return item.mesh # Simplified for brevity
```

**Parameters:**
*   `up_mode`: String. Typical value: 'Sphere mode' for planets [S1].
*   `lod_index`: Integer. Range based on terrain LODs. 0 is closest/high-res [S1].
*   `offset_along_normal`: Float. Placement adjustment [S1].

**Edge Cases and Pitfalls:**
*   **Cyclic Reference Crash**: If a scene is added to the library and the library is loaded from that same scene, Godot crashes due to cyclic reference [S1].
*   **Performance Issue**: Changing `MultiMesh.mesh` to a different LOD causes a performance issue, pending a PR merge into Godot [S1].
*   **Terrain Editing**: SDF snapping does not account for edited areas. Workaround: set instances as `persistent` so editing marks them as modified and saves positions instead of re-generating [S1].
*   **Burying/Floatation**: Instances may bury or float far away when low-resolution meshes are active, as they are moved closer to the high-resolution surface approximation [S1].
*   **Scene Complexity**: Using complex scenes with `VoxelInstanceLibrarySceneItem` can cause freeze or crash if too many instances are added [S1].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of eligible chunks and candidates. MultiMesh creation is O(1) per instance (batched), while Scene instantiation is O(N) where N is scene complexity [S1].
*   **Memory Complexity**: MultiMesh instances are memory-efficient, storing only transform and mesh index. Scene instances consume memory proportional to the scene graph size [S1].
*   **Performance**: MultiMesh is extremely fast and supports collision. Scene instances are slow and should be tuned to low numbers [S1].
*   **Strengths**: High performance for natural spawning; integrates with terrain LOD; supports persistence.
*   **Limitations**: Not suited for complex man-made structures; SDF snapping is approximate; requires specific generators for snapping; limited editor editing support [S1].
*   **When to use**: Natural spawning (grass, rocks, trees).
*   **When not to use**: Complex man-made structures; scenarios requiring precise, dynamic terrain interaction without persistence [S1].

## Background
The Voxel Instancing System is part of the Voxel Tools plugin for Godot. It was developed to address the lack of efficient instancing in earlier versions of Godot (Godot 3) and leverages Godot 4's automatic mesh LOD capabilities while providing more control over mesh and material assignment per LOD [S1]. It is used in voxel-based games for terrain decoration [S1].

## Sources
[S1] Instancing - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/instancing/, fetched 2026-10-27
