# Slice-Based Volume Rendering with HSV Transfer Function

Also called: Slice Rendering, Slice-Based Volume Rendering, Texture Array Volume Rendering

## Overview
Slice-Based Volume Rendering with HSV Transfer Function is a GPU-accelerated object-order technique that renders volumetric data by projecting a series of 2D slices onto the image plane and blending them back-to-front, while mapping scalar values to color using a Hue-Saturation-Value (HSV) transfer function [S1, unofficial][S2, unofficial]. It differs from standard 2D texture-based volume rendering by explicitly managing a texture array (sprite sheet) to simulate 3D trilinear filtering and by computing color in HSV space rather than using a pre-baked 1D RGB texture lookup [S1, unofficial]. It differs from its sibling [Novel Sampling Scheme for Slice Based Volume Rendering](953-novel-sampling-scheme-for-slice-based-volume-rendering.md) by relying on fixed sprite-sheet indexing rather than a novel sampling distribution, and from [Interpolation of Arbitrary Slices (Multi-Texture Blending)](2089-interpolation-of-arbitrary-slices-multi-texture-blending.md) by using a single texture array with calculated intra-slice coordinates instead of blending multiple distinct texture units [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset stored as a 2D texture array (sprite sheet), camera parameters (view/projection matrices), and HSV transfer function parameters (darkness, lightness, saturation, hue range).
*   **Output:** A 2D color buffer with the blended volume image.

**Data Structures:**
*   **Volume Texture:** A 2D texture array or a single large 2D texture acting as a sprite sheet.
    *   Layout: Slices are arranged in a grid. `uSlicesOverX` slices are arranged horizontally per sprite sheet row.
    *   Fields: `uNumberOfSlices` (total slices), `slicesPerSprite` (slices per texture unit if using an array, or derived from sprite dimensions), `uSlicesOverX`, `uSlicesOverY`.
*   **Vertex Buffers:**
    *   Attribute 0: `Vector3f` (3 floats) for position [S2, unofficial].
    *   Attribute 1: `Vector2f` (2 floats) for UVs (optional) [S2, unofficial].
    *   Attribute 2: `Vector3f` (3 floats) for normals (optional) [S2, unofficial].
*   **Transfer Function Parameters:**
    *   `darkness`: Global brightness/darkness multiplier [S1, unofficial].
    *   `l`: Lightness scaling factor [S1, unofficial].
    *   `s`: Saturation scaling factor [S1, unofficial].
    *   `hMin`, `hMax`: Minimum and maximum hue values [S1, unofficial].
    *   `minSos`, `minRefl`, `minAtten`, `maxSos`, `maxRefl`, `maxAtten`: Threshold parameters for slice selection/filtering (exact usage not detailed) [S1, unofficial].

**Numbered Steps:**

1.  **Geometry Setup:** Bind vertex buffers for position, UVs, and normals. Bind the volume texture to texture unit 0 [S2, unofficial].
2.  **Slice Loop:** Iterate through each slice index `i` from 0 to `numberOfSlices - 1` [S2, unofficial].
    *   Calculate normalized slice position: `slicePos = (i / numberOfSlices) - 0.5` [S2, unofficial].
    *   Calculate total slice parameter: `totalSlices = (0.5 + i) / numberOfSlices` [S2, unofficial].
    *   Pass these as shader uniforms `sliceNumber` and `totalSlices` [S2, unofficial].
    *   Draw the slice geometry (quad or box) using `glDrawElements` [S2, unofficial].
3.  **Fragment Shader - Texture Indexing:** For each fragment, compute the 3D volume position `volpos` (normalized or world space depending on implementation) [S1, unofficial].
    *   Compute original slice index: `s1Original = floor(volpos.z * uNumberOfSlices)` [S1, unofficial].
    *   Determine texture array index: `tex1Index = int(floor(s1Original / slicesPerSprite))` [S1, unofficial].
    *   Compute intra-slice coordinates:
        *   `s1 = mod(s1Original, slicesPerSprite)` [S1, unofficial].
        *   `dx1 = fract(s1 / uSlicesOverX)` [S1, unofficial].
        *   `dy1 = floor(s1 / uSlicesOverY) / uSlicesOverY` [S1, unofficial].
    *   Compute normalized texture coordinates:
        *   `texpos1.x = dx1 + (volpos.x / uSlicesOverX)` [S1, unofficial].
        *   `texpos1.y = dy1 + (volpos.y / uSlicesOverY)` [S1, unofficial].
    *   Sample the volume value from the texture array using `tex1Index`, `texpos1`, and trilinear filtering [S1, unofficial].
4.  **Fragment Shader - HSV Transfer Function:**
    *   Map the sampled scalar value to HSV space. Apply scaling factors:
        *   `H = map(value, hMin, hMax)` [S1, unofficial].
        *   `S = value * s` (Saturation scaling) [S1, unofficial].
        *   `V = value * l * darkness` (Value/Lightness scaling) [S1, unofficial].
    *   Convert HSV to RGB:
        *   Normalize hue: `hue = H / 60.0` [S1, unofficial].
        *   Determine sector: `i = int(hue)` [S1, unofficial].
        *   Fractional part: `ff = hue - float(i)` [S1, unofficial].
        *   Calculate intermediate values:
            *   `p = V * (1.0 - S)` [S1, unofficial].
            *   `q = V * (1.0 - (S * ff))` [S1, unofficial].
            *   `t = V * (1.0 - (S * (1.0 - ff)))` [S1, unofficial].
        *   Assign RGB based on sector `i` (piecewise linear interpolation) [S1, unofficial].
5.  **Blending:** Blend the resulting RGB color with the accumulated color buffer using alpha blending (back-to-front) [S1, unofficial][S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader / Geometry Setup (Conceptual)
# 1. Bind VBOs for Position (3f), UV (2f), Normal (3f)
# 2. Bind Volume Texture (2D Array or Sprite Sheet) to Unit 0
# 3. Loop i from 0 to numberOfSlices - 1:
#    a. sliceNumber = (i / numberOfSlices) - 0.5
#    b. totalSlices = (0.5 + i) / numberOfSlices
#    c. Draw Slice Geometry (Quad/Box)

# Fragment Shader
uniform vec3 uCameraPos;
uniform mat4 uMVP;
uniform int uNumberOfSlices;
uniform int uSlicesOverX;
uniform int uSlicesOverY;
uniform float uDarkness;
uniform float uLightness;
uniform float uSaturation;
uniform float uHMin;
uniform float uHMax;
uniform sampler2D uVolumeTexture; // 2D Array or Sprite Sheet

vec3 hsv2rgb(vec3 hsv) {
    float hue = hsv.x;
    float sat = hsv.y;
    float val = hsv.z;
    
    hue = hue / 60.0;
    int i = int(hue);
    float ff = hue - float(i);
    
    float p = val * (1.0 - sat);
    float q = val * (1.0 - (sat * ff));
    float t = val * (1.0 - (sat * (1.0 - ff)));
    
    if (i == 0) return vec3(val, t, p);
    if (i == 1) return vec3(q, val, p);
    if (i == 2) return vec3(p, val, t);
    if (i == 3) return vec3(p, q, val);
    if (i == 4) return vec3(t, p, val);
    if (i == 5) return vec3(val, p, q);
    return vec3(0.0); // Default
}

void main() {
    // 1. Compute 3D position in volume space
    // Note: volpos depends on MVP and fragment position
    vec4 clipPos = uMVP * vec4(gl_FragCoord.xy, 0.5, 1.0); // Simplified
    vec3 volpos = clipPos.xyz / clipPos.w; // Normalized or specific space
    
    // 2. Texture Array Indexing
    float s1Original = floor(volpos.z * float(uNumberOfSlices));
    int tex1Index = int(floor(s1Original / float(uSlicesOverX * uSlicesOverY))); // Assuming slicesPerSprite = uSlicesOverX * uSlicesOverY if single texture
    // If using actual texture array, tex1Index is used for layer selection
    
    float s1 = mod(s1Original, float(uSlicesOverX * uSlicesOverY));
    float dx1 = fract(s1 / float(uSlicesOverX));
    float dy1 = floor(s1 / float(uSlicesOverY)) / float(uSlicesOverY);
    
    float texpos1_x = dx1 + (volpos.x / float(uSlicesOverX));
    float texpos1_y = dy1 + (volpos.y / float(uSlicesOverY));
    
    // 3. Sample Volume
    // Note: Actual sampling depends on whether it's a 2D array or sprite sheet
    // For sprite sheet: texture2D(uVolumeTexture, vec2(texpos1_x, texpos1_y))
    // For 2D array: texture(uVolumeTexture, vec3(texpos1_x, texpos1_y, float(tex1Index)))
    float volumeValue = texture(uVolumeTexture, vec2(texpos1_x, texpos1_y)).r; 
    
    // 4. HSV Transfer Function
    float hue = mix(uHMin, uHMax, volumeValue);
    float sat = volumeValue * uSaturation;
    float val = volumeValue * uLightness * uDarkness;
    
    vec3 rgb = hsv2rgb(vec3(hue, sat, val));
    
    // 5. Output
    gl_FragColor = vec4(rgb, 1.0); // Alpha blending handled by GPU state
}
```

**Edge Cases and Pitfalls:**
*   **Texture Array Indexing:** Miscalculation of `tex1Index` or intra-slice coordinates (`dx1`, `dy1`) leads to incorrect slice selection or coordinate wrapping artifacts [S1, unofficial].
*   **Precision:** Floating-point precision in `floor` and `mod` operations can cause slice boundary artifacts if not handled carefully [S1, unofficial].
*   **Degenerate Input:** If `uNumberOfSlices` is 0 or `uSlicesOverX` is 0, division by zero or invalid indexing occurs [S1, unofficial].

## Subtopics

### [Slice-based Volume Rendering using Pixel Bender](5445-slice-based-volume-rendering-using-pixel-bender.md)
Uses view-aligned slices and a static 3D texture via Pixel Bender shaders. Renders volume data directly without isosurface extraction, allowing translucent cross-sections.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of slices and $M$ is the number of fragments per slice. The loop over slices in the CPU (or shader) adds overhead compared to single-pass raycasting [S2, unofficial].
*   **Memory Complexity:** $O(V)$, where $V$ is the volume size, stored in the texture array. The texture array layout requires careful packing to minimize memory overhead [S1, unofficial].
*   **Measured Performance:** Performance depends on the number of slices and the complexity of the HSV transfer function. The explicit loop over slices can be a bottleneck if not optimized [S2, unofficial].
*   **Strengths:** Simple implementation, leverages GPU texture filtering, flexible color mapping via HSV [S1, unofficial].
*   **Limitations:** Slice alignment artifacts, potential for z-fighting if slices are too thin, overhead of loop management [S2, unofficial].
*   **When to Use:** Real-time applications requiring flexible color mapping and moderate volume sizes [S1, unofficial].
*   **When Not to Use:** High-resolution volumes requiring high performance where raycasting or texture-based methods are more efficient [S2, unofficial].

## Background
*   **Origin:** The method is derived from general slice-based volume rendering techniques, adapted for GPU implementation with HSV transfer functions [S1, unofficial][S2, unofficial].
*   **History:** Developed as part of various unofficial implementations for real-time volume rendering, leveraging OpenGL and shader programming [S1, unofficial][S2, unofficial].
*   **Usage:** Used in educational contexts, prototyping, and applications requiring custom color mapping for volumetric data [S1, unofficial][S2, unofficial].

## Sources
[S1] levl-dev/VolumeRayCasting_0, https://github.com/levl-dev/VolumeRayCasting_0, fetched 2026-10-26 (unofficial)
[S2] abhishektrip/SliceBasedVolumeRendering, https://github.com/abhishektrip/SliceBasedVolumeRendering, fetched 2026-10-27 (unofficial)
