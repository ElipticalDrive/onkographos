# Orientation-Based Color Modification

Also called: HSV Color Orientation, HSV color modification, oriented fading

## Overview
Orientation-Based Color Modification simulates medical illustration techniques by altering the color, saturation, value, and transparency of volume samples based on the angle between the volumetric gradient and the viewer [S1, S2, unofficial]. It addresses the lack of depth cues in translucent volume rendering by using HSV color space to desaturate features facing the viewer and darken/saturate receding surfaces, thereby enhancing spatial understanding [S1, S2, unofficial].

## How it works
**Inputs:**
*   Volume sample color $C_{rgb}$ (RGB triplet).
*   Volumetric gradient vector $\mathbf{g}$ at the sample location.
*   Viewer direction vector $\mathbf{v}$ (pointing from the sample to the camera).

**Outputs:**
*   Modified color $C'_{rgb}$ and modified opacity $\alpha'$.

**Data Structures:**
*   HSV color space representation $(H, S, V)$, where $H \in [0, 360)$, $S \in [0, 1]$, $V \in [0, 1]$.

**Steps:**
1.  **Convert to HSV:** Transform the input RGB color $C_{rgb}$ into HSV space to obtain $(H, S, V)$ [S2, unofficial].
2.  **Calculate Orientation Angle:** Compute the angle $\theta$ between the gradient $\mathbf{g}$ and the viewer direction $\mathbf{v}$ [S1, S2, unofficial]:
    $$ \theta = \arccos\left( \frac{\mathbf{g} \cdot \mathbf{v}}{\|\mathbf{g}\| \|\mathbf{v}\|} \right) $$
    Note: $\mathbf{g}$ should be normalized if its magnitude is not inherently unit length. $\mathbf{v}$ is typically normalized.
3.  **Modify Value (V):** Decrease the Value component $V$ as the angle $\theta$ increases. This simulates "oriented fading," where surfaces facing the viewer (small $\theta$) retain brightness, and receding surfaces (large $\theta$) darken [S1, S2, unofficial].
    *   *Note:* The sources describe the qualitative effect ("decreased as the angle... increases") but do not provide a specific mathematical function (e.g., linear, exponential) for this decrease [S1, S2, unofficial].
4.  **Modify Saturation (Optional):** Desaturate surfaces oriented toward the viewer and saturate those oriented away [S2, unofficial].
    *   *Note:* The specific mapping for saturation modification is not defined in the sources [S1, S2, unofficial].
5.  **Convert to RGB:** Transform the modified HSV values $(H, S', V')$ back to RGB space to obtain $C'_{rgb}$ [S2, unofficial].
6.  **Transparency Modification:** The technique also allows modification of transparency based on orientation, though the specific formula is not detailed in the provided text [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def orientation_based_color_modification(color_rgb, gradient, viewer_direction):
    # 1. Convert RGB to HSV
    # Implementation of RGB_to_HSV is assumed as a standard utility
    h, s, v = rgb_to_hsv(color_rgb)
    
    # 2. Calculate angle between gradient and viewer
    # Normalize gradient if necessary
    g_norm = normalize(gradient)
    v_norm = normalize(viewer_direction)
    
    # Dot product, clamped to [-1, 1] for numerical stability
    dot_product = clamp(dot(g_norm, v_norm), -1.0, 1.0)
    theta = acos(dot_product)
    
    # 3. Modify Value (V)
    # Sources state V decreases as theta increases [S1, S2, unofficial]
    # Specific function not given in sources [S1, S2, unofficial]
    # Placeholder for implementation-specific fading function
    v_modified = apply_oriented_fading(v, theta)
    
    # 4. Modify Saturation (Optional)
    # Sources state desaturate facing, saturate receding [S2, unofficial]
    # Specific function not given in sources [S1, S2, unofficial]
    s_modified = apply_oriented_saturation(s, theta)
    
    # 5. Convert back to RGB
    color_rgb_modified = hsv_to_rgb(h, s_modified, v_modified)
    
    return color_rgb_modified
```

**Parameters:**
*   Fading function for $V$: Not given in the sources.
*   Saturation function for $S$: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Zero Gradient:** If $\mathbf{g}$ is zero (e.g., in uniform regions), the angle $\theta$ is undefined. A guard condition is required [G].
*   **Precision:** Numerical stability in $\arccos$ requires clamping the dot product to $[-1, 1]$ [G].

## Subtopics

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample for the color modification step, assuming constant-time HSV conversion and trigonometric operations.
*   **Memory Complexity:** $O(1)$ additional memory per sample.
*   **Strengths:** Effectively simulates traditional medical illustration techniques to enhance depth perception in translucent volumes [S1, S2, unofficial].
*   **Limitations:** The specific mathematical mappings for value and saturation modification are not provided, requiring implementation choices [S1, S2, unofficial].
*   **When to use:** When simulating illustrative styles, particularly in medical visualization, is desired [S1, S2, unofficial].
*   **When not to use:** When photorealistic rendering is required, as this method intentionally distorts color based on orientation [S1, S2, unofficial].

## Background
The method originates from the work on Non-Photorealistic Rendering of Volume Models presented at IEEE Visualization ’00 [S1, S2, unofficial]. It draws inspiration from medical illustration practices where anatomical structures oriented toward the viewer are desaturated and those oriented away are darkened and saturated [S1, S2, unofficial]. The use of HSV color space allows for intuitive application of these painterly techniques [S1, S2, unofficial].

## Sources
[S1] Non-Photorealistic Rendering of Volume Models, https://www.cs.umbc.edu/~rheingan/pubs/volillus00.pdf, fetched 2026-10-27 (unofficial)
[S2] Non-Photorealistic Rendering of Volume Models, https://userpages.cs.umbc.edu/ebert/npr/volillus.pdf, fetched 2026-10-27 (unofficial)
