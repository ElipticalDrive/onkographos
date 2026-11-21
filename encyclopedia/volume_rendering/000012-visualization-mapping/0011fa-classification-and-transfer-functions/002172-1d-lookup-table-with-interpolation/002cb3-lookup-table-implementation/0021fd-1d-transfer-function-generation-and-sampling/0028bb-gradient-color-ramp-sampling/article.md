# Gradient/Color Ramp Sampling

Also called: Color Ramp Sampling, Gradient Texture Sampling

## Overview
Gradient/Color Ramp Sampling is a method for generating a 2D color map (texture) from a set of key-value pairs defining color and opacity gradients, which is then sampled to map scalar data to RGBA values. It differs from direct 1D LUT interpolation by explicitly constructing a 2D texture where one axis represents the scalar value (mapped to color via a gradient) and the other axis often represents a secondary parameter like opacity or intensity, allowing for more complex visual mappings than simple 1D lookup [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of gradient stops (key-value pairs) defining color and optionally opacity/intensity curves. For the specific implementation cited, this involves a "sim data" channel laid out horizontally along the X-axis (color gradient) and a vertical Y-axis mapping (opacity/intensity) [S1, unofficial].
*   **Output:** A 2D texture (GradientTexture2D) containing the sampled color ramp.

**Data Structures:**
*   **Gradient Stops:** Defined by position (offset) and RGBA values.
*   **GradientTexture2D:** A 2D texture resource. If `use_hdr` is enabled, it may store channels above 1.0 [S1, unofficial].

**Steps:**
1.  **Define Gradient Stops:** Specify the color and opacity/intensity values at specific offsets (0.0 to 1.0).
2.  **Construct 2D Texture:**
    *   Map the scalar value to the X-axis. The color gradient remaps the sim data to fire color horizontally [S1, unofficial].
    *   Map the opacity/intensity to the Y-axis. In "Use Own Opacity" mode, this maps to fire opacity; otherwise, it maps to fire intensity [S1, unofficial].
3.  **Sample the Texture:**
    *   Read the luminance in float format.
    *   Truncate each channel for the color ramp, as done in Godot's `NoiseTexture2D` implementation (color.h:106-108, image.cpp:3577-3580) [S1, unofficial].
    *   Run the Gradient sampler and `channelToByte` in float, as per `Gradient::get_color_at_offset` and `Color::get_r8` (gradient.h:151-235, color.h:228) [S1, unofficial].
4.  **Handle HDR:** If `use_hdr` is true, ensure channels above 1.0 are preserved in the output texture (gradient_texture.cpp:239-270) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_gradient_ramp(scalar_value, opacity_value, gradient_stops, use_hdr=False):
    # 1. Map scalar to X, opacity to Y
    x = clamp(scalar_value, 0.0, 1.0)
    y = clamp(opacity_value, 0.0, 1.0)
    
    # 2. Sample the 2D gradient texture
    # Note: In the cited implementation, the color is mapped horizontally (X)
    # and opacity/intensity vertically (Y).
    color = sample_2d_texture(x, y, gradient_stops)
    
    # 3. Process channels
    # Read luminance in float and truncate each channel
    r = truncate_to_byte(color.r)
    g = truncate_to_byte(color.g)
    b = truncate_to_byte(color.b)
    a = truncate_to_byte(color.a)
    
    # 4. Handle HDR
    if use_hdr:
        # Keep channels above 1.0 if HDR is enabled
        # This step implies no truncation for HDR, or different handling
        pass
    else:
        # Standard truncation as per Godot's channelToByte
        r = channel_to_byte(r)
        g = channel_to_byte(g)
        b = channel_to_byte(b)
        a = channel_to_byte(a)
        
    return (r, g, b, a)

def truncate_to_byte(value):
    # Truncate float to byte range [0, 255]
    return int(clamp(value, 0.0, 1.0) * 255)

def channel_to_byte(value):
    # As per Color::get_r8
    return int(clamp(value, 0.0, 1.0) * 255)
```

**Parameters:**
*   `use_hdr`: Boolean. If true, preserves channels > 1.0. Default: False [S1, unofficial].
*   `gradient_stops`: List of (offset, color, opacity) tuples.

**Edge Cases and Pitfalls:**
*   **Precision:** Truncation can introduce quantization errors. Godot-measured ramp bytes off by one drop from 1212 to 0 of 2244 after fixing the ramp implementation [S1, unofficial].
*   **HDR:** If `use_hdr` is enabled, channels above 1.0 must be handled correctly to avoid clipping or incorrect display [S1, unofficial].
*   **Boundary Conditions:** Ensure scalar and opacity values are clamped to [0.0, 1.0] before sampling to avoid out-of-bounds access [S1, unofficial].

## Subtopics

### [Fire Color and Opacity Curves](16944-fire-color-and-opacity-curves.md)
Remaps grid simulation data to render data. Sim data channel is laid out horizontally along the X axis. Color gradient remaps sim data to fire color. Curve remaps sim data to fire opacity (in Use Own Opacity mode) or fire intensity (otherwise). Opacity/intensity is mapped vertically on the Y axis.

## Operating characteristics
*   **Time Complexity:** O(1) per sample, assuming constant-time texture lookup.
*   **Memory Complexity:** O(N) for the 2D texture, where N is the texture resolution.
*   **Performance:** The cited implementation shows significant error reduction after fixing the ramp (mean error 0.776/255 → 0.735/255 for `noisetexture2d`) [S1, unofficial].
*   **Strengths:** Allows for complex 2D mappings (color and opacity/intensity) in a single texture lookup.
*   **Limitations:** Requires pre-computation of the 2D texture. HDR handling adds complexity.

## Background
This method is used in Godot's rendering engine, specifically in `NoiseTexture2D` and `GradientTexture2D` nodes, to provide flexible color mapping for procedural textures and volumetric effects [S1, unofficial].

## Sources
[S1] feat(renderer): draw FogVolume and FogMaterial into the volumetric fog, https://github.com/Vortiago/textscene-inspector/pull/732, fetched 2026-10-26 (unofficial)
