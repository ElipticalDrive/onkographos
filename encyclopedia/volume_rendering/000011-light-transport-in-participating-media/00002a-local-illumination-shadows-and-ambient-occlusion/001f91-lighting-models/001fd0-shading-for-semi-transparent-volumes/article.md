# Shading for Semi-Transparent Volumes

Also called: Local diffuse illumination for semi-transparent volumes, Semi-transparent volume shading with local diffuse illumination, Shaded semi-transparent volume rendering

## Overview
Shading for Semi-Transparent Volumes is a technique that enables local diffuse illumination for semi-transparent volume rendering using multi-stage rasterization hardware, specifically targeting the register combiners architecture found in early 2000s GPUs [S1, unofficial][S2]. It solves the problem of computing per-voxel lighting without requiring a frame buffer copy or multi-pass rendering, which was previously necessary to handle signed gradient vectors or complex interpolation [S2]. Unlike standard isosurface shading which uses alpha-testing, this method employs alpha-blending to composite the shaded volume [S1, unofficial]. It sets itself apart from sibling methods by achieving interactive frame rates via single-pass rendering and hardware-supported dot products, though it sacrifices real trilinear interpolation to accommodate the separate texture requirement for gradients and colors [S1, unofficial][S2].

## How it works
The method relies on the principle that a surface normal approximation can be derived from the volume data gradient $\mathbf{g} = \nabla V$. In this specific hardware implementation, the gradient is not computed via finite differences in shader code but is pre-computed or stored directly in texture data to leverage the fixed-function pipeline's combiner units [S1, unofficial][S2].

### Inputs and Outputs
*   **Input 1:** A 3D volume dataset containing scalar values.
*   **Input 2:** A light direction vector $\mathbf{l}$.
*   **Input 3:** A global diffuse color $C_{diffuse}$.
*   **Output:** A shaded, semi-transparent image where each fragment's color includes ambient and diffuse contributions based on the local gradient.

### Data Structures and Layout
The technique requires two separate 2D textures (representing slices or interpolated planes) due to hardware limitations on the number of simultaneous texture inputs [S1, unofficial][S2].

1.  **Texture 0 (Gradient Texture):**
    *   **Format:** RGB.
    *   **Content:** Stores the gradient vector $\mathbf{g}$ for each voxel. The RGB channels correspond to the $x, y, z$ components of the gradient.
    *   **Note:** This texture does *not* contain color or opacity information.

2.  **Texture 1 (Color/Opacity Texture):**
    *   **Format:** RGBA (or paletted RGB with alpha).
    *   **Content:** Stores the ambient color $C_{ambient}$ and opacity $\alpha$ for each voxel.
    *   **Note:** The RGB channels represent the ambient color. The alpha channel represents the voxel's opacity. This texture can be a paletted texture of color indices to save memory [S1, unofficial][S2].

### Combiner Setup
The rendering pipeline uses two combiner stages: **General Combiner 0** and the **Final Combiner**.

**General Combiner 0:**
*   **Input A:** Texture 0 (Gradient $\mathbf{g}$).
*   **Input B:** Light Direction $\mathbf{l}$ (passed as a vertex attribute or constant).
*   **Input C:** ZERO.
*   **Operation:** Computes the dot product of the gradient and the light direction.
    $$ \text{Output}_0 = \mathbf{g} \cdot \mathbf{l} $$
*   **Note:** The hardware supports signed vector components, so no workaround for negative dot products is needed [S2].

**Final Combiner:**
*   **Input A:** Output from General Combiner 0 (the dot product $d = \mathbf{g} \cdot \mathbf{l}$).
*   **Input B:** Primary Color (Diffuse Color $C_{diffuse}$). This is typically passed as a constant color or vertex color.
*   **Input C:** ZERO.
*   **Input D:** Texture 1 (Ambient Color $C_{ambient}$ and Opacity $\alpha$).
*   **Operation:** Sums the ambient color and the product of the diffuse color and the dot product.
    $$ \text{Output}_{final} = d \cdot C_{diffuse} + (1 - d) \cdot 0 + C_{ambient} $$
    Simplified:
    $$ C_{shade} = C_{ambient} + (\mathbf{g} \cdot \mathbf{l}) C_{diffuse} $$
*   **Alpha Handling:** The alpha channel from Texture 1 is passed through to control the blending of the fragment with the background [S1, unofficial].

### Pseudocode
The following pseudocode describes the configuration of the register combiners. It assumes a fixed-function pipeline API (e.g., OpenGL 1.2+ with `ARB_register_combiners`).

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume 'gl' is the graphics context and 'tex0', 'tex1' are bound textures.

def setup_combiners(light_direction, diffuse_color):
    # Enable register combiners
    gl.enable(gl.REGISTER_COMBINERS_NV)

    # --- General Combiner 0 ---
    # Computes dot product of Gradient (Tex0) and Light Direction
    gl.combiner_input_nv(gl.COMBINER0_RGB, gl.VARIABLE_A, gl.TEXTURE0, gl.RGB)
    gl.combiner_input_nv(gl.COMBINER0_RGB, gl.VARIABLE_B, gl.CONSTANT, gl.RGB)
    # Light direction is passed as a constant color or vertex attribute mapped to B
    # For simplicity, assume light direction is set as a constant color uniform
    gl.combiner_constant_color0_nv(light_direction) 
    
    gl.combiner_input_nv(gl.COMBINER0_RGB, gl.VARIABLE_C, gl.CONSTANT, gl.RGB)
    gl.combiner_constant_color0_nv([0.0, 0.0, 0.0]) # ZERO
    
    gl.combiner_output_nv(gl.COMBINER0_RGB, gl.NONE, gl.RGB, gl.RGB, gl.RGB, gl.NONE)
    # Output is A * B = Gradient . Light

    # --- Final Combiner ---
    # Computes Ambient + Diffuse * DotProduct
    gl.combiner_input_nv(gl.COMBINER_FINAL_RGB, gl.VARIABLE_A, gl.COMBINER0, gl.RGB)
    # Variable A is the dot product from Combiner 0
    
    gl.combiner_input_nv(gl.COMBINER_FINAL_RGB, gl.VARIABLE_B, gl.CONSTANT, gl.RGB)
    # Variable B is the Diffuse Color
    gl.combiner_constant_color0_nv(diffuse_color)
    
    gl.combiner_input_nv(gl.COMBINER_FINAL_RGB, gl.VARIABLE_C, gl.CONSTANT, gl.RGB)
    gl.combiner_constant_color0_nv([0.0, 0.0, 0.0]) # ZERO
    
    gl.combiner_input_nv(gl.COMBINER_FINAL_RGB, gl.VARIABLE_D, gl.TEXTURE1, gl.RGB)
    # Variable D is the Ambient Color from Texture 1
    
    # Formula: A*B + (1-A)*C + D
    # Since C is ZERO, this simplifies to: A*B + D
    # Which is: (Gradient . Light) * Diffuse + Ambient
    gl.combiner_rgb_nv(gl.COMBINER_FINAL_RGB, gl.MULTIPLY, gl.VARIABLE_A, gl.VARIABLE_B, gl.ZERO, gl.MULTIPLY, gl.VARIABLE_C, gl.ADD)
    gl.combiner_alpha_nv(gl.COMBINER_FINAL_RGB, gl.REPLACE, gl.VARIABLE_D, gl.NONE, gl.NONE, gl.NONE, gl.NONE)
    # Alpha is taken from Texture 1 (Variable D's alpha)

def render_slice(slice_data, gradient_data):
    # Bind Gradient Texture to Unit 0
    gl.active_texture(gl.TEXTURE0)
    gl.bind_texture(gl.TEXTURE_2D, gradient_data)
    
    # Bind Color/Opacity Texture to Unit 1
    gl.active_texture(gl.TEXTURE1)
    gl.bind_texture(gl.TEXTURE_2D, slice_data)
    
    # Enable multi-texturing
    gl.enable(gl.TEXTURE_2D)
    gl.enable(gl.MULTITEXTURE)
    
    # Enable Alpha Blending for semi-transparency
    gl.enable(gl.BLEND)
    gl.blend_func(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA)
    
    # Draw the slice geometry (e.g., a quad or polygon strip)
    # The fragment shader/combiner logic runs per fragment
    draw_geometry()
```

### Parameters and Typical Values
*   **Light Direction:** Normalized 3D vector.
*   **Diffuse Color:** RGB vector, typically white or light gray.
*   **Ambient Color:** RGB vector, typically a dimmer version of the diffuse color or a base material color.
*   **Opacity:** Alpha value from Texture 1, typically in range $[0, 1]$.

### Edge Cases and Pitfalls
1.  **No Real Trilinear Interpolation:** The hardware only supports two input textures simultaneously. Therefore, it is impossible to perform real trilinear interpolation of the volume data while also accessing the gradient and color textures [S1, unofficial][S2]. The method typically uses bilinear interpolation on the 2D slice textures, which degrades image quality compared to true 3D interpolation [S1, unofficial][S2].
2.  **No Per-Voxel Ambient Color via Alpha Index:** The hardware does not provide a component-wise color lookup table. It is not possible to use the alpha channel of the gradient texture as an index to look up a per-voxel ambient color. Ambient color must be stored directly in the RGB channels of Texture 1 [S1, unofficial][S2].
3.  **Limited Lighting:** With the standard setup, only one diffuse light source can be computed per pass. A second colored light source can be added by utilizing the unused input registers (C and D) of the first combiner stage, but this requires sacrificing either trilinear interpolation or the specification of separate colors for ambient and diffuse light [S1, unofficial].
4.  **Signed Vectors:** Unlike previous software-based methods, this hardware approach natively supports signed vector components for the gradient, eliminating the need for large-scale workarounds to handle negative dot products [S2].

## Operating characteristics
*   **Time Complexity:** Single-pass rendering. The lighting calculation is performed in hardware during rasterization, resulting in $O(1)$ per fragment overhead relative to the combiner logic.
*   **Memory Complexity:** Requires two texture copies of the volume data (or slice representations): one for gradients and one for colors/opacity. This doubles the memory bandwidth requirement compared to single-texture approaches [S1, unofficial].
*   **Measured Performance:** On a GeForce 256 with a 600x600 viewport:
    *   128x128x64 volume: 22.5 fps [S1, unofficial].
    *   256x256x128 volume: 4.1 fps [S1, unofficial].
    *   256x256x256 volume: 2.0 fps [S1, unofficial].
*   **Strengths:**
    *   Interactive frame rates achieved via single-pass rendering [S1, unofficial][S2].
    *   Hardware-supported dot products avoid software computation overhead [S2].
    *   Native support for signed gradient vectors [S2].
*   **Limitations:**
    *   Limited image quality due to the absence of real trilinear interpolation [S1, unofficial][S2].
    *   Restricted to two texture inputs, limiting the complexity of lighting models (e.g., limited to one diffuse light in the basic setup) [S1, unofficial].
    *   Dependent on specific hardware features (Register Combiners) that are obsolete in modern GPUs [S1, unofficial].

## Background
The technique was introduced in the context of interactive volume rendering on standard PC graphics hardware around 2000 [S1, unofficial][S2]. It was developed to leverage the emerging multi-texturing and register combiner capabilities of GPUs like the NVIDIA GeForce 256. The method builds on the fast algorithm for displaying isosurfaces, adapting it for semi-transparent rendering by replacing alpha-testing with alpha-blending [S1, unofficial]. It was presented as an improvement over previous software-based or multi-pass hardware implementations, which suffered from poor performance due to large-scale workarounds for signed vectors or excessive texture binding operations [S1, unofficial][S2].

## Sources
[S1] Interactive Volume Rendering On PC Hardware (PDF), http://gd.tuwien.ac.at/visual/OpenQVis/Publ.2000.5.pdf, fetched 2026-10-27 (unofficial)
[S2] Interactive Volume Rendering on Standard PC Graphics ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/VolVisforPCHardware.pdf, fetched 2026-10-28
