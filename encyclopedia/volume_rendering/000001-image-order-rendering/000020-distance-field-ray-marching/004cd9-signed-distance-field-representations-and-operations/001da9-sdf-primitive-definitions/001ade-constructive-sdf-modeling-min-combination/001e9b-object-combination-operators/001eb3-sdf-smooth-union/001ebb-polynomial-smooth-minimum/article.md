# Polynomial Smooth Minimum

Also called: Blending SDF Geometry, Quilez smin, Quilez smooth min, SDF Smin, Smooth Minimum, Smooth SDF Blending

## Overview

Polynomial Smooth Minimum is a differentiable blending operator for two Signed Distance Function (SDF) values, creating a smooth union of shapes without the sharp edges inherent in the standard boolean `min` operation [S1, S2, S3, S4, S6, S8, unofficial]. It interpolates the distance values using a polynomial function controlled by a smoothing radius parameter $k$, allowing primitives to merge organically, similar to metaballs [S4, S7, unofficial]. It sets itself apart from exponential smooth minimum by using polynomial interpolation rather than logarithmic/exponential functions [S9, S10, unofficial].

## How it works

**Inputs and Outputs:**
- Input: Two scalar distance values $a$ and $b$ (the SDF values of two primitives at a point), and a scalar smoothing radius $k \ge 0$ [S5, unofficial].
- Output: A single scalar distance value $d_{smooth}$ representing the blended SDF [S1, unofficial].

**Data Structures:**
- No complex data structures are required; the operation is element-wise on scalar floats [G].

**Parameters:**
- $k$: The blending radius or smoothness factor. It controls the width of the transition zone between the two shapes [S4, S5, unofficial].
  - Typical values: 0.2 or 0.3 [S5, S8, unofficial].
  - Note: [S10, unofficial] describes an exponential variant with $k=32$, but the polynomial variants cited in [S1, S2, S4, S5, S6, S8, unofficial] use $k$ as a direct radius (e.g., 0.2). We adhere to the polynomial definitions where $k$ is a small positive radius [S1, S5, unofficial].

**Formulas:**
There are two primary polynomial formulations found in the sources. They are mathematically distinct approximations [S1, S5, unofficial].

1.  **Quilez's Quadratic Smooth Minimum (Common Implementation):**
    This is the most frequently cited "Quilez smin" in the provided sources [S1, S2, S4, S6, S8, unofficial].
    
    Let $h$ be a blending weight:
    $$ h = \text{clamp}\left(\frac{b - a + k}{2k}, 0, 1\right) $$
    Alternatively expressed as:
    $$ h = \text{clamp}\left(0.5 + 0.5 \cdot \frac{b - a}{k}, 0.0, 1.0\right) $$
    
    The result is:
    $$ d_{smooth} = a \cdot h + b \cdot (1 - h) - k \cdot h \cdot (1 - h) $$
    
    Another equivalent form found in [S5, unofficial] uses a different $h$ definition based on the absolute difference:
    $$ h = \frac{\max(k - |a - b|, 0)}{k} $$
    $$ d_{smooth} = \min(a, b) - h^2 \cdot k \cdot 0.25 $$
    
    *Note: The sources [S1, S2, S4, S6, S8, unofficial] use the first formulation (linear mix minus quadratic correction). [S5, unofficial] uses the second formulation (min minus quadratic correction). Both are cited as "Polynomial Smooth Minimum" or "Quilez smin". The first formulation is generally preferred for its symmetry and ease of derivation from linear interpolation [S1, S4, unofficial].*

2.  **Power Mean Approximation (Alternative Polynomial):**
    [S9, unofficial] cites a blending function:
    $$ d_{smooth} = \left( \frac{a^k b^k}{a^k + b^k} \right)^{1/k} $$
    This is often used for $k > 0$. However, the term "Polynomial Smooth Minimum" in the context of "Quilez smin" primarily refers to the quadratic approximation above. The power mean is computationally more expensive due to powers and roots [S9, unofficial].

**Numbered Steps (Quilez Quadratic Form):**
1.  Check if $k$ is effectively zero. If $k \le \epsilon$ (e.g., $10^{-6}$), return $\min(a, b)$ to avoid division by zero and revert to hard union [S5, unofficial].
2.  Calculate the intermediate blend factor $h$:
    $$ h = \text{clamp}\left(0.5 + 0.5 \cdot \frac{b - a}{k}, 0.0, 1.0\right) $$
3.  Calculate the linear interpolation between $a$ and $b$ using $h$. Note that `mix(b, a, h)` is $b + h(a - b) = a \cdot h + b \cdot (1 - h)$ [S4, unofficial].
4.  Subtract the quadratic correction term $k \cdot h \cdot (1 - h)$ from the linear interpolation.
5.  Return the result.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def polynomial_smooth_minimum(a: float, b: float, k: float) -> float:
    """
    Computes the polynomial smooth minimum of two SDF values a and b
    using the quadratic approximation (Quilez smin).
    
    Args:
        a: Distance value of the first primitive.
        b: Distance value of the second primitive.
        k: Smoothing radius. Must be >= 0.
        
    Returns:
        Blended distance value.
    """
    # Guard against division by zero and revert to hard min if k is negligible
    if k <= 1e-6:
        return min(a, b)
    
    # Calculate blend factor h
    # h = clamp(0.5 + 0.5 * (b - a) / k, 0.0, 1.0)
    h = 0.5 + 0.5 * (b - a) / k
    if h < 0.0:
        h = 0.0
    elif h > 1.0:
        h = 1.0
        
    # Linear interpolation: mix(b, a, h) = b + h * (a - b)
    # Which is equivalent to: a * h + b * (1 - h)
    linear_mix = a * h + b * (1.0 - h)
    
    # Subtract quadratic correction term
    result = linear_mix - k * h * (1.0 - h)
    
    return result
```

**Edge Cases and Pitfalls:**
- **$k=0$**: Division by zero in the calculation of $h$. Must be handled by falling back to `min(a, b)` [S5, unofficial].
- **Precision**: The term $k \cdot h \cdot (1 - h)$ is always non-negative. If $a$ and $b$ are very large, precision loss might occur, but typically SDFs are normalized or small [G].
- **Symmetry**: The formula is symmetric with respect to $a$ and $b$ if $h$ is calculated correctly. $S(a, b, k) = S(b, a, k)$ [G].
- **Continuity**: The function is differentiable, avoiding the sharp discontinuities of standard boolean operations [S3, S6, S8, unofficial].

## Subtopics

### Polynomial Smooth Maximum
Polynomial Smooth Maximum is the dual operator for smooth intersection of SDFs [S3, unofficial]. It is implemented by negating the inputs, applying smooth minimum, and negating the result: $\text{smax}(a, b, k) = -\text{smin}(-a, -b, k)$ [S3, unofficial].
Using the quadratic formulation from [S5, unofficial]:
$$ h = \frac{\max(k - |a - b|, 0)}{k} $$
$$ \text{smax}(a, b, k) = \max(a, b) + h^2 \cdot k \cdot 0.25 $$
This creates a smooth "merge" where the surfaces bulge outward at the intersection, effectively rounding the corner of the intersection [S5, unofficial]. It is used for smooth intersections or to create "blobby" unions where the material expands [S5, unofficial].

## Operating characteristics

- **Time Complexity**: $O(1)$. The operation involves a constant number of arithmetic operations (addition, subtraction, multiplication, division, clamp) [G].
- **Memory Complexity**: $O(1)$. No additional memory is allocated [G].
- **Strengths**:
  - Provides smooth, organic transitions between shapes [S4, S7, unofficial].
  - Differentiable, allowing for gradient-based optimization or smooth normals [S3, unofficial].
- **Limitations**:
  - The resulting distance field is an approximation rather than a true Euclidean distance field [G].
- **When to use**: When organic, metaball-like merging of primitives is desired [S4, S7, unofficial].

## Background

The Polynomial Smooth Minimum, specifically the quadratic approximation known as "Quilez smin", was popularized by Íñigo Quilez in his article "Blending SDF Geometry" [S6, S8, unofficial]. It is a standard technique in ray marching and constructive solid geometry (CSG) for SDFs [S1, S2, unofficial]. It serves as a practical alternative to the exponential smooth minimum (which uses $-\frac{\ln(e^{-ka} + e^{-kb})}{k}$) [S10, unofficial], offering a balance between smoothness and computational efficiency [G]. The method is used to create complex, organic shapes from simple primitives [S4, S7, unofficial].

## Sources

[S1] haydenparker3/3D-Sphere-Tracing, https://github.com/haydenparker3/3D-Sphere-Tracing, fetched 2026-10-26 (unofficial)
[S2] ollituohenmaa/raymarching-pathtracer, https://github.com/ollituohenmaa/raymarching-pathtracer, fetched 2026-10-26 (unofficial)
[S3] arguskern/lumina-graphics-lab, https://github.com/arguskern/lumina-graphics-lab, fetched 2026-10-26 (unofficial)
[S4] MohammadHossinzehi/sdf-raymarcher, https://github.com/MohammadHossinzehi/sdf-raymarcher, fetched 2026-10-26 (unofficial)
[S5] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
[S6] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1920/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S7] Sample - Ray Marching and Signed Distance Functions, https://github.com/ComputeNode/cyfra/pull/13, fetched 2026-10-27 (unofficial)
[S8] Ray Marching with Signed Distance Fields - Scribd, https://www.scribd.com/document/831412915/1-Ray-Marching-and-Signed-Distance-Fields, fetched 2026-10-27 (unofficial)
[S9] GPU Ray Marching Techniques, https://www.scribd.com/document/814069988/5-GPU-Ray-Marching, fetched 2026-10-28 (unofficial)
[S10] Volumetric Rendering: Signed Distance Functions - Alan Zucconi, https://www.alanzucconi.com/2016/07/01/signed-distance-functions/, fetched 2026-10-28 (unofficial)
