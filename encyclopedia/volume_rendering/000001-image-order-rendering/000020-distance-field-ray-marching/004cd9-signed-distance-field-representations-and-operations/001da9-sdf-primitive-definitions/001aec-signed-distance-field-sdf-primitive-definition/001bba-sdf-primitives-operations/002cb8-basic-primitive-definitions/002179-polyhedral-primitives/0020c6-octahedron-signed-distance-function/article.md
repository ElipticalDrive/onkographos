# Octahedron Signed Distance Function

Also called: Octahedron SDF, sdOctahedron

## Overview
Computes the exact signed distance from a 3D point to the surface of an octahedron centered at the origin with size parameter $s$. It solves the problem of ray-marching primitives in volume rendering or ray tracing where analytical distance queries are required for convergence. It is distinguished from other polyhedral primitives by its reliance on coordinate permutation to handle the octahedron's symmetry, avoiding complex case logic for face normals.

## How it works
**Inputs:** Point $p = (p_x, p_y, p_z)$ in 3D space; size $s$ (distance from center to vertex along axes).
**Outputs:** Signed distance $d$. Negative inside, positive outside, zero on surface.

**Data Structures:**
- $p$: `vec3` (float3).
- $s$: `float`.
- $m$: `float`.
- $q$: `vec3` (temporary coordinate permutation).
- $k$: `float`.

**Steps:**
1.  Compute absolute coordinates: $p \leftarrow |p|$.
2.  Compute sum minus size: $m = p_x + p_y + p_z - s$.
3.  Determine the closest face region by comparing $3p_i$ against $m$:
    -   If $3p_x < m$, set $q = (p_x, p_y, p_z)$.
    -   Else if $3p_y < m$, set $q = (p_y, p_z, p_x)$.
    -   Else if $3p_z < m$, set $q = (p_z, p_x, p_y)$.
    -   Else (central region), return $m \cdot \tan(\pi/6)$.
4.  Calculate clamp value: $k = \text{clamp}(0.5 \cdot (q_z - q_y + s), 0, s)$.
5.  Compute distance vector components:
    -   $v_x = q_x$
    -   $v_y = q_y - s + k$
    -   $v_z = q_z - k$
6.  Return $d = \sqrt{v_x^2 + v_y^2 + v_z^2}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sd_octahedron(p, s):
    """
    Signed distance to an octahedron of size s.
    p: tuple or list of 3 floats (x, y, z)
    s: float, size parameter
    """
    # Step 1: Absolute coordinates
    px, py, pz = abs(p[0]), abs(p[1]), abs(p[2])
    
    # Step 2: Sum minus size
    m = px + py + pz - s
    
    # Step 3: Determine region
    if 3.0 * px < m:
        qx, qy, qz = px, py, pz
    elif 3.0 * py < m:
        qx, qy, qz = py, pz, px
    elif 3.0 * pz < m:
        qx, qy, qz = pz, px, py
    else:
        # Central region: distance to the center plane
        return m * math.tan(math.pi / 6.0)
    
    # Step 4: Clamp value
    k = max(0.0, min(s, 0.5 * (qz - qy + s)))
    
    # Step 5 & 6: Distance calculation
    vx = qx
    vy = qy - s + k
    vz = qz - k
    
    return math.sqrt(vx*vx + vy*vy + vz*vz)
```

**Parameters:**
- $s$: Typical values depend on scene scale. Must be $> 0$.

**Edge Cases:**
- $s=0$: Degenerate point. The formula yields distance to origin if handled, but standard implementation assumes $s>0$.
- Precision: Near the central region boundary ($3p_i \approx m$), floating point errors may cause slight discontinuities in derivatives, but the distance value remains continuous.

## Subtopics

### Equilateral Triangle Signed Distance Function
Computes the signed distance to an equilateral triangle with inscribed radius $r$ in 2D. It uses coordinate rotation and clamping to map the point into a fundamental region. Based on Inigo Quilez's exact function [S2, unofficial].
**Steps:**
1.  $k = \sqrt{3.0}$.
2.  Transform $p$: $p_x \leftarrow |p_x| - r$, $p_y \leftarrow p_y + r/k$.
3.  If $p_x + k \cdot p_y > 0.0$, rotate: $p \leftarrow ((p_x - k \cdot p_y)/2, (-k \cdot p_x - p_y)/2)$.
4.  $p_x \leftarrow p_x - \text{clamp}(p_x, -2r, 0)$.
5.  Return $-|p| \cdot \text{sign}(p_y)$.

## Operating characteristics
- **Time Complexity:** $O(1)$. Constant number of arithmetic operations and comparisons.
- **Memory Complexity:** $O(1)$. No auxiliary data structures.
- **Strengths:** Exact analytical solution; fast execution; no iterative approximation required.
- **Limitations:** Only applies to regular octahedra aligned with axes. Not suitable for deformed or irregular octahedra without modification.

## Background
The octahedron SDF is a standard primitive in the family of signed distance functions for ray marching, popularized by Inigo Quilez's online articles and widely adopted in graphics programming communities. It is part of the broader set of polyhedral primitives used in implicit modeling and volumetric rendering.

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
