# Line Segment Distance

Also called: Point-to-Line Distance Calculation, Segment SDF, sdSegment

## Overview
Computes the Euclidean distance from a query point `p` to a line segment defined by endpoints `l0` and `l1`. It differs from infinite line distance by clamping the projection parameter to the segment bounds, ensuring the result is zero if the point lies on the segment. This method serves as the geometric core for signed distance field (SDF) primitives in ray marching.

## How it works
**Inputs:**
- `p`: Query point (vec3).
- `l0`: Segment start point (vec3).
- `l1`: Segment end point (vec3).

**Outputs:**
- `dist`: Euclidean distance (float).

**Data Structures:**
- None required; scalar/vector operations only.

**Steps:**
1. Compute vector `v = l1 - l0`.
2. Compute vector `w = p - l0`.
3. Compute dot product `c1 = dot(v, w)`.
4. If `c1 <= 0.0`, the closest point is `l0`. Return `length(p - l0)`.
5. Compute dot product `c2 = dot(v, v)`.
6. If `c2 <= c1`, the closest point is `l1`. Return `length(p - l1)`.
7. Compute projection parameter `b = c1 / c2`.
8. Compute projection point `pb = l0 + b * v`.
9. Return `length(p - pb)`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def line_segment_distance(p, l0, l1):
    v = l1 - l0
    w = p - l0
    c1 = dot(v, w)
    if c1 <= 0.0:
        return length(p - l0)
    c2 = dot(v, v)
    if c2 <= c1:
        return length(p - l1)
    b = c1 / c2
    pb = l0 + b * v
    return length(p - pb)
```

**Parameters:**
- None.

**Edge Cases:**
- **Degenerate Segment:** If `l0 == l1`, `c2` is 0. Division by zero occurs in step 7. [G] Guard against this by checking `c2 < epsilon` and returning `length(p - l0)` if true.
- **Precision:** Near-boundary cases may suffer from floating-point inaccuracies. [G] Use a small epsilon for comparisons if high precision is required.

## Subtopics

### [Point-to-Line Distance](5916-point-to-line-distance.md)
Computes the vector from a point to the closest point on an infinite line, or the squared Euclidean distance to that line. It serves as the geometric core for [Line Segment Distance](3058-line-segment-distance.md), which clamps this result to segment bounds. Unlike segment distance, this method assumes an infinite extent, making it suitable for directional constraints or unbounded ray approximations.

## Operating characteristics
- **Time Complexity:** O(1). Fixed number of arithmetic operations.
- **Memory Complexity:** O(1). No auxiliary data structures.
- **Strengths:** Fast, numerically stable for non-degenerate segments, easy to implement.
- **Limitations:** Degenerate segments (zero length) cause division by zero. Not suitable for infinite lines.
- **When to use:** SDF primitives for ray marching, collision detection with thin objects.
- **When not to use:** Infinite line queries, or when segment length is zero.

## Background
Originated in computational geometry for distance queries. Popularized in graphics by Inigo Quilez's SDF primitives [S1, unofficial].

## Sources
[S1] Amory0709/geresim-spike, https://github.com/Amory0709/geresim-spike, fetched 2026-10-27 (unofficial)
