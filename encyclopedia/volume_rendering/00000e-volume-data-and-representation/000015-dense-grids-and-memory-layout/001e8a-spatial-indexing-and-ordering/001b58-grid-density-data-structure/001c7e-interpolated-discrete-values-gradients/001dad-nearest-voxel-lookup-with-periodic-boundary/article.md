# Nearest Voxel Lookup with Periodic Boundary

Also called: Grid Probing, Nearest Voxel Lookup, Nearest-Neighbor Density Lookup, Nearest-Neighbor Search Interpolation, Periodic Boundary Voxel Lookup, Wrapped Nearest Neighbor Sampling

## Overview
This method retrieves the scalar density value of the single voxel containing a sample point, without interpolating neighboring values [S1, unofficial]. It differs from interpolation methods by selecting only the nearest grid vertex [S1, unofficial]. The "Periodic Boundary" variant handles out-of-bounds coordinates by wrapping them via modulo arithmetic, allowing seamless tiling or robust handling of rays outside the grid volume [S2, unofficial].

## How it works
**Inputs:**
*   `grid`: A structure containing `bounds` (a 3D vector pair `[min, max]` defining the world-space AABB), `baseResolution` (an integer scalar or 3D vector representing voxel counts per axis), and `density` (a 1D array storing voxel values) [S1, unofficial].
*   `p`: A 3D point in world space [S1, unofficial].

**Outputs:**
*   `value`: The scalar density at the nearest voxel [S1, unofficial].

**Data Structures:**
The density array follows a Z-major, Y-major, X-minor contiguous layout [S1, unofficial]. The linear memory index $I$ for coordinates $(x, y, z)$ is calculated as $I = (z \cdot N_y + y) \cdot N_x + x$ [S1, unofficial].

**Steps:**
1.  **Transform to Local Space:** Subtract the grid's minimum bound from the sample point to shift the coordinate system so the grid origin is at $(0,0,0)$ [S1, unofficial].
    $$ p_{local} = p - grid.bounds[0] $$
2.  **Normalize:** Divide by the grid's physical size to map the point to the $[0, 1]$ range [S1, unofficial].
    $$ gridSize = grid.bounds[1] - grid.bounds[0] $$
    $$ p_{norm} = p_{local} / gridSize $$
3.  **Convert to Voxel Space:** Multiply normalized coordinates by the grid resolution to obtain floating-point voxel coordinates [S1, unofficial].
    $$ p_{voxel} = p_{norm} \cdot grid.baseResolution $$
4.  **Compute Integer Indices:** Apply the floor function to the voxel coordinates to determine the integer grid indices [S1, unofficial].
    $$ x_i = \lfloor p_{voxel}.x \rfloor $$
    $$ y_i = \lfloor p_{voxel}.y \rfloor $$
    $$ z_i = \lfloor p_{voxel}.z \rfloor $$
5.  **Apply Periodic Boundary Conditions:** If the indices fall outside the valid range $[0, N-1]$, wrap them using modulo arithmetic [S2, unofficial].
    $$ x_i = x_i \pmod{N_x} $$
    $$ y_i = y_i \pmod{N_y} $$
    $$ z_i = z_i \pmod{N_z} $$
6.  **Lookup:** Calculate the linear memory index and retrieve the value [S1, unofficial].
    $$ index = (z_i \cdot N_y + y_i) \cdot N_x + x_i $$
    $$ value = grid.density[index] $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def nearest_voxel_lookup_periodic(grid, p):
    # 1. Transform to local space
    p_local = p - grid.bounds[0]
    # 2. Normalize to [0, 1]
    grid_size = grid.bounds[1] - grid.bounds[0]
    p_norm = p_local / grid_size
    # 3. Convert to voxel space
    p_voxel = p_norm * grid.baseResolution
    # 4. Compute integer indices using floor
    xi = int(math.floor(p_voxel.x)) [G]
    yi = int(math.floor(p_voxel.y)) [G]
    zi = int(math.floor(p_voxel.z)) [G]
    # 5. Apply periodic boundary conditions (modulo)
    nx, ny, nz = grid.baseResolution
    xi = xi % nx
    yi = yi % ny
    zi = zi % nz
    # 6. Calculate linear index (Z-major, Y-major, X-minor)
    index = (zi * ny + yi) * nx + xi
    # 7. Return value
    return grid.density[index]
```

**Parameters:**
*   `grid.baseResolution`: Integer or 3D vector representing voxel counts per axis [S1, unofficial].
*   `grid.bounds`: Defined by the scene geometry or asset origin [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Negative Coordinates:** Using truncation (e.g., `static_cast<int>`) instead of `floor` returns -1 for negative coordinates, causing invalid memory access [S1, unofficial]. `floor` must be used to correctly handle negative coordinates before wrapping [S1, unofficial].
*   **Boundary Exceedance:** Coordinates in voxel space must not exceed `resolution - 1`, which occurs if normalized coordinates are exactly 1.0 [S1, unofficial]. Periodic boundary conditions via modulo handle this by wrapping $N$ to $0$ [S2, unofficial].
*   **Memory Layout:** The index calculation assumes a Z-major, Y-major, X-minor layout as specified by the source formula [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial][S2, unofficial].
*   **Measured Performance:** Not given in the sources [S1, unofficial][S2, unofficial].
*   **Strengths and Limitations:** Not given in the sources [S1, unofficial][S2, unofficial].
*   **When to use:** Not given in the sources [S1, unofficial][S2, unofficial].

## Background
This technique is referred to as "probing" in libraries like OpenVDB [S1, unofficial]. The specific handling of periodic boundaries is implemented in projects like Plenoxels to manage out-of-bounds indices via modulo arithmetic [S2, unofficial].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
