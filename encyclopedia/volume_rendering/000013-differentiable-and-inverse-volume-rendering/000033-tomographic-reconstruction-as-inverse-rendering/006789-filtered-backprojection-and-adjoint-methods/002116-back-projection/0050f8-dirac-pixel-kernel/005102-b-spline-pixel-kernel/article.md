# B-Spline Pixel Kernel

Also called: B-Spline interpolator, Noyau de pixel B-Spline

## Overview
The B-Spline Pixel Kernel is a weighting function for back-projection in tomographic reconstruction that assigns contribution to a pixel proportionally to the geometric intersection length of the projection line within that pixel [S1, unofficial]. It improves upon the Dirac Pixel Kernel by accounting for the varying path lengths of rays traversing a pixel at different angles [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: Projection angle $\theta$ (in radians), projection offset $\rho$, pixel coordinates $(i, j)$ (integer indices), and pixel size $\Delta_p$ (physical width/height of a square pixel) [S1, unofficial].
*   **Output**: Kernel weight $B^0(\theta, \rho, i, j)$, a scalar value in $[0, 1]$ [S1, unofficial].

**Data Structures**
No complex data structures are required; the method operates on scalar inputs [S1, unofficial].

**Step-by-Step Method**
1.  **Calculate Distance**: Compute the perpendicular distance $e$ from the center of the pixel $(i, j)$ to the projection line defined by $(\theta, \rho)$ [S1, unofficial].
2.  **Determine Bounding Extents**: Calculate the half-widths of the trapezoidal kernel support [S1, unofficial].
3.  **Compute Weight**: Evaluate $B^0$ based on the absolute distance $|e|$ [S1, unofficial].

**Formulas and Definitions**
The kernel $B^0(\theta, \rho, i, j)$ is defined piecewise as follows [S1, unofficial]:
$$ B^0 = 1 \quad \text{if} \quad |e| \le \frac{\Delta_p (||\cos\theta| - |\sin\theta||)}{2} $$
$$ B^0 = \frac{-2|e| + (|\cos\theta| + |\sin\theta|)\Delta_p}{2\Delta_p \min(|\cos\theta|, |\sin\theta|)} \quad \text{if} \quad \frac{\Delta_p (||\cos\theta| - |\sin\theta||)}{2} < |e| \le \frac{\Delta_p (|\cos\theta| + |\sin\theta|)}{2} $$
$$ B^0 = 0 \quad \text{otherwise} $$
Here, $e = \rho - i \cos \theta - j \sin \theta$ is the signed distance from the pixel center to the projection line, and $\Delta_p$ is the pixel size [S1, unofficial]. The total bounding width of the trapezoidal support is $L + L' = \Delta_p (|\cos\theta| + |\sin\theta|)$ [S1, unofficial].

**Parameters**
$\Delta_p$ represents the physical width/height of a square pixel [S1, unofficial]. Typical values are Not given in the sources.

**Edge Cases and Pitfalls**
*   **Angle Symmetry**: The kernel is periodic with period $\pi/2$ due to the absolute values of sine and cosine [S1, unofficial]. The source derivation for the interpolation segment is performed on $[0, \pi/4[$, but the absolute value formulation generalizes it to all angles [S1, unofficial].
*   **Division by Zero**: The denominator in the interpolation case contains $\min(|\cos \theta|, |\sin \theta|)$, which approaches zero only when $\theta$ is a multiple of $\pi/2$ [S1, unofficial]. In these cases, the projection line is axis-aligned [S1, unofficial]. The limit of the expression as $\min \to 0$ should be handled carefully [S1, unofficial]. If $\theta = 0$, $\cos \theta = 1$ and $\sin \theta = 0$ [S1, unofficial]. The formula becomes $\frac{-2|e| + \Delta_p}{0}$, which is undefined [S1, unofficial]. However, geometrically, for axis-aligned lines, the intersection length is simply $\Delta_p$ if the line passes through the pixel, and $0$ otherwise [S1, unofficial]. [G] In implementation, guard against division by zero by checking if $\min(|\cos \theta|, |\sin \theta|) < \epsilon$; at these angles, the trapezoidal support collapses to a rectangle, and the weight is $1$ if $|e| \le \Delta_p/2$ and $0$ otherwise.
*   **Precision**: Floating point comparisons for $|e| \le w_{inner}$ should use a small epsilon to avoid jitter at boundaries [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def b_spline_pixel_kernel(theta, rho, i, j, delta_p):
    """
    Computes the B-Spline pixel kernel weight.
    
    Args:
        theta (float): Projection angle in radians.
        rho (float): Projection offset.
        i (int): Pixel column index.
        j (int): Pixel row index.
        delta_p (float): Pixel size.
        
    Returns:
        float: Kernel weight B^0.
    """
    # 1. Calculate distance from pixel center to projection line
    cos_t = abs(math.cos(theta))
    sin_t = abs(math.sin(theta))
    e = rho - i * cos_t - j * sin_t
    abs_e = abs(e)
    
    # 2. Calculate bounds
    inner_bound = delta_p * abs(cos_t - sin_t) / 2.0
    outer_bound = delta_p * (cos_t + sin_t) / 2.0
    
    # 3. Compute weight
    if abs_e <= inner_bound:
        return 1.0
    elif abs_e <= outer_bound:
        min_cs = min(cos_t, sin_t)
        # Guard against division by zero at axis-aligned angles
        if min_cs < 1e-9:
            return 1.0
        numerator = -2.0 * abs_e + (cos_t + sin_t) * delta_p
        denominator = 2.0 * delta_p * min_cs
        return numerator / denominator
    else:
        return 0.0
```

## Operating characteristics
*   **Complexity**: $O(1)$ time and $O(1)$ auxiliary memory per pixel per projection line [S1, unofficial].
*   **Performance**: Faster than analytical integration methods but more expensive than the Dirac kernel due to trigonometric evaluations and conditional branching [S1, unofficial].
*   **Strengths**: Reduces aliasing artifacts compared to the Dirac kernel by weighting pixels based on actual intersection length [S1, unofficial].
*   **Limitations**: Requires careful handling of edge cases at axis-aligned angles and additional arithmetic operations [S1, unofficial].

## Background
The B-Spline Pixel Kernel was first defined in 1993 by Unser et al. [S1, unofficial]. It is derived from the concept of convolving a box function (zero-order B-spline) with the projection line, resulting in a trapezoidal kernel shape that accounts for the geometric intersection length [S1, unofficial]. It is used in tomographic reconstruction algorithms to optimize back-projection accuracy [S1, unofficial].

## Sources
[S1] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
