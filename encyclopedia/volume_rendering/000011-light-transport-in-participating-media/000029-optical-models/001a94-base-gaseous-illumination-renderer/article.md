# Base Gaseous Illumination Renderer

Also called: Base Gaseous Illumination Model, Physics-based illumination of gaseous phenomena

## Overview
The Base Gaseous Illumination Renderer is the foundational volume rendering method used in volume illustration pipelines to simulate the appearance of translucent, gaseous media [S1, S2, unofficial]. It combines an opacity transfer function with an atmospheric illumination model to accumulate radiance along rays [S1, S2, unofficial]. It is distinguished from its siblings by its reliance on an atmospheric lighting model rather than standard surface shading, though the framework can be replaced by Phong illumination and linear accumulation [S1, S2, unofficial].

## How it works
**Inputs:**
*   Volume dataset $V$ containing scalar values $v_i$ for each voxel or sample point [S1, S2, unofficial].
*   Opacity transfer function parameters: $k_{os}$ (scalar controlling maximum opacity) and $k_e$ (exponent) [S1, S2, unofficial].
*   Lighting configuration: Typically a single light source positioned above the volume and slightly toward the viewer [S2, unofficial].
*   Base color assignment: Voxel colors $c$ derived from transfer functions (e.g., mimicking tissue colors) [S2, unofficial].

**Outputs:**
*   Final rendered image with shaded, segmented volume appearance [S2, unofficial].

**Data Structures:**
*   The volume is represented as a 3D grid of scalar values [S1, S2, unofficial]. For example, an abdominal CT volume may be $256 \times 256 \times 128$ voxels [S2, unofficial].
*   Each sample point $x_i$ has an associated scalar value $v_i$ [S1, S2, unofficial].

**Steps:**
1.  **Sample Volume Values:** For each ray cast through the volume, retrieve the scalar value $v_i$ at the current sample position [S1, S2, unofficial].
2.  **Compute Opacity:** Calculate the opacity $o_v$ for the sample using the power function:
    $$ o_v = (k_{os} v_i)^{k_e} $$
    where:
    *   $v_i$ is the volume sample value [S1, S2, unofficial].
    *   $k_{os}$ is a scalar controlling maximum opacity [S1, S2, unofficial].
    *   $k_e$ is an exponent [S1, S2, unofficial].
    *   If $k_e < 1$, volume differences are softened [S1, S2, unofficial].
    *   If $k_e > 1$, contrast within the volume is increased [S1, S2, unofficial].
3.  **Assign Color:** Determine the base color $c$ for the sample using a transfer function (e.g., assigning voxel colors that mimic actual tissue) [S2, unofficial].
4.  **Apply Illumination:** Render the volume using an atmospheric illumination model [S1, S2, unofficial].
5.  **Accumulate:** Composite the samples along the ray to produce the final pixel color [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def base_gaseous_illumination_renderer(volume_data, k_os, k_e, light_direction, view_direction):
    """
    Renders a volume using the base gaseous illumination model.
    
    Args:
        volume_data: 3D list or array of scalar values.
        k_os: Scalar controlling maximum opacity.
        k_e: Exponent for opacity transfer function.
        light_direction: Normalized vector for the light source.
        view_direction: Normalized vector for the camera view.
        
    Returns:
        2D list of RGBA tuples.
    """
    height, width, depth = len(volume_data), len(volume_data[0]), len(volume_data[0][0])
    image = [[(0.0, 0.0, 0.0, 0.0) for _ in range(width)] for _ in range(height)]

    for y in range(height):
        for x in range(width):
            ray_origin, ray_direction = cast_ray(x, y)  # [G] Standard ray generation from pixel coordinates
            accumulated_color = [0.0, 0.0, 0.0]
            accumulated_opacity = 0.0

            t = 0.0
            step_size = 1.0 / depth  # [G] Uniform sampling step along depth axis
            while t < depth:
                sample_pos = ray_origin + ray_direction * t
                v_i = volume_data[int(sample_pos[0])][int(sample_pos[1])][int(sample_pos[2])]

                # Compute opacity using the power function
                base = k_os * v_i
                if base < 0:
                    base = 0.0  # [G] Guard against negative base for exponentiation
                o_v = base ** k_e

                # Assign base color
                c = assign_color(v_i)  # [G] Color transfer function implementation

                # Apply atmospheric illumination
                shaded_color = apply_atmospheric_shading(c, light_direction, view_direction)  # [G] Atmospheric shading model

                # Front-to-back compositing
                alpha = o_v
                accumulated_color = [accumulated_color[i] + (1.0 - accumulated_opacity) * shaded_color[i] for i in range(3)]
                accumulated_opacity = accumulated_opacity + (1.0 - accumulated_opacity) * alpha

                if accumulated_opacity > 0.99:
                    break
                t += step_size

            image[y][x] = tuple(accumulated_color) + (accumulated_opacity,)
    return image
```

**Edge Cases and Pitfalls:**
*   **Opacity Clamping:** Ensure $v_i$ is non-negative before exponentiation if $k_e$ is not an integer, or clamp the result of $k_{os} v_i$ to $[0, 1]$ if the implementation expects normalized inputs [G].
*   **Lighting Setup:** The default setup uses a single light source above and slightly toward the viewer [S2, unofficial]. Deviating from this may require adjusting the atmospheric model parameters [G].

## Subtopics

### Atmospheric Illumination Model
The specific lighting model used in the base renderer [S1, S2, unofficial]. It simulates light scattering in gaseous media [G]. Choose this when a soft, translucent appearance is desired [G]. See [Illumination and Tone Shading Models](3857-illumination-and-tone-shading-models.md).

### Phong Illumination and Linear Accumulation
An alternative base renderer mentioned in the sources [S1, S2, unofficial]. It uses standard surface shading (Phong) and linear accumulation instead of atmospheric illumination [S1, S2, unofficial]. Choose this when a more surface-like appearance is preferred [G]. See [Illumination and Tone Shading Models](3857-illumination-and-tone-shading-models.md).

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Demonstrated on an abdominal CT volume of $256 \times 256 \times 128$ voxels [S2, unofficial].
*   **Strengths:** Simple to implement; provides a good baseline for volume illustration [S1, S2, unofficial].
*   **Limitations:** Limited to gaseous or translucent appearances [S2, unofficial].
*   **When to Use:** When creating volume illustrations that require a soft, atmospheric look; as a base for applying further illustration techniques [S1, S2, unofficial].
*   **When Not to Use:** Not given in the sources.

## Background
The Base Gaseous Illumination Renderer was introduced in the paper "Non-Photorealistic Rendering of Volume Models" by Robert Hegele et al. [S1, S2, unofficial]. It serves as the foundational rendering method for the volume illustration toolkit described in the paper [S1, S2, unofficial]. The framework is designed to be flexible, allowing for the application of various illustration techniques in arbitrary combinations [S1, S2, unofficial].

## Sources
[S1] Non-Photorealistic Rendering of Volume Models, https://www.cs.umbc.edu/~rheingan/pubs/volillus00.pdf, fetched 2026-10-27 (unofficial)
[S2] Non-Photorealistic Rendering of Volume Models, https://userpages.cs.umbc.edu/ebert/npr/volillus.pdf, fetched 2026-10-27 (unofficial)
