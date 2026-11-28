# Dirac Pixel Kernel

Also called: Dirac interpolator, Noyau de pixel Dirac

## Overview
The Dirac Pixel Kernel is a weighting function used in tomographic back-projection that assigns equal importance to all projection lines traversing a pixel, effectively treating the pixel as a binary switch for ray intersection [S1, unofficial]. It is derived directly from the continuous Radon transform equation but does not resolve sampling errors on projections [S1, unofficial]. It is distinguished from sibling kernels like the B-Spline Pixel Kernel by its binary intersection weighting rather than geometric path-length weighting [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Projection data $R_\theta(\rho)$, pixel coordinates $(i, j)$, pixel size $\Delta_p$, projection angle $\theta$, and offset $\rho$ [S1, unofficial].
*   **Output:** A binary weight $\Delta(\theta, \rho, i, j) \in \{0, 1\}$ indicating whether the projection line intersects the pixel [S1, unofficial].

**Data Structures:**
*   No complex data structures are required; the kernel operates on scalar values derived from geometric calculations [S1, unofficial].

**Method:**
1.  Compute the signed distance $e$ from the center of pixel $(i, j)$ to the projection line $(\theta, \rho)$ [S1, unofficial].
2.  Compute the angular-dependent half-width of the pixel projected onto the line's normal direction [S1, unofficial].
3.  Return 1 if the absolute distance $|e|$ is strictly less than the half-width, otherwise return 0 [S1, unofficial].

**Formulas:**
$$\Delta(\theta, \rho, i, j) = \begin{cases} 1 & \text{if } -\frac{\Delta_p (|\cos\theta| + |\sin\theta|)}{2} < e < \frac{\Delta_p (|\cos\theta| + |\sin\theta|)}{2} \\ 0 & \text{otherwise} \end{cases}$$
where $e = \rho - j \cos\theta - i \sin\theta$ is the signed distance from the pixel center to the projection line, $\Delta_p$ is the pixel size, $\theta$ is the projection angle, $\rho$ is the radial offset, and $(i, j)$ are the pixel indices [S1, unofficial].

**Parameters:**
*   $\Delta_p$: The physical size of a pixel in the reconstruction volume [S1, unofficial].
*   $\theta$: The angle of the projection [S1, unofficial].
*   $\rho$: The offset of the projection line [S1, unofficial].
*   $i, j$: The integer indices of the pixel in the reconstruction grid [S1, unofficial].
*   Typical values: Not given in the sources. [G]

**Edge cases and pitfalls:**
*   The Dirac kernel does not resolve sampling errors on projections, leading to discretization artifacts in the reconstructed image [S1, unofficial].
*   It assumes all pixels traversed by a line are equally important, ignoring the fact that rays may traverse pixels at different angles and thus have different path lengths within the pixel [S1, unofficial].
*   Precision: The strict inequality in the intersection condition may cause boundary ambiguities at exact pixel edges; in practice, floating-point comparisons should handle degenerate grazing cases with standard epsilon guards [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dirac_pixel_kernel(i, j, theta, rho, delta_p):
    """
    Computes the Dirac pixel kernel weight for a given pixel and projection line.
    
    Args:
        i (float): Pixel column index.
        j (float): Pixel row index.
        theta (float): Projection angle in radians.
        rho (float): Radial offset of the projection line.
        delta_p (float): Size of a pixel.
        
    Returns:
        float: 1.0 if the line intersects the pixel, 0.0 otherwise.
    """
    # Calculate signed distance from pixel center to projection line
    e = rho - j * math.cos(theta) - i * math.sin(theta)
    
    # Calculate the half-width of the pixel projected onto the normal of the ray
    half_width = delta_p * (abs(math.cos(theta)) + abs(math.sin(theta))) / 2.0
    
    # Check if the distance is within the half-width
    if -half_width < e < half_width:
        return 1.0
    else:
        return 0.0
```

## Subtopics

### [B-Spline Pixel Kernel](14374-b-spline-pixel-kernel.md)
The B-Spline Pixel Kernel is a weighting function for back-projection in tomographic reconstruction that assigns contribution to a pixel proportionally to the geometric intersection length of the projection line within that pixel. It improves upon the Dirac Pixel Kernel by accounting for the varying path lengths of rays traversing a pixel at different angles, thereby reducing sampling artifacts and improving Signal-to-Noise Ratio (SNR) in reconstructed images.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G]
*   **Memory Complexity:** Not given in the sources. [G]
*   **Measured performance:** Not given in the sources. [G]
*   **Scaling:** Not given in the sources. [G]
*   **Strengths:** Simple to implement and computationally efficient [G].
*   **Limitations:** Does not account for sampling errors or varying path lengths, leading to potential discretization artifacts [S1, unofficial].
*   **When to use:** When computational speed is prioritized over reconstruction quality, or as a baseline for comparison [G].
*   **When not to use:** When high-fidelity reconstruction is required and sampling artifacts must be minimized [G].

## Background
The Dirac Pixel Kernel is commonly employed in direct tomographic reconstruction methods [S1, unofficial]. It is derived from the continuous Radon transform equation [S1, unofficial]. While it is one of the most used pixel kernels due to its direct derivation, its limitations regarding sampling errors have led to the development of more advanced kernels like the B-Spline and area-based interpolators [S1, unofficial].

## Sources
[S1] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
