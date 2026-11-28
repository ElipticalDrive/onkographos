# Boundary-Safe Voxel Access

Also called: nearestPixel, pixelAt

## Overview
Boundary-Safe Voxel Access provides robust retrieval of voxel data by returning a defined null value for out-of-bounds coordinates, preventing memory access violations during volume rendering or ray marching. It differs from [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md) by clamping to a sentinel value rather than wrapping indices, and from [Nearest Neighbor Interpolation](5891-nearest-neighbor-interpolation.md) by explicitly handling the boundary check logic as a distinct access pattern rather than just a sampling filter.

## How it works
**Inputs:**
- `p`: A 3D point with floating-point coordinates $(p_x, p_y, p_z)$ representing the sample location in grid space.
- `X, Y, Z`: Integer dimensions of the voxel grid along the x, y, and z axes respectively.
- `data`: A flat array of voxel values.
- `nullval`: The value returned when coordinates are out of bounds.

**Outputs:**
- The voxel value at the nearest integer grid coordinate if within bounds, otherwise `nullval`.

**Data Structures:**
The voxel data is stored in a flat 1D array `data` of type `uint8_t*` (or equivalent scalar type). The memory layout is row-major with the x-axis as the fastest varying dimension, y as the middle, and z as the slowest.
- `ystride = X`
- `zstride = X * Y`
- Index calculation: $idx = x + y \cdot X + z \cdot X \cdot Y$

**Algorithm Steps:**
1. **Rounding:** Convert the continuous sample point $p$ to integer grid coordinates $(i_x, i_y, i_z)$ using nearest-neighbor rounding. The standard implementation uses:
   $$ i_x = \lfloor p_x + 0.5 \rfloor $$
   $$ i_y = \lfloor p_y + 0.5 \rfloor $$
   $$ i_z = \lfloor p_z + 0.5 \rfloor $$
   [G] Note: For negative coordinates, `int(p + 0.5)` in C-style casting may truncate towards zero. To ensure true mathematical rounding for negative numbers, use `floor(p + 0.5)` or `round(p)`. The source [S1, unofficial] implies simple truncation after adding 0.5, which works correctly for positive coordinates but may behave unexpectedly for negative ones depending on language semantics. We assume standard integer truncation towards zero for the pseudocode below, matching the source's `int()` notation.

2. **Boundary Check:** Verify if the rounded coordinates are within the grid limits $[0, X)$, $[0, Y)$, and $[0, Z)$.
   $$ \text{if } i_x < 0 \lor i_x \ge X \lor i_y < 0 \lor i_y \ge Y \lor i_z < 0 \lor i_z \ge Z $$

3. **Return:**
   - If out of bounds: Return `nullval`.
   - If in bounds: Calculate the linear index $idx = i_x + i_y \cdot X + i_z \cdot X \cdot Y$ and return `data[idx]`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def boundary_safe_voxel_access(p_x, p_y, p_z, X, Y, Z, data, nullval):
    # Step 1: Round to nearest integer
    # Using int() which truncates towards zero. 
    # For strict nearest-neighbor rounding including negatives, 
    # floor(p + 0.5) is safer, but we follow source [S1, unofficial] style.
    ix = int(p_x + 0.5)
    iy = int(p_y + 0.5)
    iz = int(p_z + 0.5)

    # Step 2: Check bounds
    if ix < 0 or ix >= X or \
       iy < 0 or iy >= Y or \
       iz < 0 or iz >= Z:
        return nullval

    # Step 3: Access data with stride
    # Layout: x-fastest, y-middle, z-slowest
    # ystride = X, zstride = X * Y
    index = ix + iy * X + iz * (X * Y)
    return data[index]
```

**Parameters:**
- `nullval`: Default value for out-of-bounds access. Typically 0 for density/opacity, or a specific material ID for material queries.

**Edge Cases and Pitfalls:**
- **Negative Coordinates:** The rounding method `int(p + 0.5)` behaves differently for negative numbers than `round()` in some languages. For example, `int(-0.6 + 0.5) = int(-0.1) = 0`, whereas `round(-0.6) = -1`. Implementers should verify the desired rounding behavior for points near the negative boundary.
- **Precision:** Floating-point precision errors may cause a point very close to the boundary to round to an out-of-bounds index. This is generally handled correctly by the boundary check, but may result in "holes" if `nullval` is not handled gracefully by the rendering pipeline.

## Subtopics

### Voxel Material Query with Neighbor Fallback
Retrieves the material byte at a world point, falling back to checking 6 neighboring voxels if the target voxel is air. This is useful for ensuring solid surfaces are detected even if the ray passes through the center of an air voxel adjacent to a solid one.
- **Definition:** Checks the primary voxel; if it is air, iterates through 6 neighbors to find a non-air material.
- **Steps:**
  1. Convert world point to voxel coordinates.
  2. Find the containing chunk and local vertex coordinates.
  3. Query the material at the primary coordinate.
  4. If the material is air, define 6 neighbor offsets: $(0,-1,0), (1,0,0), (-1,0,0), (0,0,1), (0,0,-1), (0,1,0)$. [S2, unofficial] Note: The source lists `(0,0,1)` twice and omits `(0,0,-1)`, but "6 neighbors" implies the full set of axial neighbors. We assume the standard 6 axial neighbors.
  5. Loop through neighbors, calculate their chunk and local vertex, and query material. Return the first non-air material found.
- **When to choose:** Use when rendering thin surfaces or when ray marching through sparse data where center-point sampling might miss solid voxels.
- **Link:** [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md)

## Operating characteristics
- **Time Complexity:** $O(1)$ for the primary access. The neighbor fallback variant is $O(k)$ where $k$ is the number of neighbors checked (typically 6), but still constant time.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Prevents segmentation faults and undefined behavior from out-of-bounds memory access. Simple to implement.
- **Limitations:** Does not provide interpolation; results in blocky artifacts if used for shading without higher-order interpolation. The neighbor fallback adds branching overhead.
- **When to use:** Essential for any volume rendering pipeline that samples outside the defined grid volume. Use the neighbor fallback variant for material identification in marching cubes or similar surface extraction algorithms where air voxels should not terminate the search if adjacent to solid geometry.

## Background
Boundary-safe access is a fundamental utility in volume rendering and grid-based simulations. The specific implementation details (flat array, x-fastest layout, `pixelAt`/`nearestPixel` naming) are documented in the `gvox` library [S1, unofficial]. The neighbor fallback strategy is observed in implementations of marching cubes on Unity, where material continuity is required across voxel boundaries [S2, unofficial].

## Sources
[S1] wagenadl/gvox, https://github.com/wagenadl/gvox, fetched 2026-10-28 (unofficial)
[S2] Javier-Garzo/Marching-cubes-on-Unity-3D, https://github.com/Javier-Garzo/Marching-cubes-on-Unity-3D, fetched 2026-10-26 (unofficial)
