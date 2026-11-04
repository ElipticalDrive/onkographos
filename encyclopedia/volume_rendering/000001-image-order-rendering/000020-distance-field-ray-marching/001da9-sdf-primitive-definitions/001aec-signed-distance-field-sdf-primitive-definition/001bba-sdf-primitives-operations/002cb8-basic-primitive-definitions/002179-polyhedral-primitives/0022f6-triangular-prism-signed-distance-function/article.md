# Triangular Prism Signed Distance Function

Also called: Signed Triangle Prism, Triangular Prism SDF, sdTriPrism

## Overview
Computes the signed distance from a 3D point to an equilateral triangular prism aligned with the Z-axis [S2, unofficial]. It exploits the symmetry of the equilateral base to provide a closed-form, branch-free solution using absolute values and max operations, avoiding iterative projection or complex geometric decomposition [G].

## How it works
**Inputs:**
- `p`: A 3D point $(p_x, p_y, p_z)$ stored as a 3-element floating-point array or tuple.
- `h`: A 2D parameter vector $(h_x, h_y)$ stored as a 2-element floating-point array or tuple.
  - $h_x$: Side-related parameter for the equilateral triangle base.
  - $h_y$: Half-height of the prism along the Z-axis.
**Outputs:**
- `d`: A single floating-point value representing the signed distance. Positive if outside, negative if inside.
**Data Structures:**
- None required beyond standard floating-point vectors [G].
**Steps:**
1. Compute the component-wise absolute value of the input point to exploit symmetry: $q = |p|$, yielding $q_x = |p_x|$, $q_y = |p_y|$, $q_z = |p_z|$ [S2, unofficial].
2. Calculate the distance to the triangular cross-section in the XY plane using the rotated coordinates [S2, unofficial].
3. Compute the Z-distance to the top/bottom faces: $d_z = q_z - h_y$ [S2, unofficial].
4. Compute the XY-distance to the triangle edges: $d_{xy} = \max(q_x \cdot 0.866025 + q_y \cdot 0.5, -q_y) - h_x \cdot 0.5$ [S2, unofficial].
5. Combine the distances: $d = \max(d_z, d_{xy})$ [S2, unofficial].
**Formulas:**
$$ q = |p| $$
$$ d = \max(q_z - h_y, \max(q_x \cdot 0.866025 + q_y \cdot 0.5, -q_y) - h_x \cdot 0.5) $$
where $0.866025 \approx \frac{\sqrt{3}}{2}$ [S2, unofficial].
**Parameters:**
- Typical values: Not given in the sources [G].
- Constants: $0.866025 \approx \frac{\sqrt{3}}{2}$, $0.5 = \frac{1}{2}$ [S2, unofficial].
**Edge Cases and Pitfalls:**
- Precision: The constant $0.866025$ is a floating-point approximation; for higher precision, use $\cos(\pi/6)$ or $\frac{\sqrt{3}}{2}$ [G].
- Boundaries: At $h_x = 0$ or $h_y = 0$, the shape degenerates to a triangle or line segment; the formula remains valid but may require epsilon handling for robust ray marching [G].
- Sign convention: The formula assumes the prism is centered at the origin with its axis aligned to Z [S2, unofficial].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdTriPrism(p, h):
    q_x = abs(p[0])
    q_y = abs(p[1])
    q_z = abs(p[2])
    d_z = q_z - h[1]
    d_xy = max(q_x * 0.866025 + q_y * 0.5, -q_y) - h[0] * 0.5
    return max(d_z, d_xy)
```

## Operating characteristics
- Time Complexity: $O(1)$. Constant number of arithmetic operations [G].
- Memory Complexity: $O(1)$. No auxiliary data structures [G].
- Strengths: Extremely fast evaluation, suitable for real-time ray marching. No branching (using `max` and `abs`) [G].
- Limitations: Only supports equilateral triangles. Orientation is fixed (aligned with Z, specific Y-axis asymmetry per source) [S2, unofficial].

## Background
The function is a standard primitive in Signed Distance Field (SDF) ray marching, popularized by shader coding communities [G]. It appears in collections like `glsl-sdf-primitives` [S2, unofficial] and is used in procedural geometry generation [G].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] marklundin/glsl-sdf-primitives, https://github.com/marklundin/glsl-sdf-primitives, fetched 2026-10-26 (unofficial)
