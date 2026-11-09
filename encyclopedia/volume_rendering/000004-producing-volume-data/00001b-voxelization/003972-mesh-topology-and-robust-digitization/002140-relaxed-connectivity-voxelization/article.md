# Relaxed Connectivity Voxelization

Also called: 26-connected voxelization of 1D input

## Overview
Relaxed Connectivity Voxelization is a method for producing a binary or labeled `UniformGrid` from effectively one-dimensional input primitives (1D, 2D, or 3D) in 3D space, ensuring the resulting voxels form a 26-connected path [S1, unofficial]. Unlike standard voxelization which may produce gaps in thin structures, this method guarantees topological connectivity for flow simulation or fill operations while minimizing voxel count compared to 6-connected approaches [S1, unofficial]. It distinguishes itself from sibling methods by supporting 2D/3D primitives that are effectively 1D, rather than being restricted to strict 1D primitives like line segments [S1, unofficial].

## How it works
**Inputs:** A set of geometric primitives (curves, surfaces, or volumes) that are effectively one-dimensional in extent.
**Outputs:** A binary `UniformGrid` where occupied voxels form a 26-connected set.

**Data Structures:**
*   **Voxel Grid:** A `UniformGrid` with resolution $R_x \times R_y \times R_z$.
*   **Intersection Target:** A set of 2D planar surfaces defined within each voxel cell. For 26-connected voxelization, the target consists of three quadrilaterals that bisect the voxel along the X, Y, and Z axes. These quadrilaterals intersect at the voxel center [S1, unofficial].

**Steps:**
1.  **Define Intersection Target:** For each voxel $V$ in the grid, define the intersection target $T_V$ as the union of three planar quadrilaterals. Each quadrilateral lies on a plane bisecting the voxel along one axis (e.g., $x=0.5$ in local voxel coordinates). The quadrilaterals connect the midpoints of the voxel edges, forming a cross-like shape in 3D [S1, unofficial].
2.  **Intersect Input with Target:** For each primitive in the input geometry, compute the intersection with the intersection targets $T_V$ of all voxels $V$ in the grid.
3.  **Mark Voxels:** If a primitive intersects the target $T_V$ of a voxel $V$, mark voxel $V$ as occupied [S1, unofficial].
4.  **Connectivity Verification:** The method relies on the geometric property that if the input intersects $T_V$, any continuous path from $V$ to a voxel outside the immediate 26-neighbors must intersect the union of the targets of the 26-neighbors $N_{26}(V)$. This union forms a closed box with side length twice that of a single voxel, ensuring no gaps in the 26-connected path [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def relaxed_connectivity_voxelization(primitives, grid_resolution, voxel_size):
    """
    Voxelize effectively 1D primitives with 26-connectivity.
    
    Args:
        primitives: List of geometric primitives (curves, surfaces, volumes).
        grid_resolution: Tuple (Rx, Ry, Rz) defining the grid dimensions.
        voxel_size: Tuple (sx, sy, sz) defining the size of each voxel.
        
    Returns:
        grid: A 3D boolean array of shape (Rx, Ry, Rz).
    """
    # Initialize empty grid
    grid = zeros(grid_resolution, dtype=bool)
    
    # Precompute voxel centers and bounds if needed for optimization
    # Not given in the sources.
    
    for primitive in primitives:
        # Iterate over all voxels in the grid
        for x in range(grid_resolution[0]):
            for y in range(grid_resolution[1]):
                for z in range(grid_resolution[2]):
                    # Define the intersection target T_V for voxel (x, y, z)
                    # T_V consists of three bisecting planes:
                    # 1. Plane x = x + 0.5 (local coordinates)
                    # 2. Plane y = y + 0.5
                    # 3. Plane z = z + 0.5
                    # Each plane is restricted to the voxel boundaries.
                    
                    # Check if primitive intersects any of the three bisecting planes
                    # within the voxel bounds.
                    
                    # Intersection with X-bisector plane
                    intersect_x = check_intersection_with_plane(primitive, 
                                                                plane_normal=(1,0,0), 
                                                                point=(x + 0.5, y, z), 
                                                                bounds=(x, y, z, x+1, y+1, z+1))
                    
                    # Intersection with Y-bisector plane
                    intersect_y = check_intersection_with_plane(primitive, 
                                                                plane_normal=(0,1,0), 
                                                                point=(x, y + 0.5, z), 
                                                                bounds=(x, y, z, x+1, y+1, z+1))
                    
                    # Intersection with Z-bisector plane
                    intersect_z = check_intersection_with_plane(primitive, 
                                                                plane_normal=(0,0,1), 
                                                                point=(x, y, z + 0.5), 
                                                                bounds=(x, y, z, x+1, y+1, z+1))
                    
                    # If intersection occurs with any part of the target, mark voxel
                    if intersect_x or intersect_y or intersect_z:
                        grid[x, y, z] = True
                        
    return grid

def check_intersection_with_plane(primitive, plane_normal, point, bounds):
    """
    Checks if a primitive intersects a specific plane within voxel bounds.
    This is a helper function. The exact implementation depends on the primitive type.
    For a line segment, this involves solving for the intersection parameter.
    For a surface, it involves checking if the surface crosses the plane within bounds.
    Not given in the sources.
    """
    # Placeholder for actual geometric intersection logic
    # [G] In a real implementation, this would use standard computational geometry routines
    # to find the intersection of the primitive with the infinite plane defined by
    # plane_normal and point, and then verify if the intersection point lies within
    # the voxel bounds.
    return False
```

**Parameters:**
*   **Connectivity:** 26-connected.
*   **Target Shape:** Three bisecting quadrilaterals.
*   **Typical Values:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Voxel Count:** The resulting voxelization may contain almost as many voxels as a cover produced by intersecting the input against the entire 3D voxel, making it potentially inefficient [S1, unofficial].
*   **Thinness:** The result is not thin in the same sense as the 2D crosshairs target. Stepping along a curve in the dominant direction may produce fewer voxels [S1, unofficial].
*   **Input Type:** The method is designed for effectively 1D input. If the input is truly 3D (e.g., a solid sphere), the method may not produce the desired topological properties [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Proportional to the number of voxels in the grid.
*   **Strengths:** Guarantees 26-connectivity for effectively 1D structures; works with 2D/3D primitives that are effectively 1D; minimizes voxel count compared to 6-connected methods.
*   **Limitations:** May produce a high number of voxels; not suitable for truly 3D objects; result is not "thin" in the strictest sense.
*   **When to Use:** When 26-connectivity is required for flow simulation or fill operations on thin structures, and the input consists of effectively 1D primitives (e.g., thick curves, thin surfaces).
*   **When Not to Use:** When 6-connectivity is sufficient; when the input is truly 3D; when minimal voxel count is critical and the input is strictly 1D (line segments).

## Background
Relaxed Connectivity Voxelization was introduced by Laine et al. in 2013 as part of a topological approach to voxelization [S1, unofficial]. It addresses the limitation of existing algorithms that were restricted to 1D primitives (e.g., line segments) by extending the concept to effectively 1D input composed of 2D or 3D primitives [S1, unofficial]. The method is used in applications requiring topological correctness, such as flow simulation and fill operations, where gaps in the voxelized representation can lead to incorrect results [S1, unofficial].

## Sources
[S1] A Topological Approach to Voxelization, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
