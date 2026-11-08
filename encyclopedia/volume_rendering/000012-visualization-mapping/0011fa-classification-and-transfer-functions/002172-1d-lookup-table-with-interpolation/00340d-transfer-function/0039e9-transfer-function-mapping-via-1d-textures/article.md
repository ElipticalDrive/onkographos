# Transfer Function Mapping via 1D Textures

Also called: Color Transfer Function, One-dimensional transfer function, Opacity Function, Parallel 1D Texture TF Mapping, Transfer Function (1D), Two-Texture Transfer Function

## Overview
This method maps voxel scalar density values to RGBA color and opacity attributes using pre-sampled 1D textures, enabling the user to define visual properties (e.g., bone opacity, blood vessel transparency) based on density thresholds [S2, unofficial]. It distinguishes itself from multi-dimensional transfer functions by relying solely on density, whereas siblings like [Gradient-based Transfer Function](1273-gradient-based-transfer-function.md) incorporate gradient magnitude to highlight boundaries [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar density value $d$ for a voxel or ray sample.
*   **Output:** A 4-component RGBA vector $(r, g, b, a)$ representing the color and opacity at that density.

**Data Structures:**
The transfer function is stored as two parallel 1D textures (or a 2D texture with height 1 due to platform limitations) [S1, S2, unofficial].

1.  **Positions Texture (Thresholds):**
    *   Format: `gl.RED` (single channel).
    *   Content: Stores the density threshold values (knots) along the x-axis.
    *   Size: $N$ texels, where $N$ is the number of knots.
2.  **Samples Texture (Mapped Values):**
    *   Format: `gl.RED` for opacity mapping or `gl.RGB` for color mapping.
    *   Content: Stores the mapped RGBA values corresponding to the positions.
    *   Size: $N$ texels, aligned with the Positions texture.

*Note: In Unity, 1D textures are unsupported. The workaround is to use a 2D texture with height=1, where the x-axis represents density and the y-axis represents opacity/color [S2, unofficial].*

**Parameters:**
*   Texture Filtering: `MIN_FILTER` and `MAG_FILTER` set to `NEAREST` [S1, unofficial].
*   Texture Wrapping: `S`, `T`, `R` wrap modes set to `CLAMP_TO_EDGE` [S1, unofficial].
*   Unpack Alignment: Set to 1 [S1, unofficial].
*   Shader Uniforms: Indices for texture units (e.g., `opacityMapSamples`, `opacityMapPositions`, `colorMapSamples`, `colorMapPositions`) [S1, unofficial].

**Steps:**
1.  **User Definition:** The user defines color knots and alpha knots in a UI box. The x-axis represents density, and the y-axis defines opacity/color [S2, unofficial].
2.  **Texture Generation:**
    *   Generate the Positions texture containing the density thresholds.
    *   Generate the Samples texture containing the corresponding RGBA values.
    *   Configure texture parameters (filtering, wrapping, alignment) [S1, S2, unofficial].
3.  **Shader Lookup:**
    *   Bind the Positions texture to one texture unit (e.g., Unit 1) and the Samples texture to another (e.g., Unit 2) [S1, unofficial].
    *   In the volume rendering shader, after computing the density $d$ of a voxel/sample:
    *   Use $d$ as the texture coordinate to look up the RGBA value in the Samples texture, using the Positions texture to define the valid range or as a parallel lookup if the implementation requires explicit threshold matching [S1, S2, unofficial].
    *   *Clarification:* Standard 1D TF mapping typically uses the density directly as the coordinate in a continuous texture. However, the "Parallel 1D Texture" approach described in [S1, unofficial] suggests using two textures: one for positions and one for samples. This implies a lookup mechanism where the density is used to index into the Positions texture to find the nearest threshold, or the density is used directly as a coordinate in a texture that stores the mapped values, with the "Positions" texture serving as a reference or for discrete knot definition. Given the description "use the density to look up the RGBA value in the transfer function texture" [S2, unofficial], the most robust implementation for a programmer is to treat the Samples texture as a 1D (or 2D height=1) texture where the x-coordinate is the density, and the value is the RGBA. The "Positions" texture may be used for discrete knot management or if the mapping is not monotonic/continuous in a simple way. For a standard implementation, the density $d$ is normalized to the texture range $[0, 1]$ and sampled from the Samples texture [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assuming standard 1D texture mapping where density maps directly to color/opacity
# Positions texture is used to define the range or knots, but lookup is typically direct
# If using the parallel texture approach for discrete knots:

def lookup_transfer_function(density, positions_tex, samples_tex, tex_width):
    """
    density: float, the scalar value of the voxel
    positions_tex: array of floats, the density thresholds (knots)
    samples_tex: array of vec4, the RGBA values corresponding to knots
    tex_width: int, number of knots
    """
    # Normalize density to [0, 1] assuming density range is known
    # Or use density directly if texture is sized to density range
    # For this implementation, we assume the texture stores the mapping
    # and we use linear interpolation between knots if needed, 
    # or nearest if NEAREST filter is used.
    
    # Standard approach: Map density to texture coordinate
    # Let's assume density is in [min_d, max_d]
    min_d = positions_tex[0]
    max_d = positions_tex[-1]
    
    if density < min_d:
        return samples_tex[0]
    if density > max_d:
        return samples_tex[-1]
        
    # Calculate index in the texture
    # If using NEAREST filter, we round to nearest knot
    # If using LINEAR, we interpolate
    # The sources specify NEAREST filter [S1, unofficial]
    
    # Map density to index [0, tex_width-1]
    # Assuming uniform distribution of knots for simplicity, 
    # or that positions_tex defines the exact indices
    # A more robust way given "Parallel 1D Texture":
    # Find the index i such that positions_tex[i] <= density < positions_tex[i+1]
    
    idx = 0
    for i in range(tex_width - 1):
        if positions_tex[i] <= density < positions_tex[i+1]:
            idx = i
            break
    else:
        idx = tex_width - 1
        
    # Return the sample at the found index
    # If NEAREST filter is used, the texture lookup itself handles this
    # If we are doing it in shader, we might just use the density as coordinate
    # in a texture that has been filled with the samples.
    
    # Simplified: Use density as coordinate in Samples texture
    # This assumes the texture was generated such that x=density maps to RGBA
    coord = (density - min_d) / (max_d - min_d)
    coord = clamp(coord, 0.0, 1.0)
    
    # In shader: texture(samples_tex, vec2(coord, 0.5)) for 2D height=1
    # Or texture(samples_tex, coord) for 1D
    
    return samples_tex[idx] # Placeholder for actual texture lookup
```

*Note: The pseudocode above is a logical reconstruction. In practice, the shader performs the texture lookup directly using the density as the coordinate [S2, unofficial]. The "Parallel" aspect in [S1, unofficial] likely refers to the separation of threshold definition and value storage, but the lookup is still density-driven.*

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel/sample for the texture lookup.
*   **Memory Complexity:** $O(N)$ where $N$ is the number of transfer function knots.
*   **Strengths:** Simple to implement, efficient, allows intuitive user interaction via color/alpha bars [S2, unofficial].
*   **Limitations:** Cannot distinguish between materials with similar densities but different boundary characteristics (requires gradient magnitude, see [Gradient-based Transfer Function](1273-gradient-based-transfer-function.md)) [S2, unofficial].
*   **When to use:** When density is the primary discriminator of material properties.
*   **When not to use:** When boundary emphasis is required or when multiple materials share similar density ranges.

## Background
This method is a fundamental technique in direct volume rendering, allowing users to define visual properties based on scalar data. It is widely used in medical visualization (e.g., distinguishing bone, blood, and skin) [S2, unofficial]. The implementation details regarding parallel 1D textures and Unity-specific workarounds are documented in unofficial sources [S1, S2, unofficial].

## Sources
[S1] termijn/webgl-volumerendering, https://github.com/termijn/webgl-volumerendering, fetched 2026-10-27 (unofficial)
[S2] Volume Rendering in Unity3D - Speaker Deck, https://speakerdeck.com/mlavik1/volume-rendering-in-unity3d, fetched 2026-10-30 (unofficial)
