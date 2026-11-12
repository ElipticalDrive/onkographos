# Parabolic Root-Finding with Gradients

Also called: Gradient-based parabola interpolation, Least-squares parabolic root finding

## Overview
Parabolic root-finding with gradients is an iterative refinement technique that estimates the zero-crossing of a scalar field $F(x)$ along a line segment by fitting a parabola to the function values and directional derivatives (gradients) at the endpoints. It solves for the root of this parabola to determine the intersection depth. It sets itself apart from its sibling, the [Secant Method for Surface Depth Refinement](996-secant-method-for-surface-depth-refinement.md), by utilizing gradient information to model curvature, thereby avoiding the need for an additional function evaluation to estimate the second derivative, while offering higher accuracy for non-linear fields.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two endpoints $x_1$ and $x_2$ on the ray, the scalar field values $F(x_1)$ and $F(x_2)$, and the gradients $\nabla F(x_1)$ and $\nabla F(x_2)$ at these points.
*   **Output:** The parameter $t \in [0, 1]$ representing the position of the root along the segment $x(t) = x_1 + t(x_2 - x_1)$.

**Data Structures:**
No complex data structures are required. The method operates on scalar values and 3D vectors.

**Steps:**
1.  **Define the Line Segment:** Parameterize the segment between $x_1$ and $x_2$ as $v(t) = x_1 + t(x_2 - x_1)$ for $t \in [0, 1]$.
2.  **Compute Directional Derivatives:** Calculate the directional derivative of $F$ along the vector $(x_2 - x_1)$ at both endpoints. The directional derivative is defined as:
    $$D_{x_2-x_1} F(x) = \nabla F(x) \cdot (x_2 - x_1)$$
    Let $d_1 = D_{x_2-x_1} F(x_1)$ and $d_2 = D_{x_2-x_1} F(x_2)$.
3.  **Fit Parabola Coefficients:** Define the parabola $F(t)$ interpolating the endpoints:
    $$F(t) = F(x_1) + (F(x_2) - F(x_1) - b)t + bt^2$$
    The derivative of this parabola is:
    $$F'(t) = F(x_2) - F(x_1) - b + 2bt$$
    Determine the coefficient $b$ by minimizing the least-squares error between the parabola's derivatives and the actual directional derivatives. The system of equations for the derivatives is:
    $$d_1 \approx F(x_2) - F(x_1) - b$$
    $$d_2 \approx F(x_1) - F(x_2) + b$$
    Solving for $b$ by averaging the two estimates yields:
    $$b = \frac{d_1 - d_2}{2}$$
4.  **Solve for Root:** Solve $F(t) = 0$ using the quadratic formula. The equation is $bt^2 + (F(x_2) - F(x_1) - b)t + F(x_1) = 0$.
    $$t = \frac{-(F(x_2) - F(x_1) - b) \pm \sqrt{(F(x_2) - F(x_1) - b)^2 - 4bF(x_1)}}{2b}$$
    Select the root $t$ that lies within the interval $[0, 1]$. The source guarantees exactly one root in this interval for valid zero-crossings.
5.  **Handle Degeneracy:** If $d_1 = d_2$, then $b=0$. The method degenerates to the linear secant method. In this case, use linear interpolation:
    $$t = \frac{-F(x_1)}{F(x_2) - F(x_1)}$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parabolic_root_finding_with_gradients(x1, x2, f1, f2, grad1, grad2):
    """
    Finds the root t in [0, 1] along the segment x1->x2.
    x1, x2: 3D vectors (endpoints)
    f1, f2: scalar field values at x1, x2
    grad1, grad2: gradient vectors at x1, x2
    """
    # 1. Direction vector
    dir_vec = x2 - x1
    
    # 2. Directional derivatives
    # D = grad . dir_vec
    d1 = dot(grad1, dir_vec)
    d2 = dot(grad2, dir_vec)
    
    # 3. Compute parabola coefficient b
    # b = (d1 - d2) / 2
    b = (d1 - d2) / 2.0
    
    # 4. Solve quadratic equation: b*t^2 + (f2 - f1 - b)*t + f1 = 0
    # Coefficients for at^2 + bt + c = 0
    a_coeff = b
    b_coeff = f2 - f1 - b
    c_coeff = f1
    
    # Handle linear case (degeneracy)
    if abs(a_coeff) < 1e-9:
        # Linear interpolation (Secant method)
        if abs(b_coeff) < 1e-9:
            return 0.5 # Fallback if both f1 and f2 are effectively equal and flat
        t = -c_coeff / b_coeff
        # Clamp to [0, 1] to handle numerical noise
        return max(0.0, min(1.0, t))
    
    # Quadratic formula
    discriminant = b_coeff**2 - 4 * a_coeff * c_coeff
    
    # Numerical stability: if discriminant is slightly negative due to precision, clamp to 0
    if discriminant < 0:
        discriminant = 0.0
        
    sqrt_disc = sqrt(discriminant)
    
    # Two possible roots
    t1 = (-b_coeff + sqrt_disc) / (2 * a_coeff)
    t2 = (-b_coeff - sqrt_disc) / (2 * a_coeff)
    
    # Select the root in [0, 1]
    # The source states there is exactly one root in the interval for valid zero-crossings.
    if 0.0 <= t1 <= 1.0:
        return t1
    elif 0.0 <= t2 <= 1.0:
        return t2
    else:
        # Fallback: return midpoint or closest boundary if no root found in interval
        # This indicates the field does not cross zero in this segment or numerical error
        return 0.5
```

**Parameters:**
*   **Tolerance:** A small epsilon (e.g., $10^{-9}$) is used to check for the linear degeneracy case ($b \approx 0$) and to clamp negative discriminants.

**Edge Cases and Pitfalls:**
*   **Equal Gradients:** If $d_1 = d_2$, $b=0$, and the quadratic formula fails. The method must fall back to linear interpolation.
*   **Precision:** If the discriminant is slightly negative due to floating-point errors, it should be clamped to zero.
*   **No Root in Interval:** If neither root lies in $[0, 1]$, the segment does not contain a zero-crossing (or the field is not monotonic/convex in a way that guarantees one).

## Subtopics

### Least-Squares Derivatives (LSDERIV)
A quadratic interpolation method that guarantees a root along an edge if the implicit function has a root, by interpolating function values and least-squares fitting derivatives. It enforces value constraints $a_0 = v_0$ and $a_0 + a_1 + a_2 = v_1$, and minimizes $(a_1 - d_0)^2 + (a_1 + 2a_2 - d_1)^2$. The resulting coefficients are $a_1 = \frac{d_0 - d_1}{2} + v_1 - v_0$ and $a_2 = \frac{d_1 - d_0}{2}$. The second order term $a_2$ vanishes if the implicit function is locally linear ($d_0 = d_1$). This solution is equivalent to the Third Order Elimination with additive offset. [S2]

## Operating characteristics
*   **Time Complexity:** $O(1)$ per segment, involving a few arithmetic operations and a square root.
*   **Memory Complexity:** $O(1)$, requiring storage for the two endpoints, function values, and gradients.
*   **Strengths:** Higher accuracy than linear interpolation for curved surfaces; avoids the cost of an additional function evaluation compared to 3-point parabolic fitting.
*   **Limitations:** Requires gradient information, which may not always be available or cheap to compute. Degenerates to linear interpolation if gradients are equal.
*   **When to use:** When gradients are available (e.g., from pre-computed textures or analytic implicit functions) and high accuracy is needed with minimal extra function evaluations.
*   **When not to use:** When gradients are unavailable and computing them is expensive, or when the field is known to be linear.

## Background
The method was introduced in "Real-Time Isosurface Extraction Using the GPU Programmable Geometry Pipeline" [S1]. It leverages the fact that gradients are often already computed for shading or are easily available in static volume textures. It is used in real-time isosurface extraction to refine intersection points on GPU pipelines.

## Sources
[S1] Real-Time Isosurface Extraction Using the GPU Programmable Geometry Pipeline, https://doi.org/10.1145/1281500.1361219, fetched 2026-11-03
[S2] Accurate Isosurface Interpolation with Hermite Data, https://ieeexplore.ieee.org/document/7335492/, fetched 2026-11-05
