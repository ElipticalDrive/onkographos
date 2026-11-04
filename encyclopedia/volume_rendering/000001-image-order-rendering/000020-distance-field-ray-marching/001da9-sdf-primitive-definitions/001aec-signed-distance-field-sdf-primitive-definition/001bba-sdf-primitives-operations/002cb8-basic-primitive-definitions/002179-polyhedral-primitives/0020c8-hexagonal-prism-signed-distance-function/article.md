# Hexagonal Prism Signed Distance Function

Also called: Hexagonal Prism SDF, Signed Hexagonal Prism, sdHexPrism

## Overview
Computes the signed Euclidean distance from a point to a hexagonal prism aligned with the Z-axis, defined by parameters $h_x$ (hexagon size) and $h_y$ (half-height) [S1, S2, unofficial]. It differs from the triangular prism by utilizing six-fold rotational symmetry to fold the query point into a fundamental sector, reducing the distance calculation to a simpler 2D case [S1, unofficial]. [S2, unofficial] provides a direct max-based formulation for a flat-topped orientation without reflection.

## How it works
**Inputs:**
- `p`: Query point $(p_x, p_y, p_z)$ [S1, S2, unofficial].
- `h`: Parameter vector $(h_x, h_y)$, where $h_x$ is the hexagon's characteristic size (radius or apothem depending on convention) and $h_y$ is the half-height of the prism [S1, S2, unofficial].

**Outputs:**
- `d`: Signed distance. Positive outside, negative inside [S1, S2, unofficial].

**Data Structures:**
- No persistent data structures. The function is stateless.

**Steps:**
1. **Absolute Value:** Compute $q = |p|$, i.e., $q_x = |p_x|, q_y = |p_y|, q_z = |p_z|$ [S1, S2, unofficial]. This exploits octant symmetry.
2. **Reflection (Fundamental Domain):** Define constant vector $k = (-\cos(\pi/6), \sin(\pi/6), \tan(\pi/6)) \approx (-0.8660254, 0.5, 0.57735)$ [S1, unofficial]. Compute $dot = k_x q_x + k_y q_y$ [S1, unofficial]. If $dot < 0$, reflect: $q_x \leftarrow q_x - 2 \cdot dot \cdot k_x$ and $q_y \leftarrow q_y - 2 \cdot dot \cdot k_y$ [S1, unofficial]. This folds the hexagon into a 30° fundamental sector.
3. **2D Hexagon Distance:** Calculate the distance to the hexagonal cross-section in the XY plane. Using the reflected point, compute the distance to the top edge segment: $d_{xy} = \text{length}(\max(q_x - \text{clamp}(q_x, -k_z h_x, k_z h_x), 0), \max(q_y - h_x, 0)) \cdot \text{sign}(q_y - h_x)$ [S1, unofficial].
4. **3D Prism Distance:** Combine with the Z-axis distance $d_z = q_z - h_y$ [S1, S2, unofficial]. Return $d = \min(\max(d_{xy}, d_z), 0) + \text{length}(\max(d_{xy}, d_z, 0))$ [S1, unofficial]. Alternatively, for a flat-topped orientation, [S2, unofficial] uses $d = \max(d_z, \max(q_x \cdot \frac{\sqrt{3}}{2} + q_y \cdot \frac{1}{2}, q_y) - h_x)$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sd_hex_prism(p, h):
    """
    Computes signed distance to a hexagonal prism aligned with the Z-axis.
    p: (3,) tuple or list, query point (px, py, pz).
    h: (2,) tuple or list, [hx, hy]. hx is hexagon size, hy is half-height.
    """
    px, py, pz = p
    hx, hy = h
    
    # 1. Absolute value for octant symmetry
    qx, qy, qz = abs(px), abs(py), abs(pz)
    
    # 2. Reflection into fundamental domain (30-degree sector)
    kx, ky, kz = -math.cos(math.pi / 6.0), math.sin(math.pi / 6.0), math.tan(math.pi / 6.0)
    dot_val = kx * qx + ky * qy
    if dot_val < 0:
        qx = qx - 2.0 * dot_val * kx
        qy = qy - 2.0 * dot_val * ky
        
    # 3. Distance to hexagonal cross-section (line segment distance)
    clamp_x = max(-kz * hx, min(qx, kz * hx))
    d_xy = math.sqrt(max(qx - clamp_x, 0.0)**2 + max(qy - hx, 0.0)**2)
    if qy > hx:
        d_xy = d_xy
    elif qy < hx:
        d_xy = -d_xy
        
    # 4. Distance to top/bottom caps
    d_z = qz - hy
    
    # 5. Combine using extruded shape SDF pattern
    return min(max(d_xy, d_z), 0.0) + math.sqrt(max(d_xy, 0.0)**2 + max(d_z, 0.0)**2)
```
*Note: [S2, unofficial] provides a simpler alternative for flat-topped orientation: `d = max(qz - hy, max(qx * 0.866025 + qy * 0.5, qy) - hx)` [S2, unofficial].*

**Edge Cases and Pitfalls:**
- Precision: Near the edges or vertices, floating-point rounding may cause slight artifacts. The reflection step avoids branch divergence in GPU implementations by using `min(dot, 0.0)` [S1, unofficial].
- Boundaries: The formula assumes the prism is centered at the origin. Off-center prisms require translating `p` by the center offset before computation [G].
- Degenerate Input: If $h_x = 0$ or $h_y = 0$, the shape degenerates to a line or disk; the formula still computes a valid distance to the degenerate geometry [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query. Involves basic arithmetic operations, `abs`, `sqrt`, `max`, `min`, and trigonometric constants.
- **Memory Complexity:** $O(1)$. No auxiliary storage or persistent state.
- **Strengths:** Exact signed distance, enabling smooth collision detection, ray marching, and CSG operations. The reflection method [S1, unofficial] is robust and orientation-agnostic for the folding step.
- **Limitations:** Orientation-dependent. The formula assumes a specific alignment; rotating the prism requires rotating the query point or adjusting the constant vector $k$. [S2, unofficial] is orientation-specific (flat-topped) but computationally cheaper.

## Background
The Hexagonal Prism SDF is part of the family of polyhedral primitives used in ray marching and constructive solid geometry (CSG) [S1, S2, unofficial]. [S1, unofficial] introduces the reflection/folding technique to exploit hexagonal symmetry, reducing the distance calculation to a fundamental sector. [S2, unofficial] provides a direct max-based formulation optimized for real-time GLSL shaders. Both are widely used in procedural modeling, real-time graphics, and collision detection.

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] marklundin/glsl-sdf-primitives, https://github.com/marklundin/glsl-sdf-primitives, fetched 2026-10-26 (unofficial)
