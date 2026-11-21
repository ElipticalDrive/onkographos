# Metric Coordinate Conversion

Also called: World Metric Coordinate Construction

## Overview
Metric Coordinate Conversion decomposes a continuous world-space position into a discrete integer meter component and a continuous fractional meter component [S1, unofficial]. This method separates large-scale spatial indexing (handled by whole meters) from high-precision local sampling (handled by fractional meters) [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: `localPosition` (float), `chunkCoordinate` (int64_t), `CHUNK_SIZE` (int64_t) [S1, unofficial].
- Output: `WorldMetricCoordinate` struct [S1, unofficial].

**Data Structures**
The `WorldMetricCoordinate` struct contains:
- `wholeMeters`: `int64_t` representing the integer meter offset [S1, unofficial].
- `fractionalMeter`: `float` representing the sub-meter offset within the current meter [S1, unofficial].

**Steps**
1. Compute the whole meter component of the local position: `localWholeMeters = floor(localPosition)` [S1, unofficial].
2. Calculate the axis-specific whole meters: `axis.wholeMeters = chunkCoordinate * CHUNK_SIZE + localWholeMeters` [S1, unofficial].
3. Calculate the fractional meter: `axis.fractionalMeter = localPosition - localWholeMeters` [S1, unofficial].
4. Assert that `axis.fractionalMeter` is in the range $[0.0, 1.0)$ [S1, unofficial].

**Formulas**
$$ \text{localWholeMeters} = \lfloor \text{localPosition} \rfloor $$
$$ \text{wholeMeters} = \text{chunkCoordinate} \times \text{CHUNK\_SIZE} + \text{localWholeMeters} $$
$$ \text{fractionalMeter} = \text{localPosition} - \text{localWholeMeters} $$

**Parameters**
- `CHUNK_SIZE`: Not given in the sources [S1, unofficial].
- `chunkCoordinate`: Not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls**
- **Fractional Precision Boundary**: The `fractionalMeter` must be strictly less than $1.0f$ [S1, unofficial]. This boundary is enforced by the `floor`-based decomposition and an explicit assertion [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from dataclasses import dataclass
import math

@dataclass
class WorldMetricCoordinate:
    whole_meters: int
    fractional_meter: float

def compute_metric_coordinate(local_position: float, chunk_coordinate: int, chunk_size: int) -> WorldMetricCoordinate:
    # Step 1: Compute whole meters
    local_whole_meters = math.floor(local_position)
    
    # Step 2: Calculate axis-specific whole meters
    whole_meters = chunk_coordinate * chunk_size + local_whole_meters
    
    # Step 3: Calculate fractional meter
    fractional_meter = local_position - local_whole_meters
    
    # Step 4: Assert fractionalMeter is in [0.0, 1.0)
    assert 0.0 <= fractional_meter < 1.0
    
    return WorldMetricCoordinate(whole_meters=whole_meters, fractional_meter=fractional_meter)
```

## Subtopics

### [Metric Coordinate Offset](12966-metric-coordinate-offset.md)
Metric Coordinate Offset shifts the world-space origin of a voxel grid by applying a delta vector to all scene entities and grid data structures. It recenteres a volume rendering scene or aligns the voxel grid with a new reference frame without altering relative geometry. It differs from Metric Coordinate Conversion by translating discrete world-space positions and spatial indices rather than performing continuous coordinate system transformations.

## Operating characteristics
Not given in the sources [S1, unofficial].

## Background
Origin: Not given in the sources [S1, unofficial].

## Sources
[S1] aaron-eugene/PlaneVoxelLab, https://github.com/aaron-eugene/PlaneVoxelLab, fetched 2026-10-27 (unofficial)
