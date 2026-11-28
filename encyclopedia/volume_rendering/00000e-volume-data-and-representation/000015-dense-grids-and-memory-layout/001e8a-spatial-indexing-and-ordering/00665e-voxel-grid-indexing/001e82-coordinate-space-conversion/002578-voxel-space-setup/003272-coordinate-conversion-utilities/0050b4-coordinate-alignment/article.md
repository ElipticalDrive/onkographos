# Coordinate Alignment
## Overview
Coordinate alignment adjusts the spatial position of a voxel-based object by shifting its coordinates so that a specific statistical property (minimum, mean, median, or maximum) of each axis aligns with a target location [S1, unofficial]. It solves the problem of centering or positioning discrete voxel data relative to a coordinate system without altering the underlying voxel topology or requiring resampling [G]. It differs from [Metric Coordinate Conversion](12960-metric-coordinate-conversion.md) by operating on raw coordinate values rather than physical units, and from [Volume Geometry Transformations](17760-volume-geometry-transformations.md) by modifying the coordinate data directly rather than metadata [G].

## How it works
**Inputs and Outputs:**
- Input: `coords`, a `data.frame` with columns `x`, `y`, and `z` representing the coordinates of the object's voxels [S1, unofficial].
- Input: `loc`, a target location vector (default: `c(0, 0, 0)`) [S1, unofficial].
- Input: `x`, `y`, `z`, strings specifying the alignment method for each axis. Valid values are `'min'`, `'mean'`, `'max'`, `'identity'`, `'median'`. Default: `'mean'` [S1, unofficial].
- Output: A transformed `data.frame` with the same dimensions as `coords`, containing the shifted coordinates [S1, unofficial].

**Data Structures:**
The input `coords` is treated as a table of points where each row represents a voxel and columns represent the spatial dimension [G]. No bit-packing is specified; standard floating-point or integer types are assumed for coordinates [G].

**Steps:**
1. For each axis $a \in \{x, y, z\}$:
   a. Calculate the statistic $S_a$ of the current coordinates $C_a$ based on the method specified in parameter $a$ [S1, unofficial].
      - If method is `'min'`: $S_a = \min(C_a)$ [G]
      - If method is `'mean'`: $S_a = \frac{1}{N} \sum_{i=1}^{N} C_{a,i}$ [G]
      - If method is `'max'`: $S_a = \max(C_a)$ [G]
      - If method is `'median'`: $S_a = \text{median}(C_a)$ [G]
      - If method is `'identity'`: $S_a = 0$ (no shift applied) [G]
   b. Calculate the shift amount $\Delta_a = \text{loc}_a - S_a$ [G].
   c. Update the coordinates: $C'_a = C_a + \Delta_a$ [G].
2. Return the updated coordinates [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coord_align(coords, loc=(0.0, 0.0, 0.0), x_method='mean', y_method='mean', z_method='mean'):
    """
    Aligns voxel coordinates to a target location.
    
    Args:
        coords: List of tuples or array-like structure with (x, y, z) values.
        loc: Tuple (x, y, z) target location.
        x_method, y_method, z_method: String, one of 'min', 'mean', 'max', 'median', 'identity'.
        
    Returns:
        List of tuples with transformed (x, y, z) values.
    """
    if not coords:
        return []
    xs = [p[0] for p in coords]
    ys = [p[1] for p in coords]
    zs = [p[2] for p in coords]
    
    def get_statistic(values, method):
        if method == 'identity':
            return 0.0
        elif method == 'min':
            return min(values)
        elif method == 'max':
            return max(values)
        elif method == 'mean':
            return sum(values) / len(values)
        elif method == 'median':
            sorted_vals = sorted(values)
            n = len(sorted_vals)
            if n % 2 == 0:
                return (sorted_vals[n//2 - 1] + sorted_vals[n//2]) / 2.0
            else:
                return sorted_vals[n//2]
        else:
            raise ValueError(f"Unknown method: {method}")

    shift_x = loc[0] - get_statistic(xs, x_method)
    shift_y = loc[1] - get_statistic(ys, y_method)
    shift_z = loc[2] - get_statistic(zs, z_method)
    
    aligned_coords = [
        (p[0] + shift_x, p[1] + shift_y, p[2] + shift_z)
        for p in coords
    ]
    
    return aligned_coords
```

**Edge Cases and Pitfalls:**
- Empty input: If `coords` is empty, statistical methods (`min`, `max`, `mean`, `median`) are undefined [G]. The implementation should handle this by returning empty output or raising an error [G].
- Single voxel: Median and mean are identical; min and max are identical [G].
- Precision: Floating-point addition may introduce minor precision errors, though typically negligible for voxel coordinates [G].

## Subtopics

### [Volume Geometry Transformations](17760-volume-geometry-transformations.md)
Modifies spatial attributes (position, size, shape) of volume geometry while preserving voxel grid topology, except for `reshape`. It operates on `VolumeGeometry` metadata rather than resampling data, allowing efficient coordinate adjustments without interpolation.

## Operating characteristics
- Time Complexity: $O(N)$ where $N$ is the number of voxels, as it requires a single pass to compute statistics and a second pass to apply shifts [G].
- Memory Complexity: $O(N)$ to store the output coordinates [G].
- Strengths: Fast, non-destructive to voxel data topology, flexible alignment targets [G].
- Limitations: Does not handle rotation or scaling; only translation [G].
- When to use: When precise positioning of voxel objects is required relative to a coordinate system or other objects [G].
- When not to use: When physical metric conversion or rotation is needed [G].

## Background
Originated in the `isocubes` R package for voxel data visualization, providing utilities to manipulate voxel coordinates for rendering [S1, unofficial]. It is part of a suite of coordinate conversion utilities including `coord_translate` and `coord_rotate` [S1, unofficial].

## Sources
[S1] isocubes: Voxel Data Visualization with Isometric Cubes, https://doi.org/10.32614/cran.package.isocubes, fetched 2026-10-26 (unofficial)
