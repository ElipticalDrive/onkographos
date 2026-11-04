# SDF Underlay & Inner Underlay

Also called: SDF Border Effect, TextMeshPro Underlay

## Overview
SDF Underlay & Inner Underlay renders a blurred, colored border either behind (underlay) or inside (inner underlay) the primary glyph face by sampling the Signed Distance Field (SDF) atlas at an offset UV coordinate, scaling the distance, and blending it with the face color using pre-multiplied alpha. It differs from standard outline methods by allowing the border to be offset in UV space independently of the glyph geometry and by supporting "inner" rendering where the border appears within the filled region of the character.

## How it works
The method extends the layered compositing pipeline by introducing a secondary sampling pass for the underlay effect. It relies on pre-computed UV offsets and scale/bias parameters derived from user-defined softness and dilation values.

### Inputs and Outputs
*   **Inputs**:
    *   `atlas`: SDF texture (typically RGBA, where Alpha channel stores signed distance).
    *   `faceColor`: The color and alpha of the primary glyph face, already computed and pre-multiplied.
    *   `underlayColor`: The desired color of the underlay border.
    *   `texcoord`: Primary UV coordinates for the face.
    *   `texcoord2`: Secondary UV coordinates for the underlay sample.
    *   `scale`: Global scaling factor for the SDF.
    *   `GradientScale`: Font-specific gradient scaling.
    *   `TextureWidth`: Width of the SDF atlas in pixels.
    *   `ScaleRatioC`: A constant ratio used for normalization (often related to pixel density or font size).
    *   Parameters: `_UnderlayOffsetX`, `_UnderlayOffsetY`, `_UnderlayDilate`, `_UnderlaySoftness`.
*   **Outputs**:
    *   `faceColor`: Updated color with the underlay blended in.

### Data Structures
The shader input structure typically contains:
*   `texcoord2.xy`: The offset UV coordinates for sampling the underlay.
*   `texcoord2.z`: A scalar value representing the scaled distance or weight (often `scale` or a derived factor).
*   `texcoord2.w`: A bias value pre-computed from dilation and softness parameters.

### Steps
1.  **Compute Offset UV**: Calculate the UV offset based on user-defined X/Y offsets, scaled by the font scale and texture dimensions.
    $$ bOffset = \left( \frac{-\text{UnderlayOffsetX} \times \text{ScaleRatioC} \times \text{GradientScale}}{\text{TextureWidth}}, \frac{-\text{UnderlayOffsetY} \times \text{ScaleRatioC} \times \text{GradientScale}}{\text{TextureHeight}} \right) $$
2.  **Compute Scale and Bias**: Determine the effective scale and bias for the underlay distance field.
    $$ bScale = \frac{\text{scale}}{1 + \text{UnderlaySoftness} \times \text{ScaleRatioC} \times \text{scale}} $$
    $$ bBias = (0.5 - \text{weight}) \times bScale - 0.5 - (\text{UnderlayDilate} \times \text{ScaleRatioC} \times 0.5 \times bScale) $$
    *Note: `weight` is typically a normalized alpha or coverage value associated with the face layer.*
3.  **Sample Distance**: Sample the SDF atlas at the offset UV coordinates.
    $$ d = \text{tex2D}(\text{atlas}, \text{texcoord2}.xy).\alpha \times \text{texcoord2}.z $$
    Here, `d` represents the scaled signed distance.
4.  **Render Standard Underlay**: Blend the underlay color behind the face. The underlay is visible where the scaled distance `d` exceeds the bias `texcoord2.w`.
    $$ \text{faceColor} \leftarrow \text{faceColor} + \text{underlayColor} \times \text{saturate}(d - \text{texcoord2}.w) \times (1 - \text{faceColor}.a) $$
5.  **Render Inner Underlay**: Blend the underlay color inside the face. This is active where the distance is *below* the bias (inside the shape) and the face alpha is not fully opaque (or to create a specific inner glow effect).
    $$ \text{faceColor} \leftarrow \text{faceColor} + \text{underlayColor} \times (1 - \text{saturate}(d - \text{texcoord2}.w)) \times \text{saturate}(1 - sd) \times (1 - \text{faceColor}.a) $$
    *Note: `sd` in the source notes likely refers to a specific distance threshold or the signed distance itself normalized. In standard implementations, this term often simplifies to checking if the point is inside the SDF contour.*

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs:
# texcoord2.xy: Offset UVs
# texcoord2.z: Scale factor (often 'scale')
# texcoord2.w: Pre-computed bias
# underlayColor: Color of the underlay
# faceColor: Current face color (pre-multiplied alpha)
# scale: Global SDF scale
# UnderlaySoftness: Blur radius
# UnderlayDilate: Expansion amount
# ScaleRatioC: Normalization constant
# TextureWidth: Atlas width
# GradientScale: Font gradient scale
# UnderlayOffsetX, UnderlayOffsetY: User offsets

# Step 1 & 2: Compute Bias (if not pre-computed by CPU)
# Note: 'weight' is often 0.5 for simple underlay or derived from face alpha
weight = 0.5 
bScale = scale / (1.0 + UnderlaySoftness * ScaleRatioC * scale)
bBias = (0.5 - weight) * bScale - 0.5 - (UnderlayDilate * ScaleRatioC * 0.5 * bScale)

# Step 3: Sample Distance
# tex2D returns a vector; we take the alpha channel as the distance value
d = tex2D(atlas, texcoord2.xy).a * texcoord2.z

# Step 4: Standard Underlay (Behind)
# saturate(x) is max(0, min(1, x))
underlay_factor = saturate(d - bBias)
faceColor = faceColor + underlayColor * underlay_factor * (1.0 - faceColor.a)

# Step 5: Inner Underlay (Inside)
# This adds color inside the shape where the distance is low (negative or small positive)
# 'sd' in source notes is ambiguous; assuming it refers to a secondary distance check or is 0 for pure inner fill
# If 'sd' is not provided, this term might be simplified to 1.0 for full inner coverage
sd = 0.0 
inner_factor = (1.0 - saturate(d - bBias)) * saturate(1.0 - sd)
faceColor = faceColor + underlayColor * inner_factor * (1.0 - faceColor.a)
```

### Parameters
*   `_UnderlayColor` (Color, default `0,0,0,0.5`): The color of the underlay border.
*   `_UnderlayOffsetX` (float, default `0`): Horizontal UV offset for the underlay sample.
*   `_UnderlayOffsetY` (float, default `0`): Vertical UV offset for the underlay sample.
*   `_UnderlayDilate` (float, default `0`): Expands (`>0`) or shrinks (`<0`) the underlay region.
*   `_UnderlaySoftness` (float, default `0`): Blur radius for the underlay edge.

### Edge Cases and Pitfalls
*   **Precision**: The bias calculation involves multiple scaling factors. If `ScaleRatioC` or `GradientScale` are very large, floating-point precision errors in `bBias` may cause jagged edges or incorrect cutoffs.
*   **Boundaries**: If `UnderlayOffsetX/Y` pushes the UV coordinates outside the atlas bounds, standard texture wrapping modes (Clamp, Repeat) will determine the behavior. Clamping is recommended to avoid sampling adjacent glyphs.
*   **Degenerate Input**: If `scale` is zero, `bScale` becomes zero, and the underlay distance `d` becomes zero, potentially rendering the underlay incorrectly depending on the bias.
*   **Alpha Overwrite**: The method uses `(1 - faceColor.a)` to blend. If the face is fully opaque (`a=1`), the underlay will not be visible, which is the intended behavior for "behind" rendering. For inner underlay, this ensures the inner color blends correctly with the face.

## Operating characteristics
*   **Time Complexity**: $O(1)$ per pixel, requiring one additional texture sample (`tex2D`) and a few arithmetic operations.
*   **Memory Complexity**: $O(1)$ additional memory per pixel for intermediate calculations.
*   **Performance**: The additional texture sample increases memory bandwidth usage. On mobile GPUs, this can be significant if the atlas is not cached in L1/L2. The arithmetic cost is negligible.
*   **Strengths**: Provides a soft, blurred border effect that integrates well with the SDF's inherent anti-aliasing. The offset capability allows for creative text effects (e.g., drop shadows, glows).
*   **Limitations**: The "inner underlay" implementation in the source notes is somewhat ambiguous regarding the `sd` variable. Performance degrades if the underlay offset causes sampling far from the primary UV, potentially missing the glyph entirely if the atlas is sparse.
*   **When to use**: When a soft, blurred border is needed for text readability or aesthetic effect, especially in UI contexts where crisp outlines are too harsh.
*   **When not to use**: When precise, hard-edged outlines are required (use standard SDF outlines instead) or when performance is critical and the additional texture sample is a bottleneck.

## Background
The method is primarily associated with the Unity UI TextMeshPro package, where it is implemented as the "Underlay" effect. It allows designers to add a blurred background to text to improve readability against complex backgrounds. The technique leverages the signed distance field's ability to represent smooth transitions, using the distance value as a mask for the underlay color.

## Sources
[S1] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
