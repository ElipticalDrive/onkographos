# Orientation-Based Color Modification

Also called: HSV Color Orientation, HSV color modification, oriented fading

## Overview
Orientation-Based Color Modification simulates medical illustration techniques by altering a volume sample's color, saturation, value, and transparency based on the angle between the volumetric gradient and the viewer [S1, S2, unofficial]. It specifically desaturates structures oriented toward the viewer and darkens/saturates those oriented away, addressing the lack of depth cues in traditional translucent volume rendering [S1, S2, unofficial]. It differs from [Lit Sphere Shading](785-lit-sphere-shading.md) by using gradient orientation rather than surface normals for shading, and from [Volume Illustration Tone Shading](805-volume-illustration-tone-shading.md) by focusing on HSV space manipulation rather than general tone mapping.

## How it works
**Inputs:**
*   Volume sample color $C_{rgb}$ (RGB triplet).
*   Volumetric gradient vector $\mathbf{g}$ at the sample location.
*   View direction vector $\mathbf{v}$ (pointing from sample to viewer).

**Outputs:**
*   Modified color $C'_{rgb}$ and modified opacity $\alpha'$.

**Data Structures:**
*   HSV representation: $H$ (Hue, $0-360^\circ$ or $0-1$), $S$ (Saturation, $0-1$), $V$ (Value, $0-1$).
*   Standard RGB to HSV conversion functions.
*   Standard HSV to RGB conversion functions.

**Steps:**
1.  **Calculate Orientation Angle:** Compute the angle $\theta$ between the gradient $\mathbf{g}$ and the view direction $\mathbf{v}$.
    $$ \theta = \arccos\left( \frac{\mathbf{g} \cdot \mathbf{v}}{\|\mathbf{g}\| \|\mathbf{v}\|} \right) $$
    Where $\mathbf{g} \cdot \mathbf{v}$ is the dot product. $\theta \in [0, \pi]$.
    *Note:* If $\|\mathbf{g}\| = 0$, $\theta$ is undefined; treat as $0$ or skip modification [G].

2.  **Convert to HSV:** Convert the input sample color $C_{rgb}$ to HSV space $(H, S, V)$.

3.  **Modify Value (V):** Decrease the Value component as $\theta$ increases. This simulates "oriented fading" where features facing the viewer appear lighter/less intense in value, or conversely, the source states "color value (V) is decreased as the angle between the gradient and the viewer increases" [S1, unofficial][S3].
    *   Let $V_{mod} = V \cdot f(\theta)$.
    *   The source describes the effect qualitatively: "decreased as the angle... increases" [S1, unofficial][S3]. A common implementation for this fading is a linear or cosine falloff, but the exact function $f(\theta)$ is not explicitly defined in the provided notes beyond the qualitative description [S1, unofficial][S3].
    *   *Interpretation:* If $\theta=0$ (gradient aligned with view), $V$ is at its original or modified base. As $\theta \to \pi/2$ or $\pi$, $V$ decreases.

4.  **Modify Saturation (S) and Hue (H):**
    *   **Desaturation:** Portions oriented toward the viewer are desaturated [S1, S2, unofficial]. This implies $S$ is reduced when $\theta$ is small.
    *   **Receding Surfaces:** Structures oriented away are darkened and saturated [S1, S2, unofficial]. This implies $S$ is increased (or maintained) and $V$ is decreased when $\theta$ is large.
    *   The exact mapping for $S$ is not given numerically. The source states the system allows modification of saturation [S1, S2, unofficial].

5.  **Modify Transparency:** The technique allows modification of transparency based on orientation [S1, S2, unofficial]. The specific formula is not provided in the notes.

6.  **Convert Back:** Convert the modified $(H, S_{mod}, V_{mod})$ back to RGB space to get $C'_{rgb}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def orientation_based_color_modification(color_rgb, gradient, view_dir):
    # 1. Calculate angle theta between gradient and view direction
    dot_product = dot(gradient, view_dir)
    norm_g = magnitude(gradient)
    norm_v = magnitude(view_dir)
    
    # Guard against zero-length gradient
    if norm_g == 0:
        return color_rgb, original_alpha # Or handle as no modification
    
    cos_theta = dot_product / (norm_g * norm_v)
    # Clamp to [-1, 1] to avoid numerical errors in acos
    cos_theta = max(-1.0, min(1.0, cos_theta))
    theta = acos(cos_theta)
    
    # 2. Convert RGB to HSV
    h, s, v = rgb_to_hsv(color_rgb)
    
    # 3. Modify Value (V)
    # Source states: V is decreased as theta increases [S1, unofficial][S3]
    # Exact function not given. Using a simple linear falloff as a placeholder for implementation.
    # Note: This is a [G] implementation detail as the source lacks the exact formula.
    # A typical "oriented fading" might use cos(theta) or a linear scale.
    # Let's assume a linear decrease from 1.0 at theta=0 to 0.0 at theta=pi/2 or similar.
    # However, to be safe and stick to the text "decreased as angle increases",
    # we apply a factor.
    value_factor = 1.0 - (theta / pi) # Linear decrease from 1 to 0 over 0 to pi
    v_mod = v * value_factor
    
    # 4. Modify Saturation (S)
    # Source states: Desaturate facing surfaces (small theta), saturate/darken receding (large theta) [S1, S2, unofficial]
    # No exact formula provided.
    # Implementation [G]: Increase saturation as theta increases.
    sat_factor = 1.0 + (theta / pi) # Increase saturation
    s_mod = min(1.0, s * sat_factor)
    
    # 5. Modify Hue (H)
    # No modification described in sources.
    h_mod = h
    
    # 6. Convert back to RGB
    color_rgb_mod = hsv_to_rgb(h_mod, s_mod, v_mod)
    
    # 7. Modify Transparency
    # Source states transparency can be modified [S1, S2, unofficial], but no formula given.
    # Not given in the sources.
    alpha_mod = original_alpha 
    
    return color_rgb_mod, alpha_mod
```

**Parameters:**
*   None explicitly defined in the sources. The behavior is driven by the angle $\theta$.

**Edge Cases:**
*   **Zero Gradient:** If $\mathbf{g} = \mathbf{0}$, the angle is undefined. [G] Handle by returning original color or treating $\theta=0$.
*   **Precision:** Clamping `cos_theta` to $[-1, 1]$ is necessary to prevent NaNs from `acos`.

## Subtopics

### Oriented Color Modification
A variant that specifically decreases the color value (V) as the angle between the gradient and the viewer increases to simulate oriented fading [S3]. It focuses on the value channel modification described in the main method.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample for the color modification step (dot product, acos, HSV conversion).
*   **Memory Complexity:** $O(1)$ additional memory per sample.
*   **Strengths:** Effectively simulates medical illustration techniques, providing intuitive depth cues through desaturation and value changes [S1, S2, unofficial]. It leverages the HSV space for intuitive control [S1, S2, unofficial].
*   **Limitations:** The exact mathematical functions for modifying Saturation and Transparency are not specified in the primary sources, requiring implementation choices [G]. Performance impact is minimal as it is a per-sample operation.
*   **When to use:** When simulating medical illustrations or artistic styles where orientation-based desaturation is desired [S1, S2, unofficial].
*   **When not to use:** When photorealistic shading is required, or when gradient information is noisy or unavailable.

## Background
The technique was introduced in the context of Non-Photorealistic Rendering (NPR) of volume models, specifically to address the lack of depth cues in traditional translucent volume rendering [S1, S2, unofficial]. It draws inspiration from medical illustration practices where anatomical structures oriented toward the viewer are desaturated and those oriented away are darkened and saturated [S1, S2, unofficial]. The work was presented at IEEE Visualization ’00 [S1, S2, unofficial] and later published in TVCG [S3].

## Sources
[S1] Non-Photorealistic Rendering of Volume Models, https://www.cs.umbc.edu/~rheingan/pubs/volillus00.pdf, fetched 2026-10-27 (unofficial)
[S2] Non-Photorealistic Rendering of Volume Models, https://userpages.cs.umbc.edu/ebert/npr/volillus.pdf, fetched 2026-10-27 (unofficial)
[S3] PDF Volume illustration: nonphotorealistic rendering of volume models ..., https://www.csee.umbc.edu/~rheingan/pubs/volillus.tvcg01.pdf, fetched 2026-10-27
