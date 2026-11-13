# Metric Coordinate Offset

Also called: World Metric Coordinate Adjustment

## Overview
Metric Coordinate Offset adjusts the world-space origin of a voxel grid by shifting all scene entities and grid data structures by a specified delta vector. It solves the problem of recentering a volume rendering scene or aligning the voxel grid with a new reference frame without altering the relative geometry of the scene. It differs from general [Metric Coordinate Conversion](12960-metric-coordinate-conversion.md) by specifically handling the translation of discrete world-space positions and associated spatial indices rather than continuous coordinate system transformations.

## How it works
**Inputs:**
- `delta`: An array of 3 floats `[dx, dy, dz]` representing the shift vector for the world origin [S2, unofficial].

**Outputs:**
- Updated mesh positions in world space.
- Updated octree block positions.
- Adjusted voxel grid internal state (implicit via octree rebase).

**Data Structures:**
- `mesh.position`: A 3D vector representing the mesh's world position.
- `mesh.isWorldMatrixFrozen`: A boolean flag indicating if the mesh's world matrix is cached/frozen.
- `octreeManager`: The manager object responsible for the spatial indexing of the voxel grid.

**Steps:**
1.  Construct a delta vector `dvec` from the input `delta` array:
    $$ \text{dvec} = (\delta_x, \delta_y, \delta_z) $$
2.  Iterate over all meshes in the scene.
3.  For each mesh, check if it has a parent. If `mesh.parent` exists, skip the mesh because its transform is relative to its parent, not the world origin [S2, unofficial].
4.  For meshes without a parent, subtract the delta vector from the mesh's position in place:
    $$ \text{mesh.position} \leftarrow \text{mesh.position} - \text{dvec} $$
5.  Check if the mesh's world matrix is frozen. If `mesh.isWorldMatrixFrozen` is true, call `mesh.freezeWorldMatrix()` again. This double-freeze operation is required to paradoxically unfreeze and re-freeze the matrix to update its internal state after the position change [S2, unofficial].
6.  Call the octree manager's rebase method with the delta vector to update the position of all octree blocks:
    $$ \text{octreeManager.rebase}(\text{dvec}) $$

**Pitfalls:**
- **Fractional Rollover:** When adding offsets to voxel coordinates, the fractional part of the meter coordinate may exceed 1.0. This must be handled by extracting the whole meter offset via `floor` and carrying it to the whole meters component [S1, unofficial].
- **Parented Meshes:** Parented meshes must be skipped during rebasing. Modifying their world position directly would break their hierarchical relationship [S2, unofficial].
- **Frozen Matrices:** Failure to re-freeze the world matrix after position updates can lead to stale transform data. The specific implementation requires a double-freeze call [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def metric_coordinate_offset(delta):
    """
    Adjusts the world metric coordinate offset for a voxel scene.
    
    Args:
        delta (list[float]): Array of 3 floats [dx, dy, dz] representing the shift vector.
    """
    # Step 1: Calculate delta vector
    dvec = Vector3(delta[0], delta[1], delta[2])
    
    # Step 2: Iterate over all meshes
    for mesh in scene.meshes:
        # Step 3: Skip parented meshes
        if mesh.parent is not None:
            continue
            
        # Step 4: Subtract delta from position
        mesh.position.subtract_in_place(dvec)
        
        # Step 5: Handle frozen world matrix
        if mesh.is_world_matrix_frozen:
            mesh.freeze_world_matrix()
            
    # Step 6: Update octree blocks
    octree_manager.rebase(dvec)
```

## Subtopics

### Origin Rebasing
Origin Rebasing is a variant of metric coordinate offset that specifically focuses on redefining the zero-point of the coordinate system for the entire scene. It sets apart from standard translation by ensuring that all spatial indices and world-space references are consistently updated relative to the new origin. It is chosen when the logical center of the volume needs to change without moving the geometry itself. See [Metric Coordinate Conversion](12960-metric-coordinate-conversion.md).

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of meshes in the scene, $O(N)$, where $N$ is the count of non-parented meshes. The octree rebase operation depends on the number of blocks in the octree.
- **Memory Complexity:** $O(1)$ auxiliary space, as the operation modifies positions in place.
- **Strengths:** Direct manipulation of world-space positions; handles hierarchical scene graphs correctly by skipping parented objects.
- **Limitations:** Requires specific handling for frozen matrices and parented meshes. Not suitable for relative coordinate adjustments within a hierarchy.
- **When to use:** When recentering a voxel grid or changing the world origin for rendering purposes.
- **When not to use:** When adjusting relative transforms of child objects or when the scene does not use a flat world-space hierarchy for the objects being moved.

## Background
The method originates from the `noa` engine implementation, which provides voxel-based world management and scene graph utilities. It addresses the need for precise coordinate adjustments in voxel-based environments where world-space positions must remain consistent with the underlying grid structure. The technique is documented in the `fenomas/noa` repository [S2, unofficial]. The fractional rollover handling is associated with the `PlaneVoxelLab` project, which deals with density field sampling and coordinate precision [S1, unofficial].

## Sources
[S1] aaron-eugene/PlaneVoxelLab, https://github.com/aaron-eugene/PlaneVoxelLab, fetched 2026-10-27 (unofficial)
[S2] fenomas/noa, https://github.com/fenomas/noa, fetched 2026-10-26 (unofficial)
