# Boundary-Enhanced Opacity

Also called: Boundary Enhancement, Gradient-based Boundary Enhancement, Gradient-based opacity enhancement, Levoy90 gradient-based shading, gradient opacity enhancement

## Overview
Boundary-Enhanced Opacity is a technique that selectively increases the opacity of volume samples based on the local gradient magnitude, thereby emphasizing boundaries between regions of different densities while making areas of constant density transparent [S1][S2, S3, unofficial]. It distinguishes itself from its parent [Classification and transfer functions](38-classification-and-transfer-functions.md) by using gradient magnitude as a multiplicative modifier to the base opacity, rather than as a primary classification criterion or a 2D lookup table input [S2, S3, unofficial]. Unlike [Silhouette Enhancement](796-silhouette-enhancement.md), which incorporates view direction to highlight surface contours, this method relies solely on scalar field discontinuities [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset $V$ defined by scalar field $f(x,y,z)$, sample positions $P_i$, and optionally a base transfer function mapping voxel values to color $c_v$ and opacity $o_v$.
*   **Output:** Enhanced opacity $o_g$ for each sample, used during ray casting.

**Data Structures:**
*   **Volume Data:** A 3D grid or set of sample points where each point $P_i = (x_i, y_i, z_i)$ has an associated scalar value $v_i = f(P_i)$ [S2, S3, unofficial].
*   **Gradient:** The spatial gradient vector $\nabla f(P_i)$ at each sample point.
*   **Normalized Gradient:** $\nabla f_n$, the unit vector of the gradient, used if directional operations are required (though standard boundary enhancement uses magnitude) [S2, S3, unofficial].

**Parameters:**
*   $k_{gc}$: Base opacity scaling factor. Controls the range of effects.
    *   $k_{gc} = 1, k_{gs} = 0$: No enhancement [S1][S2, S3, unofficial].
    *   $k_{gc} = 0$: Only areas with large gradients are visible [S1][S2, S3, unofficial].
    *   Default: Not explicitly stated as a single default, but $k_{gc}=1$ is the neutral state [S2, S3, unofficial].
*   $k_{gs}$: Gradient scaling factor. Controls the amount of enhancement.
    *   Default: $0$ (no enhancement) [S2, S3, unofficial].
    *   Typical usage: $\ge 1$ for full enhancement [S1][S2, S3, unofficial].
*   $k_{ge}$: Exponent. Controls the slope of the opacity curve [S1][S2, S3, unofficial].
    *   Typical value: $2.0$ [S1][S2, S3, unofficial].

**Steps:**
1.  **Sample Value:** For each sample point $P_i$, calculate the scalar volume value $v_i = f(P_i)$ [S2, S3, unofficial].
2.  **Gradient Calculation:** Calculate the value gradient $\nabla f(P_i)$ at $P_i$ [S2, S3, unofficial].
3.  **Normalization:** Normalize the gradient to obtain $\nabla f_n$ if required by specific variants, though the core formula uses the magnitude $\|\nabla f\|$ [S2, S3, unofficial]. Note: Source [S1] uses $\|\nabla_{fn}\|$ in the formula text but defines $\nabla_{fn}$ as the value gradient, implying the magnitude of the gradient vector. Source [S2, S3, unofficial] uses $\nabla f$ to denote the magnitude in the formula context. We interpret $\nabla f$ in the formula as the gradient magnitude $\|\nabla f(P_i)\|$ [S1][S2, S3, unofficial].
4.  **Base Opacity:** Determine the base opacity $o_v$. This is obtained by mapping $v_i$ through a standard transfer function. If no transfer function is used, $o_v$ is set to a constant for the entire volume [S2, S3, unofficial].
5.  **Enhanced Opacity Calculation:** Compute the boundary-enhanced opacity $o_g$ using the formula:
    $$ o_g = o_v \left( k_{gc} + k_{gs} \|\nabla f(P_i)\| \right)^{k_{ge}} $$
    *Note on Formula Discrepancy:* Source [S1] presents the formula as $o_g = o_v ( k_{gc} + k_{gs} (\|\nabla_{fn}\|)^{k_{gc}} )$, using $k_{gc}$ as the exponent for the gradient term. However, the parameter description in [S1] and the consistent formulas in [S2, S3, unofficial] define $k_{ge}$ as the exponent controlling the slope. The sources [S2, S3, unofficial] explicitly state $k_{ge}$ is the exponent. The formula in [S1] appears to contain a typo where $k_{gc}$ was used instead of $k_{ge}$ in the exponent position, or $k_{gc}$ serves a dual role. Given [S2, S3, unofficial] provide the parameter $k_{ge}$ specifically for this purpose and cite the same underlying work, the exponent is $k_{ge}$ [S1][S2, S3, unofficial].
6.  **Rendering:** Use $o_g$ as the final opacity for the sample during volume integration [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_boundary_enhanced_opacity(v_value, gradient_magnitude, o_v, k_gc, k_gs, k_ge):
    """
    Computes the boundary-enhanced opacity for a single sample.
    
    Args:
        v_value (float): Scalar value of the voxel f(P_i).
        gradient_magnitude (float): Magnitude of the gradient ||∇f(P_i)||.
        o_v (float): Base opacity from transfer function or constant.
        k_gc (float): Base opacity scaling factor.
        k_gs (float): Gradient scaling factor.
        k_ge (float): Exponent for the opacity curve slope.
        
    Returns:
        float: Enhanced opacity o_g.
    """
    # Calculate the term inside the parenthesis
    # Note: Sources [S2, S3, unofficial] use gradient magnitude directly.
    # Source [S1] uses ||∇fn||, which is the normalized gradient magnitude (1 if non-zero),
    # but the context of "gradient magnitude" in [S2, S3, unofficial] and typical Levoy shading
    # implies the unnormalized magnitude is used for scaling strength.
    # We follow [S2, S3, unofficial] which explicitly links k_gs to the gradient magnitude.
    
    base_term = k_gc + k_gs * gradient_magnitude
    
    # Apply exponent k_ge
    # If base_term is negative (unlikely with positive k_gs and magnitude), clamp to 0
    if base_term < 0:
        base_term = 0.0
        
    enhanced_factor = base_term ** k_ge
    
    o_g = o_v * enhanced_factor
    
    return o_g
```

**Edge Cases and Pitfalls:**
*   **Zero Gradient:** In regions of constant density, $\|\nabla f\| = 0$. If $k_{gc} = 0$, the opacity becomes 0, making the region transparent [S1][S2, S3, unofficial]. If $k_{gc} > 0$, the region retains opacity proportional to $k_{gc}^{k_{ge}}$ [S1][S2, S3, unofficial].
*   **No Transfer Function:** If no transfer function is applied, $o_v$ must be explicitly set to a constant value (e.g., 1.0) to avoid zero opacity [S2, S3, unofficial].
*   **Formula Ambiguity:** Source [S1] contains a discrepancy where $k_{gc}$ is used as the exponent in the formula text, while $k_{ge}$ is described as the exponent in the parameters. Sources [S2, S3, unofficial] consistently use $k_{ge}$ as the exponent. Implementers should use $k_{ge}$ as the exponent [S1][S2, S3, unofficial].

## Subtopics

### [Silhouette Enhancement](796-silhouette-enhancement.md)
Enhances opacity based on gradient magnitude AND view direction to highlight silhouettes. Choose when surface shape perception is critical.

### [Isovalue Contour Surface Classification](1328-isovalue-contour-surface-classification.md)
Assigns opacity based on proximity to a specific scalar value. Choose when visualizing specific isosurfaces with smooth edges.

### [Classification and Shading](2278-classification-and-shading.md)
Maps voxel values to color/opacity with lighting. Choose for general volume rendering with shading.

### [Boundary Emphasis Slider](5644-boundary-emphasis-slider.md)
A UI control for adjusting second derivative-based opacity. Choose for interactive parameter tuning.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of samples, as it requires one gradient calculation and one opacity evaluation per sample.
*   **Memory Complexity:** $O(N)$ to store gradients if precomputed, or $O(1)$ if computed on the fly (though gradient computation requires neighbor access).
*   **Performance:** The method is computationally inexpensive, adding only a few arithmetic operations per sample. It is suitable for real-time rendering if gradients are precomputed or efficiently interpolated [S2, S3, unofficial].
*   **Strengths:** Effectively removes internal clutter in homogeneous regions (e.g., organ interiors) while highlighting boundaries (e.g., tissue interfaces) [S1][S2, S3, unofficial].
*   **Limitations:** Does not provide depth cues via view direction (unlike Silhouette Enhancement) [S1]. Sensitive to noise in the gradient calculation, which can create false boundaries [G].
*   **When to Use:** When the goal is to visualize the structure of a volume by emphasizing interfaces between different materials or densities, and internal homogeneity is not of interest [S1][S2, S3, unofficial].
*   **When Not to Use:** When the internal structure of homogeneous regions is important, or when view-dependent surface cues are required [S1][S2, S3, unofficial].

## Background
The technique was introduced by Marc Levoy in 1990 as "gradient-based shading" to emphasize boundaries between data of different densities [S2, S3, unofficial]. It was later adapted by Rheingans et al. in the early 2000s for non-photorealistic volume illustration, allowing user-selectable enhancement functions via parameters $k_{gc}$, $k_{gs}$, and $k_{ge}$ [S1][S2, S3, unofficial]. It is widely used in medical volume rendering to visualize anatomical structures like lungs and vasculature [S1][S2, S3, unofficial].

## Sources
[S1] Volume illustration: nonphotorealistic rendering of volume models, https://www.csee.umbc.edu/~rheingan/pubs/volillus.tvcg01.pdf, fetched 2026-10-27
[S2] Non-Photorealistic Rendering of Volume Models, https://www.cs.umbc.edu/~rheingan/pubs/volillus00.pdf, fetched 2026-10-27 (unofficial)
[S3] Non-Photorealistic Rendering of Volume Models, https://userpages.cs.umbc.edu/ebert/npr/volillus.pdf, fetched 2026-10-27 (unofficial)
