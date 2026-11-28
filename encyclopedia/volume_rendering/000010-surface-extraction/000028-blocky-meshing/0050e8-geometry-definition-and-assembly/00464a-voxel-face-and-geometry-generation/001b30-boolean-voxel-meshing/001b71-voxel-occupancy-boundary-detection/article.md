# Voxel Occupancy Boundary Detection

Also called: Boundary Detection

## Overview
Voxel Occupancy Boundary Detection identifies voxels in a 3D occupancy grid that lie on the interface between occupied and unoccupied space, producing a boundary mask used by Boolean Voxel Meshing [971-boolean-voxel-meshing.md] to generate triangular geometry [S2, unofficial]. It operates directly on the raw boolean grid to flag boundary voxels via neighborhood checks [S1, unofficial].

## How it works
**Inputs:**
- `grid`: A 3D boolean occupancy grid of shape $(D_x, D_y, D_z)$, where `True` indicates occupied and `False` indicates empty [S2, unofficial].
- `boundarySize`: An integer radius (in voxels) from the surface to mark as boundary [S1, unofficial].

**Outputs:**
- `boundary_mask`: A 3D boolean grid of the same shape as `grid`, where `True` indicates a voxel is part of the detected boundary [S2, unofficial].

**Data Structures:**
- The grid is stored as a contiguous array or tensor [G]. For GPU implementations, it is bound as a Shader Storage Buffer Object (SSBO) [S1, unofficial]. Uniforms include `boundarySize`, `gridDims`, and `numCells` [S1, unofficial].

**Steps:**
1. **Compute Occupied Voxels:** A voxel is strictly occupied if all 8 of its corner neighbors are occupied [S2, unofficial]. For a voxel at $(x,y,z)$, this requires checking indices $(x+i, y+j, z+k)$ for $i,j,k \in \{0, 1\}$ [S2, unofficial].
2. **Compute Unoccupied Voxels:** A voxel is strictly unoccupied if all 8 of its corner neighbors are empty [S2, unofficial]. This is the negation of the logical OR of the 8 neighbors [S2, unofficial].
3. **Determine Boundary Voxels:** A voxel is on the boundary if it is neither strictly occupied nor strictly unoccupied [S2, unofficial]. This is calculated as the logical AND of the negations of the occupied and unoccupied masks [S2, unofficial].
4. **Expand Boundary (Optional):** If `boundarySize` > 1, the initial boundary mask is dilated [S1, unofficial]. The neighborhood size is adjusted to be odd if even [S1, unofficial]. Bounds are clamped to the grid dimensions [S1, unofficial]. If any voxel in the neighborhood is empty, the current voxel is marked as boundary [S1, unofficial].

**Formulas:**
Let $O[x,y,z]$ be the occupancy value at $(x,y,z)$ [G]. The masks are computed for valid interior indices where all 8 neighbors exist [S2, unofficial]:
$$
M_{occ}[x,y,z] = \bigwedge_{i,j,k \in \{0,1\}} O[x+i, y+j, z+k]
$$
$$
M_{unocc}[x,y,z] = \neg \left( \bigvee_{i,j,k \in \{0,1\}} O[x+i, y+j, z+k] \right)
$$
$$
M_{boundary}[x,y,z] = \neg M_{occ}[x,y,z] \land \neg M_{unocc}[x,y,z]
$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_occupancy_boundary_detection(grid, boundary_size=1):
    # grid: 3D list of booleans (True=occupied, False=empty)
    # boundary_size: integer radius for boundary expansion
    # Returns: boundary_mask: 3D list of booleans

    dx, dy, dz = len(grid), len(grid[0]), len(grid[0][0])

    # Initialize masks
    occupied = [[[False]*dz for _ in range(dy)] for _ in range(dx)]
    unoccupied = [[[False]*dz for _ in range(dy)] for _ in range(dx)]
    boundary = [[[False]*dz for _ in range(dy)] for _ in range(dx)]

    # Compute masks for interior voxels where all 8 neighbors exist
    for x in range(1, dx - 1):
        for y in range(1, dy - 1):
            for z in range(1, dz - 1):
                # Check all 8 corner neighbors
                neighbors = [
                    grid[x-1][y-1][z-1], grid[x-1][y-1][z],
                    grid[x-1][y][z-1],   grid[x-1][y][z],
                    grid[x][y-1][z-1],   grid[x][y-1][z],
                    grid[x][y][z-1],     grid[x][y][z]
                ]
                occ = all(neighbors)
                unocc = not any(neighbors)
                occupied[x][y][z] = occ
                unoccupied[x][y][z] = unocc
                boundary[x][y][z] = (not occ) and (not unocc)

    # Expand boundary if requested
    if boundary_size > 1:
        neighborhood_size = 2 * boundary_size + 1
        if neighborhood_size % 2 == 0:
            neighborhood_size += 1
        radius = neighborhood_size // 2

        expanded_boundary = [[[False]*dz for _ in range(dy)] for _ in range(dx)]
        for x in range(dx):
            for y in range(dy):
                for z in range(dz):
                    min_x = max(0, x - radius)
                    max_x = min(dx, x + radius + 1)
                    min_y = max(0, y - radius)
                    max_y = min(dy, y + radius + 1)
                    min_z = max(0, z - radius)
                    max_z = min(dz, z + radius + 1)

                    # Check if any voxel in the neighborhood is empty
                    has_empty = False
                    for ix in range(min_x, max_x):
                        for iy in range(min_y, max_y):
                            for iz in range(min_z, max_z):
                                if not grid[ix][iy][iz]:
                                    has_empty = True
                                    break
                        if has_empty: break
                    if has_empty:
                        expanded_boundary[x][y][z] = True
        boundary = expanded_boundary

    return boundary
```

**Parameters:**
- `boundarySize`: Integer radius (in voxels) from the surface to mark as boundary. Default is 1 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Grid Boundaries:** The 8-neighbor check requires padding or clamping to valid indices [S1, unofficial].
- **Degenerate Input:** A completely empty or completely filled grid results in an empty boundary mask, as all voxels will be classified as strictly unoccupied or strictly occupied respectively [S2, unofficial].
- **Precision:** Boolean logic is exact, but floating-point occupancy grids must be thresholded before conversion [G].

## Subtopics

### Binary Volume Cell Classification
Binary Volume Cell Classification determines if an isosurface intersects an octree leaf cell in a binary volume by checking its 8 corner voxels [S3, unofficial]. If all corners have the same value, the cell is homogeneous; if at least one corner differs, it is heterogeneous [S3, unofficial]. Heterogeneous cells contain bipolar edges (edges with endpoints of different values), which indicate isosurface intersection [S3, unofficial]. This method is used for hierarchical surface extraction where cell classification drives mesh generation [S3, unofficial].

### Grid Boundary Detection
Grid Boundary Detection determines if a voxel at $(x,y,z)$ is on the boundary of an occupied region by checking if any voxel in its neighborhood is empty [S1, unofficial]. If the neighborhood size is even, it is incremented by 1 to ensure symmetric dimensions [S1, unofficial]. The algorithm calculates min/max bounds clamped to grid dimensions and iterates through the neighborhood, returning true if any voxel is empty [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot R^3)$ where $N$ is the number of voxels and $R$ is the neighborhood radius, as each voxel is processed once with a constant number of neighbor checks [S1, unofficial].
- **Memory Complexity:** $O(N)$ for storing the input grid and output boundary mask [G].
- **Performance:** GPU implementations using compute shaders can process large grids efficiently by parallelizing the neighbor checks [S1, unofficial].
- **Strengths:** Simple, exact boolean logic; no interpolation required; suitable for binary volumes [S2, unofficial].
- **Limitations:** Only works for binary occupancy; does not handle partial voxel occupancy or scalar fields directly [S2, unofficial].
- **When to Use:** When a fast, exact boundary mask is needed for boolean meshing of binary voxel data [S1, S2, unofficial].
- **When Not to Use:** For scalar fields requiring marching cubes or for hierarchical octree-based extraction [S3, unofficial].

## Background
Voxel Occupancy Boundary Detection relies on the topological property that boundary voxels are those with mixed neighborhoods [S2, unofficial]. The method is implemented in GPU-based voxel processing pipelines, such as those in VoxelFragmentML [S1, unofficial], and differentiable volumetric rendering frameworks [S2, unofficial]. Binary cell classification variants are used in octree-based surface extraction for massive binary volumes [S3, unofficial].

## Sources
[S1] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
[S2] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S3] EXTRAÇÃO DE ISOSUPERFÍCIES DE DOMOS DE SAL EM VOLUMES BINÁRIOS MASSIVOS, https://doi.org/10.17771/pucrio.acad.51290, fetched 2026-10-26 (unofficial)
