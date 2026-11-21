# Importance Sampling of Procedural Skies using Marginal and Conditional CDFs

Also called: Nishita Sky Importance Sampling, Spherical Light Sampling via 2D CDF

## Overview
This method reduces variance and computation time when sampling directions for a procedural sky (modeled as a spherical light source) by concentrating samples in luminous regions, such as the sun, rather than using uniform sphere sampling [S1, unofficial]. It achieves this by discretizing the sky's luminosity into 2D and 1D textures representing conditional and marginal probability distributions, allowing for efficient rejection-free sampling via inverse transform sampling [S1, unofficial]. Unlike [Importance Sampling for Sphere Surface Area](2223-importance-sampling-for-sphere-surface-area.md) which focuses on geometric surface area, or [Sphere Light Shading with Importance Solid Angle Sampling](2236-sphere-light-shading-with-importance-solid-angle-sampling.md) which may use analytical approximations, this method relies on pre-integrated numerical CDFs derived directly from the procedural sky's color data [S1, unofficial].

## How it works
**Inputs:**
- Procedural sky function $L(\theta, \phi)$ returning color/radiance for azimuth $\theta$ and elevation $\phi$.
- Discretization resolution $W \times H$ (typically $180 \times 180$) [S1, unofficial].
- Random number generator producing uniform values in $[0, 1]$.

**Outputs:**
- Sampled direction vector $\mathbf{d}$ in Cartesian coordinates.
- Probability Density Function (PDF) value $p(\mathbf{d})$ for unbiased weighting.

**Data Structures:**
Four textures are precomputed from the procedural sky data before rendering:

1.  **Conditional Function Texture ($C$):** A 2D texture of size $W \times H$.
    -   Layout: $C[i, j]$ stores the magnitude (length) of the color value at column $i$ (azimuth index) and row $j$ (elevation index).
    -   $C[i, j] = \| L(\theta_i, \phi_j) \|$.
2.  **Conditional CDF Texture ($CDF_c$):** A 2D texture of size $W \times H$.
    -   Layout: $CDF_c[i, j]$ stores the cumulative sum of $C[i, k]$ for $k=0 \dots j$, normalized by the total sum of column $i$.
    -   $CDF_c[i, j] = \frac{\sum_{k=0}^{j} C[i, k]}{\sum_{k=0}^{H-1} C[i, k]}$.
3.  **Marginal Function Texture ($M$):** A 1D texture of size $W$.
    -   Layout: $M[i]$ stores the total sum of values in column $i$ of the Conditional Function texture.
    -   $M[i] = \sum_{j=0}^{H-1} C[i, j]$.
4.  **Marginal CDF Texture ($CDF_m$):** A 1D texture of size $W$.
    -   Layout: $CDF_m[i]$ stores the cumulative sum of $M[k]$ for $k=0 \dots i$, normalized by the grand total of all values in the sky.
    -   $CDF_m[i] = \frac{\sum_{k=0}^{i} M[k]}{\sum_{k=0}^{W-1} M[k]}$.

**Algorithm Steps:**

1.  **Column Selection (Azimuth):**
    Generate a random number $x_i \in [0, 1]$. Walk the 1D Marginal CDF texture $CDF_m$ to find the first index $i$ such that $CDF_m[i] > x_i$. This index $i$ corresponds to the selected column (azimuth bin).

2.  **Row Selection (Elevation):**
    Generate a second random number $x_j \in [0, 1]$. Using the selected column $i$, walk the 2D Conditional CDF texture $CDF_c$ at column $i$ to find the first index $j$ such that $CDF_c[i, j] > x_j$. This index $j$ corresponds to the selected row (elevation bin).

3.  **Coordinate Conversion:**
    Convert the discrete indices $(i, j)$ into continuous spherical coordinates:
    -   Azimuth: $\theta = \frac{i}{W} \cdot 2\pi$
    -   Elevation: $\phi = \frac{j}{H} \cdot \pi$
    *(Note: The source defines elevation range as $\pi$, mapping $[0, \pi]$ to the full sphere vertically, though standard spherical coordinates often use $[0, \pi/2]$ for hemispheres. The source explicitly states $\pi$ for elevation)* [S1, unofficial].

4.  **Direction Vector:**
    Convert $(\theta, \phi)$ from spherical to Cartesian coordinates $(x, y, z)$ to obtain the ray direction $\mathbf{d}$:
    $$
    \begin{aligned}
    x &= \sin(\phi) \cos(\theta) \\
    y &= \sin(\phi) \sin(\theta) \\
    z &= \cos(\phi)
    \end{aligned}
    $$

5.  **PDF Calculation:**
    Calculate the total PDF for the sampled direction to correct for bias:
    -   Marginal PDF: $p_{marginal}(i) = \frac{M[i]}{\sum_{all} M}$
    -   Conditional PDF: $p_{cond}(j|i) = \frac{C[i, j]}{M[i]}$
    -   Total PDF: $p(i, j) = p_{marginal}(i) \times p_{cond}(j|i)$

6.  **Unbiased Weighting:**
    Divide the sampled lighting contribution (radiance) by the total PDF $p(i, j)$ to maintain an unbiased estimator [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sample_procedural_sky(W, H, C, CDF_c, M, CDF_m, rng):
    """
    Samples a direction from a procedural sky using marginal and conditional CDFs.
    
    Args:
        W: Width of the sky texture (azimuth bins).
        H: Height of the sky texture (elevation bins).
        C: 2D array [W][H] of conditional function values (luminosity magnitude).
        CDF_c: 2D array [W][H] of conditional CDF values.
        M: 1D array [W] of marginal function values (column sums).
        CDF_m: 1D array [W] of marginal CDF values.
        rng: Function returning a uniform random float in [0, 1].
        
    Returns:
        direction: Tuple (x, y, z) Cartesian direction vector.
        pdf: Float, the probability density of the sampled direction.
    """
    
    # Step 1: Column Selection (Azimuth)
    x_i = rng()
    i = 0
    while i < W and CDF_m[i] <= x_i:
        i += 1
    # If x_i is 1.0 or CDF_m doesn't exceed it due to precision, clamp to last index
    if i >= W:
        i = W - 1

    # Step 2: Row Selection (Elevation)
    x_j = rng()
    j = 0
    while j < H and CDF_c[i][j] <= x_j:
        j += 1
    # Clamp to last index if necessary
    if j >= H:
        j = H - 1

    # Step 3: Coordinate Conversion
    theta = (i / W) * 2 * math.pi
    phi = (j / H) * math.pi

    # Step 4: Direction Vector
    sin_phi = math.sin(phi)
    cos_phi = math.cos(phi)
    cos_theta = math.cos(theta)
    sin_theta = math.sin(theta)
    
    x = sin_phi * cos_theta
    y = sin_phi * sin_theta
    z = cos_phi
    
    direction = (x, y, z)

    # Step 5: PDF Calculation
    # Total sum of M is precomputed or can be derived from CDF_m[W-1] * TotalSum
    # Since CDF_m is normalized, sum(M) is the denominator used during CDF_m creation.
    # Let S_total = sum(M). We can recover it if not passed, but typically 
    # p_marginal(i) = M[i] / S_total.
    # Note: The source defines p_marginal(i) = M(i) / sum(all M).
    
    # To calculate PDF accurately, we need the total sum of M.
    # If CDF_m is normalized, CDF_m[W-1] == 1.0.
    # We assume S_total is available or computed once.
    # For this implementation, we calculate p_marginal directly from M and its sum.
    
    sum_M = sum(M) # In practice, this is a constant precomputed value.
    
    p_marginal = M[i] / sum_M
    p_cond = C[i][j] / M[i]
    pdf = p_marginal * p_cond

    return direction, pdf
```

**Edge Cases and Pitfalls:**
-   **Bias Correction:** Importance sampling introduces bias by preferentially selecting directions. The final radiance contribution must be divided by the calculated PDF to ensure an unbiased estimator [S1, unofficial].
-   **Precision:** Walking the CDF arrays requires careful handling of floating-point comparisons. If the random number equals the CDF value exactly, the loop condition `<=` ensures the next index is selected, which is consistent with standard inverse transform sampling definitions.
-   **Boundary Conditions:** If the random number falls outside the range of the CDF (e.g., due to precision errors resulting in a value slightly $> 1.0$), the index should be clamped to the maximum valid index ($W-1$ or $H-1$) [G].

## Operating characteristics
-   **Time Complexity:** Preprocessing is $O(W \cdot H)$ to compute the textures. Sampling is $O(W + H)$ in the worst case for walking the CDF arrays, though typically much faster if the CDFs are sampled via binary search (not specified in source, source describes linear walk).
-   **Memory Complexity:** Requires storage for four textures: $2 \cdot W \cdot H$ floats for the 2D textures and $2 \cdot W$ floats for the 1D textures. For $180 \times 180$, this is negligible.
-   **Performance:** Significantly reduces the number of rays needed for the same quality compared to uniform sphere sampling by concentrating samples in luminous areas like the sun [S1, unofficial].
-   **Strengths:** Efficiently handles procedural skies where analytical PDFs are difficult to derive. Reduces variance in scenes with bright localized light sources (sun).
-   **Limitations:** Requires preprocessing time and memory for textures. Accuracy depends on the discretization resolution ($180 \times 180$ in the source). Linear walking of CDFs can be slow for high resolutions; binary search is recommended for production but not detailed in the source.
-   **When to use:** When rendering procedural skies (e.g., Nishita model) in a path tracer where uniform sampling leads to high variance and slow convergence.
-   **When not to use:** For simple directional lights (sun as a point) or when the sky is uniform.

## Background
This method was implemented by Serkan Geren in a CUDA-based volumetric path tracer, specifically for the Nishita sky model implementation [S1, unofficial]. The Nishita sky model uses Rayleigh and Mie scattering to achieve color and luminosity based on ray direction [S1, unofficial]. The sampling strategy is inspired by the pbrt section on sampling light sources, adapting the marginal/conditional decomposition technique to the 2D discretization of a spherical sky [S1, unofficial]. It was demonstrated in rendering clouds with a VPT (Volumetric Path Tracer) on an Nvidia GTX680 [S1, unofficial].

## Sources
[S1] Sampling A Procedural Sky, https://sergeneren.com/2019/04/17/sampling-a-procedural-sky/, fetched 2026-10-28 (unofficial)
