# Relaxed Connectivity Voxelization

Also called: 26-connected voxelization of 1D input

## Overview
Relaxed Connectivity Voxelization is a method for voxelizing effectively one-dimensional input (primitives that are 1D, 2D, or 3D but topologically 1D) in 3D space, ensuring the resulting voxels form a connected path (either 6-connected or 26-connected) to prevent gaps in flow simulation or fill operations. It distinguishes itself from standard voxelization by using specific 2D intersection targets within voxels to guarantee topological connectivity, and unlike algorithms restricted to 1D primitives like line segments, it handles input composed of 2D or 3D primitives [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of geometric primitives that are "effectively one-dimensional" in 3D space. This includes 1D primitives (curves, lines) as well as 2D or 3D primitives that do not separate 3D space (e.g., thin sheets or tubes) [S1, unofficial].
*   **Output:** A set of voxel indices $(i, j, k)$ representing the voxelized volume, where the set of occupied voxels forms a 6-connected or 26-connected path [S1, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 3D integer grid where each cell $(i, j, k)$ stores an occupancy flag or attribute. The grid resolution and bounding box are defined by the parent [Voxelization](12-voxelization.md) method [G].
*   **Intersection Target:** A 2D geometric structure defined within each voxel. For 26-connected voxelization, this target consists of three quadrilaterals that bisect the voxel along the x, y, and z axes, meeting at the voxel center [S1, unofficial].

**Steps:**
1.  **Define Grid:** Establish the voxel grid resolution and bounding box as per [Voxelization](12-voxelization.md) [G].
2.  **Define Intersection Target:** For each voxel, define the intersection target.
    *   For **26-connected** voxelization, the target is the union of three planar quadrilaterals. Each quadrilateral lies on a plane bisecting the voxel along one axis (e.g., $x=0.5$ in local voxel coordinates) and spans the other two dimensions. These three planes intersect at the voxel center, dividing the voxel space into 8 sub-cubes centered at the voxel corners [S1, unofficial].
    *   For **6-connected** voxelization, the target is a continuous surface on each of the 6 faces of the voxel, preventing entry/exit without intersection. Practical realizations include pyramids meeting at the center (octahedral tessellation) or surfaces pushed to the voxel boundaries (cubical tessellation) [S1, unofficial].
3.  **Intersect Input with Target:** For each primitive in the input, determine if it intersects the intersection target of any voxel.
    *   If the input primitive intersects the target within voxel $V$, mark voxel $V$ as occupied [S1, unofficial].
4.  **Verify Connectivity (Implicit):** The geometry of the intersection target ensures connectivity.
    *   For 26-connected: If input intersects the target in voxel $V$, any continuous path from $V$ to a voxel outside the immediate 26-neighbors must intersect the enclosing box formed by the targets of the 26-neighbors. This box has a side length twice that of a single voxel. Therefore, the input must intersect the target in at least one 26-neighbor of $V$, ensuring a 26-connected path [S1, unofficial].
    *   For 6-connected: A continuous 1D object cannot extend between two voxels without forming a 6-connected path of voxels in between, as the target surfaces prevent "diagonal" escape without intersection [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def relaxed_connectivity_voxelization(primitives, grid_resolution, grid_origin, grid_extent, connectivity='26'):
    """
    Voxelizes effectively 1D input with guaranteed connectivity.
    
    Args:
        primitives: List of geometric primitives (1D, 2D, or 3D but effectively 1D).
        grid_resolution: Tuple (nx, ny, nz) of integer grid dimensions.
        grid_origin: Tuple (ox, oy, oz) of world coordinates for grid start.
        grid_extent: Tuple (ex, ey, ez) of world coordinates for grid size.
        connectivity: '6' or '26'.
        
    Returns:
        occupied_voxels: Set of (i, j, k) indices.
    """
    occupied_voxels = set()
    
    # Calculate voxel size
    voxel_size_x = grid_extent[0] / grid_resolution[0]
    voxel_size_y = grid_extent[1] / grid_resolution[1]
    voxel_size_z = grid_extent[2] / grid_resolution[2]
    
    # Define intersection target based on connectivity
    if connectivity == '26':
        # Target: 3 bisecting planes (quadrilaterals)
        # In local voxel coordinates (0 to 1), these are:
        # Plane 1: x = 0.5, y in [0, 1], z in [0, 1]
        # Plane 2: y = 0.5, x in [0, 1], z in [0, 1]
        # Plane 3: z = 0.5, x in [0, 1], y in [0, 1]
        target_type = '26_bisecting_planes'
    elif connectivity == '6':
        # Target: Surfaces on voxel faces
        # Realization: Cubical tessellation (surfaces at boundaries)
        # Or Octahedral (pyramids to center)
        # For simplicity, we use the general definition: 
        # Any surface on the 6 faces that prevents diagonal escape.
        # Implementation detail: Intersect with the 6 faces of the voxel.
        target_type = '6_faces'
    else:
        raise ValueError("Connectivity must be '6' or '26'")

    # Iterate over all voxels
    for i in range(grid_resolution[0]):
        for j in range(grid_resolution[1]):
            for k in range(grid_resolution[2]):
                # Calculate voxel bounds in world space
                x_min = grid_origin[0] + i * voxel_size_x
                x_max = x_min + voxel_size_x
                y_min = grid_origin[1] + j * voxel_size_y
                y_max = y_min + voxel_size_y
                z_min = grid_origin[2] + k * voxel_size_z
                z_max = z_min + voxel_size_z
                
                voxel_center = (
                    (x_min + x_max) / 2,
                    (y_min + y_max) / 2,
                    (z_min + z_max) / 2
                )
                
                # Check intersection with each primitive
                for primitive in primitives:
                    if intersects_target(primitive, voxel_bounds=(x_min, x_max, y_min, y_max, z_min, z_max), 
                                         target_type=target_type, voxel_center=voxel_center):
                        occupied_voxels.add((i, j, k))
                        break # No need to check other primitives for this voxel
    
    return occupied_voxels

def intersects_target(primitive, voxel_bounds, target_type, voxel_center):
    """
    Checks if a primitive intersects the specific intersection target within a voxel.
    """
    x_min, x_max, y_min, y_max, z_min, z_max = voxel_bounds
    
    if target_type == '26_bisecting_planes':
        # Check intersection with the 3 bisecting planes
        # Plane 1: x = voxel_center[0]
        if primitive_intersects_plane(primitive, normal=(1, 0, 0), point=voxel_center):
            # Check if intersection point is within the voxel's y-z extent
            # (Simplified: assume primitive intersection check handles bounds)
            return True
        # Plane 2: y = voxel_center[1]
        if primitive_intersects_plane(primitive, normal=(0, 1, 0), point=voxel_center):
            return True
        # Plane 3: z = voxel_center[2]
        if primitive_intersects_plane(primitive, normal=(0, 0, 1), point=voxel_center):
            return True
        return False
        
    elif target_type == '6_faces':
        # Check intersection with the 6 faces of the voxel
        # Face 1: x = x_min
        if primitive_intersects_plane(primitive, normal=(-1, 0, 0), point=(x_min, voxel_center[1], voxel_center[2])):
            # Check if intersection is within y-z bounds
            if is_within_bounds(primitive.intersection_point, y_min, y_max, z_min, z_max):
                return True
        # Face 2: x = x_max
        if primitive_intersects_plane(primitive, normal=(1, 0, 0), point=(x_max, voxel_center[1], voxel_center[2])):
            if is_within_bounds(primitive.intersection_point, y_min, y_max, z_min, z_max):
                return True
        # Face 3: y = y_min
        if primitive_intersects_plane(primitive, normal=(0, -1, 0), point=(voxel_center[0], y_min, voxel_center[2])):
            if is_within_bounds(primitive.intersection_point, x_min, x_max, z_min, z_max):
                return True
        # Face 4: y = y_max
        if primitive_intersects_plane(primitive, normal=(0, 1, 0), point=(voxel_center[0], y_max, voxel_center[2])):
            if is_within_bounds(primitive.intersection_point, x_min, x_max, z_min, z_max):
                return True
        # Face 5: z = z_min
        if primitive_intersects_plane(primitive, normal=(0, 0, -1), point=(voxel_center[0], voxel_center[1], z_min)):
            if is_within_bounds(primitive.intersection_point, x_min, x_max, y_min, y_max):
                return True
        # Face 6: z = z_max
        if primitive_intersects_plane(primitive, normal=(0, 0, 1), point=(voxel_center[0], voxel_center[1], z_max)):
            if is_within_bounds(primitive.intersection_point, x_min, x_max, y_min, y_max):
                return True
        return False

def primitive_intersects_plane(primitive, normal, point):
    """
    [G] Placeholder for specific primitive-plane intersection logic.
    Returns True if the primitive intersects the plane defined by normal and point.
    """
    # Implementation depends on primitive type (line, triangle, etc.)
    # This is a helper function, not part of the core method definition.
    pass

def is_within_bounds(point, x_min, x_max, y_min, y_max, z_min, z_max):
    """
    [G] Placeholder for checking if a point is within voxel bounds.
    """
    return x_min <= point[0] <= x_max and y_min <= point[1] <= y_max and z_min <= point[2] <= z_max
```

**Parameters:**
*   **Connectivity:** '6' or '26'. Default is '26' for efficiency [S1, unofficial].
*   **Grid Resolution:** Determined by the parent [Voxelization](12-voxelization.md) method [G].

**Edge Cases and Pitfalls:**
*   **Inefficiency:** The resulting voxelization may contain almost as many voxels as a cover produced by intersecting the input against the entire 3D voxel, making it potentially inefficient for dense inputs [S1, unofficial].
*   **Not Thin:** The result is not "thin" in the same sense as the 2D crosshairs target. Stepping along a curve in the dominant direction may produce fewer voxels than this method [S1, unofficial].
*   **Input Type:** The method is designed for "effectively one-dimensional" input. If the input has significant 2D or 3D extent that separates space, the connectivity guarantees may not hold or may be overkill [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of voxels and primitives. For each voxel, intersection tests with all primitives are performed. This can be optimized with spatial acceleration structures (e.g., BVH) for the primitives, but the core method does not specify one.
*   **Memory Complexity:** Proportional to the number of voxels in the grid.
*   **Strengths:** Guarantees topological connectivity (6 or 26) for effectively 1D input. Handles 2D and 3D primitives that are effectively 1D, unlike some existing algorithms restricted to 1D primitives [S1, unofficial].
*   **Limitations:** Can produce a large number of voxels (not thin). May be inefficient for dense inputs.
*   **When to Use:** When topological connectivity is critical for downstream processes like flow simulation or fill operations, and the input is effectively 1D.
*   **When Not to Use:** When a thin representation is required, or when the input is not effectively 1D.

## Background
This method was introduced by M. C. Laine in the paper "A Topological Approach to Voxelization" (2013) [S1, unofficial]. It extends the 2D topological voxelization approach to 3D, specifically addressing the need for connected voxelizations of effectively 1D structures. The 26-connected variant is highlighted as more practical than the 6-connected variant for many applications due to fewer voxels, while still maintaining connectivity.

## Sources
[S1] A Topological Approach to Voxelization, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
