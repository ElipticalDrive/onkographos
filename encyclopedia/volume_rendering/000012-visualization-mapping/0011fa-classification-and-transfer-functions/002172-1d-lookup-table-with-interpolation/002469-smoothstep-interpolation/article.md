# Smoothstep Interpolation
Also called: Cubic Hermite Spline Interpolation (Smoothstep), Perlin Smoothstep, smoothstep

## Overview
Smoothstep interpolation provides a smooth, continuous transition between two values by applying a cubic Hermite spline to a normalized input. It is used in volume rendering classification to map scalar values to transfer function properties (such as opacity or color) with $C^1$ continuity, avoiding the discontinuities inherent in linear interpolation. It differs from [Linear Look-up Table Classification](2636-linear-look-up-table-classification.md) by eliminating sharp corners in the transfer function, and from [Cubic Spline Transfer Function](1811-cubic-spline-transfer-function.md) by being a fixed, parameter-free kernel rather than a piecewise function defined by control points.

## How it works
**Inputs and Outputs:**
- Input: A scalar value $x$, a lower bound $edge0$, and an upper bound $edge1$.
- Output: A scalar value $y \in [0, 1]$ representing the interpolated result.

**Data Structures:**
No complex data structures are required. The method operates on standard floating-point scalars.

**Steps:**
1. Normalize the input $x$ to the range $[0, 1]$ relative to the interval $[edge0, edge1]$.
2. Clamp the normalized value to $[0, 1]$ to handle inputs outside the defined range.
3. Apply the cubic polynomial $t^2(3 - 2t)$ to the clamped value.

**Formula:**
Let $t$ be the normalized input:
$$ t = \text{clamp}\left(\frac{x - edge0}{edge1 - edge0}, 0.0, 1.0\right) $$
The output $y$ is:
$$ y = t^2 (3.0 - 2.0 t) $$
where $\text{clamp}(v, min, max)$ returns $min$ if $v < min$, $max$ if $v > max$, and $v$ otherwise [S1, S2, unofficial].

**Parameters:**
- $edge0$: The lower bound of the transition interval.
- $edge1$: The upper bound of the transition interval.
- Typical values: These depend on the specific scalar field (e.g., density or temperature) and the desired visual effect. No universal default exists; they are tuned per transfer function segment.

**Edge Cases and Pitfalls:**
- If $edge0 = edge1$, division by zero occurs. Implementers must guard against this, typically by returning 0.0 or 1.0 based on $x$ relative to $edge0$ [G].
- Inputs $x < edge0$ result in $y = 0.0$.
- Inputs $x > edge1$ result in $y = 1.0$.
- Precision issues may arise if $x$ is very close to $edge0$ or $edge1$ due to floating-point rounding, but the clamping step ensures stability.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def smoothstep(edge0, edge1, x):
    # Guard against division by zero
    if edge0 == edge1:
        return 0.0 if x < edge0 else 1.0
    
    # Normalize and clamp
    t = (x - edge0) / (edge1 - edge0)
    t = max(0.0, min(1.0, t))
    
    # Apply cubic Hermite spline
    return t * t * (3.0 - 2.0 * t)
```

## Subtopics

### Piecewise Smoothstep
Using multiple smoothstep intervals to create complex transfer functions. Choose this when fine-grained control over multiple density ranges is needed. See [Piecewise Linear Interpolation (Lookup Table)](2643-piecewise-linear-interpolation-lookup-table.md) for the linear alternative.

### Smoothstep in Shading
Applying smoothstep for soft shadows or fog. Choose this for visual effects requiring smooth falloff. See [Shading Lookup Table Computation](3589-shading-lookup-table-computation.md).

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample. The calculation involves a few arithmetic operations and is extremely fast.
- **Memory Complexity:** $O(1)$. No additional memory is required beyond the input and output variables.
- **Performance:** On modern CPUs and GPUs, this operation is negligible in cost compared to texture lookups or ray marching steps.
- **Strengths:** Simple to implement, computationally cheap, provides smooth derivatives ($C^1$ continuity), and avoids the ringing artifacts of higher-order splines without control points.
- **Limitations:** Cannot represent sharp transitions or discontinuities. The fixed shape of the curve limits flexibility compared to cubic splines with adjustable control points.
- **When to use:** Ideal for general-purpose transfer functions where smoothness is desired but complex curve fitting is unnecessary.
- **When not to use:** When sharp edges or specific curvature control is required, use [Cubic Spline Transfer Function](1811-cubic-spline-transfer-function.md).

## Background
The smoothstep function was popularized by Ken Perlin in the context of procedural noise generation, where it is used to interpolate noise values smoothly. It is a standard component in graphics programming libraries and is widely used in volume rendering transfer functions to create smooth opacity and color ramps. The function is a cubic Hermite spline that interpolates between 0 and 1 with zero derivatives at the endpoints.

## Sources
[S1] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
[S2] mingtu149411/Noise-Cloud, https://github.com/mingtu149411/Noise-Cloud, fetched 2026-10-27 (unofficial)
