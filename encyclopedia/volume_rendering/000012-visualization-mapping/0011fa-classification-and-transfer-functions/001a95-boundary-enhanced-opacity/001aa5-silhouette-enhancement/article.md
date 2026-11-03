# Silhouette Enhancement

Also called: Oriented Feature Enhancement: Silhouettes

## Overview
Silhouette Enhancement is a technique used in volume rendering to increase the opacity of volume samples where the local gradient is nearly perpendicular to the view direction, thereby enhancing the perception of surface shape and silhouettes [S1]. It distinguishes itself from its parent, Boundary-Enhanced Opacity, which emphasizes boundaries between data of different densities based on gradient magnitude alone, by adding a directional component that specifically targets the visual cues of object edges relative to the camera [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $o_v$: The original opacity value of the current volume sample, derived from the classification and transfer function [S1].
    *   $\nabla f_n$: The normalized gradient vector of the scalar field at the current sample position [S1].
    *   $V$: The normalized view direction vector from the sample point to the camera [S1].
    *   $k_{sc}$: Parameter controlling the scaling of non-silhouette regions [S1].
    *   $k_{ss}$: Parameter controlling the amount of silhouette enhancement [S1].
    *   $k_{se}$: Parameter controlling the sharpness of the silhouette curve [S1].
*   **Outputs:**
    *   $o_s$: The silhouette-enhanced opacity value used for compositing [S1].

**Data Structures**
Not given in the sources.

**Steps**
1.  **Compute Normalized Gradient:** Calculate the gradient $\nabla f$ of the scalar field at the current sample position and normalize it to obtain $\nabla f_n$ [S1].
2.  **Compute Silhouette Factor:** Calculate the silhouette factor $S$ based on the alignment between the normalized gradient and the view direction [S1]. The source defines this using the dot product: $S = 1 - |\nabla f_n \cdot V|$ [S1]. A dot product near zero indicates orthogonality (perpendicularity) [G]. The formula uses the absolute value of the dot product subtracted from 1, yielding a value near 1 when the vectors are perpendicular (silhouette) and near 0 when they are parallel (facing the viewer) [S1].
3.  **Apply Enhancement Formula:** Compute the final opacity $o_s$ using the formula $o_s = o_v (k_{sc} + k_{ss} S)^{k_{se}}$ [S1]. Substituting $S$ gives $o_s = o_v (k_{sc} + k_{ss} (1 - |\nabla f_n \cdot V|))^{k_{se}}$ [S1].
4.  **Use Enhanced Opacity:** Use $o_s$ as the opacity for the current sample during ray casting or compositing [S1].

**Parameters and Typical Values**
Typical parameter values cited in the source for medical volume visualization are $k_{sc} = 0.9$, $k_{ss} = 50$, and $k_{se} = 0.25$ [S1].

**Edge Cases and Pitfalls**
*   **Gradient Noise:** Since the method relies on the gradient direction, noisy data can lead to erratic silhouette enhancement [S1]. Smoothing the scalar field before gradient computation may be necessary [G].
*   **View Direction Normalization:** The view direction vector $V$ must be normalized for the dot product to correctly represent the cosine of the angle [G].
*   **Formula Ambiguity:** The source text mentions "cross product" in the notes but provides a dot product formula in the text and figure caption [S1]. The implementation should follow the explicit formula provided in the source text: $(1 - |\nabla f_n \cdot V|)$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def silhouette_enhance(o_v, gradient, view_direction, k_sc, k_ss, k_se):
    """
    Applies silhouette enhancement to volume opacity.
    
    Args:
        o_v (float): Original opacity from transfer function.
        gradient (Vector3): Unnormalized gradient of the scalar field.
        view_direction (Vector3): Normalized view direction from sample to camera.
        k_sc (float): Scaling factor for non-silhouette regions.
        k_ss (float): Amount of silhouette enhancement.
        k_se (float): Sharpness of the silhouette curve.
        
    Returns:
        float: Enhanced opacity o_s.
    """
    def dot(a, b):
        return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]
    
    def magnitude(v):
        return (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    
    # Step 1: Normalize gradient
    grad_norm = magnitude(gradient)
    if grad_norm == 0:
        return o_v
    
    grad_n = [gradient[0]/grad_norm, gradient[1]/grad_norm, gradient[2]/grad_norm]
    
    # Step 2: Compute silhouette factor S
    dot_product = dot(grad_n, view_direction)
    S = 1.0 - abs(dot_product)
    
    # Step 3: Compute enhanced opacity
    base_factor = k_sc + k_ss * S
    o_s = o_v * (base_factor ** k_se)
    
    return o_s
```

## Subtopics

### Isovalue Contour Surface Classification
Emphasizes specific isovalue surfaces rather than gradient-based silhouettes. It sets itself apart by targeting exact density thresholds instead of directional cues [G]. Choose it when distinct surface boundaries are required rather than general shape cues [G]. [Isovalue Contour Surface Classification](1328-isovalue-contour-surface-classification.md)

### Classification and Shading
Combines opacity classification with surface shading techniques. It sets itself apart by adding lighting models to opacity mapping [G]. Choose it when lighting cues are needed in addition to silhouette enhancement [G]. [Classification and Shading](2278-classification-and-shading.md)

## Operating characteristics
*   **Time Complexity**: The method adds a constant number of vector operations (gradient normalization, dot product, exponentiation) per sample [G]. It does not increase the asymptotic complexity of the ray casting loop [G].
*   **Memory Complexity**: No additional memory is required beyond the storage for the gradient vector and the view direction [G].
*   **Strengths**: Effectively enhances the perception of surface shape and internal structures (e.g., honeycomb structures in liver tissue) by emphasizing edges perpendicular to the view [S1].
*   **Limitations**: Relies on accurate gradient computation; noisy data may produce artifacts [S1]. The effect is view-dependent, meaning the appearance changes as the camera rotates [S1].
*   **When to Use**: Use when the goal is to improve the readability of volumetric features and surface contours, particularly in medical imaging or scientific visualization where internal structure visibility is critical [S1].
*   **When Not to Use**: Avoid if the data is extremely noisy without prior smoothing, or if a view-independent representation is required [G].

## Background
Silhouette Enhancement was introduced in the context of Non-Photorealistic Rendering (NPR) of volume models [S1]. It draws inspiration from surface illustration techniques [S1]. The method was proposed to strengthen the cues provided by silhouette volumes, which increase the perception of volumetric features [S1]. It was applied to medical volume data, such as liver and kidney structures, to reveal fine internal details [S1].

## Sources
[S1] Non-Photorealistic Rendering of Volume Models, https://userpages.cs.umbc.edu/ebert/npr/volillus.pdf, fetched 2026-10-27
