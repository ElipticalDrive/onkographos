# Relaxed Connectivity Voxelization

Also called: 26-connected voxelization of 1D input

## Overview
Relaxed Connectivity Voxelization is a method for voxelizing effectively one-dimensional input (1D, 2D, or 3D primitives) in 3D space that ensures the resulting voxels form a connected path, preventing gaps in flow simulation or fill operations [S1, unofficial]. It produces a 26-connected path, allowing for fewer voxels than 6-connected methods while still maintaining connectivity [S1, unofficial]. Unlike existing algorithms restricted to 1D primitives (e.g., line segments), this method works for input composed of 2D or 3D primitives that are effectively 1D [S1, unofficial].

## How it works
The method relies on defining specific **intersection targets** within each voxel. If the input geometry intersects the target in a voxel $V$, connectivity is guaranteed to the neighbors.

### Inputs and Outputs
*   **Input**: A continuous geometric primitive (curve, surface, or volume) that is "effectively one-dimensional" in 3D space.
*   **Output**: A set of voxel indices $(i, j, k)$ representing the voxelized path.

### Data Structures
*   **Voxel Grid**: Standard 3D integer grid.
*   **Intersection Target**: Defined geometrically within the voxel coordinate space $[0, 1]^3$.

### Steps

#### 1. Define the Intersection Target
For **26-connected voxelization**, the intersection target consists of three quadrilaterals that bisect the voxel along all three axes [S1, unofficial].
*   Let the voxel be defined by $x, y, z \in [0, 1]$.
*   The target comprises:
    1.  The square $x = 0.5, y \in [0, 1], z \in [0, 1]$.
    2.  The square $y = 0.5, x \in [0, 1], z \in [0, 1]$.
    3.  The square $z = 0.5, x \in [0, 1], y \in [0, 1]$.
*   This configuration subdivides the space into 8 cubes centered at the voxel corners [S1, unofficial].

*(Note: For 6-connected voxelization, the target is more complex, involving surfaces on each face of the voxel. Two practical realizations are:)*
*   **Octahedral Tessellation**: Close each face with pyramids meeting at the voxel center $(0.5, 0.5, 0.5)$ [S1, unofficial].
*   **Cubical Tessellation**: Push the surfaces to the boundaries of the voxels, resulting in a 2D target consisting of only the surface of the cube [S1, unofficial].

#### 2. Intersect Input with Target
For each voxel $V$ in the grid:
1.  Compute the intersection of the input geometry with the defined intersection target $T_V$.
2.  If the intersection is non-empty, mark voxel $V$ as occupied.

#### 3. Verify Connectivity (Theoretical Guarantee)
The connectivity is guaranteed by the topology of the targets:
*   If input intersects the target in voxel $V$, parts of the intersection targets of the 26-neighbors of $V$ form a closed box with side length twice that of a single voxel [S1, unofficial].
*   Any continuous path from $V$ to voxels outside immediate neighbors must intersect this enclosing box.
*   Therefore, the path must intersect the target in $N_{26}(V)$, ensuring 26-connectivity [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def relaxed_connectivity_voxelization(input_geometry, grid_origin, grid_size, resolution):
    """
    Voxelizes effectively 1D input using 26-connected intersection targets.
    
    Args:
        input_geometry: Object with method intersects(target) -> bool
        grid_origin: (x, y, z) tuple
        grid_size: (dx, dy, dz) tuple (physical dimensions of the grid)
        resolution: (nx, ny, nz) tuple (number of voxels per axis)
        
    Returns:
        set of (i, j, k) indices
    """
    occupied_voxels = set()
    
    # Precompute voxel dimensions
    voxel_dx = grid_size[0] / resolution[0]
    voxel_dy = grid_size[1] / resolution[1]
    voxel_dz = grid_size[2] / resolution[2]
    
    for i in range(resolution[0]):
        for j in range(resolution[1]):
            for k in range(resolution[2]):
                # Define voxel bounds in world space
                x_min = grid_origin[0] + i * voxel_dx
                x_max = x_min + voxel_dx
                y_min = grid_origin[1] + j * voxel_dy
                y_max = y_min + voxel_dy
                z_min = grid_origin[2] + k * voxel_dz
                z_max = z_min + voxel_dz
                
                # Define the 26-connected intersection target in world space
                # Target 1: Bisect X
                target_x = Plane(x=0.5, bounds=(y_min, y_max, z_min, z_max))
                # Target 2: Bisect Y
                target_y = Plane(y=0.5, bounds=(x_min, x_max, z_min, z_max))
                # Target 3: Bisect Z
                target_z = Plane(z=0.5, bounds=(x_min, x_max, y_min, y_max))
                
                # Check intersection with any part of the target
                if input_geometry.intersects(target_x) or \
                   input_geometry.intersects(target_y) or \
                   input_geometry.intersects(target_z):
                    occupied_voxels.add((i, j, k))
                    
    return occupied_voxels

class Plane:
    def __init__(self, x=None, y=None, z=None, bounds=None):
        self.normal_axis = 'x' if x is not None else ('y' if y is not None else 'z')
        self.constant = x if x is not None else (y if y is not None else z)
        self.bounds = bounds # (min1, max1, min2, max2) for the other two axes

    def intersects(self, geometry):
        # Placeholder for actual geometric intersection logic
        # This would check if the input geometry crosses the plane within the bounds
        pass
```

### Parameters and Edge Cases
*   **Input Type**: The method is designed for "effectively one-dimensional" input. This includes 1D primitives (lines, curves) but also 2D or 3D primitives that are thin enough to be treated as 1D [S1, unofficial].
*   **Connectivity Choice**: 26-connected is preferred for efficiency (fewer voxels) over 6-connected [S1, unofficial].
*   **Pitfall**: The resulting voxelization may contain almost as many voxels as a cover produced by intersecting the input against the entire 3D voxel, making it potentially inefficient for thick objects [S1, unofficial].
*   **Pitfall**: The result is not "thin" in the same sense as the 2D crosshairs target. Stepping along a curve in the dominant direction may produce fewer voxels than this method [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Depends on the intersection test between the input geometry and the target planes. For complex inputs, this can be expensive.
*   **Memory Complexity**: Proportional to the number of occupied voxels.
*   **Performance**: The method may be inefficient for inputs that are not truly thin, as it can produce a dense cover [S1, unofficial].
*   **Strengths**: Works for 2D/3D primitives that are effectively 1D, unlike many existing algorithms restricted to 1D primitives [S1, unofficial]. Guarantees topological connectivity (26-connected).
*   **Limitations**: Can produce more voxels than necessary for thick objects. Not "thin" in the strict sense of minimal voxel count for curves.

## Background
This method is part of a topological approach to voxelization, specifically addressing the connectivity of voxelized paths for effectively one-dimensional input in 3D space [S1, unofficial]. It was developed to allow flow simulation or fill operations to proceed within voxelized curves without gaps, and to guarantee that visualized voxelizations are not missing pieces [S1, unofficial]. The 26-connected variant is more interesting than the 6-connected one because it avoids the overkill of a full 3D cover while still maintaining connectivity [S1, unofficial].

## Sources
[S1] A Topological Approach to Voxelization - NVIDIA, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
