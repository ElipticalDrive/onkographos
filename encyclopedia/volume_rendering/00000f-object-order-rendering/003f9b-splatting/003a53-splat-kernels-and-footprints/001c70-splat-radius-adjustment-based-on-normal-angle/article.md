# Splat Radius Adjustment Based on Normal Angle
Also called: Angle-dependent Splat Radius, Normal-based Splat Radius Adjustment
## Overview
This method adjusts the radius of individual splats to prevent visual holes that occur when surface samples are elevated in the normal direction of their base plane [S1, unofficial]. It sets the splat radius based on the angle between the splat's normal vector and the base plane's normal vector [S1, unofficial]. Unlike fixed-radius splatting, which assumes a flat distribution, this approach increases radii in areas with high local elevation variance, ensuring continuous coverage [S1, unofficial]. It builds on [Splatting](23-splatting.md) by modifying the primitive size parameter during the construction phase.
## How it works
**Inputs and Outputs:**
- Input: A set of regularly distributed samples on a base plane, where each sample has an associated normal vector $\mathbf{n}$ and an elevation value.
- Input: The sampling step size $\mu$ of the regular grid.
- Input: The normal vector of the base plane $\mathbf{n}_B$.
- Output: A radius $r$ for each splat.

**Data Structures:**
- Each splat $i$ stores its position, its normal vector $\mathbf{n}_i$, and the computed radius $r_i$.
- The base plane normal $\mathbf{n}_B$ is constant for the patch or region being processed.

**Steps:**
1. For each splat $i$, identify its normal vector $\mathbf{n}_i$ and the base plane normal $\mathbf{n}_B$.
2. Compute the dot product $\langle \mathbf{n}_i, \mathbf{n}_B \rangle$.
3. Calculate the radius $r_i$ using the formula:
   $$ r_i = \frac{\frac{1}{2}\sqrt{2}\mu}{0.1 + 0.9\langle \mathbf{n}_i, \mathbf{n}_B \rangle} $$
4. Use the computed radius $r_i$ during the rasterization of the splat [S1, unofficial].

**Parameters:**
- $\mu$: The sampling step size of the regular grid.
- $0.1$: A constant in the denominator that establishes scaling and prevents division by zero or excessively large radii when the dot product is small [S1, unofficial].
- $0.9$: A constant that controls the maximal radius increase. When $\langle \mathbf{n}_i, \mathbf{n}_B \rangle = 1$, the denominator is $1.0$, resulting in $r = \frac{1}{2}\sqrt{2}\mu$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- If samples are elevated in the normal direction of the plane, holes may occur between splats if radii are not adjusted [S1, unofficial].
- A large angle between the splat normal and the plane normal indicates large variance between neighboring elevation values, resulting in increased spatial gaps [S1, unofficial].
- The formula ensures that as the angle increases (dot product decreases), the radius increases to cover these gaps [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_splat_radius(splat_normal, base_plane_normal, grid_step_size):
    """
    Computes the radius for a splat based on the angle between its normal and the base plane normal.
    
    Args:
        splat_normal (Vector3): The normal vector of the splat.
        base_plane_normal (Vector3): The normal vector of the base plane.
        grid_step_size (float): The sampling step size of the regular grid.
        
    Returns:
        float: The computed radius for the splat.
    """
    # Compute dot product
    dot_product = dot(splat_normal, base_plane_normal)
    
    # Calculate radius using the formula
    # Numerator: (1/2) * sqrt(2) * mu
    numerator = 0.5 * sqrt(2) * grid_step_size
    
    # Denominator: 0.1 + 0.9 * <n, n_B>
    denominator = 0.1 + 0.9 * dot_product
    
    # Avoid division by zero, though the constants make it unlikely
    if denominator == 0:
        denominator = 1e-9
        
    radius = numerator / denominator
    
    return radius
```
## Subtopics

### [Automatic Size and Variance Determination for Surface Splatting](7363-automatic-size-and-variance-determination-for-surface-splatt.md)
This method improves the visual quality of surface splatting on close-up views by automatically computing the radius ($r_i$) and variance ($v_i$) for each splat to ensure hole-free rendering and smooth blending, comparable to polygon rendering. Unlike methods that use global or fixed local variance and size, this approach calculates these parameters per-splat based on local point density, balancing blurriness and aliasing without manual user adjustment. It builds on [Splat Radius Adjustment Based on Normal Angle](1264-splat-radius-adjustment-based-on-normal-angle.md) by providing the specific automatic formulas for size and variance that prevent the discontinuities and artifacts associated with improper parameter choices.

## Operating characteristics
- **Strengths:** Prevents holes between splats when samples are elevated in the normal direction of the base plane [S1, unofficial]. Adapts to local surface geometry by increasing radii where elevation variance is high [S1, unofficial].
- **Limitations:** Requires the computation of the dot product for each splat, adding a small computational overhead [S1, unofficial]. The choice of constants (0.1 and 0.9) may need tuning for different datasets to avoid over-blurring or under-covering [S1, unofficial].
- **When to use:** When rendering surface splats that have significant elevation variations relative to their base plane [S1, unofficial].
- **When not to use:** When the surface is flat and samples are not elevated, as the adjustment provides no benefit and may introduce unnecessary complexity [S1, unofficial].
## Background
This method was proposed in the context of single-pass high-quality splatting for surface reconstruction [S1, unofficial]. It addresses the issue of visual artifacts (holes) that arise when treating elevated samples as flat splats with a fixed radius [S1, unofficial]. The approach leverages the geometric relationship between the splat normal and the base plane normal to infer local surface curvature and elevation variance [S1, unofficial].
## Sources
[S1] Single-Pass High-Quality Splatting, http://nbn-resolving.de/urn:nbn:de:bsz:352-opus-22395, fetched 2026-10-26 (unofficial)
