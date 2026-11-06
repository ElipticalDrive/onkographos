# Distance to Natural Quadrics

Also called: Natural Quadrics SDF, SDF for Natural Quadrics

## Overview
Distance to Natural Quadrics provides closed-form Signed Distance Functions (SDFs) for standard solid modeling primitives: planes, spheres, cylinders, cones, and tori [S2]. It solves the problem of efficiently querying the Euclidean distance from a query point to these geometric surfaces, which is the core operation in ray marching (sphere tracing) [S3, unofficial]. Unlike constructive methods that combine simpler shapes, this method provides direct analytical solutions for primitives like tori and cones, which are essential for modeling structural elements without approximation errors [S2][S4, unofficial]. It is distinct from [Constructive SDF Modeling](871-constructive-sdf-modeling-min-combination.md) because it handles primitives that cannot be easily decomposed into unions/intersections of boxes or spheres, and from [Distance to Superquadrics](1561-distance-to-superquadrics.md) because it operates in the standard Euclidean metric rather than $L_p$ norms [S2].

## How it works
**Inputs:**
- Query point $\mathbf{p} = (p_x, p_y, p_z)$ in 3D space [G].
- Primitive-specific parameters (e.g., radius, height, normal) [G].

**Outputs:**
- Scalar signed distance $d$.
- $d < 0$: point is inside the primitive.
- $d = 0$: point is on the surface.
- $d > 0$: point is outside the primitive [S2][S3, unofficial].

**Data Structures:**
- Vectors are treated as standard 3D Cartesian coordinates [G].
- Parameters are typically stored as floats or vectors (e.g., `vec2` for torus radii) [G].

**Formulas and Steps:**

1.  **Plane:**
    Defined by a unit normal vector $\mathbf{n}$ and a point $\mathbf{p}_0$ on the plane [S2].
    $$d(\mathbf{p}) = \mathbf{n} \cdot (\mathbf{p} - \mathbf{p}_0)$$

2.  **Sphere:**
    Defined by center $\mathbf{c}$ and radius $r$ [S2].
    $$d(\mathbf{p}) = \|\mathbf{p} - \mathbf{c}\| - r$$
    For a unit sphere at the origin ($\mathbf{c}=\mathbf{0}, r=1$):
    $$d(\mathbf{p}) = \|\mathbf{p}\| - 1$$

3.  **Cylinder:**
    The source [S2] describes a unit-radius cylinder centered about the z-axis, defined by projecting into the xy-plane. [S1, unofficial] and [S3, unofficial] provide implementations for capped cylinders or axis-aligned variants.
    - *Infinite Cylinder (z-axis, radius $r$):*
      $$d(\mathbf{p}) = \sqrt{p_x^2 + p_y^2} - r$$
      (Derived from [S2] projecting to the plane perpendicular to the axis).
    - *Capped Cylinder (y-axis, radius $r$, half-height $h$):*
      Let $d_{\text{rad}} = \sqrt{p_x^2 + p_z^2} - r$ and $d_{\text{ax}} = |p_y| - h$ [S1, unofficial].
      $$d = \min(\max(d_{\text{rad}}, d_{\text{ax}}), 0) + \sqrt{\max(d_{\text{rad}}, 0)^2 + \max(d_{\text{ax}}, 0)^2}$$
      This formula returns the distance to the side, cap, or corner, whichever is nearest [S1, S3, unofficial].

4.  **Cone:**
    Defined by apex, axis, and divergence angle $\theta$ (or sin/cos of the angle) [S2][S1, unofficial].
    Let the cone axis be aligned with the y-axis, apex at origin, and height $h$. Let $c_x = \sin \theta$ and $c_y = \cos \theta$ [S1, unofficial].
    Let $\mathbf{w} = (\sqrt{p_x^2 + p_z^2}, p_y)$ and $\mathbf{q} = (h \cdot c_x/c_y, -h)$ [S1, unofficial].
    The distance is calculated by projecting $\mathbf{w}$ onto the cone surface line defined by $\mathbf{q}$ [S1, unofficial]:
    $$\mathbf{a} = \mathbf{w} - \mathbf{q} \cdot \text{clamp}\left(\frac{\mathbf{w} \cdot \mathbf{q}}{\mathbf{q} \cdot \mathbf{q}}, 0, 1\right)$$
    $$\mathbf{b} = \mathbf{w} - \mathbf{q} \cdot \left(\text{clamp}\left(\frac{w_x}{q_x}, 0, 1\right), 1\right)$$
    $$d = \sqrt{\min(\|\mathbf{a}\|^2, \|\mathbf{b}\|^2)} \cdot \text{sign}(s)$$
    Where $k = \text{sign}(q_y)$ and $s = \max(k(w_x q_y - w_y q_x), k(w_y - q_y))$ determines the inside/outside based on the cone's orientation [S1, unofficial].

5.  **Torus:**
    Defined by major radius $R$ (distance from center of tube to center of torus) and minor radius $r$ (radius of the tube) [S4, unofficial].
    Let $q_x = \sqrt{p_x^2 + p_z^2} - R$ and $q_y = p_y$ [S4, unofficial].
    $$d(\mathbf{p}) = \sqrt{q_x^2 + q_y^2} - r$$
    Typical values: $R=0.8, r=0.3$ [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sdf_plane(p, n, p0):
    dx, dy, dz = p[0] - p0[0], p[1] - p0[1], p[2] - p0[2]
    return n[0]*dx + n[1]*dy + n[2]*dz

def sdf_sphere(p, c, r):
    dx, dy, dz = p[0] - c[0], p[1] - c[1], p[2] - c[2]
    return math.sqrt(dx*dx + dy*dy + dz*dz) - r

def sdf_cylinder_capped(p, r, h):
    d_rad = math.sqrt(p[0]**2 + p[2]**2) - r
    d_ax = abs(p[1]) - h
    return min(max(d_rad, d_ax), 0.0) + math.sqrt(max(d_rad, 0.0)**2 + max(d_ax, 0.0)**2)

def sdf_torus(p, R, r):
    qx = math.sqrt(p[0]**2 + p[2]**2) - R
    qy = p[1]
    return math.sqrt(qx**2 + qy**2) - r

def sdf_cone(p, h, sin_theta, cos_theta):
    qx = h * sin_theta / cos_theta
    qy = -h
    q = (qx, qy)
    w_x = math.sqrt(p[0]**2 + p[2]**2)
    w_y = p[1]
    w = (w_x, w_y)
    dot_wq = w_x * qx + w_y * qy
    dot_qq = qx * qx + qy * qy
    t = dot_wq / dot_qq
    t = max(0.0, min(1.0, t))
    a_x = w_x - qx * t
    a_y = w_y - qy * t
    b_x = w_x - qx * max(0.0, min(1.0, w_x / qx))
    b_y = w_y - qy
    d_sq_a = a_x**2 + a_y**2
    d_sq_b = b_x**2 + b_y**2
    d = math.sqrt(min(d_sq_a, d_sq_b))
    k = -1.0 if qy < 0 else 1.0
    s = max(k * (w_x * qy - w_y * qx), k * (w_y - qy))
    return d * (1.0 if s > 0 else -1.0)
```

## Subtopics

### [Distance to Superquadrics](1561-distance-to-superquadrics.md)
Generalizes natural quadrics using $L_p$ norms to create shapes with rounded, square, or pinched cross-sections. Choose this when you need smoothable variations of primitives (e.g., squaring a cylinder) rather than exact Euclidean distances.

### [Cylinder Signed Distance Function](2739-cylinder-signed-distance-function.md)
Computes the exact Euclidean distance for a finite cylinder defined by two endpoints and a radius. Choose this for pipes and limbs where the axis is arbitrary in 3D space, whereas Natural Quadrics typically assumes axis-aligned orientations.

### [Torus Signed Distance Function](2864-torus-signed-distance-function.md)
Computes the exact Euclidean distance for a torus. While included in Natural Quadrics, this entry focuses specifically on the torus topology and its specific algebraic form, distinguishing it from other ring-like or looped primitives.

## Operating characteristics
- **Time Complexity:** $O(1)$ per primitive evaluation [G].
- **Memory Complexity:** $O(1)$ per primitive [G].
- **Performance:** Extremely fast, suitable for real-time ray marching on GPUs [S3, unofficial]. The torus and capped cylinder require square roots, which are slightly more expensive than simple additions but negligible compared to iterative methods [G].
- **Strengths:** Exact Euclidean distance ensures stable and efficient ray marching (no overshooting or jittering) [S3, unofficial]. Closed-form solutions avoid numerical instability associated with iterative solvers [G].
- **Limitations:** Limited to specific geometric shapes [G]. Primitives are defined in local coordinate space and require affine transformation of the query point $\mathbf{p}$ for arbitrary orientation or translation [G].
- **When to use:** When rendering scenes composed of standard geometric primitives and exact distance bounds are required for convergence [S2].
- **When not to use:** When the primitive is not a natural quadric (use [Distance to Superquadrics](1561-distance-to-superquadrics.md) or mesh SDFs) [G]. When the primitive is arbitrarily oriented and transforming the point is too costly (consider pre-computing transformations or using [Cylinder Signed Distance Function](2739-cylinder-signed-distance-function.md) with endpoint vectors) [G].

## Background
Signed Distance Functions for natural quadrics were formalized in the context of "Sphere Tracing," a geometric method for antialiased ray tracing of implicit surfaces [S2]. The appendix of [S2] derives these distances to aid in the implementation of sphere tracing, noting that while they may be less efficient than closed-form solutions for *simple* primitives like boxes (which can be handled by min/max operations), they are essential for higher-order constructions like blends and deformations [S2]. The torus SDF is a classic result, derived from the product of two circles [S2]. The cone SDF derivation involves trigonometric projections of the query point onto the cone's profile line [S2][S1, unofficial]. These methods are foundational in procedural geometry and ray marching tutorials [S3, S4, unofficial].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] (PDF) Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S3] Ray Marching with Signed Distance Fields - Scribd, https://www.scribd.com/document/831412915/1-Ray-Marching-and-Signed-Distance-Fields, fetched 2026-10-27 (unofficial)
[S4] Introduction to Raymarching Distance Functions, https://observablehq.com/@gre/introduction-to-raymarching-distance-functions, fetched 2026-10-29 (unofficial)
