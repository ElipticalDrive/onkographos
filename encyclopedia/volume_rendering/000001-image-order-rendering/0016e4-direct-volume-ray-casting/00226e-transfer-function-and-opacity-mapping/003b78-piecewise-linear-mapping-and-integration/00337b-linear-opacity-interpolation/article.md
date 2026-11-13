# Linear Opacity Interpolation

Also called: pokrytie

## Overview
Linear Opacity Interpolation maps scalar volume data values to opacity using linear interpolation between two threshold values [S1, unofficial]. It sets itself apart from its sibling [Piecewise Function Scalar Opacity Transfer](12435-piecewise-function-scalar-opacity-transfer.md) by restricting the mapping to a simple linear segment between two thresholds, whereas the sibling supports arbitrary piecewise definitions and non-linear combinations [S2, S1, unofficial].

## How it works
**Inputs and Outputs**:
- Input: A scalar volume value $i$ (typically `UINT16`).
- Parameters: Lower threshold $oa$ (`UINT16`), upper threshold $ob$ (`UINT16`), opacity at lower threshold $foa$ (`float`), opacity at upper threshold $fob$ (`float`).
- Output: Opacity $\alpha \in [0, 1]$ (`float`).

**Data Structures**:
No complex data structures are required; the method operates on scalar values and four parameter constants.

**Steps**:
1. Evaluate the input scalar $i$ against the thresholds $oa$ and $ob$.
2. If $i$ is outside the range $[oa, ob]$, the opacity is zero.
3. If $i$ is within the range, compute $\alpha$ using linear interpolation between $foa$ and $fob$.

**Formula**:
The opacity $\alpha$ is calculated as:
$$
\alpha = \begin{cases} 
0.0 & i < oa \lor i > ob \\
foa + (fob - foa) \frac{i - oa}{ob - oa} & \text{otherwise}
\end{cases}
$$

**Parameters**:
- $oa, ob$: Lower and upper thresholds, typically `UINT16` [S1, unofficial].
- $foa, fob$: Opacity values at the thresholds, typically `float` in the range $[0.0, 1.0]$ [S1, unofficial].

**Edge Cases and Pitfalls**:
- If $oa = ob$, the denominator becomes zero. The source code implies a standard linear interpolation function `linInt` is used, which may require a guard for this degenerate case [G].
- Input values outside the $[oa, ob]$ range result in zero opacity, effectively making those voxels transparent [S1, unofficial].

**Pseudocode**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_opacity_interpolation(i: int, oa: int, ob: int, foa: float, fob: float) -> float:
    """
    Computes opacity alpha for scalar value i using linear interpolation
    between thresholds oa and ob with opacities foa and fob.
    """
    if i < oa or i > ob:
        return 0.0
    
    # Linear interpolation
    # Guard against division by zero if oa == ob [G]
    if oa == ob:
        return foa
    
    alpha = foa + (fob - foa) * (i - oa) / (ob - oa)
    return alpha
```

## Subtopics

### [Bézier Curve Opacity Mapping](8485-b-zier-curve-opacity-mapping.md)
Maps scalar volume data values to opacity using a user-defined Bézier curve. It sets itself apart by replacing piecewise linear segments with smooth polynomial curves, allowing non-linear opacity control.

### [Piecewise Function Scalar Opacity Transfer](12435-piecewise-function-scalar-opacity-transfer.md)
Defines a mapping using a piecewise function to suppress dominant flat surfaces. It sets itself apart by supporting arbitrary piecewise definitions and non-linear combinations, unlike the simple two-threshold linear interpolation.

## Operating characteristics
- **Time Complexity**: $O(1)$ per sample, as it involves a constant number of comparisons and arithmetic operations.
- **Memory Complexity**: $O(1)$, requiring storage for only four parameters ($oa, ob, foa, fob$).
- **Strengths**: Simple to implement and fast to evaluate. Effective for isolating specific density ranges.
- **Limitations**: Cannot model complex opacity transitions or non-linear features within the selected range. Limited to a single linear segment.
- **When to use**: When a specific range of data values needs to be highlighted with a smooth opacity transition, and the rest of the data should be transparent.
- **When not to use**: When complex opacity variations or non-linear mappings are required within the data range.

## Background
The method is described in the context of parallel volume rendering implementations, specifically in the `besnik/parallel-volume-rendering` repository [S1, unofficial]. It is a fundamental component of transfer function mapping in direct volume ray casting.

## Sources
[S1] besnik/parallel-volume-rendering, https://github.com/besnik/parallel-volume-rendering, fetched 2026-10-27 (unofficial)
