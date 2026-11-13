# Transfer Function Mapping via 1D Textures

Also called: Transfer Function (1D), One-dimensional transfer function

## Overview
This method maps voxel scalar density values to RGBA color and opacity attributes using pre-sampled 1D textures, enabling the user to define visual properties based on density thresholds [S2, unofficial]. It distinguishes itself from multi-dimensional transfer functions by relying solely on density, whereas siblings like [Gradient-based Transfer Function](1273-gradient-based-transfer-function.md) incorporate gradient magnitude to highlight boundaries [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar density value $d$ for a voxel or ray sample.
*   **Output:** A 4-component RGBA vector $(r, g, b, a)$ representing the color and opacity at that density.

**Data Structures:**
The transfer function is stored as two parallel 1D textures or a single 2D texture with height 1 due to platform limitations [S1, unofficial][S3, unofficial].

1.  **Positions Texture (Thresholds):**
    *   Format: `gl.RED` (single channel).
    *   Content: Stores the density threshold values (knots) along the x-axis.
    *   Size: $N$ texels, where $N$ is the number of knots.
2.  **Samples Texture (Mapped Values):**
    *   Format: `gl.RED` for opacity mapping or `gl.RGB` for color mapping.
    *   Content: Stores the mapped RGBA values corresponding to the positions.
    *   Size: $N$ texels, aligned with the Positions texture.

*Note: In Unity, 1D textures are unsupported. The workaround is to use a 2D texture with height 1, where the x-axis represents density and the y-axis represents opacity or color [S3, unofficial].*

**Parameters:**
*   Texture Filtering: `MIN_FILTER` and `MAG_FILTER` set to `NEAREST` [S1, unofficial] or `GL_LINEAR` [S2, unofficial].
*   Texture Wrapping: `S`, `T`, `R` wrap modes set to `CLAMP_TO_EDGE` [S1, unofficial][S2, unofficial].
*   Unpack Alignment: Set to 1 [S1, unofficial].
*   Shader Uniforms: Indices for texture units (e.g., `opacityMapSamples`, `opacityMapPositions`, `colorMapSamples`, `colorMapPositions`) [S1, unofficial].

**Steps:**
1.  **User Definition:** The user defines color knots and alpha knots in a UI box where the x-axis represents density and the y-axis defines opacity or color [S2, unofficial].
2.  **Texture Generation:** Generate a 1D texture (or 2D height 1) where the x-axis is density and the texel's RGBA defines the color and opacity [S2, unofficial].
3.  **Shader Lookup:** Bind the Positions texture to one texture unit and the Samples texture to another [S1, unofficial]. In the volume rendering shader, after computing the density $d$ of a voxel or sample, use the density to look up the RGBA value in the transfer function texture [S2, unofficial].

**Formulas:**
The density $d$ is normalized to the texture coordinate range $[0, 1]$ using the known density bounds $d_{\min}$ and $d_{\max}$ [G]:
$$c = \max(0.0, \min(1.0, \frac{d - d_{\min}}{d_{\max} - d_{\min}}))$$
For a 2D height 1 texture, the lookup coordinate is $(c, 0.5)$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lookup_transfer_function(density, min_density, max_density, samples_tex, is_2d_height_1=True):
    """
    density: float, the scalar value of the voxel
    min_density: float, minimum density in the volume
    max_density: float, maximum density in the volume
    samples_tex: list of vec4, RGBA values at each knot
    is_2d_height_1: bool, True if using Unity's 2D height=1 workaround
    """
    # Normalize density to [0, 1]
    coord = (density - min_density) / (max_density - min_density)
    coord = max(0.0, min(1.0, coord))
    
    # Map coordinate to texture index
    tex_width = len(samples_tex)
    idx = int(coord * (tex_width - 1))
    idx = max(0, min(tex_width - 1, idx))
    
    # Return sample at index (NEAREST filter behavior)
    # For LINEAR filter, interpolate between samples_tex[idx] and samples_tex[idx+1]
    return samples_tex[idx]
```

**Edge Cases and Pitfalls:**
*   Density values outside the $[d_{\min}, d_{\max}]$ range must be clamped to $[0, 1]$ to prevent out-of-bounds texture sampling [G].
*   Platform limitations may require the 2D height 1 workaround instead of native 1D textures [S3, unofficial].
*   When sources specify different filtering modes, `NEAREST` [S1, unofficial] avoids interpolation artifacts but may cause banding, while `GL_LINEAR` [S2, unofficial] provides smooth transitions.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel or ray sample for the texture lookup [G].
*   **Memory Complexity:** $O(N)$ where $N$ is the number of transfer function knots [G].
*   **Limitations:** Cannot distinguish between materials with similar densities but different boundary characteristics, requiring gradient magnitude for boundary emphasis [S2, unofficial].
*   **When to use:** When density is the primary discriminator of material properties [S2, unofficial].
*   **When not to use:** When boundary emphasis is required or when multiple materials share similar density ranges [S2, unofficial].

## Background
This method is a fundamental technique in direct volume rendering, allowing users to define visual properties based on scalar data [S2, unofficial]. It is widely used in medical visualization to distinguish structures such as bone, blood vessels, and skin [S2, unofficial]. The implementation details regarding parallel 1D textures and platform-specific workarounds are documented in technical repositories and presentation decks [S1, unofficial][S3, unofficial].

## Sources
[S1] termijn/webgl-volumerendering, https://github.com/termijn/webgl-volumerendering, fetched 2026-10-27 (unofficial)
[S2] GitHub - priscilla100/medical_volume_renderer, https://github.com/priscilla100/medical_volume_renderer/, fetched 2026-10-29 (unofficial)
[S3] Volume Rendering in Unity3D - Speaker Deck, https://speakerdeck.com/mlavik1/volume-rendering-in-unity3d, fetched 2026-10-30 (unofficial)
