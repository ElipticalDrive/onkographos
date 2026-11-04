# Rectangle Signed Distance Function

Also called: 2D Rectangle SDF, sdRect

## Overview
Computes the exact Euclidean distance from a query point to the boundary of an axis-aligned rectangle centered at the origin [S1, unofficial]. It is the 2D analog of the [Signed Box Distance Function](1092-signed-box-distance-function.md) [S1, unofficial].

## How it works
**Inputs:**
- $p$: Query point in 2D space [S1, unofficial].
- $hsize$: Half-extents of the rectangle along the x and y axes [S1, unofficial].

**Outputs:**
- $d$: Signed distance [S1, unofficial].

**Data Structures:**
- Tuples or lists of two floating-point numbers for $p$ and $hsize$.

**Steps:**
1. Calculate the displacement vector $dv$ from the rectangle's boundary:
   $$dv_x = |p_x| - hsize_x, \quad dv_y = |p_y| - hsize_y$$
   where $|p_x|$ and $|p_y|$ apply the absolute value component-wise [S1, unofficial].
2. Calculate the exterior distance using the positive components of $dv$:
   $$outsideDistance = \sqrt{\max(dv_x, 0)^2 + \max(dv_y, 0)^2}$$
   where $\max(dv, 0)$ applies the maximum operator to each component with $0$, and $\|\cdot\|$ is the Euclidean length [S1, unofficial].
3. Calculate the interior distance using the closest edge:
   $$insideDistance = \min(\max(dv_x, dv_y), 0)$$
   This captures the distance to the closest edge when the point is inside the rectangle [S1, unofficial].
4. Return the sum:
   $$d = outsideDistance + insideDistance$$

**Formula:**
$$d = \|\max(dv, 0)\| + \min(\max(dv_x, dv_y), 0)$$
where $dv = |p| - hsize$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdRect(p, hsize):
    dv_x = abs(p[0]) - hsize[0]
    dv_y = abs(p[1]) - hsize[1]
    pos_x = dv_x if dv_x > 0.0 else 0.0
    pos_y = dv_y if dv_y > 0.0 else 0.0
    outside_dist = (pos_x**2 + pos_y**2)**0.5
    inside_dist = min(max(dv_x, dv_y), 0.0)
    return outside_dist + inside_dist
```

**Parameters:**
- $hsize$: Typical values depend on scene scale. Not given in the sources.

**Edge Cases:**
- **Precision:** Near the boundary, floating-point precision may cause jitter. [G] Clamping or epsilon checks are often used in ray marching loops to prevent infinite loops or z-fighting.
- **Degenerate Input:** If $hsize$ contains zero or negative values, the geometry is invalid or degenerate. Not given in the sources.

## Subtopics

### Cube Signed Distance Function
A 3D extension for a cube centered at the origin with width, height, and length equal to 2.0 (half-extents of 1.0) [S2, unofficial].
**Definition:**
$$d = \|\max(d, 0)\| + \min(\max(d_x, \max(d_y, d_z)), 0)$$
where $d = |p| - (1.0, 1.0, 1.0)$ [S2, unofficial].
**Code:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubeSDF(p):
    d_x = abs(p[0]) - 1.0
    d_y = abs(p[1]) - 1.0
    d_z = abs(p[2]) - 1.0
    pos_x = d_x if d_x > 0.0 else 0.0
    pos_y = d_y if d_y > 0.0 else 0.0
    pos_z = d_z if d_z > 0.0 else 0.0
    outside_dist = (pos_x**2 + pos_y**2 + pos_z**2)**0.5
    inside_dist = min(max(d_x, max(d_y, d_z)), 0.0)
    return outside_dist + inside_dist
```
[S2, unofficial]

## Operating characteristics
Not given in the sources.

## Background
The implementation originates from community-driven ray marching repositories [S1, unofficial][S2, unofficial].

## Sources
[S1] erbenpeter/sdf, https://github.com/erbenpeter/sdf, fetched 2026-10-26 (unofficial)
[S2] markpmlim/RaymarchingAndSDF, https://github.com/markpmlim/RaymarchingAndSDF, fetched 2026-10-27 (unofficial)
