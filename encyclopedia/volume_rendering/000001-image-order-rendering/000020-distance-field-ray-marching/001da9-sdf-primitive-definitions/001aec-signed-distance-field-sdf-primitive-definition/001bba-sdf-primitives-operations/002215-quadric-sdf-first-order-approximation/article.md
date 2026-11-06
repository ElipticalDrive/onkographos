# Quadric SDF (First-Order Approximation)
## Overview
Computes an approximate signed distance from a query point to a quadric surface defined by a general quadratic equation using a first-order Taylor approximation [S1, unofficial]. It enables ray marching of complex implicit surfaces without solving higher-order polynomials, distinguishing itself from exact SDF primitives by trading geometric precision for computational speed and algebraic simplicity [S1, unofficial].
## How it works
**Inputs:** Query point $v = (x, y, z)$; quadric coefficients $a, b, c, d, e, f, g, h, i, j$ defining the surface $ax^2 + by^2 + cz^2 + dxy + eyz + fzx + gx + hy + iz + j = 0$ [S1, unofficial].
**Outputs:** Approximate signed distance $d_{approx}$ and optionally the normalized gradient vector [S1, unofficial].

**Data Structures:**
- Coefficients: 10 floating-point values ($a$ through $j$).
- Point: 3 floating-point values ($x, y, z$).

**Steps:**
1. Evaluate the implicit function value $f_0$ at $v$:
   $$f_0 = ax^2 + by^2 + cz^2 + dxy + eyz + fzx + gx + hy + iz + j$$
2. Compute the gradient components $\nabla f = (dx, dy, dz)$:
   $$dx = \frac{\partial f}{\partial x} = 2ax + dy + fz + g$$
   $$dy = \frac{\partial f}{\partial y} = 2by + ez + dx + h$$
   $$dz = \frac{\partial f}{\partial z} = 2cz + fx + ey + i$$
   *Note: The source code uses `d` for the $xy$ coefficient and `dx` for the x-gradient component, which is a naming collision in the original implementation but mathematically distinct [S1, unofficial].*
3. Compute the gradient magnitude $f_1$:
   $$f_1 = \sqrt{dx^2 + dy^2 + dz^2}$$
4. Return the signed distance estimate:
   $$d_{approx} = \frac{f_0}{f_1}$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quadric_sdf_first_order(x, y, z, a, b, c, d, e, f, g, h, i, j):
    # 1. Evaluate implicit function f0
    f0 = (a * x * x + b * y * y + c * z * z +
          d * x * y + e * y * z + f * z * x +
          g * x + h * y + i * z + j)
    
    # 2. Compute gradient components
    # Note: 'd' is the xy coefficient, 'dx' is the x-gradient
    dx_grad = 2 * a * x + d * y + f * z + g
    dy_grad = 2 * b * y + e * z + d * x + h
    dz_grad = 2 * c * z + f * x + e * y + i
    
    # 3. Compute gradient magnitude
    f1 = math.sqrt(dx_grad**2 + dy_grad**2 + dz_grad**2)
    
    # 4. Return distance estimate
    if f1 == 0:
        return 0.0 # Guard against division by zero [G]
    return f0 / f1

def quadric_normal(x, y, z, a, b, c, d, e, f, g, h, i, j):
    dx_grad = 2 * a * x + d * y + f * z + g
    dy_grad = 2 * b * y + e * z + d * x + h
    dz_grad = 2 * c * z + f * x + e * y + i
    
    mag = math.sqrt(dx_grad**2 + dy_grad**2 + dz_grad**2)
    if mag == 0:
        return (0.0, 0.0, 0.0) # Undefined normal [G]
    return (dx_grad / mag, dy_grad / mag, dz_grad / mag)
```

**Parameters:**
- $a, b, c, d, e, f, g, h, i, j$: Coefficients of the general quadric equation [S1, unofficial].
- Typical values: Dependent on the specific quadric shape (e.g., sphere, ellipsoid, hyperboloid).

**Edge Cases and Pitfalls:**
- **Zero Gradient:** If the gradient magnitude $f_1$ is near zero (e.g., at critical points or far from the surface), the distance estimate becomes unstable or incorrect due to division by a small number [S1, unofficial]. A guard for $f_1=0$ is required [G].
- **Approximation Error:** The method is a first-order approximation. It is not exact for curved surfaces, potentially leading to jitter or incorrect convergence in ray marching if the step size is not adapted [S1, unofficial].
- **Sign Ambiguity:** The sign of the result depends on the gradient direction. Ensure consistency with the ray marching termination criteria (e.g., `abs(sdf) <= epsilon`) [S1, unofficial].

## Subtopics
## Operating characteristics
- **Time Complexity:** $O(1)$ per query. Requires 10 multiplications, 9 additions, 1 square root, and 1 division [S1, unofficial].
- **Memory Complexity:** $O(1)$ for coefficients and intermediate variables.
- **Performance:** Significantly faster than solving for exact roots of the distance function or using higher-order approximations [S1, unofficial].
- **Strengths:** Simple implementation; works for any quadric defined by the general equation; suitable for real-time rendering where exact distance is not critical [S1, unofficial].
- **Limitations:** Inaccurate far from the surface; unstable at critical points; not a true signed distance field (SDF) but an approximation [S1, unofficial].
- **When to use:** When rendering general quadric surfaces (ellipsoids, hyperboloids, paraboloids) in ray marching and exact SDFs are unavailable or too expensive [S1, unofficial].
- **When not to use:** When high precision is required (e.g., collision detection in physics engines) or when the surface is far from the query point [S1, unofficial].

## Background
Originated from the implementation in the `FunctionRenderer` project by Leesan-haruki, which implements John C. Hart's 1995 ray marching algorithm for SDFs [S1, unofficial]. The method is a practical adaptation of the gradient-based distance approximation for general quadrics, allowing them to be used in the same ray marching framework as basic primitives like spheres [S1, unofficial].

## Sources
[S1] Leesan-haruki/FunctionRenderer, https://github.com/Leesan-haruki/FunctionRenderer, fetched 2026-10-26 (unofficial)
