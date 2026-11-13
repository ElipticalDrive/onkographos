# Volume Illustration Tone Shading

Also called: Gooch tone shading for volumes, Modified Gooch tone shading for volume, Modified Gooch tone shading for volume models, Tone Shading, Tone illumination model, Volumetric tone shading

## Overview
Volume Illustration Tone Shading is a non-photorealistic illumination model that enhances shape perception in volume rendering by mapping the orientation of volume samples relative to a light source to a warm-to-cool color gradient [S1][S2][S3, unofficial]. It modifies the final color by interpolating between a warm color (for surfaces facing the light) and a cool color (for surfaces facing away), based on the signed dot product between the volume sample gradient and the light vector [S1][S2][S3, unofficial]. Unlike standard Gooch shading, this method restricts the directed illumination contribution to positive dot products only, becoming zero at orientations orthogonal to the light vector, which more closely matches familiar diffuse illumination models [S2][S3, unofficial]. It builds on [Illumination and Tone Shading Models](3857-illumination-and-tone-shading-models.md) by combining gaseous illumination with tone-based shading and directed shading [S2].

## How it works
**Inputs:**
*   Volume sample gradient $\nabla f_n$ [S2]
*   Light direction vector $L$ (unit vector) [S3, unofficial]
*   Illuminated gaseous color $I_G$ (result of transfer function and gaseous illumination calculations) [S3, unofficial]
*   Illuminated object color contribution $I_i$ (base color of the object, typically derived from transfer functions) [S3, unofficial]
*   Parameters: $k_{ta}$ (gaseous illumination weight), $k_{td}$ (directed illumination weight), $k_{ty}$ (warm tone yellow intensity), $k_{tb}$ (cool tone blue intensity) [S2]
*   Number of light sources $N_L$ [S3, unofficial]

**Outputs:**
*   Final color $c$ for the volume sample [S3, unofficial]

**Data Structures:**
*   Colors are represented as 3-component vectors $(R, G, B)$ [G]
*   Vectors are normalized where specified [G]

**Formulas:**
1.  **Warm Color ($c_w$):**
    $$c_w = (k_{ty}, k_{ty}, 0)$$
2.  **Cool Color ($c_c$):**
    $$c_c = (0, 0, k_{tb})$$
3.  **Tone Contribution ($I_t$):**
    $$I_t = \left(\frac{1.0 + \nabla f_n \cdot L}{2}\right) c_w + \left(1 - \frac{1.0 + \nabla f_n \cdot L}{2}\right) c_c$$
    *   Note: The term $\frac{1.0 + \nabla f_n \cdot L}{2}$ acts as the interpolation factor between $c_w$ and $c_c$.
4.  **Directed Illumination Component ($I_o$):**
    $$I_o = \begin{cases} k_{td} I_i (\nabla f_n \cdot L) & \text{if } \nabla f_n \cdot L > 0 \\ 0 & \text{if } \nabla f_n \cdot L \le 0 \end{cases}$$
5.  **Final Color ($c$):**
    $$c = k_{ta} I_G + \sum_{i=1}^{N_L} (I_t + I_o)$$
    *   Note: In the summation, $I_t$ and $I_o$ are calculated per light source using that light's specific $L_i$. The parameter $k_{td}$ is applied inside the calculation of $I_o$ for each light. (Note: The original source contains a typographical error in the final summation formula, repeating $k_{td}$; the implementation below and the definition of $I_o$ are consistent.)

**Parameters and Typical Values:**
*   $k_{ta}$: Controls the amount of gaseous illumination included. Typical value: 1.0 [S2][S3, unofficial].
*   $k_{td}$: Controls the amount of directed illumination included. Typical value: 0.6 [S2][S3, unofficial].
*   $k_{ty}$: Component of the warm tone color vector $c_w$. Typical value: 0.3 [S2][S3, unofficial].
*   $k_{tb}$: Component of the cool tone color vector $c_c$. Typical value: 0.3 [S2][S3, unofficial].
*   $N_L$: Number of light sources. The sources note that using more than a few lights is likely to result in confusing images [S2][S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_illustration_tone_shading(gradient, light_directions, I_G, I_i, k_ta, k_td, k_ty, k_tb):
    """
    Calculate the final color for a volume sample using Volume Illustration Tone Shading.
    
    Args:
        gradient: Gradient vector of the volume sample (vec3).
        light_directions: List of unit light direction vectors (list of vec3).
        I_G: Illuminated gaseous color (vec3).
        I_i: Illuminated object color contribution (vec3).
        k_ta: Gaseous illumination weight (float).
        k_td: Directed illumination weight (float).
        k_ty: Warm tone yellow intensity (float).
        k_tb: Cool tone blue intensity (float).
        
    Returns:
        Final color (vec3).
    """
    # Define warm and cool colors
    c_w = vec3(k_ty, k_ty, 0.0)
    c_c = vec3(0.0, 0.0, k_tb)
    
    total_tone_and_directed = vec3(0.0, 0.0, 0.0)
    
    for L in light_directions:
        # Calculate dot product
        dot_product = dot(gradient, L)
        
        # Calculate Tone Contribution (I_t)
        # Interpolation factor
        t_factor = (1.0 + dot_product) / 2.0
        I_t = t_factor * c_w + (1.0 - t_factor) * c_c
        
        # Calculate Directed Illumination Component (I_o)
        if dot_product > 0:
            I_o = k_td * I_i * dot_product
        else:
            I_o = vec3(0.0, 0.0, 0.0)
            
        total_tone_and_directed += (I_t + I_o)
        
    # Final Color
    c = k_ta * I_G + total_tone_and_directed
    
    return c
```

**Edge Cases and Pitfalls:**
*   **Multiple Lights:** Using more than a few light sources is likely to result in confusing images because humans are not used to interpreting complex illumination from many lights [S2][S3, unofficial].
*   **Orthogonal Orientations:** The directed illumination component $I_o$ becomes zero when the gradient is orthogonal to the light vector ($\nabla f_n \cdot L \le 0$). This is a deliberate design choice to match diffuse illumination models, unlike standard Gooch shading which might use the full signed dot product for the object contribution [S2][S3, unofficial].

## Subtopics

### [Gooch Shading](1670-gooch-shading.md)
Gooch Shading is a non-photorealistic illumination model for volume rendering that maps surface orientation relative to a light source to a warm-to-cool color gradient to convey shape. It is distinguished from standard Gooch shading by its specific integration with on-the-fly central difference normal estimation in rasterization pipelines, ensuring consistent visual output across different extraction methods (rasterization vs. ray-marching).

### Cool to Warm Shading
A tone-based shading model that blends between cool colors in regions facing away from the light and warm colors in regions facing the light source, often used with silhouette edges [S5, unofficial]. Unlike physically based shading (e.g., Phong), which may lose shape information in regions facing away from the light, cool-to-warm shading enhances perception of shape [S5, unofficial].

## Operating characteristics
*   **Time and memory complexity:** Not given in the sources.
*   **Measured performance:** Not given in the sources.
*   **Strengths:** Enhances shape perception by providing warm/cool cues based on orientation. Combines well with gaseous illumination and transfer functions [S3, unofficial]. The modification to use only positive dot products for directed illumination provides a more familiar diffuse lighting appearance [S2][S3, unofficial].
*   **Limitations:** Can become confusing with too many light sources [S2][S3, unofficial]. Relies on accurate gradient estimation.
*   **When to use:** When the goal is to enhance the perception of 3D shape and internal structures in volume data using a painterly, illustrative style [S2].
*   **When not to use:** When photorealism is required, or when the scene complexity with multiple lights would lead to visual confusion [S3, unofficial].

## Background
Volume Illustration Tone Shading is based on the Gooch tone shading model proposed by Gooch et al. [Gooch98] [S2]. It was adapted for volume rendering by implementing an illumination model similar to Gooch tone shading but modified for volume samples, specifically by using only the positive dot product for the illuminated object contribution [S2][S3, unofficial]. The method is part of the broader "Volume Illustration" framework, which combines direct volume rendering with non-photorealistic rendering techniques to produce a wide range of illustration styles [S2]. Examples of its application include medical volumes (liver, kidney, tomato MRI) and scientific data (iron protein) [S2].

## Sources
[S1] Interactive Multi-volume Visualization, https://doi.org/10.1007/3-540-46080-2_11, fetched 2026-10-26
[S2] PDF Volume illustration: nonphotorealistic rendering of volume models ..., https://www.csee.umbc.edu/~rheingan/pubs/volillus.tvcg01.pdf, fetched 2026-10-27
[S3] Non-Photorealistic Rendering of Volume Models, https://www.cs.umbc.edu/~rheingan/pubs/volillus00.pdf, fetched 2026-10-27 (unofficial)
[S4] Non-Photorealistic Rendering of Volume Models, https://userpages.cs.umbc.edu/ebert/npr/volillus.pdf, fetched 2026-10-27 (unofficial)
[S5] Illustrative Rendering of Particle Systems, https://doi.org/10.2312/pe.vmv.vmv13.177-185, fetched 2026-10-26 (unofficial)
