# Multi-Stage Rasterization for Shaded Isosurfaces

Also called: Fast Shaded Isosurfaces adaptation

## Overview
Multi-Stage Rasterization for Shaded Isosurfaces is a hardware-accelerated technique for rendering shaded isosurfaces from volumetric data on standard PC graphics hardware. It adapts the "Fast Shaded Isosurfaces" algorithm by utilizing multi-stage rasterization and 2D textures to compute surface normals and lighting interactions efficiently. Unlike general volume rendering methods that integrate opacity, this method focuses on extracting and shading a specific isosurface boundary. It is distinct from [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) by replacing iterative ray-marching with a two-pass rasterization pipeline that leverages the GPU's fixed-function register combiners for gradient interpolation and dot-product shading.

## How it works
The method relies on a two-pass rendering pipeline using the OpenGL register combiner architecture. It assumes the existence of a scalar field volume where the isosurface is defined by a threshold value.

**Inputs and Outputs:**
*   **Input:** A volumetric dataset represented as a sequence of 2D slices. Each slice contains scalar values (intensity) and pre-computed voxel gradients.
*   **Output:** A shaded 2D image of the isosurface projected onto the screen.

**Data Structures:**
The volume is decomposed into 2D textures.
*   **Texture 0:** Stores the voxel gradients. The RGB components hold the gradient vector $\tilde{n} = (n_x, n_y, n_z)$.
*   **Texture 1:** Stores the scalar intensity values. The Alpha component holds the intensity $I$. The RGB components are unused or set to zero.
*   **Constant Color Registers:**
    *   `const color 1`: Stores the light direction vector $\tilde{l}$.
    *   `const color 2`: Stores the diffuse light color $C_{diffuse}$.
    *   `const color 3`: Stores the ambient light color $C_{ambient}$.

**Numbered Steps:**

1.  **Gradient and Intensity Encoding:**
    Compute the voxel gradient $\tilde{n}$ for each voxel in the volume. Encode $\tilde{n}$ into the RGB components of a 2D texture representing the volume slice. Encode the scalar intensity $I$ into the Alpha component of a separate 2D texture. Note that general combiner stages support signed fixed-point values, so no scaling or biasing is required for the gradient components [S1, unofficial].

2.  **First Pass: Interpolation (General Combiner 0):**
    Render the volume slices using a multi-texture interpolation technique.
    *   **Input A:** RGB output from Texture 0 (gradient $\tilde{n}$).
    *   **Input B:** RGB output from Texture 1 (intensity $I$, mapped to alpha).
    *   **Operation:** Interpolate the gradient vector $\tilde{n}$ across the screen space based on the fragment's position relative to the slice boundaries. The result is the interpolated normal $\tilde{n}_{interp}$.
    *   **Output:** The RGB output of Combiner 0 is $\tilde{n}_{interp}$. The Alpha output is the interpolated intensity.

3.  **Second Pass: Lighting Calculation (General Combiner 1):**
    Enable the second general combiner to compute the diffuse lighting term.
    *   **Input A:** RGB output of Combiner 0 (the interpolated normal $\tilde{n}_{interp}$).
    *   **Input B:** Constant Color Register 1 (the light vector $\tilde{l}$).
    *   **Operation:** Compute the dot product $D = \tilde{n}_{interp} \cdot \tilde{l}$.
    *   **Output:** The RGB output of Combiner 1 is the dot product $D$. The Alpha component is not modified by Combiner 1 and remains the interpolated intensity from the previous stage.

4.  **Final Compositing:**
    Combine the diffuse lighting term with ambient lighting.
    *   **Input A:** RGB output of Combiner 1 (dot product $D$).
    *   **Input B:** Constant Color Register 2 (diffuse light color $C_{diffuse}$).
    *   **Input C:** Constant Color Register 3 (ambient light color $C_{ambient}$).
    *   **Operation:** Calculate final color $C_{final} = D \cdot C_{diffuse} + C_{ambient}$.
    *   **Output:** The final rendered pixel color.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode representing the register combiner setup
# Note: This is a conceptual mapping of the hardware operations described in [S1, unofficial]

def setup_combiners():
    # Combiner 0: Interpolate gradient
    combiner_0_input_A = texture_0_rgb  # Gradient n
    combiner_0_input_B = texture_1_alpha # Intensity I (used for interpolation factor)
    combiner_0_output_rgb = interpolate(combiner_0_input_A, combiner_0_input_B)
    # combiner_0_output_alpha is also interpolated intensity

    # Combiner 1: Compute dot product with light vector
    combiner_1_input_A = combiner_0_output_rgb # Interpolated normal n_interp
    combiner_1_input_B = const_color_1         # Light vector l
    combiner_1_output_rgb = dot_product(combiner_1_input_A, combiner_1_input_B)
    # combiner_1_output_alpha is unchanged (interpolated intensity)

    # Final Combiner: Add ambient light
    final_input_A = combiner_1_output_rgb      # Dot product D
    final_input_B = const_color_2              # Diffuse color C_diffuse
    final_input_C = const_color_3              # Ambient color C_ambient
    
    # Formula: C_final = D * C_diffuse + C_ambient
    final_output_rgb = (final_input_A * final_input_B) + final_input_C
    
    return final_output_rgb

def render_slice(slice_index, alpha):
    """
    Renders a single slice of the volume.
    alpha: interpolation factor between slice i and i+1
    """
    # Bind textures
    bind_texture_0(slice_gradients) # RGB = gradient
    bind_texture_1(slice_intensities) # Alpha = intensity
    
    # Execute combiner pipeline
    color = setup_combiners()
    
    # Write to framebuffer
    write_pixel(color)
```

**Parameters and Typical Values:**
*   **Light Vector $\tilde{l}$:** Typically normalized. Direction depends on camera or scene setup.
*   **Light Colors:** $C_{diffuse}$ and $C_{ambient}$ are RGB vectors.
*   **Signed Fixed-Point:** The hardware supports signed fixed-point values for combiner inputs, allowing direct storage of negative gradient components without normalization or biasing [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Single Light Source:** The method is restricted to a single light source due to the fixed-function dot product calculation in the combiner pipeline [S1, unofficial].
*   **Monochrome Display:** The description implies a monochrome or single-color shading model unless additional combiner stages are added for color mixing.
*   **Gradient Accuracy:** The quality of the shading depends on the accuracy of the pre-computed voxel gradients.

## Subtopics

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels in the screen space projection. The complexity is dominated by the rasterization and texture sampling, which are hardware-accelerated.
*   **Memory Complexity:** $O(V)$ where $V$ is the volume size, stored in 2D textures.
*   **Performance:** Enables interactive frame rates for shaded isosurfaces on standard PC hardware [S1, unofficial].
*   **Strengths:** Efficient use of fixed-function GPU hardware; avoids software ray-marching.
*   **Limitations:** Restricted to single light sources; limited to monochrome or simple shading models without additional combiner stages; requires pre-computation of gradients.

## Background
The method is an adaptation of the "Fast Shaded Isosurfaces" algorithm, designed to leverage the capabilities of standard PC graphics hardware (specifically OpenGL register combiners) for volume visualization. It was developed to provide efficient rendering of shaded isosurfaces at interactive frame rates.

## Sources
[S1] Interactive Volume Rendering on Standard PC Graphics Hardware, https://courses.cs.duke.edu/spring03/cps296.8/papers/VolVisforPCHardware.pdf, fetched 2026-10-28 (unofficial)
