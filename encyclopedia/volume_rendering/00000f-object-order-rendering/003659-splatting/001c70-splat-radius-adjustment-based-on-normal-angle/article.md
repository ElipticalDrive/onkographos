# Splat Radius Adjustment Based on Normal Angle

Also called: Angle-dependent Splat Radius, Normal-based Splat Radius Adjustment

## Overview
This method adjusts the radius of individual splats to prevent visual holes that occur when surface samples are elevated in the normal direction of their base plane [S1, unofficial]. It scales the splat radius based on the angle between the splat's normal vector and the base plane's normal vector [S1, unofficial]. Unlike fixed-radius splatting, which assumes a flat distribution, this approach increases radii in areas with high local variance (steep angles) to ensure continuous coverage [S1, unofficial]. It serves as the foundational logic for automatic size determination in surface splatting [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of regularly distributed samples on a base plane. For each sample, the method requires the sampling step size $\mu$, the splat normal vector $\mathbf{n}$, and the base plane normal vector $\mathbf{n}_B$.
*   **Output:** The adjusted radius $r$ for each splat.

**Data Structures:**
*   **Splat Attributes:** Each splat primitive stores its position, the normal vector $\mathbf{n}$, and the computed radius $r$.
*   **Base Plane Attributes:** The base plane is defined by its normal vector $\mathbf{n}_B$ and the grid sampling step size $\mu$.

**Parameters:**
*   $\mu$: The sampling step size of the regular grid [S1, unofficial].
*   Constants: $0.1$ and $0.9$ in the denominator [S1, unofficial].
    *   $0.1$: Establishes scaling and prevents division by zero or excessively large radii when the dot product is small [S1, unofficial].
    *   $0.9$: Controls the maximal radius increase [S1, unofficial].

**Formula:**
The radius $r$ is calculated as:
$$ r = \frac{\frac{1}{2}\sqrt{2}\mu}{0.1 + 0.9\langle \mathbf{n}, \mathbf{n}_B \rangle} $$
where $\langle \mathbf{n}, \mathbf{n}_B \rangle$ denotes the dot product $\mathbf{n} \cdot \mathbf{n}_B$ [S1, unofficial].

**Steps:**
1.  **Initialize Base Radius:** Determine the base radius for a flat distribution, which is $\frac{1}{2}\sqrt{2}\mu$ [S1, unofficial].
2.  **Compute Dot Product:** Calculate the dot product between the splat normal $\mathbf{n}$ and the base plane normal $\mathbf{n}_B$ [S1, unofficial].
3.  **Calculate Denominator:** Compute the scaling factor $D = 0.1 + 0.9(\mathbf{n} \cdot \mathbf{n}_B)$ [S1, unofficial].
4.  **Adjust Radius:** Divide the base radius by $D$ to obtain the final radius $r$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_splat_radius(mu, n, n_B):
    """
    Compute the adjusted splat radius based on normal angle.
    
    Args:
        mu (float): Sampling step size of the regular grid.
        n (vec3): Normal vector of the splat.
        n_B (vec3): Normal vector of the base plane.
        
    Returns:
        float: The adjusted radius r.
    """
    # Base radius for flat distribution
    base_radius = 0.5 * sqrt(2) * mu
    
    # Compute dot product of normals
    # Note: Vectors should be normalized before this step if not already
    dot_product = dot(n, n_B)
    
    # Calculate denominator with constants to prevent division by zero
    # and control maximal radius increase
    denominator = 0.1 + 0.9 * dot_product
    
    # Avoid division by zero, though denominator >= 0.1 due to constant
    if denominator < 1e-6:
        denominator = 1e-6
        
    # Compute adjusted radius
    r = base_radius / denominator
    
    return r
```

**Edge Cases and Pitfalls:**
*   **Holes from Elevation:** If samples are elevated in the normal direction of the plane without radius adjustment, holes may occur between splats [S1, unofficial].
*   **Large Angles:** A large angle between the splat normal and the plane normal indicates large variance between neighboring elevation values, resulting in increased spatial gaps [S1, unofficial]. The formula increases radii in these areas to compensate [S1, unofficial].
*   **Division by Zero:** The constant $0.1$ in the denominator prevents division by zero or excessively large radii when the dot product is small (i.e., when the splat normal is nearly perpendicular to the base plane normal) [S1, unofficial].

## Subtopics

### [Automatic Size and Variance Determination for Surface Splatting](7363-automatic-size-and-variance-determination-for-surface-splatt.md)
Improves visual quality by automatically computing radius and variance per splat to ensure hole-free rendering. It builds on this method by providing specific automatic formulas for size and variance that prevent discontinuities.

## Operating characteristics
*   **Complexity:** The calculation is $O(1)$ per splat, involving basic vector operations (dot product) and arithmetic.
*   **Strengths:** Prevents visual artifacts (holes) caused by surface elevation and normal variation without requiring manual tuning of radii [S1, unofficial].
*   **Limitations:** Requires access to both the splat normal and the base plane normal. If normals are inaccurate or missing, the adjustment may fail.
*   **When to use:** When rendering surface splats where samples are elevated from a base plane and normal information is available.
*   **When not to use:** For flat, non-elevated surfaces where a fixed radius is sufficient, or when normal information is unavailable.

## Background
This method was proposed in the context of single-pass high-quality splatting for point-based rendering [S1, unofficial]. It addresses the limitation of setting all radii to a fixed value ($\frac{1}{2}\sqrt{2}\mu$) based on regular grid distribution, which fails when samples are elevated in the normal direction [S1, unofficial]. The approach is motivated by the observation that large angles between splat and plane normals correlate with high local elevation variance, necessitating larger splat coverage [S1, unofficial].

## Sources
[S1] Single-Pass High-Quality Splatting, http://nbn-resolving.de/urn:nbn:de:bsz:352-opus-22395, fetched 2026-10-26 (unofficial)
