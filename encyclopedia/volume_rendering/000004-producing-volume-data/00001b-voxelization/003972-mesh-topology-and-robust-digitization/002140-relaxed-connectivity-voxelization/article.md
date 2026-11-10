# Relaxed Connectivity Voxelization

Also called: 26-connected voxelization of 1D input

## Overview
Relaxed Connectivity Voxelization is a method for voxelizing effectively one-dimensional input (1D, 2D, or 3D primitives) in 3D space that ensures the resulting voxels form a connected path, preventing gaps in flow simulation or fill operations [S1, unofficial]. It produces a 26-connected path, allowing for fewer voxels than 6-connected methods while still maintaining connectivity [S1, unofficial]. Unlike existing algorithms restricted to 1D primitives (e.g., line segments), this method works for input composed of 2D or 3D primitives that are effectively 1D [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A continuous geometric object composed of 1D, 2D, or 3D primitives that is "effectively one-dimensional" (cannot produce separation in 3D space) [S1, unofficial].
*   **Output:** A set of voxels in a `UniformGrid` that are 26-connected [S1, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 3D grid where each cell represents a voxel.
*   **Intersection Target:** A geometric construct defined within each voxel to determine if the input intersects the voxel. For 26-connected voxelization, the target consists of three quadrilaterals that bisect the voxel along all three axes [S1, unofficial].

**Steps:**
1.  **Define Intersection Target:** For each voxel $V$, define the intersection target as three quadrilaterals that bisect the voxel along the x, y, and z axes. This subdivides the space into cubes centered at the voxel corners [S1, unofficial].
2.  **Intersect Input with Target:** For each voxel $V$, check if the input geometry intersects the defined intersection target [S1, unofficial].
3.  **Mark Voxels:** If the input intersects the target in voxel $V$, mark $V$ as part of the voxelization [S1, unofficial].
4.  **Verify Connectivity (Implicit):** The connectivity is guaranteed by the geometry of the target. If the input intersects the target in voxel $V$, parts of the intersection targets of the 26-neighbors of $V$ form a closed box with side length twice that of a single voxel [S1, unofficial].
5.  **Ensure Path Continuity:** Any continuous path from $V$ to voxels outside immediate neighbors must intersect this enclosing box, and therefore intersect the target in $N_{26}(V)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def relaxed_connectivity_voxelization(input_geometry, grid_bounds, voxel_size):
    """
    Voxelizes effectively 1D input with 26-connectivity.
    
    Args:
        input_geometry: Continuous geometric object (1D, 2D, or 3D primitives).
        grid_bounds: Bounding box of the grid (min_x, min_y, min_z, max_x, max_y, max_z).
        voxel_size: Size of each voxel (assumed cubic).
        
    Returns:
        A set of voxel coordinates (x, y, z) that are 26-connected.
    """
    # Calculate grid dimensions
    grid_size = [
        int((grid_bounds[3] - grid_bounds[0]) / voxel_size),
        int((grid_bounds[4] - grid_bounds[1]) / voxel_size),
        int((grid_bounds[5] - grid_bounds[2]) / voxel_size)
    ]
    
    voxelized_voxels = set()
    
    # Iterate over all voxels in the grid
    for x in range(grid_size[0]):
        for y in range(grid_size[1]):
            for z in range(grid_size[2]):
                # Define the voxel's intersection target
                # The target consists of three quadrilaterals bisecting the voxel
                # along x, y, and z axes.
                # Center of the voxel
                center_x = grid_bounds[0] + (x + 0.5) * voxel_size
                center_y = grid_bounds[1] + (y + 0.5) * voxel_size
                center_z = grid_bounds[2] + (z + 0.5) * voxel_size
                
                # Define the three bisecting planes/quadrilaterals
                # 1. Plane bisecting along x-axis (constant x = center_x)
                #    This plane spans y and z within the voxel bounds
                plane_x = Plane(normal=(1, 0, 0), point=(center_x, center_y, center_z))
                
                # 2. Plane bisecting along y-axis (constant y = center_y)
                plane_y = Plane(normal=(0, 1, 0), point=(center_x, center_y, center_z))
                
                # 3. Plane bisecting along z-axis (constant z = center_z)
                plane_z = Plane(normal=(0, 0, 1), point=(center_x, center_y, center_z))
                
                # Check if input geometry intersects any of the three planes
                # within the bounds of the voxel
                intersects = False
                
                # Check intersection with plane_x
                if intersects_segment_or_primitive(input_geometry, plane_x, voxel_bounds(x, y, z, voxel_size)):
                    intersects = True
                # Check intersection with plane_y
                elif intersects_segment_or_primitive(input_geometry, plane_y, voxel_bounds(x, y, z, voxel_size)):
                    intersects = True
                # Check intersection with plane_z
                elif intersects_segment_or_primitive(input_geometry, plane_z, voxel_bounds(x, y, z, voxel_size)):
                    intersects = True
                
                if intersects:
                    voxelized_voxels.add((x, y, z))
                    
    return voxelized_voxels

def intersects_segment_or_primitive(geometry, plane, bounds):
    """
    Checks if the geometry intersects the plane within the given bounds.
    This is a placeholder for the actual intersection test.
    """
    # Implementation depends on the type of geometry (line, triangle, etc.)
    # and the plane equation.
    # For a line segment: check if endpoints are on opposite sides of the plane
    # and the intersection point lies within the bounds.
    # For a triangle: check if the triangle intersects the plane and the
    # intersection polygon overlaps with the bounds.
    # Not given in the sources. [G]
    pass

def voxel_bounds(x, y, z, voxel_size):
    """
    Returns the bounding box of the voxel at (x, y, z).
    """
    min_x = x * voxel_size
    max_x = (x + 1) * voxel_size
    min_y = y * voxel_size
    max_y = (y + 1) * voxel_size
    min_z = z * voxel_size
    max_z = (z + 1) * voxel_size
    return (min_x, min_y, min_z, max_x, max_y, max_z)

class Plane:
    def __init__(self, normal, point):
        self.normal = normal
        self.point = point
        # Plane equation: normal . (p - point) = 0
        # d = -normal . point
        self.d = -sum(n * p for n, p in zip(normal, point))
    
    def distance(self, point):
        return sum(n * p for n, p in zip(self.normal, point)) + self.d
```

**Parameters:**
*   **Voxel Size:** Determines the resolution of the output grid. Not given in the sources. [G]
*   **Grid Bounds:** Defines the spatial extent of the voxelization. Not given in the sources. [G]

**Edge Cases and Pitfalls:**
*   The resulting voxelization may contain almost as many voxels as a cover produced by intersecting the input against the entire 3D voxel, making it potentially inefficient [S1, unofficial].
*   The result is not thin in the same sense as the 2D crosshairs target; stepping along a curve in the dominant direction may produce fewer voxels [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G]
*   **Memory Complexity:** Not given in the sources. [G]
*   **Measured Performance:** Not given in the sources. [G]
*   **Strengths:** Works for input composed of 2D or 3D primitives that are effectively 1D, unlike existing algorithms restricted to 1D primitives [S1, unofficial]. Produces a 26-connected path, allowing for fewer voxels than 6-connected methods [S1, unofficial].
*   **Limitations:** The resulting voxelization may contain almost as many voxels as a cover produced by intersecting the input against the entire 3D voxel, making it potentially inefficient [S1, unofficial].
*   **When to Use:** When 26-connectivity is sufficient and the input is effectively 1D but composed of 2D or 3D primitives [S1, unofficial].
*   **When Not to Use:** When a thinner representation is required or when 6-connectivity is necessary for flow simulation [S1, unofficial].

## Background
*   **Origin:** The method is described in "A Topological Approach to Voxelization" by Laine et al. [S1, unofficial].
*   **History and Development:** The method is part of a broader topological approach to voxelization, which includes 6-connected voxelization and 26-separating voxelization [S1, unofficial].
*   **Usage:** Used in applications requiring connected voxelizations of effectively 1D structures, such as flow simulation or fill operations [S1, unofficial].

## Sources
[S1] A Topological Approach to Voxelization, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
