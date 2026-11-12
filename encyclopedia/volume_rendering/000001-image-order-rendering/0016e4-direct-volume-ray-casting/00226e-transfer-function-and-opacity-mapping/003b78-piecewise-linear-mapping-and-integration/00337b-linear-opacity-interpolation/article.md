# Linear Opacity Interpolation

Also called: pokrytie

## Overview
Linear Opacity Interpolation maps a scalar volume data value to an opacity scalar using linear interpolation between two threshold values [S1, unofficial]. It serves as the foundational opacity mapping primitive in piecewise linear transfer functions, providing a simple, computationally inexpensive way to define transparency ranges [S1, unofficial]. It differs from [Piecewise Density Approximation for Reparameterized Sampling](1139-piecewise-density-approximation-for-reparameterized-sampling.md) by operating directly on opacity rather than density, and from [Piecewise Linear Opacity Mapping with Gradient Scaling](1335-piecewise-linear-opacity-mapping-with-gradient-scaling.md) by ignoring gradient information, making it suitable only for scalar-intensity-based segmentation [S1, S2, unofficial].

## How it works
**Inputs:**
- $i$: The interpolated scalar volume value at the current sample point.
- $oa$: Lower threshold value (unsigned integer, typically `UINT16`).
- $ob$: Upper threshold value (unsigned integer, typically `UINT16`).
- $foa$: Opacity value at $oa$ (float, typically in $[0.0, 1.0]$).
- $fob$: Opacity value at $ob$ (float, typically in $[0.0, 1.0]$).

**Outputs:**
- $\alpha$: The resulting opacity scalar.

**Data Structures:**
No complex data structures are required. The parameters are stored as simple scalar variables.

**Steps:**
1. Check if the input value $i$ lies outside the range $[oa, ob]$.
2. If $i < oa$ or $i > ob$, the opacity is zero.
3. If $i$ is within the range, compute the opacity using linear interpolation between $(oa, foa)$ and $(ob, fob)$.

**Formula:**
The opacity $\alpha$ is defined as:
$$
\alpha = \begin{cases} 
0.0 & i < oa \lor i > ob \\
foa + (fob - foa) \frac{i - oa}{ob - oa} & \text{otherwise}
\end{cases}
$$

**Parameters:**
- $oa, ob$: Define the data value range where opacity is non-zero. Typical values depend on the volume's data type (e.g., 0–65535 for 16-bit unsigned integers).
- $foa, fob$: Define the opacity intensity at the boundaries. Typical values are $0.0$ (transparent) and $1.0$ (opaque), but intermediate values allow for semi-transparent blending.

**Edge Cases and Pitfalls:**
- **Division by Zero:** If $oa = ob$, the denominator $(ob - oa)$ becomes zero. The sources do not explicitly define behavior for this degenerate case, but logically, if the range has zero width, the opacity is undefined or should default to $foa$ if $i=oa$, or $0.0$ otherwise [G].
- **Precision:** Input $i$ is typically a floating-point value resulting from trilinear interpolation of volume data, while thresholds $oa, ob$ are often integers. Care must be taken to ensure consistent data types during comparison and arithmetic [S1, unofficial].
- **Discretization Correction:** As per the parent method [Piecewise Linear Mapping and Integration](9000-piecewise-linear-mapping-and-integration.md), the resulting $\alpha$ must be corrected for the step size $\Delta t$ before compositing to ensure physical consistency, using $\alpha' = 1 - (1 - \alpha)^{\Delta t}$ [S11, S32, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_opacity_interpolation(i: float, oa: int, ob: int, foa: float, fob: float) -> float:
    """
    Computes opacity alpha for a scalar value i using linear interpolation
    between thresholds oa and ob.
    
    Args:
        i: Scalar volume value at sample point.
        oa: Lower threshold (UINT16).
        ob: Upper threshold (UINT16).
        foa: Opacity at oa (float).
        fob: Opacity at ob (float).
        
    Returns:
        Opacity alpha (float).
    """
    # Check if input is outside the defined range
    if i < oa or i > ob:
        return 0.0
    
    # Handle degenerate case where range width is zero
    if ob == oa:
        # If i equals oa (and thus ob), return foa; otherwise 0.0
        # Note: Since i >= oa and i <= ob, if oa==ob, then i must equal oa.
        return foa
    
    # Linear interpolation
    # alpha = foa + (fob - foa) * (i - oa) / (ob - oa)
    alpha = foa + (fob - foa) * (i - oa) / (ob - oa)
    
    return alpha
```

## Subtopics

### [Bézier Curve Opacity Mapping](8485-b-zier-curve-opacity-mapping.md)
Replaces linear segments with a smooth Bézier curve for complex opacity transitions. Choose when smooth, non-linear opacity gradients are required.

### [Piecewise Function Scalar Opacity Transfer](12435-piecewise-function-scalar-opacity-transfer.md)
Extends linear interpolation to multiple control points for non-monotonic profiles. Choose when multiple distinct tissue types or features need separate opacity ranges.

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample. The operation involves a few comparisons and basic arithmetic operations.
- **Memory Complexity:** $O(1)$. Only five scalar parameters are required.
- **Strengths:** Extremely fast, simple to implement, and easy to tune for binary segmentation tasks (transparent vs. opaque).
- **Limitations:** Cannot represent complex, non-linear opacity transitions or multiple distinct opacity ranges without extending to piecewise functions. Ignores gradient information, which may lead to poor rendering of surfaces with similar intensities but different geometries [S2, unofficial].
- **When to Use:** Simple volume visualization where only two opacity levels are needed, or as a building block for more complex piecewise transfer functions.
- **When Not to Use:** When smooth opacity gradients are required ([Bézier Curve Opacity Mapping](8485-b-zier-curve-opacity-mapping.md)) or when multiple distinct features need independent opacity control ([Piecewise Function Scalar Opacity Transfer](12435-piecewise-function-scalar-opacity-transfer.md)).

## Background
The method is described in the context of parallel volume rendering implementations, specifically in the `besnik/parallel-volume-rendering` repository [S1, unofficial]. It is a standard technique in direct volume ray casting, forming the basis for piecewise linear transfer functions. The term "pokrytie" is used as an alias in the source material [S1, unofficial].

## Sources
[S1] besnik/parallel-volume-rendering, https://github.com/besnik/parallel-volume-rendering, fetched 2026-10-27 (unofficial)
