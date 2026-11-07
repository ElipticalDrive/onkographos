# Smooth Intersection

Also called: Round Intersection, S-Intersection, Smooth SDF Operations

## Overview
Smooth Intersection is a Constructive SDF Modeling operator that blends the overlap region of two shapes to produce rounded edges, serving as a smooth approximation of the boolean intersection operation [S1, S2, S3, S4, S5, unofficial]. Unlike sharp boolean operators, it uses a blending radius parameter to create a continuous transition between the input distance fields [S1, S2, unofficial]. It is distinct from [SDF Smooth Union](1817-sdf-smooth-union.md) (which blends the exterior of shapes) and [Linear SDF Blend](3004-linear-sdf-blend.md) (which uses linear interpolation), offering a specific curvature profile for the intersection feature [S1, S2, S4, unofficial].

## How it works
**Inputs and Outputs:**
- $a, b$: Scalar signed distance values from two primitive SDFs at a query point.
- $k$ (or $r$): Scalar blending radius/smoothing factor, $k > 0$ [S1, S2, S3, S4, unofficial].
- Output: Scalar blended distance $d_{out}$ [S1, S2, S3, S4, S5, unofficial].

**Data Structures:**
No complex data structures are required; the operation is a pure function of three scalars.

**Formulas:**
Two primary formulations are cited in the sources. They are mathematically distinct in their derivation but serve the same semantic purpose.

*Formulation 1 (Polynomial Blend):*
Defined in [S1, unofficial] and [S2, unofficial], this method uses a quadratic blend based on the difference between distances.
1. Calculate the difference: $\Delta = b - a$.
2. Calculate the blend factor $h$:
   $$ h = \text{clamp}\left(0.5 - 0.5 \frac{\Delta}{k}, 0.0, 1.0\right) $$
   Alternatively expressed as:
   $$ h = \text{clamp}\left(0.5 \left(1 - \frac{b - a}{k}\right), 0.0, 1.0\right) $$
3. Calculate the result using linear interpolation (`mix`):
   $$ d_{out} = \text{mix}(b, a, h) + k \cdot h \cdot (1.0 - h) $$
   Where $\text{mix}(x, y, t) = x(1-t) + yt$.

*Formulation 2 (Geometric/Root Blend):*
Defined in [S3, unofficial], this method uses a geometric construction involving square roots.
1. Calculate the maximum distance: $m = \max(a, b)$.
2. Check if both points are within the blending radius of the surface (i.e., inside the shapes relative to the blend zone):
   Condition: $-a < k \land -b < k$.
3. If the condition is true:
   $$ d_{out} = \max\left(m, -\left(k - \sqrt{(k+a)^2 + (k+b)^2}\right)\right) $$
4. Otherwise:
   $$ d_{out} = m $$

**Parameters:**
- $k$: The smoothing factor or blending radius. It defines the width of the transition region. It must be significantly smaller than the scale of the objects to prevent geometric distortion [S1, S2, unofficial].

**Pseudocode:**
The following implementation combines the logic from [S1, S2, unofficial] and [S3, unofficial] into a single Python-like function. Note that [S1, S2, unofficial] and [S3, unofficial] provide different algorithms; an implementer must choose one based on desired curvature properties. The code below implements Formulation 1 as it is more commonly cited for "smooth minimum" style operations in this context, but Formulation 2 is also valid.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def smooth_intersection(a: float, b: float, k: float) -> float:
    """
    Computes a smooth intersection of two SDF values a and b with blending radius k.
    Implements Formulation 1 from [S1, S2, unofficial].
    
    Args:
        a: Distance from first primitive.
        b: Distance from second primitive.
        k: Blending radius/smoothing factor. Must be > 0.
        
    Returns:
        Blended distance value.
    """
    if k <= 0:
        # Fallback to sharp intersection if k is invalid
        return max(a, b)
    
    # Formulation 1: Polynomial blend [S1, S2, unofficial]
    # h = clamp(0.5 - 0.5 * (b - a) / k, 0.0, 1.0)
    diff = b - a
    h = 0.5 - 0.5 * (diff / k)
    
    # Clamp h to [0.0, 1.0]
    if h < 0.0:
        h = 0.0
    elif h > 1.0:
        h = 1.0
        
    # mix(b, a, h) = b * (1 - h) + a * h
    mixed = b * (1.0 - h) + a * h
    
    # Result: mix + k * h * (1 - h)
    result = mixed + k * h * (1.0 - h)
    
    return result

def smooth_intersection_geometric(a: float, b: float, k: float) -> float:
    """
    Computes a smooth intersection using geometric root formulation [S3, unofficial].
    
    Args:
        a: Distance from first primitive.
        b: Distance from second primitive.
        k: Blending radius.
        
    Returns:
        Blended distance value.
    """
    m = max(a, b)
    
    # Check if both are within blending radius of surface (inside)
    # Condition: -a < k AND -b < k
    if (-a < k) and (-b < k):
        # f = max(m, -(k - sqrt((k+a)^2 + (k+b)^2)))
        term = (k + a)**2 + (k + b)**2
        root_val = (k + a)**2 + (k + b)**2
        import math
        sqrt_val = math.sqrt(root_val)
        
        inner = -(k - sqrt_val)
        result = max(m, inner)
        return result
    else:
        return m
```

**Edge Cases and Pitfalls:**
- **Lipschitz Continuity:** The operator preserves Lipschitz continuity ($|\nabla d| \le 1$) only if the gradients of the two input SDFs are orthogonal. If the angle between gradients deviates by more than $30^\circ$ or if surfaces are parallel and close, continuity is lost, causing ray marching artifacts like overshooting or jittering [S1, S2, unofficial].
- **Parameter Scale:** If $k$ is too large relative to the object size, the geometry becomes distorted [S1, S2, unofficial].
- **Disagreement:** [S1, S2, unofficial] use a polynomial blend, while [S3, unofficial] uses a geometric root blend. [S4, unofficial] describes a smooth union formula ($min(d1, d2) - h^2/4k$) which is structurally similar but applied to the minimum operator, not intersection. [S5, unofficial] references `smax`, which is a different primitive. Implementers should verify which curvature profile is required.

## Operating characteristics
- **Time Complexity:** $O(1)$ per query. The operation involves basic arithmetic and a square root (in Formulation 2).
- **Memory Complexity:** $O(1)$. No auxiliary storage.
- **Strengths:** Produces visually smooth, rounded edges for CSG intersections without requiring mesh reconstruction.
- **Limitations:** Loss of Lipschitz continuity in non-orthogonal gradient regions can break sphere tracing convergence. The blending radius $k$ must be carefully tuned.
- **When to use:** When rounded corners are desired for intersecting primitives in ray marching.
- **When not to use:** When sharp boolean edges are required, or when gradient orthogonality cannot be guaranteed (leading to marching artifacts).

## Background
Smooth Intersection operators are part of the broader field of Signed Distance Field (SDF) modeling, popularized by interactive ray marching techniques. The specific polynomial formulation is often associated with community implementations in real-time graphics [S1, S2, unofficial]. The geometric formulation appears in marching cubes contexts [S3, unofficial]. These operators are used in procedural geometry generation, VFX, and real-time rendering to create complex shapes from simple primitives.

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S3] tdhooper/glsl-marching-cubes, https://github.com/tdhooper/glsl-marching-cubes, fetched 2026-10-26 (unofficial)
[S4] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
[S5] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
