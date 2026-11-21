# Metric Coordinate Offset

Also called: World Metric Coordinate Adjustment

## Overview
Metric Coordinate Offset shifts the world-space origin of a voxel grid by applying a delta vector to all scene entities and grid data structures [S2, unofficial]. It recenteres a volume rendering scene or aligns the voxel grid with a new reference frame without altering relative geometry [S2, unofficial]. It differs from [Metric Coordinate Conversion](12960-metric-coordinate-conversion.md) by translating discrete world-space positions and spatial indices rather than performing continuous coordinate system transformations [S2, unofficial].

## How it works
**Inputs:**
- `delta`: An array of 3 floats `[dx, dy, dz]` representing the shift vector for the world origin [S2, unofficial].

**Outputs:**
- Updated mesh positions in world space and updated octree block positions [S2, unofficial].

**Data Structures:**
- `mesh.position`: A mutable sequence of 3 floats representing the mesh's world position [S2, unofficial].
- `mesh.parent`: A reference to a parent mesh, used to determine coordinate space [S2, unofficial].
- `mesh.is_world_matrix_frozen`: A boolean flag indicating if the mesh's world matrix is cached [S2, unofficial].
- `octree_manager`: An object managing the spatial indexing of the voxel grid [S2, unofficial].

**Steps:**
1. Construct a delta vector `dvec` from the input `delta` array:
   $$ \text{dvec} = (\delta_x, \delta_y, \delta_z) $$
2. Iterate over all meshes in the scene [S2, unofficial].
3. For each mesh, check if it has a parent. If `mesh.parent` exists, skip the mesh because its transform is relative to its parent, not the world origin [S2, unofficial].
4. For meshes without a parent, subtract the delta vector from the mesh's position in place:
   $$ \text{mesh.position} \leftarrow \text{mesh.position} - \text{dvec} $$
5. Check if the mesh's world matrix is frozen. If `mesh.is_world_matrix_frozen` is true, call `mesh.freeze_world_matrix()` to update its internal state after the position change [S2, unofficial].
6. Call the octree manager's rebase method with the delta vector to update the position of all octree blocks:
   $$ \text{octree\_manager.rebase}(\text{dvec}) $$

**Pitfalls:**
- **Fractional Rollover:** Adding offset may push `fractionalMeter >= 1.0f`. This is handled by extracting `wholeMeterOffset` via `floor` and carrying it to `wholeMeters` [S1, unofficial].
- **Parented Meshes:** Parented meshes must be skipped during rebasing because their transforms are relative to their parent, not the world origin [S2, unofficial].
- **Frozen Matrices:** Frozen world matrices require a double-freeze call to properly update their internal state after position changes [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def metric_coordinate_offset(delta, meshes, octree_manager):
    # Step 1: Calculate delta vector
    dvec = (delta[0], delta[1], delta[2])
    
    # Step 2: Iterate over all meshes
    for mesh in meshes:
        # Step 3: Skip parented meshes
        if mesh.parent is not None:
            continue
            
        # Step 4: Subtract delta from position in place
        mesh.position[0] -= dvec[0]
        mesh.position[1] -= dvec[1]
        mesh.position[2] -= dvec[2]
        
        # Step 5: Handle frozen world matrix
        if mesh.is_world_matrix_frozen:
            mesh.freeze_world_matrix()
            
    # Step 6: Update octree blocks
    octree_manager.rebase(dvec)
```

## Subtopics

### Origin Rebasing
Origin Rebasing is a variant of metric coordinate offset that specifically focuses on redefining the zero-point of the coordinate system for the entire scene [S2, unofficial]. It ensures that all spatial indices and world-space references are consistently updated relative to the new origin, and is chosen when the logical center of the volume needs to change without moving the geometry itself [S2, unofficial]. See [Metric Coordinate Conversion](12960-metric-coordinate-conversion.md).

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of meshes in the scene, $O(N)$, where $N$ is the count of non-parented meshes [S2, unofficial]. The octree rebase operation depends on the number of blocks in the octree [S2, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space, as the operation modifies positions in place [S2, unofficial].
- **Strengths:** Direct manipulation of world-space positions and correct handling of hierarchical scene graphs by skipping parented objects [S2, unofficial].
- **Limitations:** Requires specific double-freeze handling for frozen matrices and is not suitable for relative coordinate adjustments within a hierarchy [S2, unofficial].
- **When to use:** Recentering a voxel grid or changing the world origin for rendering purposes [S2, unofficial].
- **When not to use:** Adjusting relative transforms of child objects or when the scene does not use a flat world-space hierarchy for the objects being moved [S2, unofficial].

## Background
The method originates from the `noa` engine implementation, which provides voxel-based world management and scene graph utilities [S2, unofficial]. It addresses the need for precise coordinate adjustments in voxel-based environments where world-space positions must remain consistent with the underlying grid structure [S2, unofficial].

## Sources
[S1] aaron-eugene/PlaneVoxelLab, https://github.com/aaron-eugene/PlaneVoxelLab, fetched 2026-10-27 (unofficial)
[S2] fenomas/noa, https://github.com/fenomas/noa, fetched 2026-10-26 (unofficial)
