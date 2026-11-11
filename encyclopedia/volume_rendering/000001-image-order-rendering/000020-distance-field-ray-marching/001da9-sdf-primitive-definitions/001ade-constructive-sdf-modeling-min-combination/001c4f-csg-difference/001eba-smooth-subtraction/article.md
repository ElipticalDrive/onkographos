# Smooth Subtraction
Also called: S-Subtraction

## Overview
Smooth Subtraction provides a differentiable approximation of the Constructive SDF Modeling (CSG) difference operation, blending the subtraction of one shape from another [S1, S2, S3, unofficial]. Unlike the hard boolean difference which creates sharp edges, this method introduces a smooth transition zone controlled by a blending parameter, enabling gradient-based optimization and avoiding artifacts in differentiable rendering pipelines [S2, unofficial].

## How it works
**Inputs and Outputs**
- $d_1$: Signed distance field (SDF) value of the shape to be subtracted (the "hole" or "cutter").
- $d_2$: Signed distance field (SDF) value of the base shape.
- $k$: Blending radius or smoothness parameter, controlling the width of the transition zone [S1, S2, unofficial].
- Output: The smoothed SDF value $d$ representing the result of $d_2 \setminus d_1$ [S3, unofficial].

**Data Structures**
No complex data structures are required; the operation is element-wise on scalar SDF values.

**Algorithm**
The method computes the smooth subtraction using a mixing function based on a clamped interpolation factor $h$ [S2, unofficial].

1. Calculate the interpolation factor $h$:
   $$ h = \text{clamp}\left(0.5 - 0.5 \frac{d_2 + d_1}{k}, 0.0, 1.0\right) $$
   where $\text{clamp}(v, min, max)$ restricts $v$ to the range $[min, max]$ [S1, S2, unofficial].

2. Calculate the mixed distance $d_{mix}$:
   $$ d_{mix} = \text{mix}(d_2, -d_1, h) $$
   where $\text{mix}(a, b, t) = a(1 - t) + b t$ [S2, unofficial].

3. Calculate the final smoothed distance $d$:
   $$ d = d_{mix} + k \cdot h \cdot (1.0 - h) $$
   [S1, S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def smooth_subtraction(d1: float, d2: float, k: float) -> float:
    """
    Computes the smooth subtraction of SDF d1 from d2 with smoothing factor k.
    
    Args:
        d1: SDF of the subtracted shape.
        d2: SDF of the base shape.
        k: Smoothing factor / blending radius.
        
    Returns:
        Smoothed SDF value.
    """
    # Step 1: Calculate interpolation factor h
    # h = clamp(0.5 - 0.5 * (d2 + d1) / k, 0.0, 1.0)
    ratio = (d2 + d1) / k
    h = 0.5 - 0.5 * ratio
    if h < 0.0:
        h = 0.0
    elif h > 1.0:
        h = 1.0
        
    # Step 2: Linear mix of d2 and -d1
    # mix(a, b, t) = a * (1 - t) + b * t
    d_mix = d2 * (1.0 - h) + (-d1) * h
    
    # Step 3: Add smoothing term
    d = d_mix + k * h * (1.0 - h)
    
    return d
```
Note: Some implementations define this directly via a smooth minimum function `smax(-d1, d2, k)` [S3, unofficial]. The formula above is the explicit expansion of that operation as detailed in [S1, S2, unofficial].

**Parameters**
- $k$: The smoothing factor. Typical values depend on the scale of the scene and desired edge sharpness. If $k=0$, the operation degenerates to a hard boolean difference (potentially non-differentiable at the boundary) [S1, S2, unofficial].

**Edge Cases and Pitfalls**
- **Division by Zero**: If $k=0$, the calculation of $h$ involves division by zero. The sources imply $k > 0$ for the smooth approximation [S1, S2, unofficial].
- **Precision**: The term $k \cdot h \cdot (1.0 - h)$ ensures continuity and differentiability. At the boundaries where $h=0$ or $h=1$, the derivative is continuous [S2, unofficial].
- **Degenerate Input**: If $d_1$ and $d_2$ are both very large positive or negative values, the result is dominated by the base shape or the void, respectively, but the smoothing term remains negligible relative to the distance values.

## Operating characteristics
**Time and Memory Complexity**
- Time: $O(1)$ per sample. The operation involves basic arithmetic operations (addition, multiplication, division, clamp).
- Memory: $O(1)$. No additional storage is required beyond the input scalars.

**Strengths and Limitations**
- **Strengths**: Provides a differentiable approximation of CSG operations, essential for gradient-based shape optimization and neural rendering [S2, unofficial]. Avoids sharp discontinuities that can hinder gradient descent.
- **Limitations**: The resulting surface is not exactly at the boolean boundary but is offset by the smoothing radius $k$. This introduces a geometric error proportional to $k$ [S1, unofficial]. Not suitable for applications requiring exact boolean precision.

**When to Use**
- Use when differentiability is required (e.g., in differentiable rendering, shape optimization, or neural implicit representations) [S2, unofficial].
- Use when smooth transitions between shapes are desired for aesthetic or physical reasons.

**When Not to Use**
- Do not use when exact geometric boundaries are required.
- Do not use if computational cost of even basic arithmetic is prohibitive (rare).

## Background
Smooth Subtraction is a technique used in the context of differentiable rendering and implicit surface modeling. It allows for the optimization of shape parameters by providing gradients through boolean operations. The specific formulation using the mixing function and smoothing term is documented in unofficial repositories related to differentiable sphere tracing [S1, S2, unofficial]. The concept of using smooth minimum/maximum functions for CSG operations is well-established in SDF literature, with this specific implementation appearing in PyTorch-based sphere tracer implementations [S1, S2, unofficial]. The Rust implementation in [S3, unofficial] highlights its utility in general SDF math libraries.

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S3] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
