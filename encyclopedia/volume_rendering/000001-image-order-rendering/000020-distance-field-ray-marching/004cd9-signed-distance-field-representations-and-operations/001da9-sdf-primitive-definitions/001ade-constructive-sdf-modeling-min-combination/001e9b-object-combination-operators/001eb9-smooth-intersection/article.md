# Smooth Intersection

Also called: Round Intersection, S-Intersection, Smooth SDF Operations

## Overview
Smooth Intersection is a Constructive SDF Modeling operator that blends the overlap region of two shapes to produce rounded edges, serving as a smooth approximation of the boolean intersection operation [S1, S2, S3, S4, S5, unofficial]. Unlike sharp boolean operators, it uses a blending radius parameter to create a continuous transition between the input distance fields [S1, S2, unofficial]. It is distinct from [SDF Smooth Union](1817-sdf-smooth-union.md) (which blends the exterior of shapes) and [Linear SDF Blend](3004-linear-sdf-blend.md) (which uses linear interpolation), offering a specific curvature profile for the intersection feature [S1, S2, S4, unofficial].

## How it works
**Inputs and Outputs:**
- $a, b$: Scalar signed distance values from two primitive SDFs at a query point.
- $k$ (or $r$): Scalar blending radius or smoothing factor, $k > 0$ [S1, S2, S3, S4, unofficial].
- Output: Scalar blended distance $d_{out}$ [S1, S2, S3, S4, S5, unofficial].

**Data Structures:**
Not given in the sources.

**Numbered Steps and Formulas:**
Two primary formulations are cited in the sources [S1, S2, S3, unofficial]. They are mathematically distinct in their derivation but serve the same semantic purpose [G].

*Formulation 1 (Polynomial Blend):*
Defined in [S1, S2, unofficial], this method uses a quadratic blend based on the difference between distances.
1. Calculate the blend factor $h$:
   $$ h = \text{clamp}\left(0.5 - 0.5 \frac{b - a}{k}, 0.0, 1.0\right) $$ [S1, S2, unofficial]
2. Calculate the result using linear interpolation (`mix`):
   $$ d_{out} = \text{mix}(b, a, h) + k \cdot h \cdot (1.0 - h) $$ [S1, S2, unofficial]
   Where $\text{mix}(x, y, t) = x(1-t) + yt$ [S2, unofficial].

*Formulation 2 (Geometric/Root Blend):*
Defined in [S3, unofficial], this method uses a geometric construction involving square roots.
1. Calculate the maximum distance: $m = \max(a, b)$ [S3, unofficial].
2. Check if both points are within the blending radius of the surface:
   Condition: $-a < k \land -b < k$ [S3, unofficial].
3. If the condition is true:
   $$ d_{out} = \max\left(m, -\left(k - \sqrt{(k+a)^2 + (k+b)^2}\right)\right) $$ [S3, unofficial]
4. Otherwise:
   $$ d_{out} = m $$ [S3, unofficial]

**Parameters:**
- $k$: The smoothing factor or blending radius. It defines the width of the transition region [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
- **Lipschitz Continuity:** The operator preserves Lipschitz continuity ($|\nabla d| \le 1$) only if the gradients of the two input SDFs are orthogonal [S1, S2, unofficial]. If the angle between gradients deviates by more than $30^\circ$ or if surfaces are parallel and close, continuity is lost, causing ray marching artifacts like overshooting or jittering [S1, S2, unofficial].
- **Disagreement:** [S1, S2, unofficial] use a polynomial blend, while [S3, unofficial] uses a geometric root blend. [S4, unofficial] describes a smooth union formula ($min(d1, d2) - h^2/4k$) which is structurally similar but applied to the minimum operator, not intersection. [S5, unofficial] references `smax`, which is a different primitive. Implementers should verify which curvature profile is required [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def smooth_intersection_polynomial(a: float, b: float, k: float) -> float:
    """
    Computes a smooth intersection using the polynomial blend formulation [S1, S2, unofficial].
    """
    if k <= 0:
        return max(a, b)
    diff = b - a
    h = 0.5 - 0.5 * (diff / k)
    if h < 0.0:
        h = 0.0
    elif h > 1.0:
        h = 1.0
    mixed = b * (1.0 - h) + a * h
    return mixed + k * h * (1.0 - h)

def smooth_intersection_geometric(a: float, b: float, k: float) -> float:
    """
    Computes a smooth intersection using the geometric root formulation [S3, unofficial].
    """
    m = max(a, b)
    if (-a < k) and (-b < k):
        term = (k + a)**2 + (k + b)**2
        inner = -(k - term**0.5)
        return max(m, inner)
    return m
```

## Operating characteristics
- **Time Complexity:** $O(1)$ per query [G].
- **Memory Complexity:** $O(1)$ [G].
- **Strengths:** Produces visually smooth, rounded edges for CSG intersections without requiring mesh reconstruction [S1, S2, unofficial].
- **Limitations:** Loss of Lipschitz continuity in non-orthogonal gradient regions can break sphere tracing convergence [S1, S2, unofficial].
- **When to use:** When rounded corners are desired for intersecting primitives in ray marching [G].
- **When not to use:** When sharp boolean edges are required, or when gradient orthogonality cannot be guaranteed (leading to marching artifacts) [S1, S2, unofficial].

## Background
Smooth Intersection operators are part of the broader field of Signed Distance Field (SDF) modeling, popularized by interactive ray marching techniques [G]. The specific polynomial formulation is often associated with community implementations in real-time graphics [S1, S2, unofficial]. The geometric formulation appears in marching cubes contexts [S3, unofficial]. These operators are used in procedural geometry generation, VFX, and real-time rendering to create complex shapes from simple primitives [G].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S3] tdhooper/glsl-marching-cubes, https://github.com/tdhooper/glsl-marching-cubes, fetched 2026-10-26 (unofficial)
[S4] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
[S5] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
