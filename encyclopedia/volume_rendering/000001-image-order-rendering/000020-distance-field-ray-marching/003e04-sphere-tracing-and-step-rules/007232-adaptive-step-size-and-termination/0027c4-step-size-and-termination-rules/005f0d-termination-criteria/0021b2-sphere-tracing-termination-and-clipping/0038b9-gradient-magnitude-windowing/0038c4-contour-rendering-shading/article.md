# Contour Rendering Shading

Also called: Gradient-based Contour Shading, View-dependent Contour Shading

## Overview
Contour Rendering Shading is a view-dependent shading technique that highlights object surfaces by weighting ray samples based on the alignment between the view vector and the surface gradient [S1, unofficial][S2]. It solves the problem of visualizing distinct structural boundaries (contours) in volume data, particularly where conventional opacity transfer functions fail to separate structures with different density ranges, such as internal organs or cavities [S2]. Unlike standard shading methods that rely on lighting models, this method produces a non-photorealistic line-art effect where intensity is maximized when the view direction is tangent to the surface (gradient perpendicular to view) [S1, unofficial][S2].

## How it works
**Inputs:**
*   Ray sample position $P$ [G].
*   Normalized view vector $V$ (from camera to sample) [S1, unofficial].
*   Gradient vector $G = \nabla f(P)$ at position $P$, where $f$ is the scalar volume field [G].
*   Gradient magnitude $|g| = \|G\|$ [S1, unofficial].
*   Windowing function $w(|g|)$ output from the parent method [Gradient Magnitude Windowing](8299-gradient-magnitude-windowing.md) [S1, unofficial].
*   Exponent parameter $n$ [S1, unofficial].

**Outputs:**
*   Contour intensity $I_c$ [S1, unofficial].
*   (Optional) Opacity $\alpha_c$ for compositing [S1, unofficial].

**Data Structures:**
*   Vectors $V$ and $G$ are 3D floating-point vectors [G].
*   Scalar $I_c$ is a single floating-point value [G].

**Steps:**
1.  **Normalize Vectors:** Ensure $V$ and $G$ are unit vectors. If $G$ is zero, the contour intensity is zero [S1, unofficial].
    $$ \hat{V} = \frac{V}{\|V\|}, \quad \hat{G} = \frac{G}{\|G\|} $$
2.  **Calculate Dot Product:** Compute the dot product $\kappa$ between the normalized view vector and the normalized gradient vector [S1, unofficial].
    $$ \kappa = \hat{V} \cdot \hat{G} $$
    $\kappa = 1$ implies the view vector is aligned with the gradient (looking directly at the surface normal) [G]. $\kappa \approx 0$ implies the view vector is perpendicular to the gradient (tangent to the surface), indicating a contour edge [S1, unofficial].
3.  **Calculate Contour Weight:** Compute the view-dependent weight $s(\kappa)$ using the exponent $n$ [S1, unofficial].
    $$ s(\kappa) = (1 - \kappa)^n $$
    This term assigns high weights to samples where $\kappa \approx 0$ (contours) and low weights where $\kappa \approx 1$ [S1, unofficial].
4.  **Combine with Windowing:** Multiply the contour weight by the gradient magnitude windowing function $w(|g|)$ from the parent method to produce the final intensity [S1, unofficial].
    $$ I_c = w(|g|) \cdot s(\kappa) $$
    $w(|g|)$ ensures that only samples with significant gradient magnitudes (likely on a surface) contribute [S1, unofficial][S2].
5.  **Assign Opacity (Optional):** The intensity $I_c$ can be used directly as the opacity $\alpha$ for the ray sample during compositing, ensuring contour lines are more opaque than non-contour regions [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def contour_shading(V, G, w_gradient_magnitude, n):
    """
    Calculate contour intensity for a ray sample.
    
    Args:
        V: Normalized view vector (3D float).
        G: Gradient vector at sample point (3D float).
        w_gradient_magnitude: Output from Gradient Magnitude Windowing function.
        n: Exponent controlling contour thickness.
        
    Returns:
        Ic: Contour intensity.
    """
    # Step 1: Normalize gradient if non-zero
    if G[0] == 0 and G[1] == 0 and G[2] == 0:
        return 0.0
    
    mag_G = (G[0]**2 + G[1]**2 + G[2]**2) ** 0.5
    G_hat = (G[0] / mag_G, G[1] / mag_G, G[2] / mag_G)
    # V is assumed normalized per source requirements [S1, unofficial]
    
    # Step 2: Calculate dot product kappa
    kappa = V[0] * G_hat[0] + V[1] * G_hat[1] + V[2] * G_hat[2]
    
    # Step 3: Calculate contour weight
    # Source [S1, unofficial] defines s(kappa) = (1 - kappa)^n.
    # [G] Clamp base to non-negative to prevent complex results for fractional n
    base = max(0.0, 1.0 - kappa)
    s_kappa = base ** n
    
    # Step 4: Combine with windowing
    Ic = w_gradient_magnitude * s_kappa
    
    return Ic
```

**Parameters:**
*   $n$: Exponent controlling contour line thickness/sharpness. Higher $n$ results in thinner, sharper contours [S1, unofficial][S2].
    *   Typical Value: For 12-bit gradient quantization, $n \approx 4$ is recommended to avoid artifacts from quantization errors [S2]. No other typical value is given [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Normalization:** Vectors $V$ and $G$ must be normalized before calculating $\kappa$ to ensure $\kappa$ is in the expected range and the dot product accurately reflects the angle [S1, unofficial].
*   **Zero Gradient:** If $G = 0$, the gradient direction is undefined and the intensity is zero [S1, unofficial]. Voxels with zero gradient magnitude are typically skipped by the windowing function [S2].
*   **Quantization Artifacts:** With low-bit gradient quantization (e.g., 12-bit), high values of $n$ amplify quantization errors near contours, causing bright/dark artifacts. Use $n \approx 4$ for 12-bit data [S2].
*   **View Dependence:** The term $(1 - \kappa)^n$ is view-dependent, making pre-computation or standard empty-space skipping techniques (which rely on static voxel properties) infeasible [S2].

## Subtopics

### Non-Photorealistic Volume Rendering (NPVR) Contour Model
A specific application of contour rendering shading for visualizing object contours in medical CT scans, such as internal organs (human body, vertebrae with screws) and cavities (bronchia, frontal sinuses) [S2]. It allows simultaneous visualization of structures with different density ranges where conventional transfer functions are difficult to tune [S2]. It uses the formula $g(|\nabla(P)|) = 0$ to skip voxels with no gradient, but due to view dependence, efficient skipping is challenging [S2]. It employs an exponent $n$ (typically around 4 for 12-bit quantization) to control contour sharpness [S2].

## Operating characteristics
*   **Time Complexity:** The shading calculation is $O(1)$ per sample [G]. However, because the contour weight $s(\kappa)$ is view-dependent, conventional empty-space skipping techniques (which rely on static gradient magnitude thresholds) are infeasible, potentially increasing the number of samples processed [S2].
*   **Memory Complexity:** $O(1)$ additional memory per sample [G].
*   **Performance:** Rendering times depend on the compositing strategy. Maximum Intensity Projection (MIP) based contour rendering is faster than blending because it allows more efficient skipping of black voxels and simpler compositing [S2]. Example performance on a $256^2 \times 225$ head scan: 85ms for MIP, 150ms for blending [S2].
*   **Strengths:** Effective for visualizing boundaries and contours in complex, multi-density volumes (e.g., medical images) where standard opacity transfer functions fail [S2].
*   **Limitations:** View-dependent nature prevents static optimization of voxel skipping [S2]. Sensitive to gradient quantization errors if $n$ is too high [S2].
*   **When to Use:** When the goal is to highlight object boundaries and contours in non-photorealistic rendering, especially for medical imaging with mixed density structures [S2].
*   **When Not to Use:** When photorealistic lighting is required, or when performance is critical and the volume has large empty regions that could be skipped using static methods (since view-dependence prevents this) [S2].

## Background
Contour rendering shading was introduced as part of non-photorealistic volume rendering techniques to address the limitations of conventional transfer functions in visualizing specific structural features like contours [S1, unofficial][S2]. It builds on the concept of using gradient magnitude to identify surfaces and adds view-dependence to distinguish between front-facing surfaces and silhouette contours [S1, unofficial]. The method was developed in the context of fast visualization of object contours, particularly for medical imaging applications such as CT scans of internal organs and cavities [S2]. The specific formulation with the exponent $n$ and the combination with gradient windowing is detailed in research from the mid-2000s [S1, unofficial][S2].

## Sources
[S1] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
[S2] Fast Visualization of Object Contours by Non-Photorealistic Volume Rendering, https://doi.org/10.1111/1467-8659.00538, fetched 2026-11-01
