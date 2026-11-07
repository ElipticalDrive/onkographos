# Cube Assisted Raymarching
Also called: Cube Assisted
## Overview
Cube Assisted Raymarching is a distance-field ray marching technique that stores the size of the largest empty cube centered at each voxel instead of the largest empty sphere [S1, S2, unofficial]. It solves the problem of slow shape data generation in Sphere Assisted Raymarching by trading some step optimality for significantly faster preprocessing [S1, S2, unofficial]. While Sphere Assisted Raymarching provides optimal step sizes, Cube Assisted Raymarching is an order of magnitude faster to compute the shape data, making it suitable for larger voxel grids where preprocessing time is a bottleneck [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D voxel grid where each voxel is marked as solid or empty.
- Output: A shape texture (or distance field) where each voxel stores the side length of the largest empty cube centered at that voxel.

**Data Structures:**
1. **Voxel Grid:** A 3D array of boolean or integer values indicating occupancy.
2. **Summed Area Table (SAT):** A 3D array of the same dimensions as the voxel grid. Each entry `SAT[x][y][z]` stores the sum of all solid voxels in the rectangular prism from `(0,0,0)` to `(x,y,z)` [S1, S2, unofficial].
3. **Shape Texture:** A 3D array storing the computed cube side lengths.

**Numbered Steps:**
1. **Compute Summed Area Table:** Iterate through the voxel grid and compute the SAT. For each voxel `(x,y,z)`, the value is the sum of the current voxel's occupancy plus the sums of its neighbors in the negative x, y, and z directions, minus the overlaps [S1, S2, unofficial].
2. **Binary Search for Cube Size:** For each empty voxel `(x,y,z)`, perform a binary search to find the maximum side length `s` such that a cube of side `s` centered at `(x,y,z)` contains no solid voxels [S1, S2, unofficial].
   - The binary search range is typically `[1, max_dimension]`.
   - For a candidate side length `s`, define the cube boundaries.
   - Use the SAT to check if the cube is empty in O(1) time [S1, S2, unofficial].
3. **Store Result:** Store the maximum valid `s` in the shape texture for voxel `(x,y,z)`.

**Formulas:**
- **SAT Computation:**
  $$ SAT[x][y][z] = V[x][y][z] + SAT[x-1][y][z] + SAT[x][y-1][z] + SAT[x][y][z-1] - SAT[x-1][y-1][z] - SAT[x-1][y][z-1] - SAT[x][y-1][z-1] + SAT[x-1][y-1][z-1] $$
  Where $V[x][y][z]$ is 1 if the voxel is solid, 0 otherwise. Boundary checks are required for indices < 0.

- **Cube Area Check using SAT:**
  The number of solid voxels in a box defined by corners $(x_1, y_1, z_1)$ and $(x_2, y_2, z_2)$ is:
  $$ Count = SAT[x_2][y_2][z_2] - SAT[x_1-1][y_2][z_2] - SAT[x_2][y_1-1][z_2] - SAT[x_2][y_2][z_1-1] + SAT[x_1-1][y_1-1][z_2] + SAT[x_1-1][y_2][z_1-1] + SAT[x_2][y_1-1][z_1-1] - SAT[x_1-1][y_1-1][z_1-1] $$
  The cube is empty if $Count == 0$ [S2, unofficial].

- **Cube Boundaries:**
  For a cube of side $s$ centered at $(cx, cy, cz)$, the boundaries are:
  $$ x_1 = cx - \lfloor s/2 \rfloor, \quad x_2 = cx + \lfloor (s-1)/2 \rfloor $$
  $$ y_1 = cy - \lfloor s/2 \rfloor, \quad y_2 = cy + \lfloor (s-1)/2 \rfloor $$
  $$ z_1 = cz - \lfloor s/2 \rfloor, \quad z_2 = cz + \lfloor (s-1)/2 \rfloor $$
  [G] Note: The exact boundary calculation depends on whether the cube is aligned to voxel centers or edges. The sources imply a centered cube. A common implementation uses integer division for half-sizes.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sat(voxel_grid, dims):
    """
    Computes the 3D Summed Area Table for the voxel grid.
    voxel_grid: 3D list/array of 0 (empty) or 1 (solid).
    dims: tuple (Dx, Dy, Dz)
    Returns: 3D list/array SAT
    """
    Dx, Dy, Dz = dims
    sat = [[[0 for _ in range(Dz)] for _ in range(Dy)] for _ in range(Dx)]
    
    for x in range(Dx):
        for y in range(Dy):
            for z in range(Dz):
                val = voxel_grid[x][y][z]
                sat[x][y][z] = val
                if x > 0: sat[x][y][z] += sat[x-1][y][z]
                if y > 0: sat[x][y][z] += sat[x][y-1][z]
                if z > 0: sat[x][y][z] += sat[x][y][z-1]
                if x > 0 and y > 0: sat[x][y][z] -= sat[x-1][y-1][z]
                if x > 0 and z > 0: sat[x][y][z] -= sat[x-1][y][z-1]
                if y > 0 and z > 0: sat[x][y][z] -= sat[x][y-1][z-1]
                if x > 0 and y > 0 and z > 0: sat[x][y][z] += sat[x-1][y-1][z-1]
    return sat

def get_box_solid_count(sat, x1, y1, z1, x2, y2, z2, dims):
    """
    Returns the number of solid voxels in the box [x1, x2] x [y1, y2] x [z1, z2] using SAT.
    Indices are inclusive.
    """
    Dx, Dy, Dz = dims
    # Helper to get SAT value with boundary checks
    def get_sat(x, y, z):
        if x < 0 or y < 0 or z < 0: return 0
        if x >= Dx or y >= Dy or z >= Dz: return 0
        return sat[x][y][z]
    
    return (get_sat(x2, y2, z2) 
            - get_sat(x1-1, y2, z2) 
            - get_sat(x2, y1-1, z2) 
            - get_sat(x2, y2, z1-1) 
            + get_sat(x1-1, y1-1, z2) 
            + get_sat(x1-1, y2, z1-1) 
            + get_sat(x2, y1-1, z1-1) 
            - get_sat(x1-1, y1-1, z1-1))

def compute_cube_assisted_shape(voxel_grid, dims):
    """
    Computes the Cube Assisted shape texture.
    Returns: 3D list/array of cube side lengths.
    """
    Dx, Dy, Dz = dims
    sat = compute_sat(voxel_grid, dims)
    shape = [[[0 for _ in range(Dz)] for _ in range(Dy)] for _ in range(Dx)]
    
    max_dim = max(Dx, Dy, Dz)
    
    for x in range(Dx):
        for y in range(Dy):
            for z in range(Dz):
                if voxel_grid[x][y][z] == 1:
                    shape[x][y][z] = 0
                    continue
                
                # Binary search for max cube size
                low = 1
                high = max_dim
                best_s = 0
                
                while low <= high:
                    s = (low + high) // 2
                    # Calculate cube boundaries centered at (x,y,z)
                    # Using integer division for half-sizes
                    half_s = s // 2
                    # Adjust for even/odd sizes to keep center
                    # For a cube of size s centered at integer coordinate,
                    # the extent is roughly +/- s/2.
                    # Let's define the box as [x - floor(s/2), x + floor((s-1)/2)]
                    # This ensures the center voxel is included if s is odd,
                    # and the center is between voxels if s is even? 
                    # Actually, for a voxel-centered cube, we want the cube to 
                    # cover the voxel. A simple approach is to check if a cube 
                    # of size s fits.
                    
                    # Let's use the definition: cube extends from center - s/2 to center + s/2
                    # In integer grid, we check voxels in range.
                    x1 = x - (s // 2)
                    x2 = x + ((s - 1) // 2)
                    y1 = y - (s // 2)
                    y2 = y + ((s - 1) // 2)
                    z1 = z - (s // 2)
                    z2 = z + ((s - 1) // 2)
                    
                    # Check boundaries
                    if x1 < 0 or x2 >= Dx or y1 < 0 or y2 >= Dy or z1 < 0 or z2 >= Dz:
                        high = s - 1
                        continue
                        
                    if get_box_solid_count(sat, x1, y1, z1, x2, y2, z2, dims) == 0:
                        best_s = s
                        low = s + 1
                    else:
                        high = s - 1
                
                shape[x][y][z] = best_s
                
    return shape
```

**Parameters:**
- `epsilon`: A small value added to the step size to prevent infinite loops [S1, unofficial]. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** The binary search must handle cubes that extend beyond the voxel grid boundaries. These are considered invalid or must be clamped [G].
- **Precision:** The SAT uses integer sums, so it is exact for binary voxel grids. For non-binary grids, the SAT stores sums, and the "empty" check might need a threshold [G].
- **Degenerate Input:** A single solid voxel in a large empty grid. The SAT handles this correctly. The binary search will find the largest cube that does not include the solid voxel [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Upfront shape data generation is $O(n \log n)$, where $n$ is the number of voxels [S1, S2, unofficial]. This is an improvement over the $O(n^2)$ complexity of a simple sphere-assisted implementation [S1, S2, unofficial].
- **Memory Complexity:** Requires storing the SAT and the shape texture, both of size $O(n)$ [S1, S2, unofficial].
- **Measured Performance:** In tests on a 32x32x32 voxel set, Cube Assisted Raymarching was an order of magnitude faster to compute the shape data than Sphere Assisted Raymarching [S1, S2, unofficial]. Render times were close to Sphere Assisted Raymarching on low-end sets, but Plane Marching showed a moderate lead as set size increased [S1, S2, unofficial].
- **Strengths:** Significantly faster preprocessing than Sphere Assisted Raymarching. Still provides large steps through empty space.
- **Limitations:** Slightly suboptimal step sizes compared to Sphere Assisted Raymarching [S1, S2, unofficial].
- **When to Use:** When preprocessing time is a bottleneck and the slight loss in step optimality is acceptable.
- **When Not to Use:** When optimal step sizes are critical and preprocessing time is not a concern.

## Background
Cube Assisted Raymarching was introduced by Caleb Leak in his blog post "Raymarching Voxel Rendering" [S1, S2, unofficial]. It was developed as an optimization to the Sphere Assisted Raymarching technique, which suffered from high upfront computational cost [S1, S2, unofficial]. The technique leverages Summed Area Tables, a data structure common in computer vision, to enable fast empty space queries [S1, S2, unofficial].

## Sources
[S1] Raymarching Voxel Rendering | Caleb Leak, https://calebleak.com/posts/raymarching-voxel-rendering/, fetched 2026-10-24 (unofficial)
[S2] Raymarching Voxel Rendering, https://medium.com/@calebleak/raymarching-voxel-rendering-58018201d9d6, fetched 2026-10-25 (unofficial)
