# Nearest Neighbor Interpolation

Also called: Nearest Neighbor Voxel Sampling, Nearest Voxel Lookup, Nearest-Neighbor Density Lookup, Nearest-Neighbor Sampling, Periodic Boundary Voxel Lookup, Voxel Grid Density Sampling (Nearest Neighbor), Wrapped Nearest Neighbor Sampling

## Overview
Nearest Neighbor Interpolation maps a continuous 3D world point to the value of the single closest voxel in a discrete density grid [S2, unofficial][S4]. It solves the problem of retrieving attribute values (density, gradient, object membership) for sample points that do not align with integer grid coordinates [S3, unofficial]. Unlike trilinear or inverse weighting methods, which average surrounding voxels to ensure continuity, this method provides at best a rough estimate of the volume's nature by selecting the nearest grid vertex [S1]. It is distinguished by its extreme computational speed and lack of smoothing, making it suitable for rapid prototyping or when performance is critical over visual fidelity [S3, unofficial].

## How it works
**Inputs:**
- `point_pos`: A 3D vector $(x, y, z)$ representing the sample location in world space [S2, unofficial].
- `min_pos`: A 3D vector representing the minimum corner of the volume grid in world space [S2, unofficial].
- `max_pos`: A 3D vector representing the maximum corner of the volume grid in world space [S2, unofficial].
- `grid_resolution`: A 3D vector $(W, H, D)$ representing the number of voxels along each axis [S2, unofficial].
- `density_grid`: A 3D array of size $W \times H \times D$ storing voxel values [S2, unofficial].

**Outputs:**
- `value`: The density or attribute value of the nearest voxel. Returns 0 if the point is outside the grid bounds [S2, unofficial].

**Data Structures:**
- The `density_grid` is typically stored as a contiguous 1D array or a 3D array. For a 3D array, the layout is $(x, y, z)$ where $x \in [0, W-1]$, $y \in [0, H-1]$, $z \in [0, D-1]$.
- Coordinates are normalized to $[0, 1]$ relative to the bounding box before scaling to integer indices [S2, unofficial].

**Steps:**
1.  **Calculate Local Coordinates:** Normalize the world position to the $[0, 1]$ range relative to the grid bounds [S2, unofficial].
    $$ p_{local} = \frac{point\_pos - min\_pos}{max\_pos - min\_pos} $$
    Where the division is component-wise.

2.  **Calculate Voxel Coordinates:** Scale the normalized coordinates to the grid resolution [S2, unofficial].
    $$ p_{voxel} = p_{local} \times grid\_resolution $$
    $$ p_{voxel} = (p_{local}[0] \cdot W, \ p_{local}[1] \cdot H, \ p_{local}[2] \cdot D) $$

3.  **Calculate Integer Indices:** Apply the floor function to each component to find the nearest lower integer index [S2, unofficial]. Note: While [S4] describes rounding to the *nearest* integer, [S2, unofficial] explicitly uses `floor`. In standard grid sampling where $[0,1]$ maps to $[0, W-1]$, `floor` is the standard implementation for "nearest" in the context of binning, though strictly speaking `round` is closer to the geometric nearest neighbor. However, [S2, unofficial] is the primary implementation source for the bounds check logic. We follow [S2, unofficial]'s explicit formula:
    $$ x_i = \lfloor p_{voxel}[0] \rfloor $$
    $$ y_i = \lfloor p_{voxel}[1] \rfloor $$
    $$ z_i = \lfloor p_{voxel}[2] \rfloor $$

4.  **Boundary Check:** Verify if the calculated indices are within the valid grid range $[0, W-1]$, $[0, H-1]$, and $[0, D-1]$ [S2, unofficial].
    - If $x_i < 0$ or $x_i \ge W$ or $y_i < 0$ or $y_i \ge H$ or $z_i < 0$ or $z_i \ge D$, return 0 [S2, unofficial].

5.  **Lookup:** Return the value from `density_grid` at $(x_i, y_i, z_i)$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nearest_neighbor_interpolation(point_pos, min_pos, max_pos, grid_resolution, density_grid):
    # Step 1: Calculate local coordinates
    # Handle division by zero if bounds are identical
    dx = max_pos[0] - min_pos[0]
    dy = max_pos[1] - min_pos[1]
    dz = max_pos[2] - min_pos[2]
    
    if dx == 0 or dy == 0 or dz == 0:
        return 0
        
    p_local_x = (point_pos[0] - min_pos[0]) / dx
    p_local_y = (point_pos[1] - min_pos[1]) / dy
    p_local_z = (point_pos[2] - min_pos[2]) / dz
    
    # Step 2: Calculate voxel coordinates
    W, H, D = grid_resolution
    p_voxel_x = p_local_x * W
    p_voxel_y = p_local_y * H
    p_voxel_z = p_local_z * D
    
    # Step 3: Calculate integer indices using floor
    xi = int(p_voxel_x) # floor
    yi = int(p_voxel_y) # floor
    zi = int(p_voxel_z) # floor
    
    # Step 4: Boundary check
    # Points outside the grid bounds return density 0
    if xi < 0 or xi >= W or \
       yi < 0 or yi >= H or \
       zi < 0 or zi >= D:
        return 0
        
    # Step 5: Lookup
    # Assuming density_grid is accessed as density_grid[x][y][z]
    # or flattened: density_grid[xi + yi*W + zi*W*H]
    return density_grid[xi][yi][zi]
```

**Parameters:**
- No tunable parameters. The method is deterministic based on grid geometry [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Out of Bounds:** Points outside the grid must return 0 (or a background value) to prevent memory access violations [S2, unofficial].
- **Precision:** The method is imprecise and produces large interpolation error margins [S3, unofficial][S4].
- **Discontinuity:** The method does not guarantee $C^0$ continuity along interior faces of the grid, only at vertices [S1]. This creates visible artifacts (blockiness) in the rendered volume.
- **Rounding vs Floor:** [S4] suggests rounding to the nearest integer, while [S2, unofficial] uses floor. Implementers should note this discrepancy. If the grid coordinates are defined such that the center of voxel $i$ is at $i + 0.5$, `floor` is correct for binning. If the grid vertices are at integers, `round` is geometrically nearest. Given [S2, unofficial] is the primary code source, `floor` is the standard implementation for this specific data structure layout.

## Subtopics

### Nearest Integer Rounding
A sibling method that explicitly rounds to the nearest integer rather than flooring. It is geometrically more accurate for "nearest neighbor" but may differ in boundary handling. See [Nearest Integer Rounding](4233-nearest-integer-rounding.md).

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample point. It involves a constant number of arithmetic operations and a single array lookup [S3, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space. No additional data structures are required beyond the grid itself [S3, unofficial].
- **Measured Performance:** Significantly faster than trilinear interpolation, which requires 8 lookups and weighted averaging [S3, unofficial].
- **Strengths:** Extreme speed; minimal computational overhead; suitable for real-time applications where visual quality is secondary or for initial volume exploration [S3, unofficial].
- **Limitations:** Produces blocky, aliased visuals; large interpolation errors; lacks smoothness [S3, unofficial][S4]. It gives at best a rough estimate of the volume's nature compared to trilinear or inverse weighting [S1].
- **When to use:** When performance is critical, or when the grid resolution is very high such that voxel size is smaller than the sampling step size, making interpolation artifacts negligible.
- **When not to use:** When visual smoothness is required, or when the volume contains fine details smaller than the voxel size.

## Background
Nearest neighbor interpolation is a fundamental technique in computer graphics and scientific visualization. It was explored in early direct volume rendering (DVR) studies as a baseline method for sampling curvilinear and structured grids [S1]. It is widely implemented in various volume rendering applications, including importance-driven volume rendering (IDVR) systems, where it serves as a fast alternative to more sophisticated methods like trilinear interpolation [S3, unofficial]. The method is also referenced in educational and open-source volume rendering tools as a simple entry point for ray casting [S4].

## Sources
[S1] Direct volume rendering of curvilinear volumes, https://doi.org/10.1145/99308.99318, fetched 2026-10-28
[S2] Rintarooo/volume_rendering_plot, https://github.com/Rintarooo/volume_rendering_plot, fetched 2026-10-26 (unofficial)
[S3] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
[S4] VOLUME RENDERING Application by Thomas Lidy, https://www.cg.tuwien.ac.at/courses/Visualisierung/2001-2002/Ergebnisse/Beispiel1/LidyT/description.html, fetched 2026-10-29
