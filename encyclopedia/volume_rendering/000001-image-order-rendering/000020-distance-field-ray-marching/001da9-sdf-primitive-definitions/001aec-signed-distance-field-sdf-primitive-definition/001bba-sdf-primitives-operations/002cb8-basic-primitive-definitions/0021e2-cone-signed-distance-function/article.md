# Cone Signed Distance Function

Also called: Cone SDF, sdCone, Signed Cone, Signed Capped Cone, Exact Cone SDF, Signed Distance Function (SDF) for Cone

## Overview
The Cone Signed Distance Function computes the exact Euclidean distance from a query point to the surface of a cone, including its base cap and apex, while correctly determining the sign (positive outside, negative inside) [S4, unofficial]. It solves the problem of efficiently rendering or ray-marching conical primitives without polygonization [G]. Unlike the [Quadric SDF (First-Order Approximation)](2682-quadric-sdf-first-order-approximation.md), which approximates the cone as a smooth quadric surface, the Cone SDF provides the exact analytical distance, preserving sharp edges at the base and apex [G]. It builds on [SDF Primitives & Operations](1075-sdf-primitives-operations.md) by providing a specific primitive definition [G].

## How it works
The method reduces the 3D distance calculation to a 2D problem by exploiting rotational symmetry around the cone's axis [S4, unofficial]. The point is projected onto a 2D plane containing the axis and the radial distance, and the distance to the 2D line segment (representing the cone's side) and the base cap is computed [S4, unofficial].

### Inputs and Outputs
*   **Input**: Point $p \in \mathbb{R}^3$, base radius $r \in \mathbb{R}^+$, height $h \in \mathbb{R}^+$ [S4, unofficial].
*   **Output**: Signed distance $d \in \mathbb{R}$ [S4, unofficial].

### Data Structures
No complex data structures are required. The computation uses standard 2D and 3D vector operations [G].

### Parameters
*   $r$: Base radius of the cone [S4, unofficial].
*   $h$: Height of the cone [S4, unofficial].
*   Orientation: The implementations in [S2, unofficial] and [S4, unofficial] align the cone along the Y-axis, with the apex at the origin $(0,0,0)$ and the base centered at $y = -h$ (opening downward) [S4, unofficial]. [S5] describes a cone along the Z-axis [S5]. The pseudocode below follows the Y-axis orientation [S4, unofficial].

### Steps
1.  **2D Projection**: Project the 3D point $p$ into the 2D plane defined by the radial distance and the Y-axis [S4, unofficial].
    $$ q = (\sqrt{p_x^2 + p_z^2}, p_y) $$
2.  **Define Geometry**: Define the 2D coordinates of the apex (tip) and the base edge [S4, unofficial].
    $$ \text{tip} = (0, 0) $$
    $$ \text{base} = (r, -h) $$
3.  **Vector Definitions**:
    $$ k = (r, h) $$
    $$ m_2 = r^2 + h^2 $$
    $$ d_1 = q - \text{tip} = q $$
    $$ d_3 = q - (\text{clamp}(q_x, 0, r), -h) $$
4.  **Side Distance**: Compute the distance from $q$ to the line segment connecting the tip and the base [S4, unofficial].
    $$ \text{ba} = \text{base} - \text{tip} = (r, -h) $$
    $$ u = \text{clamp}\left(\frac{d_1 \cdot \text{ba}}{m_2}, 0, 1\right) $$
    $$ \text{side\_dist} = \| d_1 - \text{ba} \cdot u \| $$
5.  **Cap Distance**: Compute the distance from $q$ to the base cap [S4, unofficial].
    $$ \text{cap\_dist} = \| d_3 \| $$
6.  **Unsigned Distance**:
    $$ \text{dist} = \min(\text{side\_dist}, \text{cap\_dist}) $$
7.  **Inside Test**: Determine if the point is inside the cone [S4, unofficial].
    $$ \text{inside} = (q_y \le 0) \land (q_y \ge -h) \land (q_x \cdot h + q_y \cdot r \le 0) $$
8.  **Signed Distance**:
    $$ \text{result} = \begin{cases} -\text{dist} & \text{if inside} \\ \text{dist} & \text{otherwise} \end{cases} $$

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sd_cone(p_x: float, p_y: float, p_z: float, r: float, h: float) -> float:
    """
    Computes the signed distance from point p to a cone with base radius r
    and height h, aligned with the Y-axis, apex at origin, opening downward.
    """
    # 1. Project to 2D plane (radial distance, y)
    q_x = math.sqrt(p_x * p_x + p_z * p_z)
    q_y = p_y

    # 2. Define geometry in 2D
    tip_x, tip_y = 0.0, 0.0
    base_x, base_y = r, -h

    # 3. Vectors
    k_x, k_y = r, h
    m2 = k_x * k_x + k_y * k_y

    d1_x, d1_y = q_x - tip_x, q_y - tip_y

    # Clamp x to [0, r] for base projection
    clamped_q_x = max(0.0, min(r, q_x))
    d3_x, d3_y = q_x - clamped_q_x, q_y - base_y

    # 4. Side distance (distance to segment tip-base)
    ba_x, ba_y = base_x - tip_x, base_y - tip_y
    dot_d1_ba = d1_x * ba_x + d1_y * ba_y
    u = max(0.0, min(1.0, dot_d1_ba / m2))
    side_dist = math.sqrt((d1_x - ba_x * u) ** 2 + (d1_y - ba_y * u) ** 2)

    # 5. Cap distance
    cap_dist = math.sqrt(d3_x ** 2 + d3_y ** 2)

    # 6. Unsigned distance
    dist = min(side_dist, cap_dist)

    # 7. Inside test
    inside = (q_y <= 0.0) and (q_y >= -h) and ((q_x * h + q_y * r) <= 0.0)

    # 8. Return signed distance
    return -dist if inside else dist
```

### Edge Cases and Pitfalls
*   **Normalization**: In alternative formulations using a normalized vector $c$ for the cone slope [S3, unofficial], the vector $c$ must be normalized. Failure to normalize results in incorrect distances [S3, unofficial].
*   **Precision**: When $r$ or $h$ are very small, floating-point precision issues may arise in the dot product and clamp operations [G].
*   **Apex Singularity**: At the apex $(0,0,0)$, the gradient is undefined, but the distance is correctly 0 [G].
*   **Base Ring**: The implementation above handles the base cap as a flat disk [S4, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(1)$. The calculation involves a fixed number of arithmetic operations, square roots, and comparisons [G].
*   **Memory Complexity**: $O(1)$. No auxiliary data structures are needed [G].
*   **Strengths**: Exact distance calculation allows for precise ray marching and constructive solid geometry (CSG) operations [G]. It is significantly faster than polygon-based distance estimation for simple cones [G].
*   **Limitations**: The sharp edges at the base and apex can cause aliasing in ray marching if the step size is not adaptive [G]. The function is not differentiable at the edges and apex [G].
*   **When to use**: When rendering or ray-marching conical primitives with high precision is required [G].
*   **When not to use**: When a smooth approximation is preferred for aesthetic reasons or to avoid gradient discontinuities, consider the [Quadric SDF (First-Order Approximation)](2682-quadric-sdf-first-order-approximation.md) [G].

## Background
The Cone SDF is based on the analytical distance functions popularized by Inigo Quilez in his article "Distance Functions" [S1, unofficial][S4, unofficial]. The specific implementation for the capped cone along the Y-axis is widely used in GLSL-based ray marching engines [S2, unofficial]. The formulation along the Z-axis is also described in geometric ray tracing literature [S5]. The method is a staple in the [SDF Primitives & Operations](1075-sdf-primitives-operations.md) toolkit [G].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] tdhooper/glsl-marching-cubes, https://github.com/tdhooper/glsl-marching-cubes, fetched 2026-10-26 (unofficial)
[S3] marklundin/glsl-sdf-primitives, https://github.com/marklundin/glsl-sdf-primitives, fetched 2026-10-26 (unofficial)
[S4] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
[S5] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
