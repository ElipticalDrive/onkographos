# HSV Transfer Function Mapping
Also called: HSV Color Mapping
## Overview
HSV Transfer Function Mapping addresses the difficulty of simultaneously mapping data values to both color and brightness in volume rendering by decoupling these aspects. It solves the problem of perceptual uniformity in high-contrast environments by using a color space that separates intensity (luminance) from chromaticity, allowing independent transfer functions for each [S1, unofficial]. Unlike RGB-based approaches where color and intensity are entangled, this method allows the user to specify separate transfer functions for chrominance channels while computing luminance using established monochromatic algorithms [S1, unofficial]. It builds on [Color-Based Transfer Function](6581-color-based-transfer-function.md).

## How it works
**Inputs and Outputs:**
- Input: A scalar data value $d$ (e.g., voxel intensity) and user-defined transfer function parameters in HSV space.
- Output: An RGBA color value $(R, G, B, \alpha)$ for the voxel.

**Data Structures:**
- HSV Tuple: $(H, S, V)$ where $H \in [0, 1)$ (or $[0, 360)$ degrees), $S \in [0, 1]$, $V \in [0, 1]$.
- RGB Tuple: $(R, G, B)$ where each component $\in [0, 1]$.
- Opacity $\alpha$: Mapped separately or derived from $V$ depending on implementation.

**Steps:**
1. **Map Data to HSV:** Define three independent transfer functions $T_H(d)$, $T_S(d)$, and $T_V(d)$ that map the input data value $d$ to Hue, Saturation, and Value respectively. This decoupling allows intuitive control: $H$ controls color identity, $S$ controls color purity, and $V$ controls brightness/intensity [S1, unofficial].
2. **Convert HSV to RGB:** Apply a standard conversion algorithm to transform the $(H, S, V)$ tuple into an $(R, G, B)$ triple suitable for rendering [S1, unofficial].
3. **Composite:** Combine the resulting RGB color with opacity $\alpha$ (which may be mapped via $T_V$ or a separate function) to produce the final fragment color.

**Formulas:**
The conversion from HSV to RGB is performed using standard piecewise linear equations. Let $H, S, V$ be normalized to $[0, 1]$.
Let $C = V \times S$ (chroma).
Let $H' = H \times 6$.
Let $X = C \times (1 - |H' \mod 2 - 1|)$.
Let $m = V - C$.

Depending on the integer part of $H'$:
- If $0 \le H' < 1$: $(R', G', B') = (C, X, 0)$
- If $1 \le H' < 2$: $(R', G', B') = (X, C, 0)$
- If $2 \le H' < 3$: $(R', G', B') = (0, C, X)$
- If $3 \le H' < 4$: $(R', G', B') = (0, X, C)$
- If $4 \le H' < 5$: $(R', G', B') = (X, 0, C)$
- If $5 \le H' < 6$: $(R', G', B') = (C, 0, X)$

Final RGB:
$R = R' + m$
$G = G' + m$
$B = B' + m$

**Parameters:**
- Typical values for $H, S, V$ are normalized to $[0, 1]$.
- Hue is often represented in degrees $[0, 360)$ in user interfaces, requiring conversion to $[0, 1)$ for calculation.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point precision in HSV-to-RGB conversion can cause minor color banding if not handled carefully, though standard algorithms are robust.
- **Boundaries:** Hue is cyclic; $H=0$ and $H=1$ (or $360^\circ$) represent the same color (Red). Discontinuities may occur if transfer functions cross this boundary without wrapping.
- **Degenerate Input:** If $S=0$, the color is grayscale regardless of $H$. If $V=0$, the color is black.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hsv_to_rgb(h, s, v):
    """
    Converts HSV to RGB.
    h, s, v are floats in [0, 1].
    Returns r, g, b floats in [0, 1].
    """
    if s == 0:
        return v, v, v
    
    c = v * s
    h_prime = h * 6
    x = c * (1 - abs((h_prime % 2) - 1))
    m = v - c
    
    if h_prime < 1:
        r_prime, g_prime, b_prime = c, x, 0
    elif h_prime < 2:
        r_prime, g_prime, b_prime = x, c, 0
    elif h_prime < 3:
        r_prime, g_prime, b_prime = 0, c, x
    elif h_prime < 4:
        r_prime, g_prime, b_prime = 0, x, c
    elif h_prime < 5:
        r_prime, g_prime, b_prime = x, 0, c
    else:
        r_prime, g_prime, b_prime = c, 0, x
        
    r = r_prime + m
    g = g_prime + m
    b = b_prime + m
    
    return r, g, b

def apply_hsv_transfer_function(data_value, h_func, s_func, v_func):
    """
    Applies HSV transfer functions to a data value.
    h_func, s_func, v_func are callable mappings from data_value to [0, 1].
    Returns (r, g, b, alpha).
    """
    h = h_func(data_value)
    s = s_func(data_value)
    v = v_func(data_value)
    
    r, g, b = hsv_to_rgb(h, s, v)
    
    # Alpha is often mapped separately or derived from V.
    # Assuming a separate alpha function or using V as alpha for simplicity.
    alpha = v_func(data_value) # [G] Common practice, but source notes imply separate mapping capability.
    
    return r, g, b, alpha
```

## Subtopics

### [Separate Color and Intensity Transfer Functions](19963-separate-color-and-intensity-transfer-functions.md)
This method addresses the difficulty of simultaneously mapping data values to both color and brightness in volume rendering by decoupling the two aspects. It solves the problem of perceptual uniformity in high-contrast environments by using a color space that separates intensity (luminance) from chromaticity, allowing independent transfer functions for each. Unlike RGB-based approaches where color and intensity are entangled, this method allows the user to specify separate transfer functions for chrominance channels while computing luminance using established monochromatic algorithms.

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel for the HSV-to-RGB conversion and transfer function evaluation.
- **Memory Complexity:** $O(1)$ auxiliary memory per voxel.
- **Strengths:** Intuitive user interface for color selection; decoupling of color and brightness improves perceptual control [S1, unofficial].
- **Limitations:** HSV is not perceptually uniform; equal steps in H, S, or V do not correspond to equal perceptual changes. This can lead to uneven visual transitions if not carefully tuned.
- **When to use:** When users need precise control over color hue and brightness independently, or when working with data where color identity is more important than perceptual uniformity.
- **When not to use:** When perceptual uniformity is critical (e.g., scientific visualization requiring accurate magnitude representation), consider perceptually uniform color spaces like LUV or Lab.

## Background
The use of HSV (Hue, Saturation, Value) color space for transfer functions is motivated by the ease of specifying colors compared to RGB. It provides separate mappings for color and brightness, which simplifies the navigation task in the 4D transfer function space (3 axes for RGB color and one for transparency) [S1, unofficial]. Simple algorithms exist to convert the HSV values into the RGB triples used in volume rendering [S1, unofficial]. This approach is part of the broader exploration of volume rendering via transfer functions, which constitutes a navigation task in a high-dimensional space [S1, unofficial].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
