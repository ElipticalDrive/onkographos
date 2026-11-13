# Torus Implicit Function

Also called: Signed Distance Field (SDF) Torus Primitive, Signed Distance Function (SDF) for Torus, Torus SDF, Torus Scalar Field, Torus distance, sdTorus

## Overview
The Torus Implicit Function computes the signed distance from a query point to the surface of a torus, defined by a major radius and a minor radius [S1, unofficial][S6, unofficial]. It evaluates the scalar field for a specific implicit primitive [S5, unofficial]. It is distinguished from generic quadric approximations by providing an exact analytical distance in its standard form, avoiding the numerical instability of general polynomial solvers [S2, unofficial][S6, unofficial].

## How it works
### Inputs and Outputs
- **Input**: A 3D query point $p = (p_x, p_y, p_z)$ and torus parameters [S1, unofficial][S4, unofficial].
- **Output**: A scalar value $d$ representing the signed distance. $d < 0$ indicates the point is inside the torus, $d > 0$ indicates it is outside, and $d = 0$ lies on the surface [S5, unofficial].

### Data Structures
- **Point $p$**: A 3D vector $(p_x, p_y, p_z)$ [G].
- **Parameters**: 
  - Standard form: Major radius $R$ and minor radius $r$ [S1, unofficial][S6, unofficial].
  - Alternative form: Diameter $t_x = 2R$ and thickness $t_y = 2r$ [S5, unofficial].

### Formulas and Steps
Two formulations exist in the literature: the exact analytical distance (standard) and a first-order Taylor approximation [S1, unofficial][S2, unofficial][S4, unofficial][S5, unofficial][S6, unofficial].

#### 1. Exact Analytical Distance (Standard)
Computes the exact Euclidean distance to the torus surface [S1, unofficial][S4, unofficial][S5, unofficial][S6, unofficial]. Assumes the torus is centered at the origin, aligned with the Y-axis (ring in the XZ plane) [S1, unofficial].

1. **Project to the plane of the torus ring**: Calculate the radial distance from the central Y-axis:
   $$ \rho = \sqrt{p_x^2 + p_z^2} $$
   [S1, unofficial][S4, unofficial]

2. **Define the 2D cross-section point**: Map the 3D point to a 2D plane containing the torus's center and the query point's radial projection. Let $q$ be a 2D vector:
   $$ q = (\rho - R, p_y) $$
   If using diameter/thickness parameters ($t_x, t_y$):
   $$ q = (\sqrt{p_x^2 + p_z^2} - t_x, p_y) $$
   [S1, unofficial][S5, unofficial][S6, unofficial]

3. **Compute the distance**: The signed distance is the distance from the origin to $q$, minus the minor radius:
   $$ d = \|q\| - r = \sqrt{(\rho - R)^2 + p_y^2} - r $$
   [S1, unofficial][S6, unofficial]

#### 2. First-Order Approximation (Gradient-Based)
Some implementations use a first-order Taylor approximation of the implicit function $f(p)$ to estimate the distance, particularly when gradients are needed for shading or marching [S2, unofficial].

1. **Define the implicit function**: 
   $$ f(p) = (\sqrt{p_x^2 + p_y^2} - R)^2 + p_z^2 - r^2 $$
   (Assumes Z-axis alignment as per [S3, unofficial]).
   [S3, unofficial]

2. **Compute Gradient Components**: Calculate the partial derivatives $\nabla f = (df/dx, df/dy, df/dz)$ [S3, unofficial]. Let $\rho = \sqrt{p_x^2 + p_y^2}$. Handle the singularity at $\rho=0$ with a safe epsilon:
   $$ \text{safe\_}\rho = \max(\rho, 10^{-12}) $$
   $$ \text{factor} = \frac{2(\rho - R)}{\text{safe\_}\rho} $$
   $$ \frac{\partial f}{\partial x} = \text{factor} \cdot p_x, \quad \frac{\partial f}{\partial y} = \text{factor} \cdot p_y, \quad \frac{\partial f}{\partial z} = 2 p_z $$
   [S3, unofficial]

3. **Approximate Distance**: The signed distance is approximated by dividing the implicit function value by the gradient magnitude:
   $$ d \approx \frac{f(p)}{\|\nabla f\|} $$
   [S2, unofficial]

### Pseudocode
The following implementation uses the exact analytical distance, which is the standard for ray marching and SDF rendering [S5, unofficial][S6, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sd_torus(p: tuple, major_radius: float, minor_radius: float) -> float:
    """
    Computes the signed distance from point p to a torus.
    
    Args:
        p: (px, py, pz) query point coordinates.
        major_radius: Distance from the center of the tube to the center of the torus (R).
        minor_radius: Radius of the tube (r).
        
    Returns:
        Signed distance. Negative if inside, positive if outside.
    """
    px, py, pz = p
    
    # 1. Project point to the plane of the torus ring (XZ plane)
    rho = math.sqrt(px * px + pz * pz)
    
    # 2. Define the 2D point q in the cross-sectional plane
    qx = rho - major_radius
    qy = py
    
    # 3. Compute the distance from the origin to q, minus the tube radius
    return math.sqrt(qx * qx + qy * qy) - minor_radius
```

### Parameters
- **$R$ (Major Radius)**: The distance from the center of the torus to the center of the tube [S2, unofficial].
- **$r$ (Minor Radius)**: The radius of the tube [S2, unofficial].
- **$t_x$ (Diameter)**: $2R$. Used in some implementations [S5, unofficial].
- **$t_y$ (Thickness)**: $2r$. Used in some implementations [S5, unofficial].

### Edge Cases and Pitfalls
- **Singularity at Axis**: When the query point lies on the central axis of the torus ($\rho = 0$), the term $\frac{\rho - R}{\rho}$ in the gradient calculation becomes undefined. A safe epsilon (e.g., $10^{-12}$) must be used in the denominator [S3, unofficial].
- **Approximation Failure**: The first-order approximation fails or becomes inaccurate near the torus center or when the gradient magnitude approaches zero, as it divides by the gradient norm [S2, unofficial].
- **Orientation**: The standard formula assumes the torus ring lies in the XZ plane (axis along Y) [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$. The calculation involves a constant number of arithmetic operations and square roots [G].
- **Memory Complexity**: $O(1)$. No auxiliary data structures are required [G].
- **Strengths**: Provides an exact signed distance (in the standard form), which guarantees convergence for sphere tracing/ray marching [G].
- **Limitations**: The first-order approximation is less robust near singularities [S2, unofficial].
- **When to use**: When rendering a torus shape using ray marching or voxelizing a torus volume [G].
- **When not to use**: When the torus is heavily deformed or non-uniform, as the analytical solution no longer applies [G].

## Background
The signed distance function for a torus is a fundamental primitive in implicit surface rendering [S1, unofficial]. It was popularized by Inigo Quilez in his articles on distance functions [S6, unofficial]. The exact analytical solution relies on the geometric property that a torus is the locus of points at a constant distance from a circle [G]. The first-order approximation approach relates to general quadric surface handling in ray tracing [S2, unofficial].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] Leesan-haruki/FunctionRenderer, https://github.com/Leesan-haruki/FunctionRenderer, fetched 2026-10-26 (unofficial)
[S3] jakeoung/ShapeFromProjections, https://github.com/jakeoung/ShapeFromProjections, fetched 2026-10-26 (unofficial)
[S4] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S5] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
[S6] Raymarching Distance Fields: Concepts and Implementation ..., https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
