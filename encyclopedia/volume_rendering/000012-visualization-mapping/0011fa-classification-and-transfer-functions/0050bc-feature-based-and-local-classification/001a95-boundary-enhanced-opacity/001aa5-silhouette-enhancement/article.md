# Silhouette Enhancement

Also called: Oriented Feature Enhancement, Silhouette Curves, Silhouettes In Volumes, View-Vector Silhouette, silhouette volumes

## Overview
Silhouette Enhancement is a technique to draw outlines or sketches of objects in volume data by detecting silhouettes, which are strongest where the view vector is orthogonal to the surface normal vector [S1]. It increases the opacity of volume samples where the gradient is nearly perpendicular to the view direction, strengthening silhouette cues for surface shape perception [S3][S2, unofficial].

## How it works
**Inputs and Outputs:** The method takes the volume scalar field, a normalized view direction vector $V$, the original sample opacity $o_v$, and optionally the original sample color $C_v$ as inputs, and outputs the enhanced opacity $o_s$ and optionally modified color $C_s$ [G].

**Data Structures:** The method uses the volume gradient direction to approximate the surface normal and the volume gradient magnitude to detect boundaries [S1]. The normalized gradient $\nabla f_n$ is computed per sample [G].

**Numbered Steps:**
1. **Compute Normal:** Approximate the surface normal using the volume gradient direction [S1].
2. **Compute Silhouette Factor:** Calculate the alignment between the normalized gradient $\nabla f_n$ and the view direction $V$ [S4, unofficial]. The silhouette factor is defined as $S = 1 - |\nabla f_n \cdot V|$ [S3][S4, unofficial]. The dot product between the gradient and view direction nears zero when they are perpendicular [S3].
3. **Compute Enhanced Opacity:** Apply the silhouette enhancement formula to the original opacity $o_v$ [S3][S4, unofficial].
4. **Modify Color (Optional):** To create sketch effects, modify the sample color using white or black sketch colors and make non-volumetric silhouettes transparent [S3]. Shadows can be calculated based on the original volume opacity $o_v$ [S3].

**Formulas:** The silhouette-enhanced opacity is calculated as:
$$o_s = o_v (k_{sc} + k_{ss} (1 - |\nabla f_n \cdot V|))^{k_{se}}$$
where $o_s$ is the silhouette-enhanced opacity, $o_v$ is the original opacity, $\nabla f_n$ is the normalized value gradient, $V$ is the view direction vector, $k_{sc}$ controls the scaling of non-silhouette regions, $k_{ss}$ controls the amount of silhouette enhancement, and $k_{se}$ controls the sharpness of the silhouette curve [S3][S4, unofficial]. Note: Source [S2, unofficial] contains a typo in the exponent notation in the text description ($k_{sc}$ instead of $k_{se}$), but the parameter list and other sources [S3][S4, unofficial] confirm the exponent is $k_{se}$ [S2, unofficial].

**Parameters:** $k_{sc}$: Controls scaling of non-silhouette regions. Typical value: 0.9 [S3][S4, unofficial]. $k_{ss}$: Controls the amount of silhouette enhancement. Typical value: 50 [S3][S4, unofficial]. $k_{se}$: Controls the sharpness of the silhouette curve. Typical value: 0.25 [S3][S4, unofficial].

**Edge Cases and Pitfalls:** **Zero Gradient:** In regions of constant density, the gradient is zero and the normalized gradient is undefined; the implementation must handle this, typically by returning the original opacity or skipping enhancement [G]. **View Direction:** The view direction $V$ must be normalized to ensure correct dot product scaling [G]. **Parameter Sensitivity:** High $k_{ss}$ values can cause opacity to exceed 1.0 or clip [G]. The exponent $k_{se}$ affects the contrast of the silhouette lines; low values (e.g., 0.25) create softer transitions, while higher values create sharper lines [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def silhouette_enhance(gradient_magnitude, gradient_vector, view_direction, original_opacity, k_sc, k_ss, k_se):
    """
    Computes silhouette-enhanced opacity for a single volume sample.
    
    Args:
        gradient_magnitude: float, magnitude of the volume gradient ||grad(f)||
        gradient_vector: tuple (gx, gy, gz), the volume gradient vector
        view_direction: tuple (vx, vy, vz), normalized view vector
        original_opacity: float, the base opacity o_v
        k_sc: float, scaling factor for non-silhouette regions
        k_ss: float, enhancement amount
        k_se: float, sharpness exponent
        
    Returns:
        float: enhanced opacity o_s
    """
    # Step 1: Compute normalized gradient
    # Guard against zero magnitude to avoid division by zero [G]
    if gradient_magnitude < 1e-9:
        return original_opacity
    
    nx = gradient_vector[0] / gradient_magnitude
    ny = gradient_vector[1] / gradient_magnitude
    nz = gradient_vector[2] / gradient_magnitude
    
    # Step 2: Compute silhouette factor S = 1 - |grad_n . view|
    dot_product = nx * view_direction[0] + ny * view_direction[1] + nz * view_direction[2]
    silhouette_factor = 1.0 - abs(dot_product)
    
    # Step 3: Compute enhanced opacity
    # Formula: o_s = o_v * (k_sc + k_ss * S) ^ k_se
    base_term = k_sc + k_ss * silhouette_factor
    enhanced_opacity = original_opacity * (base_term ** k_se)
    
    return enhanced_opacity
```

## Subtopics

### Sketching
A specific application where non-silhouette regions are made transparent and silhouette regions are colored white or black to create a pen-and-ink look. Shadows are calculated based on original opacity [S3].

### Edge Coloring
Modifying the color of samples based on the silhouette term, often darkening edges [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, assuming the gradient is pre-computed or computed via finite differences [G].
*   **Memory Complexity:** $O(1)$ additional memory per sample for storing the normalized gradient if pre-computed [G].
*   **Strengths:** Effectively emphasizes surface shape and orientation cues, which are critical for perceiving 3D structure in volumes [S2, unofficial]. Can create artistic sketch styles [S3].
*   **Limitations:** Relies on the accuracy of the gradient estimation. Noisy data can produce noisy silhouettes [G]. Performance is similar to standard volume rendering but with slightly higher computational cost due to gradient and dot product calculations [G].
*   **When to use:** When the goal is to emphasize boundaries and surface orientation, particularly in medical illustration or artistic visualization [S1].
*   **When not to use:** When photorealism is required, or when the volume data is very noisy and gradient-based methods fail [G].

## Background
The technique was introduced by Ebert and Rheingans in 2000 [S1]. It is inspired by the art of silhouette drawing, named after Étienne de Silhouette (1709–1767) [S1]. The method extends surface silhouette techniques to volumes by using the volume gradient to approximate surface normals [S1]. It is commonly used in medical illustration to convey shape and structure [S1].

## Sources
[S1] Volume illustration for medicine and flows, https://doi.org/10.1145/1185657.1185694, fetched 2026-10-28
[S2] PDF Volume illustration: nonphotorealistic rendering of volume models ..., https://www.csee.umbc.edu/~rheingan/pubs/volillus.tvcg01.pdf, fetched 2026-10-27 (unofficial)
[S3] Non-Photorealistic Rendering of Volume Models, https://www.cs.umbc.edu/~rheingan/pubs/volillus00.pdf, fetched 2026-10-27
[S4] Non-Photorealistic Rendering of Volume Models, https://userpages.cs.umbc.edu/ebert/npr/volillus.pdf, fetched 2026-10-27 (unofficial)
