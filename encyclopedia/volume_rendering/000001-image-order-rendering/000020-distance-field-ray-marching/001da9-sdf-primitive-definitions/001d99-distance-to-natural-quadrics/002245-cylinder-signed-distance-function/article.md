# Cylinder Signed Distance Function

Also called: Capped Cylinder SDF, Cylinder SDF, Finite Cylinder SDF, Signed Capped Cylinder, Signed Cylinder, Signed Distance Function (SDF) for Cylinder, sdCappedCylinder, sdCylinder

## Overview
The Cylinder Signed Distance Function (SDF) computes the exact Euclidean distance from a query point to a finite cylinder defined by two endpoints and a radius [S1, unofficial]. Unlike infinite cylinder variants, the capped version correctly handles the flat end caps and their corners, returning negative values inside the volume and positive values outside [S2, S4, unofficial]. It is distinct from the [Torus Signed Distance Function](2864-torus-signed-distance-function.md) (which models a ring) and [Distance to Superquadrics](1561-distance-to-superquadrics.md) (which offers smoothable variations but not exact Euclidean distance for standard primitives) [S2, unofficial][S5].

## How it works
**Inputs:**
* $p$: Query point (Vector3) [S1, unofficial].
* $c_1, c_2$: Cylinder endpoints (Vector3) [S1, unofficial].
* $r$: Cylinder radius (scalar) [S1, unofficial].

**Outputs:**
* $d$: Signed distance (scalar). Negative if inside, positive if outside, zero on surface [S2, unofficial].

**Data Structures:**
* Standard 3D vector ($x, y, z$) and scalar types. No special bit-packing required for the core function [G].

**Steps:**
1. **Compute Axis Geometry:** Calculate the axis direction vector $dir = c_2 - c_1$ and its squared length $len\_sq = dir \cdot dir$ [S1, S4, unofficial].
2. **Handle Degeneracy:** If $len\_sq < \epsilon$ (degenerate cylinder), return $length(p - c_1) - r$ [G].
3. **Project Point:** Compute the unclamped projection parameter $t = \frac{(p - c_1) \cdot dir}{len\_sq}$ [S1, unofficial].
4. **Compute Closest Point:** Calculate the closest point on the finite axis segment: $closest = c_1 + dir \cdot clamp(t, 0.0, 1.0)$ [S1, S4, unofficial].
5. **Calculate Radial Distance:** Compute the vector from the closest axis point to the query point: $diff = p - closest$. The radial distance is $side\_dist = length(diff) - r$ [S1, S4, unofficial].
6. **Determine Signed Distance:** Apply the piecewise logic based on $t$ and $side\_dist$, where $h = length(dir)$ is the cylinder height [S1, unofficial]:
   * If $0 \le t \le 1$: return $side\_dist \ge 0 \ ? \ side\_dist \ : \ min(side\_dist, min(h \cdot t, h \cdot (1 - t)))$ [S1, unofficial].
   * If $t > 1$: return $side\_dist \ge 0 \ ? \ \sqrt{side\_dist^2 + h^2(t-1)^2} \ : \ h(t-1)$ [S1, unofficial].
   * If $t < 0$: return $side\_dist \ge 0 \ ? \ \sqrt{side\_dist^2 + h^2(-t)^2} \ : \ h(-t)$ [S1, unofficial].

**Axis-Aligned Simplification:**
For a cylinder aligned with the Y-axis (or Z-axis) with radius $r$ and half-height $h$, the distance can be computed using a 2D projection trick structurally identical to the box SDF [S2, S3, S4, unofficial]:
1. $q = (length(p_{xz}) - r, |p_y| - h)$ [S2, S4, unofficial].
2. $d = max(q, 0.0)$ [S2, S4, unofficial].
3. $inside = min(max(q_x, q_y), 0.0)$ [S2, S4, unofficial].
4. $outside = length(d)$ [S2, S4, unofficial].
5. Return $outside + inside$ [S2, S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sd_capped_cylinder(p, c1, c2, r):
    """
    Computes signed distance to a capped cylinder.
    p: query point (Vec3)
    c1, c2: endpoints (Vec3)
    r: radius (float)
    """
    axis = c2 - c1
    len_sq = dot(axis, axis)
    
    # Handle degenerate case (point cylinder)
    if len_sq < 1e-12:
        return length(p - c1) - r

    # Project p onto axis
    pa = p - c1
    t = dot(pa, axis) / len_sq
    
    # Closest point on the segment
    t_clamped = max(0.0, min(1.0, t))
    closest = c1 + axis * t_clamped
    
    # Radial distance
    diff = p - closest
    side_dist = length(diff) - r
    
    h = length(axis)
    
    # Piecewise signed distance logic
    if 0.0 <= t <= 1.0:
        if side_dist >= 0.0:
            return side_dist
        else:
            return min(side_dist, min(h * t, h * (1.0 - t)))
    elif t > 1.0:
        if side_dist >= 0.0:
            return sqrt(side_dist**2 + h**2 * (t - 1.0)**2)
        else:
            return h * (t - 1.0)
    else: # t < 0.0
        if side_dist >= 0.0:
            return sqrt(side_dist**2 + h**2 * (-t)**2)
        else:
            return h * (-t)
```

## Subtopics

### Ring Signed Distance Function
Computes the signed distance to a ring or annulus with radius $r$ and thickness $th$ [S4, unofficial]. The formula projects the query point into 2D, measures the radial deviation from $r$, and subtracts half the thickness: $|\|p\| - r| - th \cdot 0.5$ [S4, unofficial]. Implementation: `return (p.length() - r).abs() - th * 0.5` [S4, unofficial].

### Bézier Curve Signed Distance Function
Calculates the distance to the global offset surface of a cubic Bézier space curve [S5]. The method defines the curve parametrically as $p(u)$ and finds the closest point $p(u)$ to the query point $x$ by solving the orthogonality condition $(x - p(u)) \cdot p'(u) = 0$ [S5]. This condition converts into a degree-five 1D Bézier curve, which is solved efficiently using standard root-finding techniques [S5].

## Operating characteristics
**Time and Memory Complexity:** O(1) time and O(1) memory per query point [G]. The computation involves a constant number of vector operations, dot products, and square roots [S1, S2, unofficial].
**Measured Performance:** Runs efficiently on modern GPUs and CPUs; the axis-aligned 2D projection variant is slightly faster due to avoiding branch-heavy logic and using simpler arithmetic [S2, S4, unofficial].
**Strengths and Limitations:** Provides exact Euclidean distance, making it robust for ray marching, collision detection, and CSG operations [S1, S2, unofficial]. The arbitrary-axis implementation requires careful handling of degenerate inputs (zero-length cylinders) and floating-point precision near the caps [G]. It is more expensive than infinite cylinder SDFs but necessary for finite geometry [S2, unofficial].
**When to Use:** Use when exact distance to a finite, arbitrarily oriented cylinder is required for rendering or physics [S1, S2, unofficial]. Avoid when infinite extent is sufficient or when approximate distance fields are acceptable for performance [S2, unofficial].

## Background
Originated as part of the standard set of signed distance functions for natural quadrics used in implicit surface rendering and ray marching [S1, S2, unofficial]. Popularized by GLSL shader libraries and real-time rendering pipelines for modeling pipes, limbs, and structural elements [S2, S4, unofficial]. The mathematical formulation for arbitrary-axis cylinders relies on projecting query points onto the central segment and handling cap corners via Pythagorean distance [S1, unofficial]. The axis-aligned 2D projection trick generalizes the box SDF to cylindrical coordinates [S2, S4, unofficial]. It is widely used in distance-field ray marching for antialiased implicit surface rendering [S5].

## Sources
[S1] Leesan-haruki/FunctionRenderer, https://github.com/Leesan-haruki/FunctionRenderer, fetched 2026-10-26 (unofficial)
[S2] marklundin/glsl-sdf-primitives, https://github.com/marklundin/glsl-sdf-primitives, fetched 2026-10-26 (unofficial)
[S3] markpmlim/RaymarchingAndSDF, https://github.com/markpmlim/RaymarchingAndSDF, fetched 2026-10-27 (unofficial)
[S4] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
[S5] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
