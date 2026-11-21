# Sphere-Voxel Collision Detection via Displacement Averaging

Also called: sphereIntersect

## Overview
A collision detection method for a sphere moving through a voxel grid that calculates a displacement vector representing the push-out needed to resolve penetration by averaging displacements from all intersected voxel faces [S1, unofficial]. It differs from simple boolean occupancy tests by providing a continuous resolution vector, allowing the sphere to be pushed out of the voxel structure rather than merely detecting overlap [S1, unofficial].

## How it works
**Inputs:**
- `sphere`: A sphere defined by a center point (`THREE.Vector3Like`) and a radius (`number`).
- `voxelmapCollider`: An object providing access to voxel data, specifically `getVoxel(voxelIndex)` which returns an `EVoxelStatus` enum (e.g., `EMPTY`, `FULL`).
- `voxelmapCollider.chunkSize`: Dimensions of the voxel chunks.
- `voxelmapCollider.voxelsChunkOrdering`: String defining the memory layout of voxels within a chunk (e.g., "xyz").

**Outputs:**
- `null` if no collision is detected.
- An object `{ normal: THREE.Vector3Like, depth: number }` where `normal` is the unit vector of the average displacement and `depth` is the magnitude of that average displacement.

**Data Structures:**
- `displacements`: An array of `THREE.Vector3Like` objects storing individual face push-out vectors.
- `voxel`: A `THREE.Vector3Like` representing integer voxel coordinates `(x, y, z)`.
- `sphereCenterProjection2dX/Y/Z`: Objects with `x` and `y` fields representing 2D projections of the sphere center onto specific voxel faces.

**Algorithm Steps:**

1.  Initialize an empty list `displacements` of type `THREE.Vector3Like[]`.

2.  Define a helper function `addDisplacementIfNeeded(sphereCenterProjection)`:
    a. Calculate `projectionToCenter = sphere.center - sphereCenterProjection`.
    b. Calculate `distanceFromCenter = projectionToCenter.length()`.
    c. If `distanceFromCenter < sphere.radius`:
        i. Calculate `pushVector = projectionToCenter.normalize() * (sphere.radius - distanceFromCenter)`.
        ii. Push `pushVector` to `displacements`.

3.  Calculate the bounding voxel coordinates containing the sphere:
    a. `voxelFrom = floor(sphere.center - sphere.radius)`.
    b. `voxelTo = floor(sphere.center + sphere.radius)`.

4.  Iterate through all voxels in the bounding box:
    a. For `voxel.z` from `voxelFrom.z` to `voxelTo.z`:
    b.   For `voxel.y` from `voxelFrom.y` to `voxelTo.y`:
    c.     For `voxel.x` from `voxelFrom.x` to `voxelTo.x`:
        i. Calculate local sphere center relative to voxel: `localSphereCenter = sphere.center - voxel`.
        ii. Calculate 2D projections of the local sphere center onto the voxel faces using `this.pointSquareProjection`:
            - `sphereCenterLocalProjection2dX` (using z, y components).
            - `sphereCenterLocalProjection2dY` (using x, z components).
            - `sphereCenterLocalProjection2dZ` (using x, y components).
        iii. Calculate global 2D projection coordinates for each face by adding voxel indices:
            - `sphereCenterProjection2dX = { x: sphereCenterLocalProjection2dX.x + voxel.z, y: sphereCenterLocalProjection2dX.y + voxel.y }`
            - `sphereCenterProjection2dY = { x: sphereCenterLocalProjection2dY.x + voxel.x, y: sphereCenterLocalProjection2dY.y + voxel.z }`
            - `sphereCenterProjection2dZ = { x: sphereCenterLocalProjection2dZ.x + voxel.x, y: sphereCenterLocalProjection2dZ.y + voxel.y }`
        iv. Check if current voxel is full: `voxelIsFull = (voxelmapCollider.getVoxel(voxel) !== EVoxelStatus.EMPTY)`.
        v. Check neighbors for fullness:
            - `voxelBelowIsFull`: status of `(voxel.x, voxel.y - 1, voxel.z)`
            - `voxelAboveIsFull`: status of `(voxel.x, voxel.y + 1, voxel.z)`
            - `voxelLeftIsFull`: status of `(voxel.x - 1, voxel.y, voxel.z)`
            - `voxelRightIsFull`: status of `(voxel.x + 1, voxel.y, voxel.z)`
            - `voxelBackIsFull`: status of `(voxel.x, voxel.y, voxel.z - 1)`
            - `voxelFrontIsFull`: status of `(voxel.x, voxel.y, voxel.z + 1)`
        vi. If `voxelIsFull !== voxelBelowIsFull`, call `addDisplacementIfNeeded({ x: sphereCenterProjection2dY.x, y: voxel.y, z: sphereCenterProjection2dY.y })`.
        vii. If `voxelIsFull !== voxelAboveIsFull`, call `addDisplacementIfNeeded({ x: sphereCenterProjection2dY.x, y: voxel.y + 1, z: sphereCenterProjection2dY.y })`.
        viii. If `voxelIsFull !== voxelLeftIsFull`, call `addDisplacementIfNeeded({ x: voxel.x, y: sphereCenterProjection2dX.y, z: sphereCenterProjection2dX.x })`.
        ix. If `voxelIsFull !== voxelRightIsFull`, call `addDisplacementIfNeeded({ x: voxel.x + 1, y: sphereCenterProjection2dX.y, z: sphereCenterProjection2dX.x })`.
        x. If `voxelIsFull !== voxelBackIsFull`, call `addDisplacementIfNeeded({ x: sphereCenterProjection2dZ.x, y: sphereCenterProjection2dZ.y, z: voxel.z })`.
        xi. If `voxelIsFull !== voxelFrontIsFull`, call `addDisplacementIfNeeded({ x: sphereCenterProjection2dZ.x, y: sphereCenterProjection2dZ.y, z: voxel.z + 1 })`.

5.  After loops, if `displacements.length > 0`:
    a. Sum all displacements into `totalDisplacement`.
    b. Calculate `averageDisplacement = totalDisplacement / displacements.length`.
    c. Calculate `totalDepth = averageDisplacement.length()`.
    d. Return `{ normal: averageDisplacement.normalize(), depth: totalDepth }`.

6.  If `displacements.length` is 0, return `null`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sphere_voxel_collision(sphere, voxelmap_collider):
    """
    sphere: { center: Vector3, radius: float }
    voxelmap_collider: object with getVoxel(voxel: Vector3) -> EVoxelStatus
    Returns: { normal: Vector3, depth: float } or None
    """
    displacements = []

    def add_displacement_if_needed(sphere_center_projection):
        projection_to_center = sphere['center'] - sphere_center_projection
        distance_from_center = projection_to_center.length()
        if distance_from_center < sphere['radius']:
            push_vector = projection_to_center.normalize() * (sphere['radius'] - distance_from_center)
            displacements.append(push_vector)

    # Calculate bounding voxel coordinates
    voxel_from = floor(sphere['center'] - sphere['radius'])
    voxel_to = floor(sphere['center'] + sphere['radius'])

    for z in range(voxel_from.z, voxel_to.z + 1):
        for y in range(voxel_from.y, voxel_to.y + 1):
            for x in range(voxel_from.x, voxel_to.x + 1):
                voxel = Vector3(x, y, z)
                
                # Local sphere center relative to voxel
                local_sphere_center = sphere['center'] - voxel
                
                # Calculate 2D projections of the local sphere center onto the voxel faces
                # Note: pointSquareProjection is assumed to be a method on the collider or helper
                sphere_center_local_proj_2d_x = point_square_projection(local_sphere_center, 'x') # uses z,y
                sphere_center_local_proj_2d_y = point_square_projection(local_sphere_center, 'y') # uses x,z
                sphere_center_local_proj_2d_z = point_square_projection(local_sphere_center, 'z') # uses x,y
                
                # Calculate global 2D projection coordinates for each face
                sphere_center_proj_2d_x = Vector2(sphere_center_local_proj_2d_x.x + z, sphere_center_local_proj_2d_x.y + y)
                sphere_center_proj_2d_y = Vector2(sphere_center_local_proj_2d_y.x + x, sphere_center_local_proj_2d_y.y + z)
                sphere_center_proj_2d_z = Vector2(sphere_center_local_proj_2d_z.x + x, sphere_center_local_proj_2d_z.y + y)
                
                # Check if current voxel is full
                voxel_is_full = voxelmap_collider.get_voxel(voxel) != EVoxelStatus.EMPTY
                
                # Check neighbors for fullness
                voxel_below_is_full = voxelmap_collider.get_voxel(Vector3(x, y - 1, z)) != EVoxelStatus.EMPTY
                voxel_above_is_full = voxelmap_collider.get_voxel(Vector3(x, y + 1, z)) != EVoxelStatus.EMPTY
                voxel_left_is_full = voxelmap_collider.get_voxel(Vector3(x - 1, y, z)) != EVoxelStatus.EMPTY
                voxel_right_is_full = voxelmap_collider.get_voxel(Vector3(x + 1, y, z)) != EVoxelStatus.EMPTY
                voxel_back_is_full = voxelmap_collider.get_voxel(Vector3(x, y, z - 1)) != EVoxelStatus.EMPTY
                voxel_front_is_full = voxelmap_collider.get_voxel(Vector3(x, y, z + 1)) != EVoxelStatus.EMPTY
                
                # Add displacements if there is a boundary between full and empty
                if voxel_is_full != voxel_below_is_full:
                    add_displacement_if_needed(Vector3(sphere_center_proj_2d_y.x, y, sphere_center_proj_2d_y.y))
                
                if voxel_is_full != voxel_above_is_full:
                    add_displacement_if_needed(Vector3(sphere_center_proj_2d_y.x, y + 1, sphere_center_proj_2d_y.y))
                
                if voxel_is_full != voxel_left_is_full:
                    add_displacement_if_needed(Vector3(x, sphere_center_proj_2d_x.y, sphere_center_proj_2d_x.x))
                
                if voxel_is_full != voxel_right_is_full:
                    add_displacement_if_needed(Vector3(x + 1, sphere_center_proj_2d_x.y, sphere_center_proj_2d_x.x))
                
                if voxel_is_full != voxel_back_is_full:
                    add_displacement_if_needed(Vector3(sphere_center_proj_2d_z.x, sphere_center_proj_2d_z.y, z))
                
                if voxel_is_full != voxel_front_is_full:
                    add_displacement_if_needed(Vector3(sphere_center_proj_2d_z.x, sphere_center_proj_2d_z.y, z + 1))

    if len(displacements) > 0:
        total_displacement = sum(displacements)
        average_displacement = total_displacement / len(displacements)
        total_depth = average_displacement.length()
        return {
            'normal': average_displacement.normalize(),
            'depth': total_depth
        }
    
    return None
```

**Parameters:**
- `sphere.radius`: The radius of the sphere. Typical values depend on the scene scale.
- `voxelmap_collider.chunkSize`: Dimensions of the voxel chunks. Not given in the sources.
- `voxelmap_collider.voxelsChunkOrdering`: String defining the memory layout of voxels within a chunk. Not given in the sources.

**Edge Cases and Pitfalls:**
- If `displacements` is empty, the function returns `null`, indicating no collision.
- Precision issues may arise if `distanceFromCenter` is very close to `sphere.radius`.
- The algorithm assumes `pointSquareProjection` is available and correctly implemented. [G]
- The algorithm assumes `floor` is applied to the bounding box calculation. [G]
- The algorithm assumes `normalize` handles zero-length vectors gracefully or that they do not occur. [G]

## Operating characteristics
- **Time Complexity:** Proportional to the number of voxels in the sphere's bounding box, which is $O(r^3)$ where $r$ is the sphere radius in voxel units.
- **Memory Complexity:** Proportional to the number of intersected faces, which is $O(r^2)$.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance scales with the cube of the sphere radius.
- **Strengths:** Provides a continuous displacement vector for collision resolution, allowing for smooth movement and penetration recovery.
- **Limitations:** Requires checking multiple neighbors for each voxel in the bounding box, which can be computationally expensive for large spheres or dense voxel grids.
- **When to use:** When continuous collision resolution is needed for spheres in a voxel environment.
- **When not to use:** When only boolean collision detection is required, or when performance is critical and the sphere is very large.

## Background
The method is described in the `aresrpg-engine` repository, an unofficial project [S1, unofficial]. It is used for collision detection in voxel-based games or simulations.

## Sources
[S1] aresrpg/aresrpg-engine, https://github.com/aresrpg/aresrpg-engine, fetched 2026-10-27 (unofficial)
