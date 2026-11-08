# Linear Opacity Interpolation

Also called: pokrytie

## Overview
Linear Opacity Interpolation maps scalar volume data values to opacity using linear interpolation between two threshold values [S1, unofficial]. It serves as the simplest form of piecewise linear mapping, providing a direct, computationally inexpensive way to define opacity transitions without the complexity of multi-segment functions or gradient scaling [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: scalar volume data value $i$ (typically unsigned integer, e.g., `UINT16`).
- Parameters: lower threshold $o_a$ (`UINT16`), upper threshold $o_b$ (`UINT16`), opacity at lower threshold $f_o_a$ (`float`), opacity at upper threshold $f_o_b$ (`float`).
- Output: resulting opacity $\alpha$ (`float`).

**Data Structures:**
No complex data structures are required. The method relies on four scalar parameters.

**Steps:**
1. Check if the input value $i$ is outside the range $[o_a, o_b]$.
2. If $i < o_a$ or $i > o_b$, return $0.0$.
3. Otherwise, compute the linear interpolation between $f_o_a$ and $f_o_b$ based on the position of $i$ within $[o_a, o_b]$.

**Formula:**
$$
\alpha = \begin{cases} 
0.0 & i < o_a \lor i > o_b \\
f_o_a + (f_o_b - f_o_a) \frac{i - o_a}{o_b - o_a} & \text{otherwise}
\end{cases}
$$

**Parameters:**
- $o_a, o_b$: Lower and upper thresholds. Type: `UINT16` [S1, unofficial].
- $f_o_a, f_o_b$: Opacity values at the thresholds. Type: `float`. Typical range: $0.0$ to $1.0$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** If $o_a = o_b$, the denominator in the interpolation formula becomes zero. The source code implies a check or assumption that $o_a \neq o_b$ for valid interpolation [S1, unofficial]. In implementation, this case should be handled explicitly (e.g., returning $f_o_a$ or $0.0$).
- **Input Range:** Values outside $[o_a, o_b]$ are clamped to $0.0$ opacity [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_opacity_interpolation(i: int, oa: int, ob: int, foa: float, fob: float) -> float:
    """
    Computes opacity alpha for input value i using linear interpolation between thresholds oa and ob.
    
    Args:
        i: Scalar volume data value (UINT16 compatible).
        oa: Lower threshold (UINT16).
        ob: Upper threshold (UINT16).
        foa: Opacity at oa (float).
        fob: Opacity at ob (float).
        
    Returns:
        Opacity alpha (float).
    """
    # Check if input is outside the threshold range
    if i < oa or i > ob:
        return 0.0
    
    # Handle degenerate case where thresholds are equal
    if oa == ob:
        # If thresholds are equal, the range has zero width.
        # Typically, this might return the opacity at that point if i == oa,
        # or 0.0 if strictly outside. Based on the formula, division by zero occurs.
        # A safe default is to return foa if i == oa, else 0.0.
        if i == oa:
            return foa
        else:
            return 0.0
            
    # Linear interpolation
    # alpha = foa + (fob - foa) * (i - oa) / (ob - oa)
    alpha = foa + (fob - foa) * (i - oa) / (ob - oa)
    
    return alpha
```

## Subtopics

### [Bézier Curve Opacity Mapping](8485-b-zier-curve-opacity-mapping.md)
Maps scalar volume data values to opacity using a user-defined Bézier curve, allowing non-linear opacity control across a selected data range.

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel/sample. The operation involves a few comparisons and basic arithmetic.
- **Memory Complexity:** $O(1)$ for parameters. No additional memory structures are needed.
- **Strengths:** Extremely fast, simple to implement, and easy to understand. Suitable for real-time applications where computational resources are limited.
- **Limitations:** Limited flexibility in shaping the opacity function. Cannot model complex opacity transitions or multiple distinct material ranges within a single linear segment.
- **When to use:** When a simple, direct opacity transition is sufficient, or as a building block for more complex piecewise functions.
- **When not to use:** When complex opacity shaping or multiple material distinctions are required within the same data range.

## Background
The method is described in the context of parallel volume rendering implementations, specifically in the `besnik/parallel-volume-rendering` repository [S1, unofficial]. It is a fundamental technique in direct volume ray casting, forming the basis for transfer function mapping in many volume rendering pipelines. The term "pokrytie" is noted as an alias, likely derived from Slavic languages meaning "coverage" or "opacity" [S1, unofficial].

## Sources
[S1] besnik/parallel-volume-rendering, https://github.com/besnik/parallel-volume-rendering, fetched 2026-10-27 (unofficial)
