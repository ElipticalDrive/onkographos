# Orientation-Based Color Modification

Also called: HSV Color Orientation, HSV color modification, oriented fading

## Overview
Orientation-Based Color Modification simulates medical illustration techniques by altering the color, saturation, value, and transparency of volume samples based on the angle between the volumetric gradient and the viewer [S1, S2, unofficial]. It addresses the lack of depth cues in translucent volume rendering by using HSV color space to desaturate features facing the viewer and darken/saturate receding surfaces, thereby enhancing spatial understanding [S1, S2, unofficial].

## How it works
**Inputs:**
*   Volume sample color $C_{\text{rgb}}$ (RGB triplet) [S1, S2, unofficial].
*   Volumetric gradient vector $\mathbf{g}$ at the sample location [S1, S2, unofficial].
*   Viewer direction vector $\mathbf{v}$ (pointing from the sample to the camera) [S1, S2, unofficial].

**Outputs:**
*   Modified color $C'_{\text{rgb}}$ and modified opacity $\alpha'$ [S1, S2, unofficial].

**Data Structures:**
*   HSV color space representation $(H, S, V)$, where $H \in [0, 360)$, $S \in [0, 1]$, $V \in [0, 1]$ [G].

**Steps:**
1.  **Convert to HSV:** Transform the input RGB color $C_{\text{rgb}}$ into HSV space to obtain $(H, S, V)$ [S2, unofficial].
2.  **Calculate Orientation Angle:** Compute the angle $\theta$ between the gradient $\mathbf{g}$ and the viewer direction $\mathbf{v}$ [S1, S2, unofficial]:
    $$ \theta = \arccos\left( \frac{\mathbf{g} \cdot \mathbf{v}}{\|\mathbf{g}\| \|\mathbf{v}\|} \right) $$
    Normalize $\mathbf{g}$ and $\mathbf{v}$ if their magnitudes are not inherently unit length [G].
3.  **Modify Value (V):** Decrease the Value component $V$ as the angle $\theta$ increases, simulating oriented fading where surfaces facing the viewer retain brightness and receding surfaces darken [S1, S2, unofficial]. The exact mathematical function for this decrease is not specified in the sources [S1, S2, unofficial].
4.  **Modify Saturation (Optional):** Desaturate surfaces oriented toward the viewer and saturate those oriented away [S2, unofficial]. The specific mapping for saturation modification is not defined in the sources [S1, S2, unofficial].
5.  **Convert to RGB:** Transform the modified HSV values $(H, S', V')$ back to RGB space to obtain $C'_{\text{rgb}}$ [S2, unofficial].
6.  **Transparency Modification:** Modify opacity based on orientation, though the specific formula is not detailed in the provided text [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def orientation_based_color_modification(color_rgb, gradient, viewer_direction):
    # 1. Convert RGB to HSV
    h, s, v = rgb_to_hsv(color_rgb)
    
    # 2. Calculate angle between gradient and viewer
    g_norm = gradient / max(norm(gradient), 1e-8)
    v_norm = viewer_direction / max(norm(viewer_direction), 1e-8)
    dot_product = clamp(dot(g_norm, v_norm), -1.0, 1.0)
    theta = acos(dot_product)
    
    # 3. Modify Value (V)
    # Sources do not specify the exact fading function [S1, S2, unofficial]
    v_modified = v * (1.0 - theta / 3.14159)  # Standard linear approximation [G]
    
    # 4. Modify Saturation (Optional)
    # Sources do not specify the exact saturation function [S1, S2, unofficial]
    s_modified = s * (1.0 + theta / 3.14159)  # Standard linear approximation [G]
    
    # 5. Convert back to RGB
    color_rgb_modified = hsv_to_rgb(h, s_modified, v_modified)
    
    return color_rgb_modified

def rgb_to_hsv(c):
    # Standard RGB to HSV conversion [G]
    r, g, b = c[0], c[1], c[2]
    max_c = max(r, g, b)
    min_c = min(r, g, b)
    diff = max_c - min_c
    if max_c == 0:
        return 0.0, 0.0, 0.0
    h = 0.0
    if max_c == r:
        h = 60.0 * ((g - b) / diff) % 360.0
    elif max_c == g:
        h = 60.0 * (2.0 + (b - r) / diff)
    else:
        h = 60.0 * (4.0 + (r - g) / diff)
    s = diff / max_c
    return h, s, max_c

def hsv_to_rgb(h, s, v):
    # Standard HSV to RGB conversion [G]
    c = v * s
    x = c * (1.0 - abs((h / 60.0) % 2.0 - 1.0))
    m = v - c
    if h < 60.0:
        r, g, b = c, x, 0.0
    elif h < 120.0:
        r, g, b = x, c, 0.0
    elif h < 180.0:
        r, g, b = 0.0, c, x
    elif h < 240.0:
        r, g, b = 0.0, x, c
    elif h < 300.0:
        r, g, b = x, 0.0, c
    else:
        r, g, b = c, 0.0, x
    return (r + m, g + m, b + m)
```

**Parameters:**
*   Fading function for $V$: Not given in the sources [S1, S2, unofficial].
*   Saturation function for $S$: Not given in the sources [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Gradient:** If $\mathbf{g}$ is zero (e.g., in uniform regions), the angle $\theta$ is undefined; a guard condition is required to prevent division by zero [G].
*   **Precision:** Numerical stability in $\arccos$ requires clamping the dot product to $[-1, 1]$ [G].

## Subtopics

### Oriented Color Modification
A minor variant that specifically decreases the color value (V) as the angle between the gradient and the viewer increases to simulate oriented fading [S3].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample for the color modification step, assuming constant-time HSV conversion and trigonometric operations [G].
*   **Memory Complexity:** $O(1)$ additional memory per sample [G].
*   **Strengths:** Effectively simulates traditional medical illustration techniques to enhance depth perception in translucent volumes [S1, S2, unofficial].
*   **Limitations:** The specific mathematical mappings for value and saturation modification are not provided, requiring implementation choices [S1, S2, unofficial].
*   **When to use:** When simulating illustrative styles, particularly in medical visualization, is desired [S1, S2, unofficial].

## Background
The method originates from the work on Non-Photorealistic Rendering of Volume Models presented at IEEE Visualization ’00 [S1, S2, unofficial]. It draws inspiration from medical illustration practices where anatomical structures oriented toward the viewer are desaturated and those oriented away are darkened and saturated [S1, S2, unofficial]. The use of HSV color space allows for intuitive application of these painterly techniques [S1, S2, unofficial].

## Sources
[S1] Non-Photorealistic Rendering of Volume Models, https://www.cs.umbc.edu/~rheingan/pubs/volillus00.pdf, fetched 2026-10-27 (unofficial)
[S2] Non-Photorealistic Rendering of Volume Models, https://userpages.cs.umbc.edu/ebert/npr/volillus.pdf, fetched 2026-10-27 (unofficial)
[S3] PDF Volume illustration: nonphotorealistic rendering of volume models ..., https://www.csee.umbc.edu/~rheingan/pubs/volillus.tvcg01.pdf, fetched 2026-10-27
