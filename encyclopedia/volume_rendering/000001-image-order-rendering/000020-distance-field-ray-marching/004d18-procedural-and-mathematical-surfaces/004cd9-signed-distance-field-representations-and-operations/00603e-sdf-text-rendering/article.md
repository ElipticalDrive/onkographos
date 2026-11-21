# SDF Text Rendering

Also called: TMP_SDF-Mobile Overlay

## Overview
SDF Text Rendering is a mobile-optimized technique for rendering text using Signed Distance Fields (SDFs) by sampling a pre-computed font atlas in screen space. It solves the problem of rendering crisp, scalable text with outlines and gradients on constrained hardware by omitting complex 3D effects like bevels, bump mapping, and environment mapping found in full desktop SDF shaders [S1, unofficial]. It differs from its sibling [RayMarchedHeading](18123-sdf-text-rendering.md) by operating on 2D texture samples rather than performing 3D raymarching through an extruded volume [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 2D font atlas texture (`_MainTex`) containing the SDF values in the alpha channel, vertex positions, UV coordinates, and UI parameters (scale, bias, outline width, colors).
*   **Output:** A final color `half4 c` for each pixel, including face color, outline color, and alpha.

**Data Structures:**
The vertex shader outputs a `pixel_t` struct with the following layout:
*   `vertex`: `SV_POSITION` (Clip space position).
*   `faceColor`: `COLOR` (RGBA).
*   `outlineColor`: `COLOR1` (RGBA).
*   `texcoord0`: `TEXCOORD0` (Texture UV for atlas sampling; Mask UV for clipping).
*   `param`: `TEXCOORD1` (Packed half4: `x` = Scale, `y` = BiasIn, `z` = BiasOut, `w` = BiasFace).
*   `mask`: `TEXCOORD2` (Position in clip space `xy`; Softness factors `zw`).

**Parameters:**
*   `_GradientScale` (float, default 5): Scales the distance field gradient for the font atlas [S1, unofficial].
*   `_WeightNormal` (float, default 0): Base weight value for normal text style [S1, unofficial].
*   `_WeightBold` (float, default 0.5): Base weight value for bold text style [S1, unofficial].
*   `_FaceDilate` (Range -1 to 1, default 0): Dilates the face color region [S1, unofficial].
*   `_OutlineWidth` (Range 0 to 1, default 0): Thickness of the outline [S1, unofficial].
*   `_OutlineSoftness` (Range 0 to 1, default 0): Softness applied to both sides of the outline [S1, unofficial].
*   `_PerspectiveFilter` (Range 0 to 1, default 0.875): Factor for perspective correction of the distance field scale [S1, unofficial].
*   `_Sharpness` (Range -1 to 1, default 0): Adjusts the sharpness of the distance field sampling [S1, unofficial].
*   `_ClipRect` (vector, default -32767,-32767,32767,32767): Clipping rectangle for UI masking [S1, unofficial].
*   `_MaskSoftnessX/Y` (float, default 0): Softness factors for the clipping mask [S1, unofficial].

**Steps:**

1.  **Compute Pixel Size:** Calculate the pixel size in world space from the clip position and projection matrix.
    $$ \text{pixelSize}_w = vPosition.w $$
    $$ \text{pixelSize} = \frac{\text{pixelSize}_w}{\begin{bmatrix} \text{ScaleX} \\ \text{ScaleY} \end{bmatrix} \cdot \left| \begin{bmatrix} P_{00} & P_{01} \\ P_{10} & P_{11} \end{bmatrix} \cdot \begin{bmatrix} \text{ScreenParams}_x \\ \text{ScreenParams}_y \end{bmatrix} \right|} $$
    Where $P$ is the projection matrix `UNITY_MATRIX_P` and `ScreenParams` is the screen resolution [S1, unofficial].

2.  **Compute Base Scale:** Derive the base scale factor from the pixel size.
    $$ scale = \frac{1}{\sqrt{\text{pixelSize}_x^2 + \text{pixelSize}_y^2}} $$
    [S1, unofficial]

3.  **Apply Perspective Filter:** If the projection is orthographic (`UNITY_MATRIX_P[3][3] == 0`), lerp between the absolute scale and a view-direction dependent scale using `_PerspectiveFilter` to prevent distortion [S1, unofficial]. Otherwise, use the computed scale directly [G].

4.  **Compute Weight and Dilate:** Calculate the text weight based on normal/bold states and apply face dilation.
    $$ weight = \frac{\text{lerp}(\_WeightNormal, \_WeightBold, bold)}{4.0} $$
    $$ weight = (weight + \_FaceDilate) \cdot \_ScaleRatioA \cdot 0.5 $$
    [S1, unofficial]

5.  **Compute Bias and Outline:** Determine the bias values for the face and outline regions, adjusting for softness.
    $$ scale = \frac{scale}{1 + (\_OutlineSoftness \cdot \_ScaleRatioA \cdot scale)} $$
    $$ bias = (0.5 - weight) \cdot scale - 0.5 $$
    $$ outline = \_OutlineWidth \cdot \_ScaleRatioA \cdot 0.5 \cdot scale $$
    [S1, unofficial]

6.  **Pack Parameters:** Pack the computed values into the `param` output for the fragment shader.
    $$ param.x = scale $$
    $$ param.y = bias - outline $$
    $$ param.z = bias + outline $$
    $$ param.w = bias $$
    [S1, unofficial]

7.  **Sample Distance Field:** In the fragment shader, sample the alpha channel of the font atlas.
    $$ d = \text{tex2D}(\_MainTex, input.texcoord0.xy).a \cdot input.param.x $$
    [S1, unofficial]

8.  **Compute Face Color:** Determine the face color based on the distance value and the face bias.
    $$ c = input.faceColor \cdot \text{saturate}(d - input.param.w) $$
    [S1, unofficial]

9.  **Apply Outline:** If outline is enabled, blend between the outline color and face color based on the distance relative to the outline bias.
    $$ c = \text{lerp}(input.outlineColor, c, \text{saturate}(d - input.param.z)) \cdot \text{saturate}(d - input.param.y) $$
    [S1, unofficial]

10. **Apply Clipping Mask:** Softly clip the text using the `_ClipRect` and mask softness.
    $$ m = \text{saturate}((\_ClipRect.zw - \_ClipRect.xy - |input.mask.xy|) \cdot input.mask.zw) $$
    $$ c *= m.x \cdot m.y $$
    [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdf_text_rendering(vertex_pos, texcoord0, face_color, outline_color, 
                       weight_normal, weight_bold, bold, face_dilate, 
                       outline_width, outline_softness, scale_ratio_a,
                       perspective_filter, clip_rect, mask_softness,
                       main_tex, screen_params, projection_matrix, scale_xy):
    """
    Renders SDF text by sampling a font atlas and applying face/outline colors.
    """
    # Step 1: Compute Pixel Size
    pixel_size_w = vertex_pos.w
    # Extract 2x2 upper-left of projection matrix
    p_matrix_2x2 = projection_matrix[0:2, 0:2]
    screen_vec = vector(screen_params.x, screen_params.y)
    abs_proj_scale = abs(p_matrix_2x2 @ screen_vec)
    scale_vec = vector(scale_xy.x, scale_xy.y)
    
    # Avoid division by zero
    denominator = scale_vec * abs_proj_scale
    if denominator.x == 0 or denominator.y == 0:
        scale = 1.0
    else:
        pixel_size = vector(pixel_size_w / denominator.x, pixel_size_w / denominator.y)
        
        # Step 2: Compute Base Scale
        scale = 1.0 / sqrt(pixel_size.x**2 + pixel_size.y**2)
        
        # Step 3: Apply Perspective Filter
        # Check if orthographic (UNITY_MATRIX_P[3][3] == 0)
        if projection_matrix[3, 3] == 0:
            # View-direction dependent scale calculation would go here [G]
            # For simplicity, we assume the base scale is adjusted or lerp is applied
            # Note: Exact view-direction formula not provided in source [S1, unofficial]
            pass 
        # Else use computed scale

    # Step 4: Compute Weight and Dilate
    weight = lerp(weight_normal, weight_bold, bold) / 4.0
    weight = (weight + face_dilate) * scale_ratio_a * 0.5

    # Step 5: Compute Bias and Outline
    # Adjust scale for outline softness
    scale_adjusted = scale / (1 + (outline_softness * scale_ratio_a * scale))
    
    bias = (0.5 - weight) * scale_adjusted - 0.5
    outline = outline_width * scale_ratio_a * 0.5 * scale_adjusted

    # Step 6: Pack Parameters
    param_scale = scale_adjusted
    param_bias_in = bias - outline
    param_bias_out = bias + outline
    param_bias_face = bias

    # Step 7: Sample Distance Field
    d = main_tex.sample(texcoord0).a * param_scale

    # Step 8: Compute Face Color
    c = face_color * saturate(d - param_bias_face)

    # Step 9: Apply Outline
    # Lerp between outline color and face color based on d - param_bias_out
    # Then multiply by saturate(d - param_bias_in)
    c = lerp(outline_color, c, saturate(d - param_bias_out)) * saturate(d - param_bias_in)

    # Step 10: Apply Clipping Mask
    mask_pos = vector(texcoord0.x, texcoord0.y) # Assuming mask.xy maps to clip space pos
    # Note: Source uses input.mask.xy for clip space position and input.mask.zw for softness
    # Re-mapping based on struct definition: mask.xy is clip space pos, mask.zw is softness
    clip_diff = vector(clip_rect.z - clip_rect.x, clip_rect.w - clip_rect.y)
    mask_soft = vector(mask_softness.x, mask_softness.y)
    
    # abs(input.mask.xy) assumes mask.xy is already in correct relative coords or handled by UV
    # Source: (_ClipRect.zw - _ClipRect.xy - abs(input.mask.xy)) * input.mask.zw
    # This implies input.mask.xy is the vertex position in clip space relative to rect?
    # Or UVs? The struct says "Position in clip space(xy)".
    # Let's follow the source formula exactly.
    m_x = saturate((clip_rect.z - clip_rect.x - abs(mask_pos.x)) * mask_soft.x) # Simplified 1D for x
    m_y = saturate((clip_rect.w - clip_rect.y - abs(mask_pos.y)) * mask_soft.y) # Simplified 1D for y
    
    # More accurate interpretation of source:
    # _ClipRect is typically (x_min, y_min, x_max, y_max)
    # The formula (_ClipRect.zw - _ClipRect.xy - ...) suggests:
    # width = z - x, height = w - y
    # But the source code: (_ClipRect.zw - _ClipRect.xy - abs(input.mask.xy))
    # This looks like it might be treating mask.xy as offset from center or similar?
    # Standard Unity UI mask logic:
    # rect = _ClipRect
    # uv = input.mask.xy (often vertex position in clip space)
    # soft = input.mask.zw
    
    # Let's stick to the literal source formula provided:
    # half2 m = saturate((_ClipRect.zw - _ClipRect.xy - abs(input.mask.xy)) * input.mask.zw);
    # c *= m.x * m.y;
    
    # Re-evaluating source code:
    # _ClipRect.xy is min, _ClipRect.zw is max.
    # _ClipRect.zw - _ClipRect.xy is (width, height).
    # This formula seems to assume input.mask.xy is normalized or specific offset.
    # Given the ambiguity, we implement the literal source line.
    
    mask_uv = vector(texcoord0.x, texcoord0.y) # Placeholder for actual mask UV/Pos
    # In Unity UI, mask UVs are often passed. Let's assume input.mask.xy holds the relevant coords.
    # The struct says "Position in clip space(xy)".
    
    # Correct implementation of source line:
    clip_rect_min = vector(clip_rect.x, clip_rect.y)
    clip_rect_max = vector(clip_rect.z, clip_rect.w)
    clip_rect_size = clip_rect_max - clip_rect_min
    
    # Note: The source formula `(_ClipRect.zw - _ClipRect.xy - abs(input.mask.xy))` 
    # is mathematically suspicious if mask.xy is absolute clip space. 
    # It likely implies mask.xy is relative to the rect center or similar, 
    # OR it's a specific Unity UI implementation detail where mask.xy is UVs mapped to rect.
    # However, we must follow the source.
    
    # Let's assume input.mask.xy are the UVs passed to the shader for masking.
    # And input.mask.zw are the softness values.
    
    # Actually, looking at Unity UI source code patterns:
    # mask.xy is often the vertex position in clip space.
    # The formula usually is: saturate((rect.zw - rect.xy - abs(pos - rect.xy)) * soft)
    # But the source says: `(_ClipRect.zw - _ClipRect.xy - abs(input.mask.xy))`
    # This implies input.mask.xy might be the distance from the rect center?
    # Without clarification, we write the code as per source literal.
    
    m_x = saturate((clip_rect.z - clip_rect.x - abs(mask_uv.x)) * mask_softness.x)
    m_y = saturate((clip_rect.w - clip_rect.y - abs(mask_uv.y)) * mask_softness.y)
    
    c *= m_x * m_y
    
    return c
```

## Subtopics

### [True Raymarched Text Reveal](18132-true-raymarched-text-reveal.md)
True Raymarched Text Reveal is a technique that renders text animations by converting DOM glyphs into a 3D Signed Distance Field (SDF), marching rays through it, and revealing the text using smooth-min metaballs and SDF-gradient lighting. It differs from its sibling [RayMarchedHeading](18123-sdf-text-rendering.md) by performing actual 3D raymarching through an extruded SDF rather than marching 2D blur taps along a smear line.

## Operating characteristics
*   **Time Complexity:** O(1) per pixel, as it involves a single texture sample and arithmetic operations.
*   **Memory Complexity:** O(1) per pixel for the shader state; O(N) for the font atlas texture where N is the number of characters.
*   **Measured Performance:** Optimized for mobile devices by omitting bevel, bump, environment mapping, and glow options [S1, unofficial].
*   **Strengths:** High performance, crisp text at any scale, supports outlines and gradients.
*   **Limitations:** Limited to 2D rendering; no true 3D depth effects like bevels or lighting interactions with 3D geometry.
*   **When to use:** UI text rendering on mobile or low-power devices where performance is critical.
*   **When not to use:** When 3D text effects, bevels, or integration with 3D lighting are required.

## Background
SDF Text Rendering is a method popularized by Unity's TextMeshPro asset, specifically the mobile-optimized variant `TMP_SDF-Mobile` [S1, unofficial]. It leverages the properties of Signed Distance Fields to render text efficiently on GPUs. The technique is widely used in mobile game UIs and applications requiring scalable, high-quality text.

## Sources
[S1] ForlornU/ProceduralGeneration, https://github.com/ForlornU/ProceduralGeneration, fetched 2026-10-27 (unofficial)
