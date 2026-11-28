# Smooth Subtraction
Also called: S-Subtraction

## Overview
Smooth Subtraction provides a smooth approximation of the boolean subtraction operation (A minus B) for two signed distance fields, controlled by a blending parameter $k$ [S2, unofficial]. It blends the subtraction of one shape from another to approximate the CSG difference [S1, unofficial].

## How it works
**Inputs and Outputs**
- $d_1$: Signed distance field (SDF) value of the shape to be subtracted [S1, S2, unofficial].
- $d_2$: Signed distance field (SDF) value of the base shape [S1, S2, unofficial].
- $k$: The smoothing factor, also referred to as the blending radius or smoothness parameter [S1, S2, unofficial].
- Output: The smoothed SDF value $d$ representing the result of the subtraction [S1, S2, unofficial].

**Data Structures**
Not given in the sources. [G]

**Algorithm**
The method computes the smooth subtraction using a mixing function based on a clamped interpolation factor $h$ [S1, S2, unofficial].

1. Calculate the interpolation factor $h$:
   $$ h = \text{clamp}\left(0.5 - 0.5 \frac{d_2 + d_1}{k}, 0.0, 1.0\right) $$
   where $\text{clamp}(v, \text{min}, \text{max})$ restricts $v$ to the range $[\text{min}, \text{max}]$ [S1, S2, unofficial].

2. Calculate the mixed distance $d_{\text{mix}}$:
   $$ d_{\text{mix}} = \text{mix}(d_2, -d_1, h) $$
   where $\text{mix}(a, b, t) = a \cdot (1 - t) + b \cdot t$ [S2, unofficial].

3. Calculate the final smoothed distance $d$:
   $$ d = d_{\text{mix}} + k \cdot h \cdot (1.0 - h) $$
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
    ratio = (d2 + d1) / k
    h = 0.5 - 0.5 * ratio
    if h < 0.0:
        h = 0.0
    elif h > 1.0:
        h = 1.0
        
    # Step 2: Linear mix of d2 and -d1
    d_mix = d2 * (1.0 - h) + (-d1) * h
    
    # Step 3: Add smoothing term
    d = d_mix + k * h * (1.0 - h)
    
    return d
```
Some implementations define this directly via a smooth minimum function `smax(-d1, d2, k)` [S3, unofficial].

**Parameters**
- $k$: The smoothing factor. Typical values: Not given in the sources. [G]

**Edge Cases and Pitfalls**
- **Division by Zero**: If $k=0$, the calculation of $h$ involves division by zero. A guard value or minimum threshold for $k$ is required in practice [G].
- **Precision and Boundaries**: Not given in the sources. [G]
- **Degenerate Input**: Not given in the sources. [G]

## Operating characteristics
**Time and Memory Complexity**
Not given in the sources. [G]

**Strengths and Limitations**
Not given in the sources. [G]

**When to Use**
Not given in the sources. [G]

**When Not to Use**
Not given in the sources. [G]

## Background
Smooth Subtraction is a technique used in the context of implicit surface modeling and CSG operations [S1, S2, unofficial]. The specific formulation using the mixing function and smoothing term is documented in [S1, S2, unofficial]. A direct Rust implementation is provided in [S3, unofficial].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S3] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
