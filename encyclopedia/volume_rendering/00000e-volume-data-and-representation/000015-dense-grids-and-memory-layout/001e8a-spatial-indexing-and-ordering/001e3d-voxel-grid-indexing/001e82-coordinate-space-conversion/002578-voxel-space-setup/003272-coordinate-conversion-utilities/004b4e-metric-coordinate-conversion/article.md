# Metric Coordinate Conversion

Also called: World Metric Coordinate Construction

## Overview
Metric Coordinate Conversion decomposes a continuous world-space position into a discrete integer meter component and a continuous fractional meter component. This method solves the problem of separating large-scale spatial indexing (handled by whole meters) from high-precision local sampling (handled by fractional meters). It differs from [Sliced Volume Coordinate Mapping](8287-sliced-volume-coordinate-mapping.md) by operating on absolute world coordinates rather than slice-relative indices, and from [Voxel Index Computation](11238-voxel-index-computation.md) by preserving sub-meter precision rather than truncating to voxel indices.

## How it works
**Inputs and Outputs**
- Input: `localPosition` (float), `chunkCoordinate` (int64_t), `CHUNK_SIZE` (int64_t).
- Output: `WorldMetricCoordinate` struct.

**Data Structures**
The `WorldMetricCoordinate` struct contains:
- `wholeMeters`: `int64_t` representing the integer meter offset.
- `fractionalMeter`: `float` representing the sub-meter offset within the current meter.

**Steps**
1. Compute the whole meter component of the local position: `localWholeMeters = floor(localPosition)`.
2. Calculate the axis-specific whole meters: `axis.wholeMeters = chunkCoordinate * CHUNK_SIZE + localWholeMeters`.
3. Calculate the fractional meter: `axis.fractionalMeter = localPosition - localWholeMeters`.
4. Assert that `axis.fractionalMeter` is in the range $[0.0, 1.0)$.

**Formulas**
$$
\text{localWholeMeters} = \lfloor \text{localPosition} \rfloor
$$
$$
\text{wholeMeters} = \text{chunkCoordinate} \times \text{CHUNK\_SIZE} + \text{localWholeMeters}
$$
$$
\text{fractionalMeter} = \text{localPosition} - \text{localWholeMeters}
$$

**Parameters**
- `CHUNK_SIZE`: Not given in the sources.
- `chunkCoordinate`: Not given in the sources.

**Edge Cases and Pitfalls**
- **Fractional Precision Boundary**: The `fractionalMeter` must be strictly less than $1.0f$. This is handled by the `floor`-based decomposition and an assertion [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_metric_coordinate(local_position: float, chunk_coordinate: int64, chunk_size: int64) -> WorldMetricCoordinate:
    # Compute whole meters
    local_whole_meters = floor(local_position)
    
    # Calculate axis.wholeMeters
    whole_meters = chunk_coordinate * chunk_size + local_whole_meters
    
    # Calculate axis.fractionalMeter
    fractional_meter = local_position - local_whole_meters
    
    # Assert fractionalMeter is in [0.0f, 1.0f)
    assert 0.0 <= fractional_meter < 1.0
    
    return WorldMetricCoordinate(whole_meters=whole_meters, fractional_meter=fractional_meter)
```

## Subtopics

### [Metric Coordinate Offset](12966-metric-coordinate-offset.md)
Metric Coordinate Offset adjusts the world-space origin of a voxel grid by shifting all scene entities and grid data structures by a specified delta vector. It solves the problem of recentering a volume rendering scene or aligning the voxel grid with a new reference frame without altering the relative geometry of the scene. It differs from general [Metric Coordinate Conversion](12960-metric-coordinate-conversion.md) by specifically handling the translation of discrete world-space positions and associated spatial indices rather than continuous coordinate system transformations.

## Operating characteristics
Not given in the sources.

## Background
Origin: Not given in the sources.

## Sources
[S1] aaron-eugene/PlaneVoxelLab, https://github.com/aaron-eugene/PlaneVoxelLab, fetched 2026-10-27 (unofficial)
