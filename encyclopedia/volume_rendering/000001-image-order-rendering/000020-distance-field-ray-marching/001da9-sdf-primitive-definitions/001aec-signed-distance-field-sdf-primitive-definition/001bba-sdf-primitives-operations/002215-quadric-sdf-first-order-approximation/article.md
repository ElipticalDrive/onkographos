# Quadric SDF (First-Order Approximation)
## Overview
Computes an approximate signed distance from a query point to a quadric surface defined by a general quadratic equation, using a first-order Taylor approximation [S1, unofficial]. It enables ray marching against arbitrary quadrics without deriving closed-form distance functions, trading geometric accuracy for implementation simplicity [S1, unofficial].
## How it works
**Inputs:**
- Query point $v = (x, y, z)$.
- Quadric coefficients $a, b, c, d, e, f, g, h, i, j$ defining the surface $ax^2 + by^2 + cz^2 + dxy + eyz + fzx + gx + hy + iz + j = 0$ [S1, unofficial].

**Outputs:**
- Approximate signed distance $d_{approx}$ [S1, unofficial].

**Data Structures:**
- Coefficients are stored as six quadratic terms ($a,b,c,d,e,f$), three linear terms ($g,h,i$), and one constant ($j$) [S1, unofficial]. Standard double-precision floats are used [S1, unofficial].

**Steps:**
1. Evaluate the implicit function value $f_0$ at $v$:
   $$f_0 = ax^2 + by^2 + cz^2 + dxy + eyz + fzx + gx + hy + iz + j$$
2. Compute the gradient components $\nabla f = (dx, dy, dz)$:
   $$dx = 2ax + dy + fz + g$$
   $$dy = 2by + ez + dx + h$$
   $$dz = 2cz + fx + ey + i$$
3. Compute the gradient magnitude $f_1$:
   $$f_1 = \sqrt{dx^2 + dy^2 + dz^2}$$
4. Return the approximation:
   $$d_{approx} = \frac{f_0}{f_1}$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def quadric_sdf_first_order(x, y, z, a, b, c, d, e, f, g, h, i, j):
    # 1. Evaluate implicit function
    f0 = (a * x * x + b * y * y + c * z * z +
          d * x * y + e * y * z + f * z * x +
          g * x + h * y + i * z + j)
    
    # 2. Compute gradient components
    dx = 2 * a * x + d * y + f * z + g
    dy = 2 * b * y + e * z + d * x + h
    dz = 2 * c * z + f * x + e * y + i
    
    # 3. Compute gradient magnitude
    f1 = math.sqrt(dx * dx + dy * dy + dz * dz)
    
    # 4. Return approximation
    if f1 == 0.0:
        return 0.0  # Guard against division by zero [G]
    return f0 / f1
```

**Parameters:**
- $a, b, c, d, e, f, g, h, i, j$: Coefficients of the quadric equation [S1, unofficial]. Typical values are not given in the sources.

**Edge Cases and Pitfalls:**
- **Zero Gradient:** If the gradient magnitude $f_1$ is near zero (e.g., at critical points or far from the surface), the distance estimate becomes unstable or incorrect [S1, unofficial]. A guard against division by zero is required [G].
- **Approximation Error:** The first-order approximation is only accurate near the surface [G]. Far from the surface, the error grows, potentially causing ray marching to overshoot or undershoot [G].
- **Sign Ambiguity:** The sign of $f_0$ determines the side of the surface, but the approximation may flip sign incorrectly if the gradient is small [G].

## Subtopics

### [Principal Axis Transformation for Quadratic Surfaces](4071-principal-axis-transformation-for-quadratic-surfaces.md)
Transforms the quadric coefficients to align with principal axes, simplifying the distance calculation. Choose when the quadric is rotated or sheared.

## Operating characteristics
- **Time Complexity:** $O(1)$ per query [G].
- **Memory Complexity:** $O(1)$ for coefficients [G].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear with the number of query points [G].
- **Strengths:** Simple to implement; works for any quadric defined by a general equation [G].
- **Limitations:** Approximate distance; unstable near critical points; less accurate than exact SDFs [G].
- **When to use:** When exact SDFs are unavailable or too complex to derive; for simple quadrics where high precision is not critical [G].
- **When not to use:** When high precision is required; near critical points of the quadric [G].
## Background
Originated from the use of first-order Taylor approximations for implicit surface distance estimation [S1, unofficial]. It is used in ray marching for rendering quadric surfaces [S1, unofficial].
## Sources
[S1] Leesan-haruki/FunctionRenderer, https://github.com/Leesan-haruki/FunctionRenderer, fetched 2026-10-26 (unofficial)
