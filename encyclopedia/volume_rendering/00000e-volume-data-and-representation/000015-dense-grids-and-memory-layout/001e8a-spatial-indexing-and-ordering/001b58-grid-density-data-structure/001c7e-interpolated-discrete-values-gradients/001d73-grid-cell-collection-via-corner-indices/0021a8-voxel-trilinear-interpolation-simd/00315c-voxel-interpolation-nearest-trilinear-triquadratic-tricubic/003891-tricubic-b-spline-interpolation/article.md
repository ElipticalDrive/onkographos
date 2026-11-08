# Tricubic B-Spline Interpolation
## Overview
A GPU-accelerated interpolation method using uniform cubic B-splines to sample a 3D volume texture, providing smoother results than linear interpolation while being faster than full tricubic convolution [S1, unofficial]. It sets itself apart from standard tricubic convolution by using a specific optimized weight grouping and coordinate transformation that reduces the number of texture fetches and arithmetic operations, making it suitable for real-time volume rendering on WebGL-compatible hardware [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A 3D volume texture (stored in a 2D texture array or similar format to work around WebGL limitations [S1, unofficial]), a 3D coordinate `coord` in normalized space $[0, 1]$, and the dimensions of the volume `nrOfVoxels`.
- Output: A scalar value (float) representing the interpolated density or color at `coord`.

**Data Structures:**
- `coord`: `vec3` in $[0, 1]^3$.
- `nrOfVoxels`: `vec3` containing the width, height, and depth of the volume grid.
- `texture3Dfrom2D`: A custom sampler function that maps 3D grid coordinates to a 2D texture array or packed 2D texture format.

**Steps:**
1.  **Shift Coordinate:** Transform the normalized coordinate to grid space, shifting the origin to the center of the first voxel.
    $$ \text{coord\_grid} = \text{coord} \times \text{nrOfVoxels} - 0.5 $$
2.  **Index and Fraction:** Compute the integer voxel index and the fractional offset within that voxel.
    $$ \text{index} = \lfloor \text{coord\_grid} \rfloor $$
    $$ \text{fraction} = \text{coord\_grid} - \text{index} $$
    $$ \text{one\_frac} = 1.0 - \text{fraction} $$
3.  **Compute Weights:** Calculate the four B-spline basis weights for each axis based on the fraction $f$.
    $$ w_0 = \frac{1}{6} (1-f)^3 $$
    $$ w_1 = \frac{2}{3} - \frac{1}{2} f^2 (2-f) $$
    $$ w_2 = \frac{2}{3} - \frac{1}{2} (1-f)^2 (2-(1-f)) $$
    $$ w_3 = \frac{1}{6} f^3 $$
    *Note: The source implementation groups these weights for optimization.*
4.  **Group Weights:**
    $$ g_0 = w_0 + w_1 $$
    $$ g_1 = w_2 + w_3 $$
5.  **Compute Sample Coordinates:** Calculate the normalized sample coordinates for the two rows of voxels along each axis. Let $mult = 1.0 / \text{nrOfVoxels}$.
    $$ h_0 = mult \times \left( \frac{w_1}{g_0} - 0.5 + \text{index} \right) $$
    $$ h_1 = mult \times \left( \frac{w_3}{g_1} + 1.5 + \text{index} \right) $$
6.  **Interpolate along X (First Y/Z Plane):**
    Fetch two samples along the X-axis for the first Y/Z plane defined by $h_0$.
    $$ \text{tex000} = \text{texture3Dfrom2D}(h_0).r $$
    $$ \text{tex100} = \text{texture3Dfrom2D}(\text{vec3}(h_1.x, h_0.y, h_0.z)).r $$
    Interpolate along X:
    $$ \text{tex000} = \text{mix}(\text{tex100}, \text{tex000}, g_0.x) $$
7.  **Interpolate along X (Second Y/Z Plane):**
    Fetch two samples along the X-axis for the second Y/Z plane defined by $h_0$ in Y/Z but $h_1$ in Y.
    $$ \text{tex010} = \text{texture3Dfrom2D}(\text{vec3}(h_0.x, h_1.y, h_0.z)).r $$
    $$ \text{tex110} = \text{texture3Dfrom2D}(\text{vec3}(h_1.x, h_1.y, h_0.z)).r $$
    Interpolate along X:
    $$ \text{tex010} = \text{mix}(\text{tex110}, \text{tex010}, g_0.x) $$
8.  **Interpolate along Y (First Z Plane):**
    $$ \text{tex000} = \text{mix}(\text{tex010}, \text{tex000}, g_0.y) $$
9.  **Interpolate along X (Third Y/Z Plane):**
    Fetch two samples along the X-axis for the third Y/Z plane defined by $h_0$ in X/Y but $h_1$ in Z.
    $$ \text{tex001} = \text{texture3Dfrom2D}(\text{vec3}(h_0.x, h_0.y, h_1.z)).r $$
    $$ \text{tex101} = \text{texture3Dfrom2D}(\text{vec3}(h_1.x, h_0.y, h_1.z)).r $$
    Interpolate along X:
    $$ \text{tex001} = \text{mix}(\text{tex101}, \text{tex001}, g_0.x) $$
10. **Interpolate along X (Fourth Y/Z Plane):**
    Fetch two samples along the X-axis for the fourth Y/Z plane defined by $h_1$ in Y/Z.
    $$ \text{tex011} = \text{texture3Dfrom2D}(\text{vec3}(h_0.x, h_1.y, h_1.z)).r $$
    $$ \text{tex111} = \text{texture3Dfrom2D}(h_1).r $$
    Interpolate along X:
    $$ \text{tex011} = \text{mix}(\text{tex111}, \text{tex011}, g_0.x) $$
11. **Interpolate along Y (Second Z Plane):**
    $$ \text{tex001} = \text{mix}(\text{tex011}, \text{tex001}, g_0.y) $$
12. **Final Interpolation along Z:**
    $$ \text{result} = \text{mix}(\text{tex001}, \text{tex000}, g_0.z) $$
    Return `result`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tricubic_b_spline_interpolation(coord, nr_of_voxels, texture_sampler):
    """
    coord: vec3 in [0, 1]
    nr_of_voxels: vec3 (width, height, depth)
    texture_sampler: function that takes a vec3 coordinate and returns a float (r component)
    """
    # 1. Shift coordinate
    coord_grid = coord * nr_of_voxels - 0.5
    
    # 2. Index and Fraction
    index = floor(coord_grid)
    fraction = coord_grid - index
    one_frac = 1.0 - fraction
    
    # 3. Compute Weights
    f = fraction
    w0 = (1.0 / 6.0) * (one_frac ** 3)
    w1 = (2.0 / 3.0) - (0.5 * f ** 2 * (2.0 - f))
    w2 = (2.0 / 3.0) - (0.5 * (one_frac ** 2) * (2.0 - one_frac))
    w3 = (1.0 / 6.0) * (f ** 3)
    
    # 4. Group Weights
    g0 = w0 + w1
    g1 = w2 + w3
    
    # 5. Compute Sample Coordinates
    mult = 1.0 / nr_of_voxels
    h0 = mult * ((w1 / g0) - 0.5 + index)
    h1 = mult * ((w3 / g1) + 1.5 + index)
    
    # 6. Interpolate along X (First Y/Z Plane)
    tex000 = texture_sampler(h0)
    tex100 = texture_sampler(vec3(h1.x, h0.y, h0.z))
    tex000 = mix(tex100, tex000, g0.x)
    
    # 7. Interpolate along X (Second Y/Z Plane)
    tex010 = texture_sampler(vec3(h0.x, h1.y, h0.z))
    tex110 = texture_sampler(vec3(h1.x, h1.y, h0.z))
    tex010 = mix(tex110, tex010, g0.x)
    
    # 8. Interpolate along Y (First Z Plane)
    tex000 = mix(tex010, tex000, g0.y)
    
    # 9. Interpolate along X (Third Y/Z Plane)
    tex001 = texture_sampler(vec3(h0.x, h0.y, h1.z))
    tex101 = texture_sampler(vec3(h1.x, h0.y, h1.z))
    tex001 = mix(tex101, tex001, g0.x)
    
    # 10. Interpolate along X (Fourth Y/Z Plane)
    tex011 = texture_sampler(vec3(h0.x, h1.y, h1.z))
    tex111 = texture_sampler(h1)
    tex011 = mix(tex111, tex011, g0.x)
    
    # 11. Interpolate along Y (Second Z Plane)
    tex001 = mix(tex011, tex001, g0.y)
    
    # 12. Final Interpolation along Z
    return mix(tex001, tex000, g0.z)
```

**Parameters:**
- `nrOfVoxels`: The dimensions of the 3D volume texture. Typically `uResolution` [S1, unofficial].

**Edge Cases and Pitfalls:**
- The implementation uses `texture3Dfrom2D` instead of standard `texture3D`, implying the 3D volume is stored in a 2D texture array or similar format to work around WebGL limitations or performance constraints [S1, unofficial].
- Boundary conditions for coordinates outside $[0, 1]$ are not explicitly defined in the source steps, but standard clamping or wrapping should be applied before the shift step if necessary [G].
- Division by zero in weight calculation if $g_0$ or $g_1$ becomes zero. This occurs if all weights are zero, which is theoretically impossible for valid fractions, but numerical precision issues should be monitored [G].

## Operating characteristics
**Time and Memory Complexity:**
- Time: Constant time $O(1)$ per sample, with a fixed number of texture fetches (8) and arithmetic operations.
- Memory: $O(1)$ auxiliary memory. The volume data size depends on the resolution.

**Measured Performance:**
- Specific performance metrics are not provided in the sources [S1, unofficial].

**Scaling:**
- Scales linearly with the number of samples. The complexity is independent of the volume resolution, as it relies on texture sampling hardware.

**Strengths and Limitations:**
- Strengths: Smoother results than trilinear interpolation; faster than full tricubic convolution due to optimized weight grouping and reduced fetches [S1, unofficial].
- Limitations: Requires a specific texture layout (2D texture array or similar) to work around WebGL limitations [S1, unofficial]. Not as smooth as higher-order splines or convolution-based methods.

**When to Use:**
- Real-time volume rendering on WebGL-compatible hardware where smoothness is desired but full tricubic convolution is too slow [S1, unofficial].

**When Not to Use:**
- When highest possible smoothness is required (use higher-order splines or convolution).
- When hardware supports standard `texture3D` efficiently and performance is not a bottleneck (standard tricubic may be simpler to implement).

## Background
**Origin:**
- The method is described in the `sharevol` project by OKaluza [S1, unofficial].

**History and Development:**
- The specific optimization using grouped weights and coordinate transformation for B-spline interpolation in WebGL is attributed to the `sharevol` repository [S1, unofficial].

**Where it is Used:**
- WebGL-based volume rendering applications [S1, unofficial].

## Sources
[S1] OKaluza/sharevol, https://github.com/OKaluza/sharevol, fetched 2026-10-27 (unofficial)
