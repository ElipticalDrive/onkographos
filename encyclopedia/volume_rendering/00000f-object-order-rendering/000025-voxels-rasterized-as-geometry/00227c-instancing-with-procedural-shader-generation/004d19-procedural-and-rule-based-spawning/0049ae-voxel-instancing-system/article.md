# Voxel Instancing System

Also called: VoxelInstancer

## Overview
The Voxel Instancing System is a method for procedurally spawning numerous 3D objects (instances) onto a voxel terrain surface. It solves the problem of efficiently rendering natural features like grass, rocks, and trees by decoupling instance generation from the voxel chunk structure, allowing for high-density placement. It distinguishes itself from sibling methods like [Block LOD Instancing](12572-block-lod-instancing.md) by supporting two distinct instance types: highly optimized `Multimesh` instances for bulk rendering and flexible `Scene` instances for interactive objects, while utilizing a procedural generator with Signed Distance Field (SDF) snapping for surface alignment.

## How it works

### Inputs and Outputs
*   **Input**: A voxel terrain with an associated `VoxelGenerator` (must support series generation for SDF snapping, e.g., `VoxelGeneratorGraph`) and a `VoxelStream` (e.g., `VoxelStreamSQLite` for persistence).
*   **Input**: A `VoxelInstanceLibrary` resource defining spawn rules, item types, and LOD configurations.
*   **Input**: A `VoxelInstancer` node linked to the terrain and library.
*   **Output**: A set of 3D instances positioned on the terrain surface, respecting LOD constraints and persistence rules.

### Data Structures

**VoxelInstanceLibrary**
A resource assigned to the `VoxelInstancer.library` property. It contains a list of `VoxelInstanceLibraryItem` objects.

**VoxelInstanceLibraryItem**
Base class for spawn rules. Fields:
*   `lod_index` (int): Determines the terrain chunk resolution level for spawning. `0` is high-res/close; higher indices are low-res/distant.
*   `offset_along_normal` (float): Distance to offset the instance along the surface normal after snapping.
*   `snap_to_generator_sdf_enabled` (bool): Toggle for SDF-based surface snapping.
*   `persistent` (bool): If true, instance position/ID is saved to the stream and retained during terrain edits.
*   `generator` (object): Procedural logic evaluating spawn conditions (e.g., noise thresholds).

**VoxelInstanceLibraryMultiMeshItem**
Inherits from `VoxelInstanceLibraryItem`. Fields:
*   `mesh` (Mesh): LOD0 mesh.
*   `lod1` (Mesh): LOD1 mesh.
*   `lod2` (Mesh): LOD2 mesh.
*   `lod3` (Mesh): LOD3 mesh.
*   `mesh_lod_settings` (object): Contains ratios (0-1) relative to terrain LOD view distance to switch between meshes. Includes option to hide instances beyond max LOD distance.

**VoxelInstanceLibrarySceneItem**
Inherits from `VoxelInstanceLibraryItem`. Fields:
*   `scene` (PackedScene): The scene resource to instantiate. Root must be `Node3D`.

### Algorithm Steps

1.  **Library Initialization**: The `VoxelInstancer` loads the `VoxelInstanceLibrary`. For each `VoxelInstanceLibraryItem`, the system prepares the spawn logic. If the item is a `MultiMeshItem` derived from a scene, the scene is converted to a `MultiMesh` format at runtime or editor time [S1].
2.  **Chunk Iteration**: The system iterates through active terrain chunks. For each chunk, it calculates the chunk's `lod_level`.
3.  **Eligibility Check**: For each item in the library, the system checks if `chunk.lod_level == item.lod_index`. Only chunks matching the item's LOD index are eligible for spawning that item [S1, Parent].
4.  **Candidate Generation**: The item's `generator` evaluates procedural conditions (e.g., density, noise) at candidate positions within the eligible chunk. Manual position editing is not supported; positions are defined solely by these rules [Parent].
5.  **Surface Snapping (Optional)**: If `snap_to_generator_sdf_enabled` is true:
    *   The system queries the `VoxelGenerator` for Signed Distance Field (SDF) values at the candidate position and nearby points.
    *   **Requirement**: The generator must support series generation (arbitrary floating-point queries) [S1].
    *   If at least 2 nearby samples are available, the system interpolates to find the zero-crossing (surface) and snaps the instance position closer to the surface. This is an interpolation approximation, not exact ray-casting [Parent].
    *   *Pitfall*: Snapping may cause instances to "bury" or "float" far away if low-resolution meshes are active, as the SDF estimation may not align with the coarse mesh geometry [S1].
6.  **Normal Offset**: The instance position is adjusted by `offset_along_normal` along the interpolated surface normal vector to compensate for mesh imprecision [Parent].
7.  **Upward Alignment**: The instance rotation is aligned to the terrain's upward direction. If `VoxelInstancer.up_mode` is set to 'Sphere mode', alignment is based on the local origin of the terrain (suitable for planets) [S1].
8.  **Persistence Handling**:
    *   If `persistent` is true, the system checks the `VoxelStream`. If the instance ID exists in the stream for the current block, the saved position is used.
    *   If the terrain block has been edited, persistent instances are retained; non-persistent instances are regenerated [Parent].
    *   *Limitation*: If the terrain is modified, non-persistent instances in the edited area are regenerated. Persistent instances in edited areas are saved but do not regenerate their positions based on new terrain geometry unless explicitly handled by the stream logic [S1, Parent].
9.  **Instance Creation**:
    *   For `MultiMeshItem`: A `MultiMeshInstance` is created or updated with the calculated transform. Mesh LOD is selected based on distance ratios [S1].
    *   For `SceneItem`: A new `Node3D` instance of the scene is created as a child of the `VoxelInstancer`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_instancing_system(terrain, library, instancer):
    """
    Main loop for voxel instancing.
    terrain: Object with get_active_chunks() and get_sdf(position)
    library: VoxelInstanceLibrary containing list of items
    instancer: VoxelInstancer node managing rendering
    """
    
    # 1. Iterate through active terrain chunks
    for chunk in terrain.get_active_chunks():
        chunk_lod = chunk.lod_level
        
        # 2. Filter items eligible for this chunk's LOD
        eligible_items = [item for item in library.items if item.lod_index == chunk_lod]
        
        for item in eligible_items:
            
            # 3. Procedural Generation
            candidates = item.generator.evaluate(chunk.bounds)
            
            for candidate in candidates:
                
                # 4. Surface Snapping
                final_position = candidate.position
                final_normal = Vector3(0, 1, 0) # Default up
                
                if item.snap_to_generator_sdf_enabled:
                    # Query SDF at candidate and neighbors
                    sdf_val = terrain.get_sdf(candidate.position)
                    sdf_neighbors = [terrain.get_sdf(candidate.position + offset) for offset in NEIGHBOR_OFFSETS]
                    
                    # Interpolate to surface if enough samples
                    if len(sdf_neighbors) >= 2:
                        final_position = interpolate_surface(candidate.position, sdf_val, sdf_neighbors)
                        final_normal = compute_normal(candidate.position, sdf_neighbors)
                
                # 5. Normal Offset
                final_position += final_normal * item.offset_along_normal
                
                # 6. Upward Alignment
                if instancer.up_mode == 'SPHERE':
                    # Align to local terrain origin
                    up_vector = -final_position.normalized()
                else:
                    up_vector = Vector3(0, 1, 0)
                    
                final_rotation = align_to_up(final_rotation, up_vector)
                
                # 7. Persistence Check
                if item.persistent:
                    saved_pos = instancer.stream.get_instance_position(item.id, chunk.id)
                    if saved_pos is not None:
                        final_position = saved_pos
                        # Note: Does not regenerate position on edit, retains saved pos
                    else:
                        # New instance, save it
                        instancer.stream.save_instance(item.id, chunk.id, final_position)
                
                # 8. Create/Update Instance
                if item.type == 'MULTIMESH':
                    mesh_lod = select_mesh_lod(item.meshes, final_position.distance_to(camera))
                    instancer.update_multimesh_item(item.id, final_position, final_rotation, mesh_lod)
                elif item.type == 'SCENE':
                    instancer.create_scene_instance(item.scene, final_position, final_rotation)
```

### Parameters and Typical Values
*   `lod_index`: Integer. `0` for close/high-res, higher for distant/low-res.
*   `offset_along_normal`: Float. Small positive value (e.g., 0.01-0.1) to prevent z-fighting or sinking.
*   `up_mode`: Enum. 'Sphere mode' for planets, 'World Up' for flat terrain.
*   `Mesh LOD Ratios`: Floats between 0 and 1, relative to terrain LOD view distance.

### Edge Cases and Pitfalls
*   **Cyclic Reference Crash**: Loading a library from a scene that is also included in the library causes a crash [S1].
*   **Performance Issue**: Changing `MultiMesh.mesh` to a different LOD causes performance issues until a specific PR is merged [S1].
*   **Snapping Limitations**: SDF snapping requires a generator supporting series generation (e.g., `VoxelGeneratorGraph`). It does not work correctly if the terrain is modified, as estimations do not account for edits unless instances are persistent [S1, Parent].
*   **Low-Res Artifacts**: Instances may appear buried or floating when low-resolution meshes are active due to SDF/mesh mismatch [S1].
*   **Scene Instance Limits**: High numbers of `SceneItem` instances can cause freezes or crashes due to overhead. Default density is lower than multimesh [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time**: Proportional to the number of active chunks and the density of instances per chunk. Scene instances have significantly higher per-instance overhead than Multimesh instances [S1].
*   **Memory**: Multimesh instances are memory-efficient, allowing extremely numerous objects. Scene instances consume more memory per instance due to full scene graph duplication [S1].

### Measured Performance
*   **Multimesh**: Can handle extremely numerous instances with optional collision. Suitable for large-scale natural spawning [S1].
*   **Scene**: Slower. Recommended for low numbers. Complex scenes can lead to freeze/crash [S1].

### Scaling
*   Scales with terrain LOD. Higher LOD indices spawn on larger chunks, covering larger ranges but with lower detail [S1].
*   Mesh LOD reduces vertex count over distance within visible meshes, offering an alternative to Visibility Ranges which requires duplicating data [S1].

### Strengths and Limitations
*   **Strengths**: High performance for natural features (grass, rocks). Flexible instance types (Multimesh vs Scene). Procedural generation with SDF snapping. Persistence support.
*   **Limitations**: Not suited for complex man-made structures. Manual editing of instance positions is not supported. Snapping fails on edited terrain for non-persistent instances. Requires specific generator support for SDF.

### When to Use
*   Use for natural spawning (grass, rocks, trees, semi-random foliage) [S1].
*   Use Multimesh for bulk, non-interactive objects.
*   Use Scene instances for interactive objects (scripts, animations) where performance allows.

### When Not to Use
*   Do not use for complex man-made structures like houses or villages (unless scene instances are sparse and simple) [S1].
*   Do not use if terrain editing is frequent and persistent instances are not desired, as snapping will be inaccurate [S1].

## Background

### Origin
The Voxel Instancing System is part of the Voxel Tools library for the Godot Engine. It was developed to address the lack of efficient instancing in Godot 3, leveraging Godot 4's automatic mesh LOD capabilities while providing finer control over mesh and material usage per LOD [S1].

### History and Development
*   Initially designed to support natural spawning on voxel terrains.
*   Added support for `VoxelGeneratorGraph` to enable SDF snapping.
*   Added `VoxelStreamSQLite` support for persistent instance saving.
*   Development is ongoing, with API stability for custom generators noted as a future concern [S1].

### Usage
Primarily used in Godot-based voxel games for terrain decoration. It is integrated into the Godot editor via the `VoxelInstancer` node and `VoxelInstanceLibrary` resource [S1].

## Sources

[S1] Instancing - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/instancing/, fetched 2026-10-27
