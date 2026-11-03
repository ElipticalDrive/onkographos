# CSG Difference
Also called: SDF Difference, Subtraction of SDFs

## Overview
CSG Difference is a Constructive Solid Geometry operator that subtracts the first implicit function from the second, producing a shape that is solid where the second function is solid, except where the first is solid [S1, unofficial]. It combines two Signed Distance Fields (SDFs) into a single field to represent the volume of the second object minus the first.

## How it works
The method takes two SDF functions, $S_a$ (the subtractor) and $S_b$ (the base), and a query point $p$ [S1, unofficial]. It outputs a new SDF value $S(p)$ representing the distance to the nearest surface of the combined shape, and optionally the gradient for shading [S1, unofficial].

**Inputs:**
- $S_a(p)$: Scalar distance to object A at point $p$.
- $S_b(p)$: Scalar distance to object B at point $p$.
- $V_a(p)$: Vector gradient (normal) of $S_a$ at $p$.
- $V_b(p)$: Vector gradient (normal) of $S_b$ at $p$.

**Data Structures:**
- Scalar fields: Single floating-point values per point.
- Vector fields: 3D vectors $(x, y, z)$ representing the gradient.

**Steps:**
1. Evaluate the scalar distance for both objects at the query point $p$:
   $$d_a = S_a(p)$$
   $$d_b = S_b(p)$$
2. Compute the combined scalar distance using the maximum function:
   $$S(p) = \max(d_b, -d_a)$$ [S1, unofficial]
3. Compute the combined vector gradient (normal):
   $$V(p) = \max(V_b(p), -V_a(p))$$ [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vec_max(a, b):
    return tuple(max(x, y) for x, y in zip(a, b))

def csg_difference_scalar(S_a, S_b, p):
    d_a = S_a(p)
    d_b = S_b(p)
    return max(d_b, -d_a)

def csg_difference_vector(V_a, V_b, p):
    g_a = V_a(p)
    g_b = V_b(p)
    return vec_max(g_b, tuple(-x for x in g_a))
```

**Parameters:**
- None for the standard hard difference.

**Edge Cases:**
- Precision issues may arise when $d_a \approx 0$ or $d_b \approx 0$ near the boundary of the cutout [G].

## Subtopics

### Smooth Subtraction
A smooth approximation of the boolean subtraction operation for two SDFs, controlled by a blending parameter $k$ [S2, unofficial]. It is chosen when a sharp CSG cutout is undesirable and a gradual transition is preferred [G]. The formula is:
$$\text{subtraction}(d_1, d_2, k) = \text{mix}(d_2, -d_1, h) + k \cdot h \cdot (1 - h)$$
where $h = \text{clamp}(0.5 - 0.5 \cdot \frac{d_2 + d_1}{k}, 0., 1.)$ and $\text{mix}(a, b, t) = a \cdot (1 - t) + b \cdot t$ [S2, unofficial].
- $d_1$: Distance to object A (subtractor) [S2, unofficial].
- $d_2$: Distance to object B (base) [S2, unofficial].
- $k$: Blending radius or smoothness parameter [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, requiring two SDF evaluations and basic arithmetic [G].
- **Memory Complexity:** $O(1)$, no additional storage required beyond the two input fields [G].
- **Measured performance with hardware, resolution, and scene:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Efficiently creates complex shapes with holes; compatible with standard ray marching [G].
- **Limitations:** The hard difference can produce sharp internal corners which may require smaller step sizes in ray marching to avoid overshooting the zero-crossing [G]. Smooth subtraction introduces a parameter $k$ that must be tuned for visual quality [S2, unofficial].
- **When to use it and when not:** Use for hard cutouts or boolean subtraction in implicit modeling; avoid when smooth transitions or filleted edges are required, in which case the smooth variant should be used instead [G].

## Background
CSG operations are fundamental in implicit modeling and SDF-based rendering [G]. The specific scalar and vector formulations for difference are documented in implementations of isosurface rendering from SDFs [S1, unofficial]. The smooth variant is utilized in differentiable rendering pipelines, such as those using PyTorch3D, to allow gradient-based optimization of scene parameters [S2, unofficial].

## Sources
[S1] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
[S2] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
