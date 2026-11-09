# SDF Underlay & Inner Underlay

Also called: SDF Border Effect, TextMeshPro Underlay

## Overview
SDF Underlay & Inner Underlay renders a blurred, colored border either behind (underlay) or inside (inner underlay) the primary glyph face by sampling the Signed Distance Field (SDF) atlas at an offset UV coordinate, scaling the distance, and blending it with the face color using pre-multiplied alpha [S1, unofficial]. It differs from standard outline methods by allowing the border to be offset in UV space independently of the glyph geometry and by supporting "inner" rendering where the border appears within the filled region of the character [S1, unofficial].

## How it works
### Inputs and Outputs
*   **Inputs**: `atlas` (SDF texture), `faceColor` (pre-multiplied alpha), `underlayColor`, `texcoord2` (UV offset `.xy`, scale factor `.z`, bias `.w`), `scale`, `GradientScale`, `TextureWidth`, `TextureHeight`, `ScaleRatioC`, `UnderlayOffsetX`, `UnderlayOffsetY`, `UnderlayDilate`, `UnderlaySoftness`, `weight` [S1, unofficial].
*   **Outputs**: `faceColor` (updated with underlay blended) [S1, unofficial].

### Data Structures
The shader input structure packs the offset UV coordinates in `texcoord2.xy`, a scalar scale factor in `texcoord2.z`, and a pre-computed bias value in `texcoord2.w` [S1, unofficial].

### Steps
1.  Compute offset UV: `bOffset = (-UnderlayOffsetX * ScaleRatioC * GradientScale / TextureWidth, -UnderlayOffsetY * ScaleRatioC * GradientScale / TextureHeight)` [S1, unofficial].
2.  Compute underlay scale/bias: `bScale = scale / (1 + UnderlaySoftness * ScaleRatioC * scale)` and `bBias = (0.5 - weight) * bScale - 0.5 - (UnderlayDilate * ScaleRatioC * 0.5 * bScale)` [S1, unofficial].
3.  Pass to pixel shader [S1, unofficial].
4.  Sample distance: `d = sample_texture(atlas, texcoord2.xy).a * texcoord2.z` [S1, unofficial].
5.  Standard underlay: `faceColor += underlayColor * saturate(d - texcoord2.w) * (1 - faceColor.a)` [S1, unofficial].
6.  Inner underlay: `faceColor += underlayColor * (1 - saturate(d - texcoord2.w)) * saturate(1 - sd) * (1 - faceColor.a)` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def saturate(x):
    return max(0.0, min(1.0, x))

def sample_texture(tex, uv):
    # Standard bilinear texture sampling returning a 4-component vector [r, g, b, a]
    return tex.bilinear_sample(uv)

def compute_sdf_underlay(atlas, faceColor, underlayColor, texcoord2, scale, GradientScale, TextureWidth, TextureHeight, ScaleRatioC, UnderlayOffsetX, UnderlayOffsetY, UnderlayDilate, UnderlaySoftness, weight):
    # Step 1: Compute offset UV
    bOffset = (-UnderlayOffsetX * ScaleRatioC * GradientScale / TextureWidth, -UnderlayOffsetY * ScaleRatioC * GradientScale / TextureHeight)

    # Step 2: Compute scale and bias
    bScale = scale / (1.0 + UnderlaySoftness * ScaleRatioC * scale)
    bBias = (0.5 - weight) * bScale - 0.5 - (UnderlayDilate * ScaleRatioC * 0.5 * bScale)

    # Step 3 & 4: Sample distance at offset UV
    d = sample_texture(atlas, texcoord2.xy).a * texcoord2.z

    # Step 5: Standard underlay (behind)
    faceColor = faceColor + underlayColor * saturate(d - texcoord2.w) * (1.0 - faceColor.a)

    # Step 6: Inner underlay (inside)
    # sd is not defined in the sources [G]
    sd = 0.0
    faceColor = faceColor + underlayColor * (1.0 - saturate(d - texcoord2.w)) * saturate(1.0 - sd) * (1.0 - faceColor.a)

    return faceColor
```

### Parameters
*   `_UnderlayColor` (Color, default `0,0,0,0.5`): Border color [S1, unofficial].
*   `_UnderlayOffsetX` (float, default `0`): Horizontal UV offset for the underlay sample [S1, unofficial].
*   `_UnderlayOffsetY` (float, default `0`): Vertical UV offset for the underlay sample [S1, unofficial].
*   `_UnderlayDilate` (float, default `0`): Expands or shrinks the underlay region [S1, unofficial].
*   `_UnderlaySoftness` (float, default `0`): Blur radius for the underlay edge [S1, unofficial].

### Edge Cases and Pitfalls
Not given in the sources.

## Operating characteristics
*   **Time Complexity**: $O(1)$ per pixel, requiring one additional texture sample and a few arithmetic operations [S1, unofficial].
*   **Memory Complexity**: $O(1)$ additional memory per pixel for intermediate calculations [S1, unofficial].
*   **Performance**: The additional texture sample increases memory bandwidth usage, which can be significant on mobile GPUs if the atlas is not cached in L1/L2 [S1, unofficial].
*   **Strengths**: Provides a soft, blurred border effect that integrates well with the SDF's inherent anti-aliasing [S1, unofficial].
*   **Limitations**: The inner underlay implementation relies on an undefined `sd` parameter, and performance degrades if the underlay offset causes sampling far from the primary UV [S1, unofficial].
*   **When to use**: When a soft, blurred border is needed for text readability or aesthetic effect [S1, unofficial].
*   **When not to use**: When precise, hard-edged outlines are required [S1, unofficial].

## Background
The method is primarily associated with the Unity UI TextMeshPro package, where it is implemented as the "Underlay" effect to improve readability against complex backgrounds [S1, unofficial]. The technique leverages the signed distance field's ability to represent smooth transitions, using the distance value as a mask for the underlay color [S1, unofficial].

## Sources
[S1] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
