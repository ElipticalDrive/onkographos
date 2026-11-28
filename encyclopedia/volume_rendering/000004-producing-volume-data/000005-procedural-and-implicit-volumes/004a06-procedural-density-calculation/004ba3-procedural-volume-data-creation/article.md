# Procedural Volume Data Creation
## Overview
Procedural Volume Data Creation generates volumetric datasets algorithmically rather than from scanned or simulated physical data [S1, unofficial]. It solves the problem of creating complex, testable, or artistic volume data without external assets [S1, unofficial]. It differs from sibling methods like [Height Fog](14108-height-fog.md) by using constructive solid geometry (CSG) Boolean operations on an empty voxel grid to create complex landscapes, whereas height fog typically generates density based on altitude functions [S1, unofficial][G].
## How it works
**Inputs:** An empty voxel volume grid; a user-defined range for sphere parameters (size and position) [S1, unofficial].
**Outputs:** A voxel volume where each cell stores a scalar density value [G].
**Data Structures:** The volume is represented as a 3D array of voxels. Each voxel contains a scalar density value. The CSG operations modify these values based on the intersection with primitive shapes (spheres) [S1, unofficial].
**Steps:**
1. Initialize an empty 3D voxel grid with zero density [G].
2. Define a user-specified range for sphere radii ($R_{min}, R_{max}$) and positions ($X_{min}, X_{max}, Y_{min}, Y_{max}, Z_{min}, Z_{max}$) [S1, unofficial].
3. For each sphere $i$ in a large set (e.g., thousands):
   a. Randomly sample a radius $r_i$ from $[R_{min}, R_{max}]$ [S1, unofficial].
   b. Randomly sample a center position $c_i = (x_i, y_i, z_i)$ from the defined spatial range [S1, unofficial].
   c. Determine the bounding box of the sphere.
   d. For each voxel $v$ within the bounding box:
      i. Calculate the distance $d$ from the voxel center to $c_i$.
      ii. If $d < r_i$, apply a Boolean operation (add or subtract) to the voxel's density. For addition, $density_v \leftarrow density_v + 1$ (or a specific material value). For subtraction, $density_v \leftarrow density_v - 1$ (or clamp to zero) [S1, unofficial].
4. Return the populated voxel grid [G].
**Parameters:**
- Sphere count: "Thousands" [S1, unofficial].
- Sphere size: Random values within a user-defined range [S1, unofficial].
- Sphere position: Random values within a user-defined range [S1, unofficial].
- Voxel size: Typically 1.0, but must match the raycasting step size to avoid integration errors [G].
**Edge Cases and Pitfalls:**
- Overlapping spheres: The method uses Boolean operations, so overlaps result in additive or subtractive density, potentially creating non-physical density values if not clamped [S1, unofficial].
- Boundary conditions: Spheres near the volume boundary may be partially clipped [G].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_procedural_volume(grid_width, grid_height, grid_depth, 
                             sphere_count, 
                             pos_min, pos_max, 
                             radius_min, radius_max, 
                             operation='add',
                             voxel_size=1.0):
    """
    Creates a procedural volume using CSG operations on spheres.
    
    Args:
        grid_width, grid_height, grid_depth: Dimensions of the voxel grid.
        sphere_count: Number of spheres to generate.
        pos_min, pos_max: Tuple of (x, y, z) for the bounding box of sphere centers.
        radius_min, radius_max: Floats for the range of sphere radii.
        operation: String, 'add' or 'subtract'.
        voxel_size: Physical size of each voxel (default 1.0).
        
    Returns:
        A 3D numpy-like array of densities.
    """
    import random
    import math
    
    # Initialize empty volume
    volume = [[[0.0 for _ in range(grid_depth)] for _ in range(grid_height)] for _ in range(grid_width)]
    
    for _ in range(sphere_count):
        # Random sphere parameters
        cx = random.uniform(pos_min[0], pos_max[0])
        cy = random.uniform(pos_min[1], pos_max[1])
        cz = random.uniform(pos_min[2], pos_max[2])
        r = random.uniform(radius_min, radius_max)
        
        # Determine voxel bounds for this sphere
        # Fixed off-by-one: +1 ensures the last intersected voxel is included
        x_start = max(0, int(math.floor((cx - r) / voxel_size)))
        x_end = min(grid_width, int(math.ceil((cx + r) / voxel_size)) + 1)
        y_start = max(0, int(math.floor((cy - r) / voxel_size)))
        y_end = min(grid_height, int(math.ceil((cy + r) / voxel_size)) + 1)
        z_start = max(0, int(math.floor((cz - r) / voxel_size)))
        z_end = min(grid_depth, int(math.ceil((cz + r) / voxel_size)) + 1)
        
        for x in range(x_start, x_end):
            for y in range(y_start, y_end):
                for z in range(z_start, z_end):
                    # Calculate distance from voxel center to sphere center
                    # Voxel center is at (index + 0.5) * voxel_size
                    dx = ((x + 0.5) * voxel_size) - cx
                    dy = ((y + 0.5) * voxel_size) - cy
                    dz = ((z + 0.5) * voxel_size) - cz
                    dist_sq = dx*dx + dy*dy + dz*dz
                    
                    if dist_sq < r*r:
                        if operation == 'add':
                            volume[x][y][z] += 1.0
                        elif operation == 'subtract':
                            volume[x][y][z] -= 1.0
                            
    return volume
```
## Subtopics

### Normal Map Volume Generation
A node that generates a new volume based on an input volume to provide normals for lighting calculations in slicing or raycasting [S2, unofficial]. It outputs a volume where `color.rgb` contains the normal vector and `color.a` contains the original scalar value [S2, unofficial]. The `voxelFormat` parameter defines the output format, such as "xyzs" [S2, unofficial]. This is used when advanced visualization with correct lighting is required, as opposed to basic intensity visualization [S2, unofficial].

### Cylindrical Volume Generation with Periodic Boundary Mirroring
A technique to create a cylindrical simulation domain with vertical periodic boundary conditions by generating a subvolume and mirroring it along the vertical axis [S3]. It extracts a cubic subvolume of 256^3 voxels, removes voxels outside a radius of 124 voxels slice by slice, and mirrors the resulting cylindrical volume along its vertical axis to double the length, yielding a final domain of diameter 248 voxels and length 512 voxels [S3].

## Operating characteristics
- **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of voxels in the bounding boxes of all spheres and $S$ is the number of spheres [G]. Since spheres overlap and bounds are checked, it is efficient for sparse volumes but can be dense if spheres cover the entire grid [G].
- **Memory Complexity:** $O(W \cdot H \cdot D)$ for the voxel grid, where $W, H, D$ are grid dimensions [G].
- **Strengths:** Generates complex, organic-looking terrains or structures algorithmically without external data [S1, unofficial]. Suitable for testing algorithms as it creates "complex enough" landscapes [S1, unofficial].
- **Limitations:** The resulting density values are arbitrary (Boolean sums) and may not represent physical properties. Not suitable for physically accurate simulations [G].
- **When to use:** When testing volume rendering algorithms or creating artistic procedural landscapes [S1, unofficial].
- **When not to use:** When physically accurate density data is required [G].
## Background
Originated in the context of visualizing large procedural volumetric terrains [S1, unofficial]. The method was employed to verify algorithm feasibility by generating complex test data on the fly, rather than pre-computing entire terrains [S1, unofficial]. It is similar to other procedural methods but focuses on CSG operations for terrain synthesis [S1, unofficial].
## Sources
[S1] Visualizing Large Procedural Volumetric Terrains Using Nested Clip-Boxes, https://waseda.repo.nii.ac.jp/records/19543, fetched 2026-10-26 (unofficial)
[S2] instantreality 1.0 - tutorial - Volume Rendering, https://doc.instantreality.org/tutorial/volume-rendering/, fetched 2026-10-28 (unofficial)
[S3] Improved dynamic imaging of multiphase flow by constrained tomographic reconstruction., https://doi.org/10.1038/s41598-021-91776-1, fetched 2026-10-26
