# Multi-Stage Rasterization for Shaded Isosurfaces

Also called: Fast Shaded Isosurfaces adaptation

## Overview
Multi-Stage Rasterization for Shaded Isosurfaces adapts the Fast Shaded Isosurfaces method for PC hardware using multi-stage rasterization and 2D textures to efficiently render shaded isosurfaces [S1]. It replaces iterative ray-marching with a two-pass register combiner pipeline that interpolates voxel gradients and computes dot-product lighting, extracting a specific isosurface boundary instead of integrating opacity [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset represented as a sequence of 2D slices containing scalar values and pre-computed voxel gradients [S1].
*   **Output:** A shaded 2D image of the isosurface projected onto the screen [S1].

**Data Structures:**
The volume is decomposed into 2D textures [S1].
*   **Texture 0:** Stores the voxel gradients in its RGB components [S1].
*   **Texture 1:** Stores the scalar intensity values in its Alpha component [S1].
*   **Constant Color Registers:** Store the light direction vector in register 2, the diffuse light color, and the ambient light color [S1].

**Numbered Steps:**
1.  **Gradient and Intensity Encoding:** Compute the voxel gradient for each voxel and write it to the RGB components of a 2D texture representing the volume [S1]. Code the scalar intensity into the Alpha component of the texture [S1].
2.  **First Pass: Interpolation (General Combiner 0):** Apply the first general combiner stage to interpolate intermediate slices [S1]. Map the RGB output of Texture 0 (gradient $\tilde{n}$) to variable A [S1]. The combiner interpolates the gradient vector across the screen space based on the fragment's position relative to the slice boundaries [S1]. The result is the interpolated normal $\tilde{n}_{interp}$ [S1].
3.  **Second Pass: Lighting Calculation (General Combiner 1):** Enable the second general combiner to compute the dot product $A \bullet B$ [S1]. Map the RGB output of Combiner 0 (the interpolated normal $\tilde{n}_{interp}$) to variable A [S1]. Map Constant Color Register 2 (the light vector $\tilde{l}$) to variable B [S1]. The operation computes the dot product $D = \tilde{n}_{interp} \cdot \tilde{l}$ in the RGB portion [S1]. The Alpha component is not modified by Combiner 1 and remains the interpolated intensity [S1].
4.  **Final Compositing:** Combine the diffuse lighting term with ambient lighting [S1]. The final combiner calculates the output color using the dot product and the constant color registers for diffuse and ambient light [S1].

**Formulas:**
*   $\tilde{n}$: voxel gradient vector.
*   $\tilde{l}$: light direction vector.
*   $I$: scalar intensity value.
*   $\alpha$: interpolation factor between slice $i$ and slice $i+1$.
*   $D$: dot product $\tilde{n}_{interp} \cdot \tilde{l}$.
*   $C_{diffuse}, C_{ambient}$: RGB light color vectors.
*   $C_{final}$: output pixel color.

**Parameters and Typical Values:**
*   **Light Vector $\tilde{l}$:** Direction depends on camera or scene setup [S1].
*   **Light Colors:** $C_{diffuse}$ and $C_{ambient}$ are RGB vectors [S1].
*   **Signed Fixed-Point:** The hardware supports signed fixed-point values for combiner inputs, allowing direct storage of negative gradient components without scaling and biasing [S1].

**Edge Cases and Pitfalls:**
*   **Single Light Source:** The method is restricted to a single light source due to the fixed-function dot product calculation in the combiner pipeline [S1].
*   **Monochrome Display:** The algorithm is restricted to monochrome display only [S1].
*   **Gradient Accuracy:** The quality of the shading depends on the accuracy of the pre-computed voxel gradients [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_shaded_isosurface(slices, light_vector, diffuse_color, ambient_color, alpha):
    """
    Simulates the multi-stage register combiner pipeline for shaded isosurfaces.
    slices: list of 2D arrays, each (H, W, 4). RGB=gradient, Alpha=intensity.
    light_vector: (3,) array. Mapped to constant color register 2.
    diffuse_color, ambient_color: (3,) arrays.
    alpha: float, interpolation factor between slice i and i+1.
    """
    final_image = None
    for i in range(len(slices) - 1):
        s_curr = slices[i]
        s_next = slices[i + 1]
        
        # Combiner 0: Interpolate intermediate slices
        # Input A: RGB from Texture 0 (gradient n)
        n_curr = s_curr[:, :, :3]
        n_next = s_next[:, :, :3]
        n_interp = (1.0 - alpha) * n_curr + alpha * n_next
        
        # Alpha output: interpolated intensity
        i_curr = s_curr[:, :, 3]
        i_next = s_next[:, :, 3]
        i_interp = (1.0 - alpha) * i_curr + alpha * i_next
        
        # Combiner 1: Compute dot product A . B
        # Input A: n_interp, Input B: light_vector (const color register 2)
        # RGB portion calculates dot product, Alpha portion is unused/unchanged
        D = n_interp[:, :, 0] * light_vector[0] + n_interp[:, :, 1] * light_vector[1] + n_interp[:, :, 2] * light_vector[2]
        
        # Final Combiner: C_final = D * C_diffuse + C_ambient
        final_rgb = D[:, :, None] * diffuse_color + ambient_color
        
        final_image = final_rgb
    return final_image
```

## Subtopics

### [K-Buffer Isosurface Extraction](6210-k-buffer-isosurface-extraction.md)
Extracts flat-shaded isosurfaces by adapting the K-Buffer rasterization technique to replace 3D pre-integrated volume tables with a 2D lookup table for boolean detection between front and back scalar values. It is chosen when simple, fast hardware boolean extraction is preferred over continuous gradient-based shading. See [K-Buffer Isosurface Extraction](6210-k-buffer-isosurface-extraction.md).

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels in the screen space projection [S1].
*   **Performance:** Enables interactive frame rates for shaded isosurfaces on standard PC hardware [S1].
*   **Strengths:** Efficient use of fixed-function GPU hardware; avoids software ray-marching [S1].
*   **Limitations:** Restricted to single light sources; limited to monochrome display; requires pre-computation of gradients [S1].

## Background
The method is an adaptation of the "Fast Shaded Isosurfaces" algorithm, designed to leverage the capabilities of standard PC graphics hardware (specifically OpenGL register combiners) for volume visualization [S1]. It was developed to provide efficient rendering of shaded isosurfaces at interactive frame rates [S1].

## Sources
[S1] Interactive Volume Rendering on Standard PC Graphics Hardware, https://courses.cs.duke.edu/spring03/cps296.8/papers/VolVisforPCHardware.pdf, fetched 2026-10-28
