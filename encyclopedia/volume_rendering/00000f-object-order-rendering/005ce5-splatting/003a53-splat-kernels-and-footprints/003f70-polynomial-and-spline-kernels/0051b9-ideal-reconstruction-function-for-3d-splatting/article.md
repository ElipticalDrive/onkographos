# Ideal Reconstruction Function for 3D Splatting

Also called: Optimized 3D splat filter, Splatting with optimized 3D reconstruction filter, Tri-cubic reconstruction kernel, Westover splatting

## Overview
The Ideal Reconstruction Function for 3D Splatting is a rotationally symmetric, compact-support kernel designed to reconstruct 3D volume data via splatting with minimal visible artifacts [S1, unofficial]. It solves the problem of "Mach bands" and visible individual splat boundaries that occur when approximating smooth volume data with polygon collections or non-optimized kernels [S1, unofficial]. Unlike Gaussian splats, this kernel has a well-defined finite overlap extent and goes to zero in a minimum extent, providing high-quality smooth overlap from any viewing direction at the cost of higher performance penalties for large screen-space splats [S1, unofficial]. It is distinct from its siblings by being mathematically optimized to minimize the relative variance of a constant cube reconstruction in 3D, rather than relying on standard spline continuity constraints alone [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid with intensity values; a viewing direction; a splat center $(0,0,0)$ in local coordinates [S1, unofficial].
*   **Output:** A 3D reconstruction kernel function $h(x,y,z)$ and its 2D projected footprint $f(x,y)$ for rasterization [S1, unofficial].

**Data Structures:**
The kernel is defined by two scalar parameters $s$ and $t$, and two cubic polynomial coefficient sets [S1, unofficial].
*   $t$: Support radius (outer boundary) [S1, unofficial].
*   $s$: Transition radius (inner boundary) [S1, unofficial].
*   $g(r)$: Radial function dependent on $r = \sqrt{x^2 + y^2 + z^2}$ [S1, unofficial].

**Parameters:**
*   Support radius $t = 1.556228$ [S1, unofficial].
*   Transition radius $s = 0.889392$ [S1, unofficial].
*   Inner polynomial coefficients (derived from optimization): $a=0.557526$, $b=-1.157743$, $c=0.671033$ [S1, unofficial].
*   Outer polynomial coefficients (derived from optimization): $d=0.067599$, $e=-0.282474$ [S1, unofficial].

**Numbered Steps:**
1.  Calculate the radial distance $r$ from the splat center for any point $(x,y,z)$:
    $$ r = \sqrt{x^2 + y^2 + z^2} $$
2.  Evaluate the kernel value $h(x,y,z)$ using the radial function $g(r)$:
    *   If $0 < r \le s$:
        $$ g(r) = 0.557526 - 1.157743 r^2 + 0.671033 r^3 $$
    *   If $s < r \le t$:
        $$ g(r) = 0.067599 (r - t)^2 - 0.282474 (r - t)^3 $$
    *   If $r > t$:
        $$ g(r) = 0 $$
    *   The kernel is $C^1$ continuous at the origin and at the transition point $s$ [S1, unofficial].
3.  To rasterize, compute the 2D footprint $f(x,y)$ by integrating the 3D kernel along the view direction (z-axis):
    $$ f(x,y) = \int_{-\infty}^{\infty} h(x,y,z) \, dz $$
    This integral has a closed-form solution because $h$ is composed of low-degree polynomials in $r^2+z^2$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Mach Bands:** Using polygon approximations (15-21 triangles) for splats causes visible Mach bands and individual splat visibility because they do not overlap smoothly [S1, unofficial]. This kernel eliminates this by providing smooth, analytic overlap [S1, unofficial].
*   **Precision:** The maximum relative deviation from the mean for a constant cube reconstruction is small ($0.00534$), but intensities may need scaling to maintain accuracy in 8-bit color buffers [S1, unofficial].
*   **Viewing Direction:** The kernel is rotationally symmetric, ensuring accuracy from any viewing direction, unlike axis-aligned approximations [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ideal_reconstruction_kernel(x, y, z):
    """
    Computes the value of the ideal 3D reconstruction kernel at (x, y, z).
    Parameters are fixed constants from the optimization in [S1, unofficial].
    """
    t = 1.556228
    s = 0.889392
    
    r_sq = x*x + y*y + z*z
    r = r_sq ** 0.5
    
    if r > t:
        return 0.0
    elif r == 0.0:
        return 0.557526
    elif r <= s:
        return 0.557526 - 1.157743 * r_sq + 0.671033 * (r_sq * r)
    else:
        diff = r - t
        return 0.067599 * (diff * diff) - 0.282474 * (diff * diff * diff)

def compute_footprint(x, y):
    """
    Computes the 2D footprint f(x,y) by integrating h(x,y,z) over z.
    The closed-form integral of the piecewise cubic radial function is required.
    Not given in the sources.
    """
    # Not given in the sources.
    pass
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Higher performance penalty for large screen-space splats compared to Gaussian splats due to the finite support and complex footprint calculation [S1, unofficial].
*   **Quality:** Minimizes relative variance of constant cube reconstruction to $0.00119$ [S1, unofficial]. Maximum projected footprint deviation is approximately $1/256$ of the mean, fitting within 8-bit color accuracy [S1, unofficial].
*   **When to use:** [G] When high-quality, artifact-free volume rendering is required and polygon-based splatting is insufficient.
*   **When not to use:** [G] When performance is critical and large screen-space splats are common, or when infinite support (Gaussian) is acceptable for smoother falloff.

## Background
The method was developed by Laur and Hanrahan, building on Max's 2D ideal reconstruction function [Max92] [S1, unofficial]. While Max's function was designed for 2D image reconstruction, Laur and Hanrahan extended it to 3D signals to replace polygon approximations of splats in volume rendering [S1, unofficial]. The optimization minimizes the relative variance of a constant cube reconstruction, ensuring smooth overlap from any viewing direction [S1, unofficial]. It was implemented using the fast compositing hardware of the SGI VGX [S1, unofficial].

## Sources
[S1] Texture splats for 3D vector and scalar field visualization, https://digital.library.unt.edu/ark:/67531/metadc1339952/, fetched 2026-10-26 (unofficial)
