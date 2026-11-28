# Block LOD Instancing

Also called: LOD-based Spawning

## Overview
Block LOD Instancing is a method for spawning voxel instances based on the resolution level (LOD) of the underlying terrain chunks. It solves the problem of efficiently populating large voxel worlds by assigning instance types to specific distance ranges via chunk resolution. It differs from standard instancing by tying spawn eligibility directly to the terrain's hierarchical LOD structure rather than a fixed global radius, allowing larger items to spawn on distant, low-resolution chunks while smaller items populate high-resolution areas.

## How it works
**Inputs and Outputs:**
*   **Input:** A `VoxelInstanceLibrary` containing items with a `lod_index` property; a voxel terrain with hierarchical LOD chunks.
*   **Output:** A set of transformed instances (position, rotation, scale) placed on the terrain surface.

**Data Structures:**
*   `VoxelInstanceLibraryItem`: Contains `lod_index` (integer).
    *   `lod_index = 0`: Spawns on LOD0 chunks (highest resolution, closest range).
    *   `lod_index > 0`: Spawns on progressively lower-resolution, larger chunks.

**Steps:**
1.  **Chunk Eligibility:** Iterate through all loaded voxel chunks in the terrain. For each `VoxelInstanceLibraryItem`, check if the chunk's current resolution level matches the item's `lod_index` [S1].
2.  **Candidate Generation:** Evaluate the item's generator conditions (e.g., noise thresholds) at candidate positions within the eligible chunks [Parent].
3.  **Surface Snapping (Optional):**
    *   If `snap_to_generator_sdf_enabled` is true, query the `VoxelGenerator` for Signed Distance Field (SDF) values at the candidate position and nearby samples [S1].
    *   Interpolate between samples to find the zero-crossing (surface) [S1].
    *   *Pitfall:* This is an interpolation approximation, not exact ray-casting. It may fail or produce errors if the terrain is edited after generation, as the generator's cached data may not reflect recent changes [S1].
4.  **Normal Offset:** Displace the instance position along the interpolated surface normal by `offset_along_normal` [S1].
5.  **Alignment:**
    *   If `up_mode` is 'Sphere mode', rotate instances to align with the local origin of the terrain (suitable for planets) [S1].
    *   Otherwise, align with a fixed world up vector [S1].
6.  **Instance Creation:** Create instances as either `MultiMesh` (high-performance, static) or `Scene` (dynamic, scriptable) types [S1].
7.  **Persistence:** If `persistent` is true and a `VoxelStream` is active, save the instance's ID and position. Editing the terrain marks these instances as modified, preventing regeneration and saving their new positions [S1].

**Parameters:**
*   `lod_index`: Determines the chunk resolution for spawning. Value 0 spawns on LOD0 chunks (closest range, fades quickly). Higher indexes spawn on bigger chunks (larger range) [S1].
*   `offset_along_normal`: Parameter in the generator. Depends on the asset; designing assets with part of their bottom sunk into the ground provides margin of error [S1].

**Edge Cases and Pitfalls:**
*   **Placement Imprecision:** Larger `lod_index` values are less precise because they work on lower-resolution meshes, potentially causing instances to float above or sink into the ground near sharp terrain changes (ridges, crevices, caves) [S1].
*   **Abrupt Disappearance:** Instances may disappear abruptly at LOD boundaries. Mitigation involves fading meshes based on distance from the camera using a custom shader or using ground textures of similar colors to blend [S1].

## Subtopics

### Offset Along Normal Placement
Adjusts instance placement by offsetting along the surface normal to correct for floating or sinking caused by low-resolution mesh imprecision. The `offset_along_normal` parameter depends on the asset; designing assets with part of their bottom sunk into the ground provides a margin of error [S1].

## Operating characteristics
*   **Time Complexity:** Generation is faster for higher `lod_index` values because they operate on lower-resolution meshes, avoiding full-precision LOD calculations [S1].
*   **Memory Complexity:** Depends on the number of instances spawned per chunk.
*   **Strengths:** Efficiently populates large distances by leveraging coarse terrain geometry; separates instance density by distance via LOD hierarchy.
*   **Limitations:** Placement accuracy degrades significantly with higher `lod_index` values, leading to visual artifacts (floating/sinking) on complex terrain [S1].
*   **When to use:** Best for large-scale vegetation or objects where exact surface contact is less critical than performance and visibility range. Use higher `lod_index` for big items seen from far away, and lower indexes for smaller items [S1].

## Background
Block LOD Instancing is part of the Voxel Tools ecosystem, specifically designed for voxel-based terrain rendering. It leverages the inherent hierarchical structure of voxel terrains to optimize instance generation.

## Sources
[S1] Instancing - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/instancing/, fetched 2026-10-27
