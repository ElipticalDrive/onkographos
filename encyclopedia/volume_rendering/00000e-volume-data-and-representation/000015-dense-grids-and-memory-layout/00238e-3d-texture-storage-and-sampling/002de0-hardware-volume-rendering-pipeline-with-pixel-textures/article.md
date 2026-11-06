# Hardware Volume Rendering Pipeline with Pixel Textures

Also called: Dependent Texture Classification, Dependent Texture Volume Rendering, Hardware Volume Rendering Pipeline (Pixel Texture Based), Pixel Texture Classification, Pixel Texture Shading on SGI V Series, SGI Octane2 Hardware Volume Rendering with Pixel Textures, VGH/QN Pipeline

## Overview
This method implements volume rendering on hardware that supports the OpenGL pixel texture (dependent texture) extension, specifically targeting architectures like the SGI Octane 2 with V-series graphics cards [S1, S2]. It solves the problem of applying multi-dimensional transfer functions (classification and shading) by using color fragments from a rendered slice to generate texture coordinates for a 3D lookup texture, effectively replacing the fragment color with the result of the transfer function evaluation [S1, S2]. It differs from per-fragment dependent texture reads (e.g., GeForce3) by requiring an auxiliary scratch buffer for intermediate storage, as the pixel texture operation does not occur directly during rasterization on the target hardware [S1].

## How it works
### Inputs and Outputs
**Inputs:**
1.  **VGH Data Volume:** A 3D texture encoding Data Value ($V$), Gradient Magnitude ($G$), and Second Derivative ($H$) in the RGB color components. Each component uses 8-bit resolution [S1, S2].
2.  **Quantized Normal (QN) Volume:** A 3D texture encoding surface normal direction as a 16-bit unsigned short, split into two 8-bit color components (Red and Green) [S1, S2].
3.  **Shade Volume:** A 3D texture containing pre-computed lighting information (RGB), spatially coincident with the VGH volume [S1, S2].
4.  **Transfer Function (TF):** Defined as a 3D pixel texture. For classification, it maps $(V, G, H)$ to $(RGBA)$. For shading, it maps quantized normals to lighting values [S1, S2].

**Outputs:**
*   Final composited image in the Frame Buffer.

### Data Structures and Layout
1.  **VGH Texture:**
    *   Format: 3D Texture.
    *   Channels: Red = $V$ (8-bit), Green = $G$ (8-bit), Blue = $H$ (8-bit) [S1, S2].
    *   Resolution: 256 entries per axis for $V$ and $G$; limited resolution for $H$ (see Parameters) [S2].
2.  **QN Texture:**
    *   Format: 3D Texture.
    *   Channels: Red = Upper 8 bits of normal, Green = Lower 8 bits of normal [S1, S2].
    *   Encoding: 16-bit unsigned short quantized normal [S1, S2].
3.  **Shade Texture:**
    *   Format: 3D Texture.
    *   Channels: RGB lighting values [S1, S2].
4.  **Scratch Buffer:**
    *   An auxiliary framebuffer or texture buffer used to store the intermediate classified slice before compositing [S1].

### Parameters
*   **Interpolation:** Nearest-neighbor interpolation is used for pixel texture lookups [S1].
*   **Second Derivative ($H$) Resolution:** Limited to three "sheets" representing negative, zero, and positive values [S2].
*   **Boundary Emphasis:** A slider controlling the opacity scaling on the outer sheets ($H \neq 0$) relative to the central sheet ($H = 0$) [S2].
*   **Quantization:** 16-bit quantization for normals [S1].

### Algorithm Steps
The pipeline processes the volume slice-by-slice. Let $S$ be the current slice index.

1.  **Render VGH Slice to Scratch Buffer:**
    *   Render the slice from the VGH data volume into the scratch buffer.
    *   The fragment color at this stage is $(V/255, G/255, H_{scaled}/255)$, where $H_{scaled}$ is the second derivative value scaled and biased to fit the $[0, 1]$ range required for texture coordinates [S1, S2].

2.  **Apply Classification Pixel Texture:**
    *   Perform a `glCopyPixels` (or equivalent) operation from the scratch buffer to itself (or a target texture) using the pixel texture extension.
    *   The hardware uses the RGB color fragments from Step 1 as texture coordinates into the Classification 3D Texture.
    *   The lookup returns a new RGBA value $(R_{out}, G_{out}, B_{out}, A_{out})$ which replaces the original fragment color [S1, S2].
    *   *Note on H-axis:* The classification texture uses three control points for the $H$ axis. The opacity on the center sheet ($H=0$) is set by classification widgets. The opacity on the outer sheets ($H>0, H<0$) is scaled by the boundary emphasis slider [S2].

3.  **Apply Shading (Optional):**
    *   If shading is enabled:
        a. Render the corresponding slice from the **Shade Volume** into the scratch buffer (or a temporary buffer).
        b. Modulate (multiply) the classified slice (from Step 2) with the shading slice.
        c. Result: $(R_{out} \cdot R_{shade}, G_{out} \cdot G_{shade}, B_{out} \cdot B_{shade}, A_{out})$ [S1].
    *   *Alternative Shading Path (QN):* If using the Quantized Normal volume, render the QN slice to the scratch buffer and apply a Shading Pixel Texture lookup to generate the lighting values, then multiply with the classified slice [S1].

4.  **Composite to Frame Buffer:**
    *   Copy the final slice from the scratch buffer to the Frame Buffer.
    *   Blend the slice with the existing content in the Frame Buffer using standard alpha blending (back-to-front or front-to-back depending on implementation, typically back-to-front for hardware accumulation) [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volume_slice(vgh_texture, shade_texture, qn_texture, classification_tf, shading_tf, scratch_buffer, frame_buffer, slice_index, use_shading, use_qn_shading):
    """
    Renders a single slice of the volume using the hardware pixel texture pipeline.
    """
    
    # 1. Render VGH slice to scratch buffer
    # The fragment color is V, G, H normalized to [0,1]
    # H must be scaled/biased to maintain sign information for interpolation
    h_scaled = scale_and_bias_second_derivative(vgh_texture[slice_index].H)
    
    render_to_buffer(vgh_texture[slice_index], scratch_buffer, 
                     color_output=(vgh_texture[slice_index].V/255.0, 
                                   vgh_texture[slice_index].G/255.0, 
                                   h_scaled))
    
    # 2. Apply Classification Pixel Texture
    # Use nearest-neighbor interpolation
    classified_slice = apply_pixel_texture_lookup(scratch_buffer, classification_tf, interpolation='nearest')
    
    # 3. Apply Shading if enabled
    if use_shading:
        if use_qn_shading:
            # Render QN slice to scratch buffer
            render_to_buffer(qn_texture[slice_index], scratch_buffer, 
                             color_output=(qn_texture[slice_index].Red/255.0, 
                                           qn_texture[slice_index].Green/255.0, 0.0))
            
            # Apply Shading Pixel Texture to get lighting values
            shading_values = apply_pixel_texture_lookup(scratch_buffer, shading_tf, interpolation='nearest')
            
            # Modulate classified slice with shading values
            final_slice = multiply_colors(classified_slice, shading_values)
        else:
            # Use pre-computed Shade Volume
            render_to_buffer(shade_texture[slice_index], scratch_buffer, 
                             color_output=shade_texture[slice_index].RGB)
            
            # Modulate classified slice with shading slice
            final_slice = multiply_colors(classified_slice, scratch_buffer)
    else:
        final_slice = classified_slice
        
    # 4. Composite to Frame Buffer
    # Blend final_slice into frame_buffer using alpha blending
    blend_into_frame_buffer(final_slice, frame_buffer, alpha=final_slice.A)

def scale_and_bias_second_derivative(h_value):
    """
    Scales and biases the signed second derivative to [0, 1] for texture coordinate usage.
    Maintains sign information for proper interpolation across the zero-crossing.
    """
    # Implementation detail: Map [-max, max] to [0, 1] such that 0 maps to 0.5
    # This allows the 3-sheet structure (negative, zero, positive) to interpolate correctly.
    # Not given in sources: Exact scaling factor depends on data range.
    return (h_value + max_abs_h) / (2 * max_abs_h)
```

### Edge Cases and Pitfalls
1.  **Scratch Buffer Size:** When updating the shade volume in hardware, the quantized normal slices must be smaller than the scratch buffer’s dimensions [S1].
2.  **Hardware Flexibility:** Some hardware implementations do not support generating the shading pixel texture by rendering a point with its normal to the frame buffer and copying it [S1].
3.  **Performance Optimization:** If the transfer function is manipulated, the viewpoint is static, allowing the pixel texture operation to be used only on the portion of the volume currently changing. Conversely, when the viewpoint rotates, the transfer function is static [S1].
4.  **Texture Size:** Hardware flexibility in scaling color channel values allows the number of elements along a dimension of the pixel texture to differ from the number of bit planes in the component that generated the texture coordinate, preventing prohibitively large 3D pixel textures [S2].

## Subtopics

### [Multi-Pass/Multi-Texture Shading](5653-multi-pass-multi-texture-shading.md)
Multi-Pass/Multi-Texture Shading is a technique for adding interactive shading to the hardware volume rendering pipeline by decoupling classification and lighting calculations into separate rendering passes. It distinguishes itself from single-pass methods by utilizing multiple textures and pipeline stages to handle complex lighting models, such as gradient-based enhancement, without significant performance penalties.

## Operating characteristics
*   **Time Complexity:** Per-slice rendering involves two main texture lookups (classification and shading) and one blend operation. The cost is dominated by the memory bandwidth required to read/write the scratch buffer and the pixel texture lookup latency.
*   **Memory Complexity:** Requires storage for VGH, QN, and Shade volumes, plus a scratch buffer. The VGH and QN volumes are 3D textures with 8-bit components.
*   **Measured Performance:** On SGI Octane 2 with V-series cards, the method requires a scratch buffer, leading to expensive copy operations. On nVidia GeForce3, dependent texture reads on a per-fragment basis eliminate the scratch buffer, significantly improving performance [S1].
*   **Strengths:** Allows complex multi-dimensional transfer functions (V, G, H) to be evaluated efficiently in hardware. Supports interactive manipulation of transfer functions and viewpoint [S1, S2].
*   **Limitations:** Requires auxiliary buffers on older hardware (SGI V-series). Limited to separable transfer functions on hardware with only 2D dependent texture reads (GeForce3) [S1].

## Background
This method was developed for interactive volume rendering using multi-dimensional transfer functions on SGI Octane 2 systems with V-series graphics cards and nVidia GeForce3 series graphics adaptors [S1]. It leverages the OpenGL pixel texture (dependent texture) extension to perform arbitrary function evaluation via lookup tables, enabling real-time classification and shading of 3D volumes [S1, S2]. The approach was presented in the paper "Interactive Volume Rendering Using Multi-Dimensional Transfer Functions" by Kindlmann et al. [S1, S2].

## Sources
[S1] Interactive Volume Rendering Using Multi-Dimensional Transfer Functions, https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
[S2] Interactive Volume Rendering Using Multi-Dimensional Transfer Functions, https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
