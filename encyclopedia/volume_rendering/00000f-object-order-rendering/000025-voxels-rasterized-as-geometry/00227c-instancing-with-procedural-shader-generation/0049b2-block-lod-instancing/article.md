# Block LOD Instancing

Also called: LOD-based Spawning

## Overview
Block LOD Instancing is a voxel terrain instancing strategy where the spatial resolution of the spawning grid is determined by the terrain's Level of Detail (LOD) index. It solves the performance bottleneck of generating high-frequency instances (like grass or rocks) across large, detailed terrains by deferring precise placement to lower-resolution meshes at distance. It differs from [Voxel Instancing](9449-voxel-instancing.md) by explicitly tying instance density and precision to the terrain's LOD hierarchy rather than a fixed voxel grid, and from [Surface-based Instancing with Noise Modulation](1582-surface-based-instancing-with-noise-modulation.md) by relying on geometric mesh resolution rather than procedural noise for placement distribution.

## How it works
**Inputs and Outputs:**
*   **Input:** Voxel terrain data with multiple LOD levels; `VoxelInstanceLibraryItem` configuration containing `lod_index`; instance assets (meshes).
*   **Output:** Transformed instance matrices positioned on the terrain surface.

**Data Structures:**
*   `VoxelInstanceLibraryItem`: Contains `lod_index` (integer).
    *   `lod_index = 0`: Spawns on highest-resolution chunks (LOD0).
    *   `lod_index > 0`: Spawns on progressively lower-resolution chunks (larger cell size).

**Steps:**
1.  **Determine Spawn Resolution:** The instancer selects the terrain mesh resolution based on the item's `lod_index`. Higher indices correspond to coarser meshes covering larger spatial ranges.
2.  **Grid Generation:** Instances are placed on the grid points of the selected LOD mesh. This allows for rapid generation at large distances where full-precision geometry is not yet loaded or is too expensive to process.
3.  **Position Assignment:** The instance position is set to the vertex or grid point of the coarse LOD mesh.
4.  **Orientation:** The instance is aligned to the surface normal of the coarse mesh. If the terrain is spherical, the `up_mode` is set to `Sphere` to align items with the local origin [S1].
5.  **Refinement (Optional):** If `snap_to_generator_sdf_enabled` is active, the position is refined by querying the `VoxelGenerator`'s Signed Distance Field (SDF) at the coarse position to find the true surface, assuming the coarse mesh is a good starting point. This requires at least 2 nearby SDF samples [S1].

**Parameters:**
*   `lod_index`: Integer. Determines the chunk resolution.
    *   Typical values: `0` for close-range/high-precision; `1, 2, ...` for far-range/low-precision.
    *   Strategy: Use high `lod_index` for large items visible from afar; use low `lod_index` for small items [S1].
*   `offset_along_normal`: Float. Offsets the instance along the surface normal to correct for floating/sinking. Depends on asset design (e.g., sinking the bottom into the ground) [S1].
*   `snap_to_generator_sdf_enabled`: Boolean. Enables SDF-based snapping. Slows generation but improves accuracy for `lod_index > 0` [S1].

**Pitfalls:**
*   **Placement Imprecision:** Larger `lod_index` values result in lower precision. Instances may float above or sink into the ground near sharp terrain features (ridges, crevices, caves) because they are placed on a simplified mesh [S1].
*   **Abrupt Disappearance:** Instances may vanish abruptly at LOD boundaries. Mitigation: Use custom shaders to fade meshes based on distance and use ground textures similar to the instance color [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spawn_instances(terrain, item_config):
    lod_idx = item_config.lod_index
    # Select mesh resolution based on LOD index
    # Higher lod_idx -> coarser mesh
    mesh = terrain.get_lod_mesh(lod_idx)
    
    for vertex in mesh.vertices:
        # Base position from coarse mesh
        pos = vertex.position
        normal = vertex.normal
        
        # Apply offset to mitigate floating/sinking
        pos += normal * item_config.offset_along_normal
        
        # Optional: Refine position using SDF if enabled
        if item_config.snap_to_generator_sdf_enabled:
            # Query VoxelGenerator SDF at pos
            # Approximate surface by finding zero-crossing or closest point
            # Requires at least 2 nearby samples
            refined_pos = snap_to_sdf(terrain.generator, pos)
            pos = refined_pos
            
        # Orientation alignment
        if terrain.mode == "Sphere":
            # Align with local origin
            up_vector = normalize(pos)
        else:
            up_vector = normal
            
        matrix = build_transform_matrix(pos, up_vector, vertex.tangent)
        render_instance(item_config.asset, matrix)
```

## Subtopics

### Offset Along Normal Placement
Adjusts instance placement by offsetting along the surface normal to correct for floating or sinking caused by low-resolution mesh imprecision. It depends on the asset design, where sinking part of the asset's bottom into the ground provides a margin of error. The parameter `offset_along_normal` is used in the generator [S1].

## Operating characteristics
*   **Time Complexity:** Faster than full-precision instancing for large areas because it operates on lower-resolution meshes for distant instances. SDF snapping adds computational cost [S1].
*   **Memory Complexity:** Depends on the terrain LOD storage. No additional instance buffers beyond standard instancing.
*   **Strengths:** Efficiently handles large terrains by reducing instance generation cost at distance; leverages existing terrain LOD system.
*   **Limitations:** Placement precision decreases with higher LOD indices; instances may misalign with sharp terrain features; requires careful asset design (offsets) to avoid visual artifacts.
*   **When to Use:** Large-scale terrain with many small instances (grass, rocks) where perfect precision at distance is not critical.
*   **When Not to Use:** Scenes requiring high-precision placement across all distances or near sharp geometric features without SDF snapping.

## Background
Block LOD Instancing is part of the Voxel Tools ecosystem, specifically designed for voxel-based terrain rendering. It integrates with the terrain's LOD system to balance performance and visual fidelity. The method is documented in the Voxel Tools documentation [S1].

## Sources
[S1] Instancing - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/instancing/, fetched 2026-10-27
