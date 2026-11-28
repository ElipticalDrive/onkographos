# Voxel Grid Bounding Box Calculation

Also called: Bounding Box Calculation, Min-Max Voxel Extent, Spatial Extent Computation

## Overview
Computes the minimum and maximum voxel coordinates occupied by a set of input voxels to determine the spatial extent of the dataset [S1, unofficial]. It returns inclusive index bounds `[[xmin, ymin, zmin], [xmax, ymax, zmax]]` where indices are absolute and may be negative [S2, unofficial]. This method serves as the foundational step for determining the active region of sparse voxel data, enabling efficient memory allocation and culling in downstream volume rendering pipelines.

## How it works
**Inputs and Outputs**
- Input: `voxels`, a collection (e.g., slice or array) of 3D integer voxel coordinates (`IVec3`).
- Output: A tuple `(min, max)` where `min` and `max` are `IVec3` structures representing the inclusive lower and upper bounds of the voxel set.

**Data Structures**
- `IVec3`: A 3-component integer vector with fields `x`, `y`, `z`.
- `IVec3::MAX`: The maximum representable integer value for each component (e.g., `(2147483647, 2147483647, 2147483647)`).
- `IVec3::MIN`: The minimum representable integer value for each component (e.g., `(-2147483648, -2147483648, -2147483648)`).
- `IVec3::ZERO`: The zero vector `(0, 0, 0)`.

**Algorithm Steps**
1. Validate the input voxel array. If the array is empty, handle the edge case as defined below [S2, unofficial].
2. Initialize `min` to `IVec3::MAX` and `max` to `IVec3::MIN` [S1, unofficial].
3. Iterate through each voxel `v` in the `voxels` collection:
   a. Update `min` component-wise: `min.x = min(min.x, v.x)`, `min.y = min(min.y, v.y)`, `min.z = min(min.z, v.z)` [S1, unofficial].
   b. Update `max` component-wise: `max.x = max(max.x, v.x)`, `max.y = max(max.y, v.y)`, `max.z = max(max.z, v.z)` [S1, unofficial].
4. Return the tuple `(min, max)` [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_min_max(voxels):
    """
    Computes the bounding box of a set of voxel coordinates.
    
    Args:
        voxels: A list or array of IVec3 tuples (x, y, z).
        
    Returns:
        A tuple (min, max) where min and max are IVec3 tuples.
    """
    # Edge case: Empty input
    if not voxels:
        # Source [S1, unofficial] returns (0,0,0) for empty input.
        # Source [S2, unofficial] suggests raising a ValueError.
        # Implementation choice: Return (0,0,0) to match [S1, unofficial] behavior, 
        # but note that this may cause downstream size=1 errors if not handled separately [S1, unofficial].
        return ((0, 0, 0), (0, 0, 0))
    
    # Initialize bounds to extreme values
    # Assuming standard 32-bit signed integers for IVec3
    MAX_INT = 2147483647
    MIN_INT = -2147483648
    
    min_coord = (MAX_INT, MAX_INT, MAX_INT)
    max_coord = (MIN_INT, MIN_INT, MIN_INT)
    
    for voxel in voxels:
        x, y, z = voxel
        # Update min
        if x < min_coord[0]: min_coord = (x, min_coord[1], min_coord[2])
        elif x == min_coord[0]:
            if y < min_coord[1]: min_coord = (x, y, min_coord[2])
            elif y == min_coord[1]:
                if z < min_coord[2]: min_coord = (x, y, z)
        
        # Update max
        if x > max_coord[0]: max_coord = (x, max_coord[1], max_coord[2])
        elif x == max_coord[0]:
            if y > max_coord[1]: max_coord = (x, y, max_coord[2])
            elif y == max_coord[1]:
                if z > max_coord[2]: max_coord = (x, y, z)
                
    return (min_coord, max_coord)
```

**Parameters**
- `voxels`: The set of voxel coordinates. No specific size limits are given, but performance scales linearly with the number of voxels [S1, unofficial].

**Edge Cases and Pitfalls**
- **Empty Input**: If the input voxel array is empty, the method returns `(0,0,0)` for both min and max [S1, unofficial]. This can cause downstream bounds calculations to produce a size of 1 instead of 0 if not handled separately [S1, unofficial]. Alternatively, some implementations raise a `ValueError` for empty inputs [S2, unofficial].
- **Negative Indices**: The returned indices are absolute and may be negative, reflecting the coordinate system of the voxel grid [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$, where $N$ is the number of voxels in the input array. Each voxel is visited exactly once [S1, unofficial].
- **Memory Complexity**: $O(1)$ auxiliary space. Only two `IVec3` variables are maintained regardless of input size [S1, unofficial].
- **Strengths**: Extremely fast and memory-efficient for sparse data. It does not require densifying the voxel grid into a dense array, unlike some other bounding box methods [S2, unofficial].
- **Limitations**: The result is sensitive to empty inputs, requiring explicit handling to avoid incorrect size calculations [S1, unofficial]. It does not account for voxel size or spacing, only integer indices [S2, unofficial].

## Background
This method is a fundamental primitive in sparse voxel processing libraries. It is used in `sparsecubes.measure` to compute the `bounding_box` of voxel sets [S2, unofficial]. It is also utilized in `svo-rs` for pathfinding and spatial queries within sparse voxel octrees [S1, unofficial]. The method is often a prerequisite for operations like `connected_components` or `volume` calculation, which require knowing the spatial extent of the data [S2, unofficial].

## Sources
[S1] mkmarek/svo-rs, https://github.com/mkmarek/svo-rs, fetched 2026-10-26 (unofficial)
[S2] navis-org/sparse-cubes, https://github.com/navis-org/sparse-cubes, fetched 2026-10-26 (unofficial)
