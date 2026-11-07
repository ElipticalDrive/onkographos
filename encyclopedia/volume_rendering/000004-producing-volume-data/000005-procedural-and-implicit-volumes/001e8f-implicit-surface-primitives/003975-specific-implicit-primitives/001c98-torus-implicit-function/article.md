# Torus Implicit Function

Also called: Signed Distance Field (SDF) Torus Primitive, Signed Distance Function (SDF) for Torus, Torus SDF, Torus Scalar Field, Torus distance, sdTorus

## Overview
The Torus Implicit Function computes the signed distance from a query point to a torus surface defined by a major radius and a minor radius [S1, unofficial][S6]. It is distinguished from [Metaball Field Evaluation](1017-metaball-field-evaluation.md) by providing a single, non-blending primitive, and from [Superelliptic Blends](2051-superelliptic-blends.md) by its specific geometric derivation for a ring shape rather than a generalized superquadric [S6].

## How it works
**Inputs and Outputs:**
*   Input: Query point $p = (p_x, p_y, p_z)$ [S1, S5, unofficial].
*   Input: Torus parameters. Two conventions exist:
    1.  Major radius $R$ and minor radius $r$ [S1, S3, unofficial][S6].
    2.  Diameter $t_x = 2R$ and thickness $t_y = 2r$ [S5, unofficial].
*   Output: Signed distance $d$. Negative if inside, positive if outside, zero on the surface [S5, unofficial].

**Data Structures:**
The function is stateless and evaluates per-point [G]. Parameters are typically stored as two floats representing the major and minor radii [S5, unofficial].

**Steps:**
The exact signed distance function is derived by projecting the point onto the plane of the torus ring and computing the distance to the center of the tube [S6].

1.  Compute the radial distance of the point from the axis of revolution. For a torus aligned with the Y-axis (ring in the XZ plane), this is $\rho = \sqrt{p_x^2 + p_z^2}$ [S1, S5, unofficial].
2.  Construct a 2D vector $q$ representing the point's position relative to the center of the tube's cross-section in the radial-vertical plane:
    $$ q = (\rho - R, p_y) $$
    If using diameter/thickness parameters:
    $$ q = (\sqrt{p_x^2 + p_z^2} - t_x, p_y) $$
    [S1, S5, unofficial].
3.  Compute the Euclidean length of $q$:
    $$ \|q\| = \sqrt{q_x^2 + q_y^2} $$
    [S1, S5, unofficial].
4.  Subtract the minor radius (or half-thickness) to get the signed distance:
    $$ d = \|q\| - r $$
    If using thickness:
    $$ d = \|q\| - t_y $$
    [S1, S5, unofficial].

**Approximation Note:**
Some sources describe a first-order Taylor approximation for generic quadrics, computing $f_0/f_1$ where $f_1$ is the gradient magnitude [S2, unofficial]. This approximation fails or becomes inaccurate near the torus center or when the gradient magnitude approaches zero [S2, unofficial]. The exact geometric derivation is standard for tori [S1, unofficial][S6].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sd_torus(p_x, p_y, p_z, major_radius, minor_radius):
    """
    Computes the signed distance from point p to a torus aligned with the Y-axis.
    
    Args:
        p_x, p_y, p_z: Coordinates of the query point.
        major_radius: Distance from the center of the tube to the center of the torus (R).
        minor_radius: Radius of the tube (r).
        
    Returns:
        Signed distance. Negative inside, positive outside.
    """
    # 1. Compute radial distance from the Y-axis
    rho = math.sqrt(p_x * p_x + p_z * p_z)
    
    # 2. Construct 2D vector q in the radial-vertical plane
    q_x = rho - major_radius
    q_y = p_y
    
    # 3. Compute length of q
    q_length = math.sqrt(q_x * q_x + q_y * q_y)
    
    # 4. Subtract minor radius
    return q_length - minor_radius

def sd_torus_diameter_thickness(p_x, p_y, p_z, diameter, thickness):
    """
    Computes signed distance using diameter and thickness parameters.
    Note: Source [S5, unofficial] labels these as diameter/thickness but uses them directly 
    as radii in the distance formula.
    """
    rho = math.sqrt(p_x * p_x + p_z * p_z)
    q_x = rho - diameter
    q_y = p_y
    q_length = math.sqrt(q_x * q_x + q_y * q_y)
    return q_length - thickness
```

**Edge Cases and Pitfalls:**
*   **Singularity at Axis:** When $p_x = 0$ and $p_z = 0$, $\rho = 0$. The distance formula remains well-defined, but gradient-based normal computation requires a guard to avoid division by zero [S3, unofficial].
*   **Gradient Division:** If using the first-order approximation method ($f_0/f_1$), the gradient magnitude $f_1$ approaches zero near the torus center or specific singularities, causing instability [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per point [G].
*   **Memory Complexity:** $O(1)$ [G].
*   **Strengths:** Provides an exact signed distance suitable for sphere tracing and ray marching [S5, unofficial][S6].
*   **Limitations:** Only applies to toroidal shapes [S6].
*   **When to use:** Rendering toroidal geometry via ray marching [S5, unofficial].
*   **When not to use:** Complex boolean operations without blending [S6].

## Background
The signed distance function for a torus is a standard primitive in ray marching and implicit surface rendering [S5, unofficial]. It is derived from the geometric definition of a torus as a surface of revolution [S6]. The exact formulation is widely attributed to Inigo Quilez's work on distance functions [S5, unofficial]. The first-order approximation method for quadrics is discussed in general implicit surface rendering contexts [S2, unofficial].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] Leesan-haruki/FunctionRenderer, https://github.com/Leesan-haruki/FunctionRenderer, fetched 2026-10-26 (unofficial)
[S3] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S4] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
[S5] Raymarching Distance Fields: Concepts and Implementation ..., https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
[S6] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
