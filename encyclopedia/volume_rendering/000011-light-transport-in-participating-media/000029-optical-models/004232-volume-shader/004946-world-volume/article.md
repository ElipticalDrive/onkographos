# World Volume

Also called: World Volume Shader, Global Volume

## Overview
A World Volume applies a volume shader to the entire scene space, effectively filling the universe with participating media [S1]. It solves the problem of rendering global atmospheric effects without requiring explicit geometry to bound the volume [S1]. Unlike sibling methods that operate on specific objects, the World Volume assumes the background is infinitely far away, making it distinct from bounded volume objects [S1]. It is primarily suited for dark scenes or night-time rendering, as it can override or interact with world surface shaders and sun lights in ways that may obscure them if not configured correctly [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume shader definition (containing scattering, absorption, and emission coefficients) applied to the world context [S1].
*   **Output:** The final rendered image where the entire space is treated as a participating medium [S1].

**Data Structures:**
The implementation relies on the standard Volume Shader data structures defined in [Volume Shader](10695-volume-shader.md). No additional global data structures are specified for the World Volume itself beyond the shader parameters [S1].

**Steps:**
1.  The renderer initializes the view ray.
2.  Instead of terminating the ray at a world boundary or background plane, the renderer treats the entire ray length as traversing the volume [S1].
3.  The volume shader is evaluated along the ray to compute transmittance and accumulated color [S1].
4.  The background is assumed to be at infinite distance [S1].

**Parameters:**
*   **Volume Shader:** Defined by the user, containing scattering, absorption, and emission properties [S1].
*   **Background Distance:** Assumed to be infinite [S1].

**Edge Cases and Pitfalls:**
*   **Infinite Background Assumption:** The method assumes the background is infinitely far away. This is accurate for distant light sources like the sun but inaccurate for effects like fog or atmospheric scattering where the distance between light sources and objects involves significant empty space [S1].
*   **Scene Darkness:** Because the world background is assumed to be infinitely far away, using a World Volume can cause the world surface shader or sun lights to have no effect if the volume shader is used, potentially resulting in a dark scene [S1].
*   **Geometry Holes:** If the scene geometry has holes or normals point in the wrong direction, the renderer may be unable to determine the inside or outside of volumes, similar to issues with glass refraction [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Conceptual implementation based on [S1]
def render_world_volume(view_ray, volume_shader):
    # The world volume fills the entire space.
    # The background is assumed to be infinitely far away.
    
    # Initialize accumulated color and transmittance
    accumulated_color = vec3(0.0)
    transmittance = 1.0
    
    # Sample the volume along the ray
    # Note: Specific sampling strategy depends on the underlying renderer
    # and is not detailed in the sources [S1].
    # The volume shader is applied to the entire space [S1].
    
    for sample in sample_ray(view_ray):
        # Evaluate volume shader at sample point
        sigma_a = volume_shader.absorption(sample.position)
        sigma_s = volume_shader.scattering(sample.position)
        emission = volume_shader.emission(sample.position)
        
        # Update transmittance and accumulated color
        # Standard volume rendering equation integration
        # (Implementation details of integration are not provided in [S1])
        pass
        
    # Since background is infinitely far, no background color is added
    # unless the volume is transparent and the renderer handles infinite
    # background differently [S1].
    
    return accumulated_color
```

## Subtopics

### Front-to-Back Blending
A technique used in conjunction with World Volume rendering to composite the final image [S2, unofficial]. It involves sampling front-to-back (f2b) and back-to-front (b2f) accumulated volume textures at normalized screen coordinates ($u\_screen\_size$) [S2, unofficial]. The colors are blended using the formula $color = f2b + b2f \times (1.0 - f2b.a)$ [S2, unofficial]. Finally, a transfer function is applied to the resulting color: $gl\_FragmentColor = texture(u\_transfer, vec2(color.r, 0.0))$ [S2, unofficial]. This method requires three sampler2D inputs: `u_f2b` for the front-to-back accumulated volume texture, `u_b2f` for the back-to-front accumulated volume texture, and `u_transfer` for the transfer function texture [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Useful for night-time or dark scenes where global atmospheric effects are desired without explicit geometry [S1].
*   **Limitations:** Inaccurate for modeling fog or atmospheric scattering due to the infinite background assumption, which ignores the empty space between light sources and objects [S1]. Can cause scenes to become dark if it overrides world surface shaders or sun lights [S1].
*   **When to use:** Night-time scenes or dark environments [S1].
*   **When not to use:** Fog or atmospheric scattering effects where the distance between light sources and objects is significant [S1].

## Background
The World Volume concept is documented in the Blender Manual [S1]. It is part of the volume shader system, which allows materials to occupy space and interact with light through scattering, absorption, and emission [S1]. The specific pitfall regarding the infinite background assumption highlights the distinction between global volumes and bounded volume objects [S1].

## Sources
[S1] Volumes — Blender Manual, https://www.blender.org/manual/nb/3.6/render/materials/components/volume.html, fetched 2026-10-26
[S2] Pointifix/Vis2_Schott2010, https://github.com/Pointifix/Vis2_Schott2010, fetched 2026-10-26 (unofficial)
