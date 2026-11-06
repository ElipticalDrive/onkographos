# Multi-Pass/Multi-Texture Shading

Also called: Multi-Pass Shading, Multi-Texture Shading

## Overview
Multi-Pass/Multi-Texture Shading is a technique for adding interactive shading to the hardware volume rendering pipeline by decoupling classification and lighting calculations into separate rendering passes [S1]. It distinguishes itself from single-pass methods by utilizing multiple textures and pipeline stages to handle complex lighting models, such as gradient-based enhancement, without significant performance penalties [S1].

## How it works
The method operates on a pipeline that maintains separate data and shading volumes. The core inputs are the rotated volume data and the updated shading parameters. The outputs are the final shaded volume image.

**Data Structures and Layout:**
The pipeline relies on specific 3D texture formats:
1.  **VGH Texture (Data Volume):** Encodes three quantities in its color components, using 8 bits per component [S1]:
    *   $V$: Data value.
    *   $G$: Gradient magnitude.
    *   $H$: Second derivative (Hessian).
2.  **QN Texture (Quantized Normal Volume):** Encodes normal direction [S1].
    *   Uses a 16-bit quantization scheme.
    *   Stored in two 8-bit color components of a 3D texture.
    *   Interpolation: Nearest-neighbor is used for pixel texture lookups to avoid redundancy and truncation errors inherent in hardware interpolation of normals [S1].
3.  **Dependent Textures:**
    *   **VG Dependent Texture:** Used for classification based on Value and Gradient.
    *   **H Dependent Texture:** Used for classification based on the Second Derivative.
4.  **Normal Texture:** Used specifically for shading calculations [S1].

**Pipeline Stages (GeForce3 Example):**
The hardware pipeline utilizes four-way multi-texturing [S1]:
1.  **VGH Texture:** Primary data lookup.
2.  **VG Dependent Texture:** Classification lookup.
3.  **H Dependent Texture:** Classification lookup.
4.  **Normal Texture:** Lighting/shading lookup.

The central box of the pipeline is the **Register Combiner stage** [S1]. This stage combines the outputs of the textures. Two specific stages are noted:
*   **Blend VG&H Color:** Combines colors based on VG and H. This stage is often skipped ("rarely executed") because color is rarely varied along the second derivative axis [S1].
*   **Multiply VG&H Alpha:** Combines alpha values. This stage is **required** because the 3D transfer function must be composed separably as a 2D $\times$ 1D function [S1].

**Algorithm Steps:**
1.  **Rotate Volume:** Apply the view transformation to the volume data.
2.  **Update Shade Volume:** Recalculate or update the shading volume (e.g., normals) after the rotation [S1].
3.  **Re-render:** Execute the rendering pass using the multi-texture setup described above [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_pass_shading_pipeline(volume_data, view_matrix):
    # 1. Rotate the volume data to align with the view
    rotated_volume = rotate(volume_data, view_matrix)
    
    # 2. Update the shading volume (normals) based on the rotated data
    # This step is critical because hardware interpolation of normals 
    # is avoided via nearest-neighbor lookup on pre-quantized normals.
    shade_volume = update_shade_volume(rotated_volume)
    
    # 3. Setup Textures
    # VGH: 8-bit V, 8-bit G, 8-bit H
    tex_vgh = create_3d_texture(rotated_volume.v, rotated_volume.g, rotated_volume.h)
    
    # QN: 16-bit normal encoded in two 8-bit channels
    tex_normal = create_3d_texture(shade_volume.normals_q16)
    
    # Dependent textures for separable transfer function
    tex_vg = create_3d_texture(rotated_volume.v, rotated_volume.g)
    tex_h = create_3d_texture(rotated_volume.h)
    
    # 4. Configure Register Combiner
    # Stage 1: Multiply VG&H Alpha (Required for separable 2Dx1D transfer function)
    alpha = multiply_alpha(tex_vg.alpha, tex_h.alpha)
    
    # Stage 2: Blend VG&H Color (Optional, often skipped)
    if vary_color_along_h():
        color = blend_color(tex_vg.color, tex_h.color)
    else:
        color = tex_vg.color
        
    # Stage 3: Lighting using Normal Texture
    # Normals are looked up via nearest-neighbor to preserve quantization
    lit_color = apply_lighting(color, tex_normal, view_matrix)
    
    # 5. Render
    render(lit_color, alpha)
```

**Edge Cases and Pitfalls:**
*   **Normal Interpolation:** Hardware interpolation of normals causes redundancy and truncation. Therefore, nearest-neighbor interpolation must be used for the normal texture lookup [S1].
*   **Quantization:** Normals are 16-bit quantized. Using 8-bit per-axis 2D nearest neighbor pixel textures is equivalent to 16-bit 1D nearest neighbor pixel textures [S1].
*   **Transfer Function Composition:** The transfer function must be separable (2D $\times$ 1D) to fit the pipeline structure, requiring the alpha multiplication stage [S1].

## Subtopics

### [Hardware-Accelerated Non-Photorealistic Volume Rendering](5847-hardware-accelerated-non-photorealistic-volume-rendering.md)
Extends this shading approach to non-photorealistic rendering (NPR) using paletted textures for tone shading and silhouettes, decoupling lighting from base opacity via two passes per polygon.

## Operating characteristics
*   **Performance:** Adds interactive shading with a "negligible increase in rendering cost" compared to unshaded rendering [S1].
*   **Complexity:** Requires managing multiple texture units (four-way multi-texturing on GeForce3) and register combiner stages [S1].
*   **Limitations:** Relies on hardware-specific features (Register Combiners, Dependent Textures). The requirement for separable transfer functions limits the complexity of the classification function [S1].
*   **Precision:** Uses 8-bit for V, G, H and 16-bit for normals. This quantization is a trade-off for performance and hardware compatibility [S1].

## Background
This method was developed as part of the Interactive Volume Rendering project, modifying the traditional hardware volume rendering pipeline to support interactive shading [S1]. It was implemented on hardware such as the Octane2 and GeForce3, leveraging extensions like dependent textures [S1].

## Sources
[S1] Interactive Volume Rendering Using Multi-Dimensional Transfer Functions, https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
