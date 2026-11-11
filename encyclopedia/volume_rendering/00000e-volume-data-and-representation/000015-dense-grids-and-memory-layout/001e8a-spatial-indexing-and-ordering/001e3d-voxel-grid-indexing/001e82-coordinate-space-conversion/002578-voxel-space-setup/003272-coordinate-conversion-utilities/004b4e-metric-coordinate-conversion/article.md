# Metric Coordinate Conversion

Also called: World Metric Coordinate Construction

## Overview
Metric Coordinate Conversion decomposes a continuous world-space position into a discrete integer component (whole meters) and a continuous fractional component (sub-meter precision). This decomposition enables efficient spatial indexing and caching in voxel-based rendering systems by separating coarse grid alignment from fine-grained sampling offsets. It differs from [Sliced Volume Coordinate Mapping](8287-sliced-volume-coordinate-mapping.md) by operating on absolute metric units rather than slice indices, and from [Voxel Index Computation](11238-voxel-index-computation.md) by preserving sub-voxel precision rather than truncating to integer voxel coordinates.

## How it works
**Inputs:**
- `localPosition`: A 3D vector representing the position in world space (or local chunk space, depending on context).
- `chunkCoordinate`: An integer representing the current chunk or block index along the axis (used to compute absolute whole meters).
- `CHUNK_SIZE`: An integer constant defining the size of a chunk in meters (or voxels, depending on the specific implementation context; here assumed to be meters for metric consistency).

**Outputs:**
- `WorldMetricCoordinate`: A struct containing `wholeMeters` and `fractionalMeter`.

**Data Structure:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct WorldMetricCoordinate {
    int64_t wholeMeters;   // Signed 64-bit integer for whole meter offset
    float fractionalMeter; // 32-bit float for sub-meter precision [0.0, 1.0)
};
```

**Steps:**
1. **Compute Whole Meters:** Calculate the integer part of the local position using floor: `localWholeMeters = floor(localPosition)`.
2. **Calculate Absolute Whole Meters:** Combine the chunk coordinate with the local whole meters: `axis.wholeMeters = chunkCoordinate * CHUNK_SIZE + localWholeMeters`.
3. **Compute Fractional Meter:** Extract the fractional part: `axis.fractionalMeter = localPosition - localWholeMeters`.
4. **Validation:** Assert that `axis.fractionalMeter` is in the range `[0.0f, 1.0f)`. This is guaranteed by the floor-based decomposition for positive values, but requires care for negative values if `floor` is not used correctly (standard `floor` handles negatives correctly, e.g., `floor(-0.5) = -1`, so `-0.5 - (-1) = 0.5`).

**Pitfalls:**
- **Fractional Precision Boundary:** The `fractionalMeter` must be strictly less than `1.0f`. Floating-point errors could theoretically push it to `1.0f` or slightly above. The assertion catches this, but implementation should ensure `floor` is used, not truncation towards zero.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convert_to_metric_coordinate(local_position: float, chunk_coordinate: int, chunk_size: int) -> dict:
    """
    Converts a local position to a metric coordinate structure.
    
    Args:
        local_position: The position in local space (float).
        chunk_coordinate: The integer index of the current chunk (int).
        chunk_size: The size of the chunk in meters (int).
        
    Returns:
        A dictionary with 'wholeMeters' (int64) and 'fractionalMeter' (float).
    """
    # Step 1: Compute whole meters from local position
    local_whole_meters = math.floor(local_position)
    
    # Step 2: Calculate absolute whole meters
    # Note: chunkCoordinate * CHUNK_SIZE assumes chunk_coordinate is in units of chunks
    # and chunk_size is the size of each chunk in meters.
    whole_meters = chunk_coordinate * chunk_size + local_whole_meters
    
    # Step 3: Calculate fractional meter
    fractional_meter = local_position - local_whole_meters
    
    # Step 4: Assert boundary condition
    # fractional_meter must be in [0.0, 1.0)
    if not (0.0 <= fractional_meter < 1.0):
        raise ValueError(f"Fractional meter {fractional_meter} is out of bounds [0.0, 1.0)")
        
    return {
        "wholeMeters": int(whole_meters),
        "fractionalMeter": fractional_meter
    }
```

## Subtopics

### Metric Coordinate Offset
Adds an offset to a `WorldMetricCoordinate`. The offset is decomposed into whole and fractional parts. The fractional part is added to `fractionalMeter`. If the sum exceeds `1.0f`, the excess is carried over to `wholeMeters` via `floor`. This prevents fractional rollover errors. [S1, unofficial]

## Operating characteristics
- **Time Complexity:** O(1) per axis.
- **Memory Complexity:** O(1) for the struct.
- **Strengths:** Provides exact sub-meter precision while maintaining integer-based spatial hashing or chunking.
- **Limitations:** Requires careful handling of floating-point precision near boundaries. Not suitable for direct voxel indexing without further conversion.

## Background
Originated in the PlaneVoxelLab project by Aaron Eugene, this method is part of a broader system for efficient volume rendering using dense grids. It facilitates the mapping of continuous world coordinates to discrete spatial structures. [S1, unofficial]

## Sources
[S1] aaron-eugene/PlaneVoxelLab, https://github.com/aaron-eugene/PlaneVoxelLab, fetched 2026-10-27 (unofficial)
