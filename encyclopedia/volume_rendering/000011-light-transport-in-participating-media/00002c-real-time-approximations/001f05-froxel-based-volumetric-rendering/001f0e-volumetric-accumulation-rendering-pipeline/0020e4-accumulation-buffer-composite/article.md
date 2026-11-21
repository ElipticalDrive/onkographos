# Accumulation Buffer Composite

Also called: Alpha-Averaged Composite, Spot Noise Composite

## Overview
The Accumulation Buffer Composite is a post-processing step that averages accumulated color samples from a stochastic volume renderer and blends the result with the background using a reveal factor [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**:
    *   `accumbuf`: A 2D texture storing accumulated RGB and alpha per pixel [S1, unofficial].
    *   `countbuf`: A 2D texture storing the reveal factor (0.0 to 1.0) per pixel [S1, unofficial].
*   **Output**:
    *   `color_out`: A 4-tuple containing the final average color and background alpha [S1, unofficial].

**Data Structures**
The method operates on per-pixel data fetched from the two input textures [S1, unofficial].

**Steps**
1.  Fetch the accumulated color and alpha from `accumbuf` at the current fragment coordinate: $\text{accum} = \text{accumbuf}[\text{frag\_coord}]$ [S1, unofficial].
2.  Fetch the reveal factor from `countbuf` at the current fragment coordinate: $\text{reveal} = \text{countbuf}[\text{frag\_coord}]$ [S1, unofficial].
3.  If the reveal factor is approximately $1.0$, discard the fragment, as the volume is fully opaque and no compositing is needed [S1, unofficial].
4.  If any component of the accumulated RGB is infinite, replace the entire RGB tuple with the accumulated alpha value to prevent NaN propagation during division [S1, unofficial].
5.  Compute the average color by dividing the accumulated RGB by the accumulated alpha, clamped to `EPSILON` to avoid division by zero [S1, unofficial].
6.  Output the final color with the background alpha: $\text{color\_out} = (\text{avg.rgb}, 1.0 - \text{reveal})$ [S1, unofficial].

**Formulas**
$$ \text{avg} = \frac{\text{accum.rgb}}{\max(\text{accum.a}, \epsilon)} $$
where $\text{accum.rgb}$ is the accumulated red, green, and blue values, $\text{accum.a}$ is the accumulated alpha value, and $\epsilon$ is a small positive constant [S1, unofficial].

**Parameters**
*   `EPSILON`: $0.00001$. Used for numerical stability in division and approximate equality checks [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Division by Zero**: Prevented by using $\max(\text{accum.a}, \text{EPSILON})$ in the denominator [S1, unofficial].
*   **Infinity Propagation**: Detected via $\text{isinf}(\max3(|\text{accum.rgb}|))$ and replaced with $\text{accum.a}$ to avoid NaNs in the final output [S1, unofficial].
*   **Reveal Factor Approximation**: The check for $\text{reveal} == 1.0$ uses an approximate equality function to account for floating-point precision issues [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

EPSILON = 0.00001

def is_approximately_equal(a, b):
    return abs(a - b) <= (abs(a) if abs(a) < abs(b) else abs(b)) * EPSILON

def max3(v):
    return max(v[0], v[1], v[2])

def accumulate_buffer_composite(accumbuf, countbuf, frag_coord):
    # Step 1-2: Fetch accumulated color/alpha and reveal factor
    accum = accumbuf[frag_coord]  # Returns (r, g, b, a)
    reveal = countbuf[frag_coord] # Returns float

    # Step 3: If volume is fully opaque, discard fragment
    if is_approximately_equal(reveal, 1.0):
        return None

    # Step 4: Handle infinity in accumulated RGB
    if any(math.isinf(c) for c in accum[:3]):
        accum = (accum[3], accum[3], accum[3], accum[3])

    # Step 5: Compute average color
    avg_rgb = tuple(c / max(accum[3], EPSILON) for c in accum[:3])

    # Step 6: Output final color with background alpha
    return (avg_rgb[0], avg_rgb[1], avg_rgb[2], 1.0 - reveal)
```

## Subtopics

### Empirical Volume Shading with Indirect Light Approximation
An alternative sibling method that approximates indirect lighting in volumetric scenes. Choose this when indirect lighting effects are critical and stochastic noise is less of a concern than lighting accuracy. [Empirical Volume Shading with Indirect Light Approximation](4092-empirical-volume-shading-with-indirect-light-approximation.md)

## Operating characteristics

## Background
The method originates from the `volumetric-spot-noise` repository [S1, unofficial].

## Sources
[S1] bottledspace/volumetric-spot-noise, https://github.com/bottledspace/volumetric-spot-noise, fetched 2026-10-27 (unofficial)
