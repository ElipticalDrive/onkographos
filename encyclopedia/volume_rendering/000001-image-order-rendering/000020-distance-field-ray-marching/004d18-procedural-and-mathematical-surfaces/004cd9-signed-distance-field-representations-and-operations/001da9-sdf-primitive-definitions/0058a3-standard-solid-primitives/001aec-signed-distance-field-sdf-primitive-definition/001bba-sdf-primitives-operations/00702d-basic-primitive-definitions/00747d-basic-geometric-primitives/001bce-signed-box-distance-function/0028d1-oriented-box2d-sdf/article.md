# Oriented Box2D SDF
Also called: Oriented Box SDF, Rotated Box2D SDF

## Overview
Computes the signed Euclidean distance from a 2D point to a line segment of arbitrary length and thickness, effectively representing a 2D box with rounded corners oriented along the segment axis. It differs from the [Unsigned Box Distance Function](3750-unsigned-box-distance-function.md) by providing signed output and from the [Signed Box Distance Function](1092-signed-box-distance-function.md) by operating in 2D with a specific segment-based orientation rather than a general 3D axis-aligned or oriented box. It serves as a primitive for constructing 2D shapes in ray marching or implicit modeling.

## How it works
**Inputs:**
- Point $p \in \mathbb{R}^2$.
- Segment endpoints $a, b \in \mathbb{R}^2$.
- Full thickness $th \in \mathbb{R}$ (width of the box perpendicular to the segment).

**Outputs:**
- Signed distance $d \in \mathbb{R}$.

**Data Structures:**
- Vectors are 2D floats. No complex packing required.

**Steps:**
1. Calculate segment length $l = \|b - a\|$.
2. **Edge Case:** If $l \le 10^{-6}$, the segment is degenerate. Return the distance to a circle centered at $a$ with radius $th \cdot 0.5$.
   $$d = \text{circle}(p - a, th \cdot 0.5)$$
   Where $\text{circle}(v, r) = \|v\| - r$.
3. Calculate unit direction vector $d_{vec} = (b - a) / l$.
4. Calculate center of segment $c = (a + b) \cdot 0.5$.
5. Calculate vector from center to point $q = p - c$.
6. Rotate $q$ into the local frame aligned with $d_{vec}$. The local x-axis aligns with the segment, and the y-axis is perpendicular.
   $$local.x = d_{vec}.x \cdot q.x + d_{vec}.y \cdot q.y$$
   $$local.y = -d_{vec}.y \cdot q.x + d_{vec}.y \cdot q.y$$
   *Note: The source [S1, unofficial] specifies `local.y = -d.y * q.x + d.x * q.y`. This is a 90-degree rotation.*
7. Calculate half-extents $h = (l \cdot 0.5, th \cdot 0.5)$.
8. Compute the 2D box distance using the `box2d` function:
   $$d = \text{box2d}(local, h)$$
   Where $\text{box2d}(v, h) = \|\max(v, 0)\| + \min(\max(v_x, v_y), 0)$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def oriented_box2d_sdf(p, a, b, th):
    # Step 1: Segment length
    diff = b - a
    l = length(diff)
    
    # Step 2: Degenerate case
    if l <= 1e-6:
        return length(p - a) - (th * 0.5)
    
    # Step 3: Direction
    d_vec = diff / l
    
    # Step 4 & 5: Center and relative vector
    center = (a + b) * 0.5
    q = p - center
    
    # Step 6: Rotate into local frame
    # local.x is projection onto segment direction
    # local.y is projection onto perpendicular direction
    local_x = d_vec.x * q.x + d_vec.y * q.y
    local_y = -d_vec.y * q.x + d_vec.x * q.y
    
    # Step 7: Half extents
    h_x = l * 0.5
    h_y = th * 0.5
    
    # Step 8: Box distance
    # max(v, 0) applies max component-wise
    dx = max(local_x, 0.0)
    dy = max(local_y, 0.0)
    
    dist = length([dx, dy])
    
    # min(max(vx, vy), 0)
    min_component = min(max(local_x, local_y), 0.0)
    
    return dist + min_component
```

**Parameters:**
- `th`: Typical values depend on scene scale. Must be positive.
- `1e-6`: Threshold for degenerate segment detection [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Degenerate Segment:** If $a \approx b$, the direction vector is undefined. The method falls back to a circle [S1, unofficial].
- **Precision:** Division by small $l$ causes instability. The threshold $10^{-6}$ mitigates this [S1, unofficial].
- **Thickness:** If $th$ is zero, the shape becomes a line segment. The distance function still works but represents a 1D object in 2D space.

## Subtopics

### Deformed Box Signed Distance Function
A variant representing a glass panel with variable thickness and surface perturbations. It constructs an SDF by deforming an axis-aligned box SDF using elevation profiles $f$ and $g$ for the front and back faces [S2]. The thickness $t(\mathbf{x})$ is defined as $t + f(\mathbf{x}_x, \mathbf{x}_y)$ for $\mathbf{x}_z > 0$ and $t + g(\mathbf{x}_x, \mathbf{x}_y)$ otherwise, where $f$ and $g$ are differentiable functions (e.g., procedural noise) [S2]. This allows modeling hand-blown glass with bubbles and surface irregularities [S2]. However, replacing $t$ with $t(\mathbf{x})$ in the standard box SDF definition creates a discontinuity around $x_z = 0$ and inaccurate distance evaluation near the center plane [S2].

## Operating characteristics
- **Time Complexity:** $O(1)$. Constant number of arithmetic operations.
- **Memory Complexity:** $O(1)$. No auxiliary data structures.
- **Strengths:** Efficient primitive for 2D ray marching; handles arbitrary orientation and thickness; robust to degenerate inputs.
- **Limitations:** Only applicable in 2D; assumes uniform thickness along the segment; not suitable for complex 3D box orientations directly (requires 3D extension).
- **When to use:** When rendering 2D shapes with rounded corners defined by a central spine or segment.
- **When not to use:** For 3D boxes or non-uniform thickness profiles.

## Background
The Oriented Box2D SDF is derived from computational geometry libraries for signed distance functions, specifically documented in the `sdf-math-rs` repository [S1, unofficial]. It is used in procedural generation and ray marching of 2D implicit surfaces. The underlying box distance formula $\|\max(\mathbf{q}, 0)\| + \min(\max(\mathbf{q}_x, \mathbf{q}_y), 0)$ is a standard definition for 2D box SDFs, often attributed to Inigo Quilez [S2][S1, unofficial]. The specific handling of degenerate segments as circles is a practical implementation detail from [S1, unofficial].

## Sources
[S1] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
[S2] Parabolic Sphere Tracing of Signed Distance Fields for Old Glass Modelling and Rendering, https://doi.org/10.1109/tvcg.2025.3613853, fetched 2026-11-05
