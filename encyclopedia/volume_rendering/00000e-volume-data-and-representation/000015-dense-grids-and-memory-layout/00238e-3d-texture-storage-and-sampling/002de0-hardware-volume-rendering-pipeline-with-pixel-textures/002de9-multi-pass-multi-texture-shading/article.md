# Multi-Pass/Multi-Texture Shading

Also called: Multi-Pass Shading, Multi-Texture Shading

## Overview
Multi-Pass/Multi-Texture Shading is a method for adding interactive shading to the hardware volume rendering pipeline using multiple rendering passes and textures [S1]. It distinguishes itself from single-pass approaches by decoupling shading calculations into a separate pass, achieving this with a negligible increase in rendering cost [S1].

## How it works
The pipeline maintains separate data and shading volumes [S1]. Inputs are the rotated volume data and the updated shading parameters [S1]. The output is the final shaded volume image [S1].

**Data Structures and Layout:**
The pipeline relies on specific 3D texture formats [S1]:
1. **VGH Texture (Data Volume):** Encodes three quantities in its color components, using 8 bits per component [S1]:
   * $V$: Data value.
   * $G$: Gradient magnitude.
   * $H$: Second derivative (Hessian).
2. **QN Texture (Quantized Normal Volume):** Encodes normal direction as a 16-bit unsigned short in two 8-bit color components of a 3D texture [S1].
3. **Dependent Textures:** The GeForce3 pipeline uses four-way multi-texturing with VGH, VG Dependent Texture, H Dependent Texture, and Normal Texture [S1].
4. **Normal Texture:** Lighting/shading lookup [S1].
Hardware interpolation of normals causes redundancy and truncation, so nearest-neighbor interpolation is used for all pixel texture lookups [S1].

**Pipeline Stages (GeForce3 Example):**
The central pipeline stage is the register combiner [S1]. It processes the four textures as follows:
* **VGH Texture:** Primary data lookup [S1].
* **VG Dependent Texture:** Classification lookup based on Value and Gradient [S1].
* **H Dependent Texture:** Classification lookup based on the Second Derivative [S1].
* **Normal Texture:** Lighting/shading lookup [S1].
Two specific combiner stages are defined:
* **Blend VG&H Color:** Combines colors based on VG and H [S1]. This stage is rarely executed because color is rarely varied along the second derivative axis [S1].
* **Multiply VG&H Alpha:** Combines alpha values [S1]. This stage is required because the 3D transfer function must be composed separably as a 2D × 1D function [S1].

**Algorithm Steps:**
1. **Rotate Volume:** Apply the view transformation to the volume data [S1].
2. **Update Shade Volume:** Recalculate or update the shading volume (e.g., normals) after the rotation [S1].
3. **Re-render:** Execute the rendering pass using the multi-texture setup described above [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_pass_shading_pipeline(volume_data, view_matrix):
    # 1. Rotate volume data to align with view [S1]
    rotated_data = [transform_coord(v, view_matrix) for v in volume_data]

    # 2. Update shade volume (normals) after rotation [S1]
    # Sources do not specify the exact normal computation algorithm
    shade_volume = compute_quantized_normals(rotated_data)

    # 3. Setup 3D textures [S1]
    # VGH: 8-bit V, 8-bit G, 8-bit H
    tex_vgh = Texture3D(rotated_data.v, rotated_data.g, rotated_data.h)
    # QN: 16-bit normal encoded in two 8-bit channels
    tex_normal = Texture3D(shade_volume.normals_q16)
    # Dependent textures for separable transfer function
    tex_vg = Texture3D(rotated_data.v, rotated_data.g)
    tex_h = Texture3D(rotated_data.h)

    # 4. Configure Register Combiner [S1]
    # Stage 1: Multiply VG&H Alpha (Required for separable 2Dx1D transfer function)
    alpha = tex_vg.alpha * tex_h.alpha

    # Stage 2: Blend VG&H Color (Optional, rarely executed)
    if vary_color_along_h:
        color = tex_vg.color + tex_h.color
    else:
        color = tex_vg.color

    # Stage 3: Lighting using Normal Texture
    # Nearest-neighbor sampling to avoid redundancy/truncation from hardware interpolation [S1]
    lit_color = apply_lighting(color, tex_normal, view_matrix)

    # 5. Re-render the volume [S1]
    render(lit_color, alpha)

def transform_coord(v, matrix):
    return [sum(v[i] * matrix[i][j] for i in range(3)) for j in range(3)]

def compute_quantized_normals(data):
    # Compute gradients, normalize, quantize to 16-bit unsigned short
    # stored in two 8-bit channels, using nearest-neighbor sampling [S1]
    return data

def Texture3D(*channels):
    return {"channels": channels}

def vary_color_along_h():
    return False

def apply_lighting(color, normal_tex, view_matrix):
    return color

def render(color, alpha):
    pass
```

**Edge Cases and Pitfalls:**
* **Normal Interpolation:** Hardware interpolation of normals causes redundancy and truncation [S1]. Therefore, nearest-neighbor interpolation must be used for the normal texture lookup [S1].
* **Quantization:** Normals are 16-bit quantized [S1]. Using 8-bit per-axis 2D nearest neighbor pixel textures is equivalent to 16-bit 1D nearest neighbor pixel textures [S1].
* **Transfer Function Composition:** The transfer function must be separable (2D × 1D) to fit the pipeline structure, requiring the alpha multiplication stage [S1].

## Subtopics

### [Hardware-Accelerated Non-Photorealistic Volume Rendering](5847-hardware-accelerated-non-photorealistic-volume-rendering.md)
A parallel technique for interactive scalar volume visualization that uses paletted textures for tone shading and silhouettes. It sets itself apart by employing two rendering passes per view-aligned polygon to decouple lighting from base opacity, and by filtering paletted textures after color lookup in RGBA space to enable non-linear mappings with trilinear interpolation. Choose it when non-photorealistic enhancement is required over standard photorealistic rendering.

## Operating characteristics
* **Performance:** Adds interactive shading with a negligible increase in rendering cost compared to unshaded rendering [S1].
* **Complexity:** Requires managing multiple texture units (four-way multi-texturing on GeForce3) and register combiner stages [S1].
* **Limitations:** Relies on hardware-specific features (Register Combiners, Dependent Textures) [S1]. The requirement for separable transfer functions limits the complexity of the classification function [S1].
* **Precision:** Uses 8-bit for V, G, H and 16-bit for normals [S1].

## Background
This method was developed as part of the Interactive Volume Rendering project, modifying the traditional hardware volume rendering pipeline to support interactive shading [S1]. It was implemented on hardware such as the Octane2 and GeForce3, leveraging extensions like dependent textures [S1].

## Sources
[S1] Interactive Volume Rendering Using Multi-Dimensional Transfer Functions, https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
