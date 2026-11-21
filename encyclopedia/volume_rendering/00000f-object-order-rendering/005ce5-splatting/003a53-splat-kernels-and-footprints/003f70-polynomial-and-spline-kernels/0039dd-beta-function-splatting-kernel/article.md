# Beta Function Splatting Kernel

Also called: Beta distribution splatting, Beta footprint

## Overview
The Beta Function Splatting Kernel is a 2D kernel function used in volume rendering that replaces the standard Normal (Gaussian) distribution with a Beta function to control the footprint shape [S1, unofficial]. It solves the problem of tuning visual parameters (opacity, zoom, shading) by offering smaller footprints that maintain equivalent visual quality while reducing rendering time [S1, unofficial]. It sets itself apart from Gaussian splats by allowing independent control over the curve's openness, height, and width, resulting in more compact support and faster execution [S1, unofficial].

## How it works
**Inputs:**
*   $x$: Normalized radial distance from the splat center, mapped to the range $[0, 1]$.
*   $a, b$: Shape parameters controlling the curve profile.
*   $k_{scale}$: Scaling factor for the kernel height.

**Outputs:**
*   $W$: The kernel weight/value at distance $x$.

**Data Structures:**
*   The kernel is defined by the scalar parameters $a, b, k_{scale}$.
*   The footprint radius is fixed by the domain of $x \in [0, 1]$.

**Formula:**
The kernel value $W$ is calculated as:
$$ W(x) = k_{scale} \cdot x^a \cdot (1 - x)^b $$
where $x \in [0, 1]$ [S1, unofficial].

**Parameters:**
*   $a, b$: Shape parameters. Identical values ($a=b$) produce symmetric curves. Lower values produce "more opened" (wider/flatter) curves [S1, unofficial].
*   $k_{scale}$: Scaling factor applied to adjust the range/height of the curve [S1, unofficial].

**Steps:**
1.  Calculate the normalized distance $x$ from the splat center to the sample point, ensuring $x \in [0, 1]$.
2.  Compute the base Beta function value: $x^a (1 - x)^b$.
3.  Multiply by the scaling factor $k_{scale}$ to obtain the final weight $W$.
4.  If $x < 0$ or $x > 1$, the weight is 0 (compact support).

**Edge Cases and Pitfalls:**
*   **Holes at Boundaries:** Holes may appear when voxel opacity decreases. This requires dynamic changes in footprint size to maintain continuity between neighbor voxels, especially at feature boundaries in tagged volumes [S1, unofficial].
*   **Precision:** For very small $a$ or $b$, numerical stability near $x=0$ or $x=1$ should be considered [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def beta_kernel(x, a, b, k_scale):
    """
    Computes the Beta function splatting kernel weight.
    
    Args:
        x (float): Normalized distance from center, in [0, 1].
        a (float): Shape parameter for the start of the curve.
        b (float): Shape parameter for the end of the curve.
        k_scale (float): Scaling factor for the kernel height.
        
    Returns:
        float: Kernel weight W.
    """
    if x < 0.0 or x > 1.0:
        return 0.0
    
    # Compute x^a * (1-x)^b
    # Note: 0^0 is treated as 1 in this context for continuity if a,b > 0
    term1 = x ** a
    term2 = (1.0 - x) ** b
    
    return k_scale * term1 * term2
```

## Subtopics

### [Deformable Beta Splatting](9228-deformable-beta-splatting.md)
An explicit radiance field method that allows the Beta kernel's spatial shape to deform during training via a learnable parameter, better modeling sharp boundaries and high-frequency details compared to fixed-shape kernels.

### [Universal Beta Splatting (UBS)](10589-universal-beta-splatting-ubs.md)
An N-dimensional splatting kernel that replaces the Gaussian distribution with a Beta function to enable independent per-dimension shape control, addressing limitations of coupled Gaussian distributions in methods like 6DGS.

## Operating characteristics
*   **Performance:** Beta footprints are smaller than Normal distribution footprints while providing equivalent visual quality, resulting in faster execution (e.g., twice as fast in tested examples) [S1, unofficial].
*   **Tuning:** Easier to tune visual parameters like opacity and zoom compared to Gaussian kernels [S1, unofficial].
*   **Limitations:** Requires careful management of footprint size to avoid holes when opacity varies, particularly at feature boundaries [S1, unofficial].

## Background
The Beta Function Splatting Kernel was proposed in the context of splatting multiresolution volume data using a feature graph [S1, unofficial]. It was introduced to improve rendering performance by using smaller footprints that are easier to tune under visual changes, such as zooming and shading functions [S1, unofficial].

## Sources
[S1] Splatting multiresolution volume data using the feature graph, http://hdl.handle.net/2117/86461, fetched 2026-10-26 (unofficial)
