# Block LOD Instancing

Also called: LOD-based Spawning

## Overview
Block LOD Instancing is a procedural placement method for voxel terrains that determines instance spawn locations based on the terrain's Level of Detail (LOD) hierarchy rather than fixed world coordinates. It solves the performance bottleneck of generating high-density instances (e.g., grass, rocks) across large terrains by restricting detailed placement to high-resolution chunks and coarse placement to low-resolution chunks. It differs from standard voxel instancing by decoupling instance density from geometric resolution, allowing distant, low-LOD chunks to host simplified or sparse instances while maintaining visual continuity.

## How it works
The method relies on a configuration object, `VoxelInstanceLibraryItem`, which defines the procedural rules for spawning. The core logic maps an integer `lod_index` to specific terrain chunks, ensuring that instances are generated only on chunks matching that resolution level.

### Inputs and Outputs
*   **Input**: `VoxelInstanceLibraryItem` configuration, terrain LOD mesh data, and optionally a `VoxelGenerator` for SDF queries.
*   **Output**: A set of instance transforms (position, rotation, scale) placed on the terrain surface.

### Data Structures
The `VoxelInstanceLibraryItem` contains the following fields:
*   `lod_index` (int): Determines the chunk resolution for spawning.
*   `offset_along_normal` (float): Adjusts position along the surface normal.
*   `snap_to_generator_sdf_enabled` (bool): Toggles SDF-based surface snapping.
*   `persistent` (bool): Indicates if position is saved to the voxel stream.
*   `generator` (object): The procedural logic defining spawn conditions.

### Algorithm Steps
1.  **Chunk Selection**: The system iterates through active terrain chunks. It selects only those chunks whose resolution level matches the item's `lod_index`.
    *   `lod_index = 0` selects LOD0 chunks (closest range, highest resolution).
    *   Higher `lod_index` values select larger, lower-resolution chunks covering a wider range.
2.  **Position Sampling**: For each selected chunk, the system samples the chunk's mesh geometry to determine candidate spawn positions.
3.  **Surface Snapping (Optional)**: If `snap_to_generator_sdf_enabled` is true:
    *   The system queries the `VoxelGenerator` for Signed Distance Field (SDF) values at the candidate position and nearby points.
    *   It requires at least 2 nearby samples to interpolate the surface distance.
    *   The instance position is adjusted to minimize the SDF value, snapping it to the approximate surface. This is an approximation, not exact ray-casting.
4.  **Normal Offset**: The instance position is adjusted by `offset_along_normal` along the surface normal vector. This compensates for floating or sinking caused by mesh imprecision.
5.  **Persistence Check**: If `persistent` is true, the instance ID and position are saved to the voxel stream (e.g., `VoxelStreamSQLite`). If the terrain is edited, persistent instances in modified blocks are retained; non-persistent instances are regenerated.
6.  **Rendering**: Instances are rendered using GPU-instanced `MultiMesh` buffers for performance or standard scene nodes for flexibility. `MultiMesh` items support LOD switching based on distance ratios relative to the terrain LOD view distance.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spawn_instances(item_config, terrain_chunks):
    instances = []
    target_lod = item_config.lod_index
    
    # Select chunks matching the target LOD
    relevant_chunks = [c for c in terrain_chunks if c.lod_level == target_lod]
    
    for chunk in relevant_chunks:
        # Sample mesh vertices or grid points within the chunk
        candidates = chunk.get_surface_samples()
        
        for pos in candidates:
            # Check procedural conditions (density, noise) defined in item_config.generator
            if not item_config.generator.is_valid(pos):
                continue
            
            # Apply SDF snapping if enabled
            if item_config.snap_to_generator_sdf_enabled:
                # Query SDF at pos and neighbors
                sdf_val = item_config.generator.query_sdf(pos)
                # Interpolate using at least 2 nearby samples to find surface
                pos = snap_to_sdf(pos, item_config.generator)
            
            # Apply normal offset
            normal = chunk.get_normal_at(pos)
            pos = pos + normal * item_config.offset_along_normal
            
            # Create instance transform
            instance = create_instance(pos, item_config)
            
            # Handle persistence
            if item_config.persistent:
                save_to_stream(instance.id, instance.position)
            
            instances.append(instance)
            
    return instances
```

### Parameters and Edge Cases
*   **`lod_index`**: Value 0 spawns on LOD0 chunks (closest range). Higher indexes spawn on bigger chunks. Larger indexes are less precise because they operate on lower-resolution meshes, potentially causing instances to float above or sink into the ground near sharp terrain changes (ridges, crevices, caves) [S1].
*   **`offset_along_normal`**: Depends on the asset. Designing assets with part of their bottom sunk into the ground provides a margin of error for placement imprecision [S1].
*   **`snap_to_generator_sdf_enabled`**: Preferably used when `lod_index > 0`, as LOD0 already has maximum precision. It incurs a performance cost due to SDF queries [S1].
*   **Persistence Constraint**: Only specific streams (e.g., `VoxelStreamSQLite`) support saving persistent instances. Manual editing of instance positions is not supported; locations are defined solely by procedural rules [S1].
*   **Terrain Modification**: Snapping does not work correctly if the terrain is modified; instances in edited areas will not regenerate unless set to `persistent` [S1].

## Subtopics

### Offset Along Normal Placement
A variant that adjusts instance placement by offsetting along the surface normal to correct for floating or sinking caused by low-resolution mesh imprecision. It uses the `offset_along_normal` parameter, which depends on the asset design. Designing assets with part of their bottom sunk into the ground provides a margin of error. It is a lightweight alternative to SDF snapping [S1].

## Operating characteristics
*   **Performance**: Significantly faster than full-precision placement for distant objects because it operates on lower-resolution meshes for higher `lod_index` values [S1].
*   **Precision**: Decreases as `lod_index` increases. Larger indexes are less precise, leading to potential visual artifacts (floating/sinking) near sharp terrain features [S1].
*   **Memory**: Persistent instances require storage in the voxel stream, increasing memory usage. Non-persistent instances are regenerated each frame or chunk update, consuming CPU/GPU time but less storage [S1].
*   **Use Case**: Ideal for large-scale terrain with high-density vegetation or debris where performance is critical and minor placement inaccuracies are acceptable at a distance. Not suitable for precise architectural placement or areas with frequent terrain modification unless persistence is used [S1].

## Background
Block LOD Instancing is part of the Voxel Tools ecosystem, specifically designed for procedural terrain generation in real-time applications. It leverages the inherent LOD structure of voxel terrains to optimize instance generation. The method was developed to address the scalability issues of traditional instancing systems when applied to vast, dynamically generated voxel worlds.

## Sources
[S1] Instancing - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/instancing/, fetched 2026-10-27
