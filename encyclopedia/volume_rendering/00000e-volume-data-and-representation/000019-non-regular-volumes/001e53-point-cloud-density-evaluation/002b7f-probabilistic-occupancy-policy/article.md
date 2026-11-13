# Probabilistic Occupancy Policy
Also called: Intensity-based Occupancy Thresholding, Signed Intensity Occupancy Policy

## Overview
The Probabilistic Occupancy Policy determines the occupancy status of a point cloud point based on its intensity field value, mapping values near 0.0 to free space and values near 1.0 to occupied space [S1, unofficial]. It distinguishes itself from the Signed Intensity Occupancy Policy by using a symmetric tolerance range around both 0 and 1 rather than a single threshold near zero [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A point $p$ from a point cloud, which may contain an 'intensity' field.
- Output: An `OccupancyStatus` enum value: `FREE`, `OCCUPIED`, or `UNKNOWN`.

**Data Structures:**
The method uses a simple enumeration for status:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
enum OccupancyStatus { FREE = 0, OCCUPIED = 1, UNKNOWN = 2 };
```
The point $p$ is assumed to have a method or field access `p.intensity` returning a `float`; if the field does not exist, the method returns `UNKNOWN` [S1, unofficial].

**Numbered Steps:**
1. Check if the point $p$ has an 'intensity' field. If not, return `UNKNOWN` [S1, unofficial].
2. Let $v$ be the value of the intensity field.
3. If $|v| < \text{TOLLERANCE}$, return `FREE` [S1, unofficial].
4. If $v > (1.0 - \text{TOLLERANCE})$, return `OCCUPIED` [S1, unofficial].
5. Otherwise, return `UNKNOWN` [S1, unofficial].

**Formulas:**
- Free condition: $|v| < \text{TOLLERANCE}$ [S1, unofficial].
- Occupied condition: $v > 1.0 - \text{TOLLERANCE}$ [S1, unofficial].
- Where $v$ is the intensity value and $\text{TOLLERANCE}$ is the tolerance constant [S1, unofficial].

**Parameters:**
- `TOLLERANCE`: A static constant float with value `0.2`, defining the tolerance range for determining if an intensity value is close to 0 or 1 [S1, unofficial]. The source code uses the spelling `TOLLERANCE` (double L) [S1, unofficial].

**Edge Cases and Pitfalls:**
- The implementation is deterministic thresholding despite the probabilistic name [S1, unofficial].
- Values in $[0.2, 0.8]$ are considered unknown [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class OccupancyStatus:
    FREE = 0
    OCCUPIED = 1
    UNKNOWN = 2

def get_occupancy_status(p):
    if not hasattr(p, 'intensity'):
        return OccupancyStatus.UNKNOWN
    v = p.intensity
    TOLLERANCE = 0.2
    if abs(v) < TOLLERANCE:
        return OccupancyStatus.FREE
    if v > (1.0 - TOLLERANCE):
        return OccupancyStatus.OCCUPIED
    return OccupancyStatus.UNKNOWN
```

## Subtopics

### [Signed Intensity Occupancy Policy](52-signed-intensity-occupancy-policy.md)
Uses a single threshold near zero to distinguish free from occupied, ignoring the upper bound of 1. Choose when intensity values are expected to be negative for free space and positive for occupied space.

### [Neighbor Counting](2905-neighbor-counting.md)
Determines occupancy based on the density of neighboring points. Choose when intensity fields are unavailable or unreliable.

### [BrickDensityRegion Point Splatting](3956-brickdensityregion-point-splatting.md)
Uses a spatial brick structure for density evaluation. Choose for large-scale point clouds requiring efficient spatial queries.

### [Chamfer Distance](2446-chamfer-distance.md)
A metric for comparing point clouds, not an occupancy policy. Included as a sibling evaluation method.

### [Mesh/Point Cloud Evaluation Metrics](4797-mesh-point-cloud-evaluation-metrics.md)
General metrics for evaluating reconstruction quality.

## Operating characteristics
Time and memory complexity, measured performance, scaling, strengths, limitations, and usage guidelines are not given in the sources [S1, unofficial].

## Background
The Probabilistic Occupancy Policy is part of the `basic_next_best_view` package, developed by R. Monica [S1, unofficial]. It is used in robotics for next-best-view planning, where occupancy information from point clouds is needed to evaluate potential viewpoints [S1, unofficial]. The policy is one of several occupancy policies available in the package, alongside the Signed Intensity Occupancy Policy [S1, unofficial].

## Sources
[S1] RMonica/basic_next_best_view, https://github.com/RMonica/basic_next_best_view, fetched 2026-10-24 (unofficial)
