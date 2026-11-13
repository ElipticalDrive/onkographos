# Fast Paint Texture

Also called: Fast Paint Texturing, Hertzmann’s fast paint texture

## Overview
Fast Paint Texture is a stylization technique that simulates 3D paint undulations and overlap shadows on 2D strokes by generating a height field from stroke height maps and overlap counts, rendered via Phong shading, thereby avoiding the computational cost of 3D mesh generation [S1]. It distinguishes itself from sibling methods like [Interactive Volume Illustration](5272-interactive-volume-illustration-hatching.md) by explicitly modeling the physical accumulation of paint layers through a 2D height field rather than relying solely on surface normals or simple shading, and from [Visual Complexity (VC) Algorithm](14351-visual-complexity-vc-algorithm.md) by providing the specific texturing pass mechanism used within the VC framework to create embossed strokes [S1].

## How it works
**Inputs:** A list of strokes, where each stroke has an associated color, opacity map, and height map. The background is initialized to black with full opacity.
**Outputs:** A final RGB image with simulated paint depth and lighting.

**Data Structures:**
1.  **Color Buffer:** Stores the RGB components of the base stroke colors. Initialized to $(0,0,0)$.
2.  **Overlap Weight Buffer:** Stores the red component of the color buffer after the Overlap Pass. Represents the count of overlapping strokes. Initialized to $0$.
3.  **Height Buffer:** Stores the red component of the color buffer after the Height Pass. Represents the height map value of the last drawn stroke. Initialized to $0$.

**Steps:**

1.  **Color Pass:** Draw each stroke colored and textured with an opacity map. Save the RGB components in the Color Buffer.
    *   Stroke color is set to $(1,1,1,\alpha)$.
    *   Background is $(0,0,0,1)$.
    *   This pass records the base color information.

2.  **Overlap Pass:** Draw each stroke white and textured with the same opacity maps used in the Color Pass. Alpha blend with transparency $0 < \alpha \le 1$ to count overlapping strokes. Save the red component of the color buffer in the Overlap Weight Buffer.
    *   The red component $R_{i,k}$ at pixel $p_i$ after drawing stroke $k$ is calculated as:
        $$
        R_{i,k} = \begin{cases} 
        0 & \text{if } k = 0 \\
        \alpha + R_{i,k-1}(1 - \alpha) & \text{if } k > 0
        \end{cases}
        $$
    *   This compactly defines the count of overlapping strokes [S1].

3.  **Height Pass:** Draw each stroke white and textured with the height map paired with the opacity map. Each pixel $p_i$ contains the grey value (height value) of the last stroke drawn at $p_i$. Save the red component of the color buffer in the Height Buffer.
    *   This pass records the local texture height of the most recent stroke.

4.  **Height Field Calculation:** For each pixel $p_i$, compute the final height value $h(p_i)$ as the sum of the stroke’s height map value at $p_i$ plus a weight representing the number of strokes overlapping at $p_i$.
    $$
    h(p_i) = \text{height\_map}(p_i) + \text{overlap\_weight}(p_i)
    $$
    *   Calculate per-pixel normals using directional derivatives of the height field [S1].

5.  **Final Rendering:** For each pixel $p_i$, combine the stroke’s diffuse color components (from Color Pass) and the surface normal (from Height Pass) using the Phong model to obtain the final RGB color value.
    *   The diffuse component for the red channel is:
        $$
        C_i^R = I_a R_k^a O_i^R d_R + f_{att} I_p^R O_i^R d_R (N_i \cdot L) + \dots
        $$
    *   Similar calculations are performed for Green and Blue channels [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fast_paint_texture(strokes, width, height):
    # Initialize buffers
    color_buffer = np.zeros((height, width, 3))
    overlap_buffer = np.zeros((height, width))
    height_buffer = np.zeros((height, width))
    
    # Pass 1: Color Pass
    for stroke in strokes:
        # Draw stroke with its color and opacity map
        # Save RGB to color_buffer
        pass

    # Pass 2: Overlap Pass
    for stroke in strokes:
        # Draw stroke white with opacity map
        # Alpha blend into overlap_buffer (using Red channel logic)
        # R_new = alpha + R_old * (1 - alpha)
        pass

    # Pass 3: Height Pass
    for stroke in strokes:
        # Draw stroke white with height map and opacity map
        # Save height value to height_buffer
        pass

    # Compute final height field and normals
    final_height_field = height_buffer + overlap_buffer
    normals = compute_normals(final_height_field)

    # Final Rendering
    final_image = np.zeros((height, width, 3))
    for y in range(height):
        for x in range(width):
            # Get base color from color_buffer
            base_color = color_buffer[y, x]
            # Get normal from computed normals
            normal = normals[y, x]
            # Apply Phong shading
            final_image[y, x] = phong_shading(base_color, normal)
            
    return final_image
```

**Parameters:**
*   **Alpha ($\alpha$):** Transparency value used in blending passes. Range: $0 < \alpha \le 1$ [S1].

**Edge Cases and Pitfalls:**
*   **Stroke Orientation Approximation:** Bézier curves approximate, not interpolate, control points. The curve may not pass through control points B and C but is pulled in their direction. Curved strokes are only drawn where the underlying data attribute mapped to orientation varies rapidly [S1].
*   **Precision:** The overlap count relies on the specific alpha blending formula. Deviations from the standard $R_s \alpha + R_d (1-\alpha)$ blending may result in incorrect stroke counts [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of strokes and image resolution, as it requires three passes over the stroke list and one pass over the image for shading.
*   **Memory Complexity:** Requires storage for three buffers (Color, Overlap, Height) of size proportional to the image resolution.
*   **Strengths:** Avoids rendering expensive 3D mesh models of strokes while producing similar visual results (shadows/undulations) by using a 2D height field and lighting [S1].
*   **Limitations:** Relies on the assumption that stroke overlap can be approximated by alpha blending, which may not perfectly represent physical paint accumulation in all cases.

## Background
Fast Paint Texture was introduced by Hertzmann in 2002 as a method to simulate the visual properties of paint on a canvas, specifically undulations and overlap shadows, without the computational overhead of 3D geometry [S1]. It is widely used in non-photorealistic volume rendering, particularly within the Visual Complexity (VC) algorithm, to create embossed strokes that enhance the perception of depth and texture in volumetric data [S1].

## Sources
[S1] Engaging viewers through nonphotorealistic visualizations, https://doi.org/10.1145/1274871.1274886, fetched 2026-10-28
