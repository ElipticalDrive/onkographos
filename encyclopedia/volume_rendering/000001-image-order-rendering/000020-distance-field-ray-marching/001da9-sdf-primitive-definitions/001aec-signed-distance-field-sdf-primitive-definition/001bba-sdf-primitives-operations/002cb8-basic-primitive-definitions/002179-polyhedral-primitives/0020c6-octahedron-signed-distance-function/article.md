# Octahedron Signed Distance Function
Also called: Octahedron SDF, sdOctahedron

## Overview
Computes the exact signed distance from a query point to the surface of a regular octahedron centered at the origin with a specified size parameter [S1, unofficial].

## How it works
**Inputs and outputs**:
- Input: Query point $p \in \mathbb{R}^3$, size parameter $s \in \mathbb{R}^+$ [S1, unofficial].
- Output: Signed distance $d \in \mathbb{R}$ [S1, unofficial].

**Data structures**:
- None required beyond standard 3D vectors [G].

**Steps**:
1. Compute the absolute value of the query point: $p' = |p|$. Let components be $p_x, p_y, p_z$ [S1, unofficial].
2. Calculate the sum minus size: $m = p_x + p_y + p_z - s$ [S1, unofficial].
3. Determine the closest face region by comparing $3p_i$ against $m$ [S1, unofficial]:
   - If $3p_x < m$, set $q = (p_x, p_y, p_z)$ [S1, unofficial].
   - Else if $3p_y < m$, set $q = (p_y, p_z, p_x)$ [S1, unofficial].
   - Else if $3p_z < m$, set $q = (p_z, p_x, p_y)$ [S1, unofficial].
   - Else, the point is in the central region; return $m \cdot \tan(\pi/6)$ [S1, unofficial].
4. For the selected $q = (q_x, q_y, q_z)$, calculate the clamp value: $k = \text{clamp}(0.5 \cdot (q_z - q_y + s), 0, s)$ [S1, unofficial].
5. Calculate the distance vector components: $v_x = q_x$, $v_y = q_y - s + k$, $v_z = q_z - k$ [S1, unofficial].
6. Return the Euclidean norm: $d = \sqrt{v_x^2 + v_y^2 + v_z^2}$ [S1, unofficial].

**Pseudocode**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sd_octahedron(p, s):
    """
    Computes the signed distance to an octahedron of size s.
    p: tuple or list of 3 floats (x, y, z)
    s: float, size parameter
    """
    # Step 1: Absolute value
    px, py, pz = abs(p[0]), abs(p[1]), abs(p[2])
    
    # Step 2: Sum minus size
    m = px + py + pz - s
    
    # Step 3: Determine region
    if 3 * px < m:
        qx, qy, qz = px, py, pz
    elif 3 * py < m:
        qx, qy, qz = py, pz, px
    elif 3 * pz < m:
        qx, qy, qz = pz, px, py
    else:
        # Central region
        return m * math.tan(math.pi / 6)
    
    # Step 4: Clamp calculation
    k = max(0.0, min(s, 0.5 * (qz - qy + s)))
    
    # Step 5: Distance vector
    vx = qx
    vy = qy - s + k
    vz = qz - k
    
    # Step 6: Norm
    return math.sqrt(vx*vx + vy*vy + vz*vz)
```

**Parameters**:
- $s$: The size parameter of the octahedron [S1, unofficial].

**Edge cases**:
- $s=0$: Returns distance to origin [G].
- Precision: The `clamp` and comparisons are robust, but floating point errors near the boundaries ($3p_i = m$) may cause slight discontinuities in derivatives, though the distance value remains continuous [G].

## Subtopics

### Equilateral Triangle Signed Distance Function
Computes the signed distance to an equilateral triangle with inscribed radius $r$ in 2D [S2, unofficial].
1. Calculate $k = \sqrt{3.0}$ [S2, unofficial].
2. Transform $p_x = |p_x| - r$, $p_y = p_y + r / k$ [S2, unofficial].
3. If $p_x + k \cdot p_y > 0.0$, set $p = ((p_x - k \cdot p_y) \cdot 0.5, (-k \cdot p_x - p_y) \cdot 0.5)$ [S2, unofficial].
4. Transform $p_x = p_x - \text{clamp}(p_x, -2.0 \cdot r, 0.0)$ [S2, unofficial].
5. Return $-|p| \cdot \text{sign}(p_y)$ [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
