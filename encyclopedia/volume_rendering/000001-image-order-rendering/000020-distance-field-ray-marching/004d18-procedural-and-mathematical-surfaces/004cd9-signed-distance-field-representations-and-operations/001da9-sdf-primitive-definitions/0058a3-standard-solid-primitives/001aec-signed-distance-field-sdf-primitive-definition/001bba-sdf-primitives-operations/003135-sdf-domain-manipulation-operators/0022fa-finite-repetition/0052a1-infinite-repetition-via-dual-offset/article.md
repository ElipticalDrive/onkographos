# Infinite Repetition via Dual Offset

Also called: Hexagonal Honeycomb Repetition, Hexagonal Honeycomb Repetition via Dual Offset Repetition, OR-Composite Honeycomb, Two-Cell Honeycomb

## Overview
Infinite Repetition via Dual Offset creates a perfect, interlocking hexagonal honeycomb pattern by combining two separate rectangular repetition grids that are offset from each other [S1, unofficial]. Unlike the skew method, which applies a single coordinate transformation, this method uses a boolean OR operation (minimum of distances) on two distinct distance functions to ensure that the "interlocking" parts of adjacent hexagons are correctly rendered without clipping [S1, unofficial]. It is computationally more expensive than skew-based approaches but allows for different shape functions or properties (such as color) per column [S1, unofficial].

## How it works
**Inputs:**
- Point $p$ in 3D space (or 2D for the tiling plane) [S1, unofficial].
- Side length $s$ of the regular hexagon [S1, unofficial].

**Outputs:**
- Signed distance $d$ to the nearest hexagon boundary [S1, unofficial].

**Data Structures:**
- No complex structures required. The method relies on standard vector arithmetic and two calls to a rectangular repetition function [S1, unofficial].

**Steps:**
1.  **Define the two repetition patterns:** The honeycomb is formed by two interleaved rectangular grids [S1, unofficial]. Let the vertical repetition step be $h_y$ and the horizontal step be $h_x$ [S1, unofficial]. For a regular hexagon with side length $s$: the horizontal period is $h_x = 2s$ [G]. The vertical period is $h_y = 2\sqrt{3}s$ [S1, unofficial]. This is derived from the height of an equilateral triangle with side $s$ ($\frac{\sqrt{3}}{2}s$) multiplied by 4 [S1, unofficial]. The two grids are offset. Typically, one grid is aligned with the origin, and the other is shifted to ensure their cells interlock [S1, unofficial]. A common implementation places the centers of the two rectangular cells at Grid 1: $(0, 0)$ and Grid 2: $(s, \frac{\sqrt{3}}{2}s)$ or equivalent offset that ensures the hexagons from Grid 1 and Grid 2 touch at their edges to form the honeycomb [G].
2.  **Calculate the distance function for the first pattern ($d_1$):** Apply the rectangular repetition operator to point $p$ using periods $(h_x, h_y)$ centered at the origin [S1, unofficial]. Let $p_1$ be the wrapped point [G]. Calculate the signed distance from $p_1$ to the hexagon shape defined within this cell [S1, unofficial]. Let this be $d_1$.
3.  **Calculate the distance function for the second pattern ($d_2$):** Apply the rectangular repetition operator to point $p$ using the same periods $(h_x, h_y)$ but with the offset center for the second grid [S1, unofficial]. Let $p_2$ be the wrapped point relative to this second grid's origin [G]. Calculate the signed distance from $p_2$ to the hexagon shape [S1, unofficial]. Let this be $d_2$.
4.  **Combine using minimum operation:** The final distance is the minimum of the two distances:
    $$ d = \min(d_1, d_2) $$
    This corresponds to the boolean OR operation for signed distance fields, ensuring that if a point is inside or near a hexagon from either grid, the distance reflects the nearest boundary [S1, unofficial].

**Parameters:**
- $s$: Side length of the hexagon. Typical value: $1.0$ [S1, unofficial].
- $h_x = 2s$ [G].
- $h_y = 2\sqrt{3}s \approx 3.464s$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Clipping in Single-Grid Approaches:** If one attempts to create a honeycomb by simply shifting every other row by half the cell width (brick-like layout) using a single distance function per cell, the interlocking parts of tightly packed hexagons will extend outside the defined rectangular repetition cell [S1, unofficial]. Since a single distance function defines the shape within one cell, these protruding parts are clipped/cut off at the cell boundaries [S1, unofficial]. The Dual Offset method avoids this by ensuring the "missing" parts of one cell are covered by the adjacent cell from the other grid [S1, unofficial].
- **Precision:** Standard floating-point precision is sufficient for typical scene scales [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Vector2:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    def __sub__(self, other):
        return Vector2(self.x - other.x, self.y - other.y)

def repeat_rect(p, period_x, period_y):
    """Wraps point p into the fundamental domain of a rectangular grid."""
    x = p.x % period_x
    y = p.y % period_y
    if x > period_x / 2:
        x -= period_x
    if y > period_y / 2:
        y -= period_y
    return Vector2(x, y)

def hexagon_sdf(p, s):
    """Signed distance to a regular hexagon centered at origin with side length s [G]."""
    q = Vector2(abs(p.x), abs(p.y))
    if q.x > q.y:
        d = q.x * 0.5 + q.y * math.sqrt(3)/2 - s * math.sqrt(3)/2
    else:
        d = q.y - s
    return d

def infinite_repetition_dual_offset(p, side_length):
    """Computes the distance to an infinite honeycomb pattern using dual offset repetition."""
    s = side_length
    h_x = 2 * s
    h_y = 2 * math.sqrt(3) * s
    
    p1 = repeat_rect(p, h_x, h_y)
    d1 = hexagon_sdf(p1, s)
    
    offset = Vector2(s, math.sqrt(3)/2 * s)
    p2 = repeat_rect(p - offset, h_x, h_y)
    d2 = hexagon_sdf(p2, s)
    
    return min(d1, d2)
```

## Subtopics

### Hexagonal Honeycomb Repetition via Offset Rows
A variant that creates a honeycomb pattern by shifting every other row of a rectangular grid by half the cell width, creating a brick-like layout [S1, unofficial]. It is simpler to conceptualize but suffers from a critical pitfall: when using convex shapes like hexagons that are tightly packed, the 'interlocking' parts of the shape may extend outside the defined rectangular repetition cell [S1, unofficial]. Since a single distance function defines the shape within one cell, these protruding parts are clipped/cut off at the cell boundaries, resulting in incorrect geometry [S1, unofficial].

### Hexagonal Honeycomb Repetition via Skew Transformation
A variant that applies a 2D skew transformation matrix that shears the xy-plane by 30 degrees while maintaining unit vector lengths [S1, unofficial]. This transformation is applied to the input coordinates before passing them to the infinite repetition function [S1, unofficial]. A pitfall is that the primitive shape itself must be defined in the skewed coordinate space; for example, a right-angled triangle in the skewed space becomes an equilateral triangle in the final rendered space [S1, unofficial]. It is computationally lighter than the dual offset method because it requires only one transformation and one distance function evaluation, whereas the alternative requires two distance function evaluations and a boolean OR [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Requires calculating two distance functions and combining them with a minimum operation, making it approximately twice the cost of a single repetition pattern [S1, unofficial].
- **Memory Complexity:** Minimal additional memory; only requires storing the offset parameters [G].
- **Strengths:** Produces a perfect, interlocking honeycomb without clipping artifacts [S1, unofficial]. Allows different shape functions or properties (e.g., color, material) to be applied to each column/grid independently [S1, unofficial].
- **Limitations:** More computationally expensive than the skew method [S1, unofficial]. Requires careful tuning of the offset to ensure proper interlocking [G].
- **When to use:** When perfect interlocking is required and different properties per column are desired [S1, unofficial].
- **When not to use:** When performance is critical and the skew method suffices for the visual requirement [S1, unofficial].

## Background
Originated from practical shader art and procedural geometry techniques, specifically discussed in the context of Sphere Tracing and domain operations [S1, unofficial]. The method is highlighted as a solution to the clipping issues inherent in simple row-offsetting methods for hexagonal tiling [S1, unofficial]. It leverages the property that the union of two offset rectangular grids can form a hexagonal lattice [S1, unofficial].

## Sources
[S1] SphereTracing:領域操作関数]ハニカム構造, https://qiita.com/keim_at_si/items/856f995418192118bedc, fetched 2026-10-26 (unofficial)
