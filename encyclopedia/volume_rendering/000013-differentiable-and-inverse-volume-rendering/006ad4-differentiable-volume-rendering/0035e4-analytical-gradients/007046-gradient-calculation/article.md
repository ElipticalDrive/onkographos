# Gradient Calculation


## Overview
Gradient calculation computes the vector field of partial derivatives for a scalar field, pointing in the direction of the greatest rate of decrease of the field's value [S1, unofficial]. In volume rendering, this vector is used for shading to simulate surface-like lighting effects on volumetric data [S1, unofficial]. It is distinguished from numerical finite-difference approximations by its potential for analytical derivation when the field is defined by a differentiable function or transformation, ensuring consistency with differentiable rendering pipelines [S2].

## How it works
**Inputs:** A scalar field function $f(x, y, z)$ or a transformed field defined by parameters; a query point $\mathbf{p} = (x, y, z)$ [G].
**Outputs:** The gradient vector $\nabla f$ at $\mathbf{p}$ [G].

**Data Structures:**
*   **Scalar Field:** Represented as a function $f: \mathbb{R}^3 \to \mathbb{R}$ or a discrete grid where interpolation is applied [G].
*   **Gradient Vector:** A 3D vector $\mathbf{g} = (g_x, g_y, g_z)$ [G].

**Standard Cartesian Gradient:**
For a scalar field $f$, the gradient is defined as:
$$
\nabla f = \left( \frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z} \right)
$$
where $\frac{\partial f}{\partial x}$, $\frac{\partial f}{\partial y}$, and $\frac{\partial f}{\partial z}$ are the partial derivatives with respect to the spatial coordinates [S1, unofficial].

**Pseudocode for Standard Gradient:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vector3:
    """Simple 3D vector container for gradient computation."""
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z
    def __truediv__(self, scalar):
        return Vector3(self.x / scalar, self.y / scalar, self.z / scalar)
    def magnitude(self):
        return (self.x**2 + self.y**2 + self.z**2)**0.5
    def normalize(self):
        m = self.magnitude()
        return self / m if m > 0 else Vector3(0, 0, 0)

def compute_gradient_cartesian(field_fn, deriv_fn, point):
    """
    Computes the gradient of a scalar field at a point using analytical partial derivatives.
    
    Args:
        field_fn: Callable f(x, y, z) returning a scalar.
        deriv_fn: Callable returning (df_dx, df_dy, df_dz) at (x, y, z).
        point: Tuple (x, y, z).
        
    Returns:
        Vector3 representing the gradient.
    """
    x, y, z = point
    dx, dy, dz = deriv_fn(x, y, z)
    return Vector3(dx, dy, dz)
```

**Möbius Transformation of Gradients:**
When the scalar field is defined in a kernel space and transformed to world space via a Möbius transformation $L$, the gradient must be transformed accordingly [S2].

**Inputs:**
*   $q$: Point in kernel space (quaternion) [S2].
*   $g$: Gradient at $q$ in kernel space (pure quaternion) [S2].
*   $L$: Möbius transformation defined by quaternions $a, b, c, d$ [S2].
*   $q_w$: Corresponding point in world space, $q_w = L(q)$ [S2].

**Outputs:**
*   $g_w$: Gradient at $q_w$ in world space (pure quaternion) [S2].

**Steps:**
1.  Define the gradient $g_w$ as the differential change of $q_w$ in response to a differential change of $q$ along $g$:
    $$
    g_w = \frac{d\mathcal{L}(q+g\epsilon)}{d\epsilon} \bigg|_{\epsilon=0}
    $$
2.  Expand the Möbius transformation $L(q) = [a q + b][c q + d]^{-1}$ [S2].
3.  Apply the product rule and substitute $\epsilon = 0$ [S2].
4.  Apply the reciprocal rule for quaternions: $\frac{df^{-1}(x)}{dx} = -f^{-1}(x) \frac{df(x)}{dx} f^{-1}(x)$ [S2].
5.  Factor the result to isolate $g$ [S2].

**Formula:**
The general form is:
$$
g_w = \left[ a - (aq + b)(cq + d)^{-1}c \right] g(cq + d)^{-1}
$$
If $q_w$ is known, this simplifies to:
$$
g_w = (a - q_w c) g(cq + d)^{-1} \quad (3)
$$
where $q_w = [a q + b][c q + d]^{-1}$ [S2].

**Pitfalls:**
*   **Normalization:** The resulting gradient $g_w$ must be normalized to unit length if it represents a pure direction, as the transformation may scale it [S2].
*   **Non-commutativity:** Quaternion multiplication is non-commutative; the order of terms in the formula is critical [S2].
*   **Singularity:** The transformation is undefined if $cq + d = 0$ [S2].

**Usage:**
Used to obtain world-space gradients from kernel-space gradients when $L$ transforms from kernel space to world space [S2].

## Subtopics

### Möbius Transformation of Gradients
A method to compute the gradient $g_w$ of a transformed distance field at point $q_w = L(q)$, given the gradient $g$ at point $q$ in kernel space and the Möbius transformation $L$ [S2]. It sets itself apart by handling non-linear conformal mappings of spatial derivatives, essential for kernel-based rendering techniques [S2]. Choose it when rendering volumes defined by kernel functions that are transformed via Möbius mappings to world space [S2]. Link: [Kernel-Reflection Sequences](https://doi.org/10.2312/stag.20161364).

## Operating characteristics
Time complexity: Not given in the sources.
Memory complexity: Not given in the sources.
Strengths: Not given in the sources.
Limitations: Not given in the sources.
When to use: Not given in the sources.
When not to use: Not given in the sources.

## Background
Gradient calculation is a fundamental concept in vector calculus applied to computer graphics [G]. In volume rendering, it was identified as a trivial but effective method for shading scalar fields by mapping density to opacity and velocity to color, allowing insights into fluid flow patterns [S1, unofficial]. The specific application of Möbius transformations to gradients in the context of kernel-based volume rendering was detailed in work on Kernel-Reflection Sequences [S2].

## Sources
[S1] Qwen version of Ct Volume Rendering, https://www.flyriver.com/q/ct-volume-rendering, fetched 2026-10-29 (unofficial)
[S2] Kernel-Reflection Sequences, https://doi.org/10.2312/stag.20161364, fetched 2026-10-26
