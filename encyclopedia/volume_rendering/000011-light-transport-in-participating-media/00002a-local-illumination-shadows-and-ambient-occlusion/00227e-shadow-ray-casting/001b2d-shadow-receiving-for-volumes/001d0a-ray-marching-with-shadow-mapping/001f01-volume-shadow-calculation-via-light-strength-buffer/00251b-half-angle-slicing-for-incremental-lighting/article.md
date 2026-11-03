# Half-Angle Slicing for Incremental Lighting

Also called: Two-Pass Volumetric Shadow Setup

## Overview
Half-Angle Slicing is a technique for computing volumetric shadows efficiently by using a pixel buffer to accumulate light attenuation from the light's point of view, thereby avoiding the visual artifacts (such as blurry shadows and dark images) associated with volumetric shadow maps [S1]. It sets apart from its sibling [Volume Shadow Calculation via Light Strength Buffer](1914-volume-shadow-calculation-via-light-strength-buffer.md) by utilizing a shared slicing axis that is geometrically aligned between the viewer and the light source, allowing a single set of slices to be rendered from both perspectives in a two-pass loop per slice [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volume data, transfer function (mapping scalar value to color $C$ and opacity $A$), eye view direction, light view direction, light color $C_L$, desired sampling distance along the observer view direction $d_v$ [S1].
*   **Outputs:** Final image in the eye buffer [S1].

**Data Structures:**
1.  **Eye Buffer:** Standard framebuffer for the final composite image [S1].
2.  **Light Buffer:** A texture buffer used to accumulate light attenuation, initialized to the light color $C_L$ [S1].
3.  **Volume Data Texture:** 3D texture containing the scalar field [S1].
4.  **Slice Geometry:** Proxy geometry (e.g., a plane or box) representing the current slice [S1].

**Parameters:**
*   $C_L$: Light color [S1].
*   $d_v$: Desired sampling distance along the observer view direction [S1].
*   $\theta$: Angle between the observer view direction and the light view direction [S1].
*   $d_s$: Slice spacing, calculated to maintain constant sampling rate [S1].

**Formulas:**
The slice spacing $d_s$ is determined by the angle $\theta$ between the observer and light view directions to ensure a constant sampling rate relative to the observer:
$$ d_s = \frac{d_v}{\cos(\theta/2)} $$
[S1]

**Algorithm Steps:**
1.  **Initialization:** Clear the eye buffer and initialize the light buffer to the light color $C_L$ [S1].
2.  **Determine Slice Axis:** Compute the proxy geometry in object space [S1].
    *   Calculate the dot product of the light direction ($\vec{L}$) and the view direction ($\vec{V}$) [S1].
    *   If $\vec{L} \cdot \vec{V} > 0$: Set the slice direction to the vector halfway between the light and view directions [S1]. Render the volume front-to-back for the eye using the **Under** operator [S1]. Render front-to-back for the light using the **Over** operator [S1].
    *   If $\vec{L} \cdot \vec{V} \le 0$: Set the slice direction to the vector halfway between the light and the *inverted* view directions [S1]. Render the volume back-to-front for the eye using the **Over** operator [S1]. Render front-to-back for the light using the **Over** operator [S1].
3.  **Loop over Slices:** For each slice along the determined axis:
    *   **Pass 1 (Eye Rendering):** Render and blend the slice into the eye buffer [S1]. Project the slice vertices to the light buffer using the light's modelview and projection matrices [S1]. Convert the resulting vertex positions to 2D texture coordinates based on the size of the light's viewport and the light buffer [S1]. Bind the light buffer as a texture [S1]. In the fragment shader, evaluate the transfer function for reflective color $C$ and opacity $A$ [S1]. Multiply $C$ by the color sampled from the light buffer ($C_L$) [S1]. Blend $C$ and $A$ into the eye buffer using the appropriate operator (Under or Over) determined in Step 2 [S1].
    *   **Pass 2 (Light Buffer Update):** Render and blend the slice into the light buffer using the **Over** operator [S1]. In the fragment shader, evaluate the transfer function for the alpha component and set the fragment color to 0 [S1].
4.  **Finalize:** The eye buffer contains the final image with volumetric shadows [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(v1, v2):
    return sum(a * b for a, b in zip(v1, v2))

def normalize(v):
    length = math.sqrt(sum(x * x for x in v))
    return [x / length for x in v]

def clamp(val, min_val, max_val):
    return max(min_val, min(max_val, val))

def half_angle_slicing(volume_data, transfer_function, eye_view, light_view, light_color, dv, light_buffer_size):
    # 1. Initialization
    eye_buffer = [0.0] * (light_buffer_size[0] * light_buffer_size[1] * 4)
    light_buffer = [light_color] * (light_buffer_size[0] * light_buffer_size[1] * 4) # Initialized to light color CL [S1]

    # 2. Determine Slice Axis and Direction
    dot_product = dot(normalize(light_view), normalize(eye_view))
    dot_product = clamp(dot_product, -1.0, 1.0) # Fix precision bug for acos [G]

    if dot_product > 0:
        slice_axis = normalize([light_view[i] + eye_view[i] for i in range(3)])
        eye_blend_op = "UNDER"
        light_blend_op = "OVER"
        eye_render_order = "FRONT_TO_BACK"
    else:
        slice_axis = normalize([light_view[i] - eye_view[i] for i in range(3)]) # Inverted view direction [S1]
        eye_blend_op = "OVER"
        light_blend_op = "OVER"
        eye_render_order = "BACK_TO_FRONT"

    # Calculate slice spacing
    theta = math.acos(dot_product)
    ds = dv / math.cos(theta / 2)

    # 3. Loop over slices
    slices = generate_slices(volume_data, slice_axis, ds)

    for slice_obj in slices:
        # Pass 1: Render to Eye Buffer
        light_tex_coords = project_to_light_buffer(slice_obj.vertices, light_view, light_buffer_size)
        light_buffer_texture = light_buffer
        render_slice_to_buffer(
            slice_obj=slice_obj,
            buffer=eye_buffer,
            volume_data=volume_data,
            transfer_function=transfer_function,
            light_tex_coords=light_tex_coords,
            blend_op=eye_blend_op,
            order=eye_render_order
        )

        # Pass 2: Render to Light Buffer
        render_slice_to_buffer(
            slice_obj=slice_obj,
            buffer=light_buffer,
            volume_data=volume_data,
            transfer_function=transfer_function,
            light_tex_coords=None,
            blend_op="OVER",
            order="FRONT_TO_BACK",
            color_override=0.0
        )

def generate_slices(volume_data, axis, spacing):
    # Returns a list of slice objects representing planes along the axis [G]
    pass

def project_to_light_buffer(vertices, light_view, light_buffer_size):
    # Converts 3D vertex positions to 2D texture coordinates based on light's viewport and buffer size [S1]
    pass

def render_slice_to_buffer(slice_obj, buffer, volume_data, transfer_function, light_tex_coords, blend_op, order, color_override=None):
    for fragment in slice_obj.rasterize():
        value = sample_volume(volume_data, fragment.world_position)
        C, A = transfer_function(value)

        if light_tex_coords is not None:
            C_L = sample_texture(light_tex_coords)
            C = [C[i] * C_L[i] for i in range(3)]

        if color_override is not None:
            C = [color_override, color_override, color_override]

        blend_into_buffer(buffer, fragment.position, C, A, blend_op)

def sample_volume(volume_data, pos):
    # Standard 3D texture lookup [G]
    pass

def sample_texture(tex_coords):
    # Standard 2D texture lookup [G]
    pass

def blend_into_buffer(buffer, pos, C, A, op):
    # Blends color C and alpha A into buffer at pos using UNDER or OVER operator [S1]
    pass
```

**Edge Cases and Pitfalls:**
*   **Angle Adjustment:** As the angle between the observer and light changes, the slice distance $d_s$ must be adjusted to maintain a constant sampling rate [S1].
*   **Scattering:** The method ignores contributions from light scattering within the volume, which can result in dark and unrealistic images [S1].

## Operating characteristics
*   **Time Complexity:** Proportional to the number of slices times the number of fragments per slice, requiring two rendering passes per slice [S1].
*   **Memory Complexity:** Requires storage for the eye buffer, the light buffer, and the volume data, where the light buffer size depends on the light's viewport and resolution [S1].
*   **Strengths:** Avoids the artifacts of volumetric shadow maps (blurry shadows, dark images) by using a pixel buffer for attenuation accumulation and efficiently reuses slice geometry for both eye and light views [S1].
*   **Limitations:** Does not account for light scattering, leading to potentially dark images, and performance depends on the number of slices and the resolution of the light buffer [S1].
*   **When to use:** When real-time volumetric shadows are needed and shadow map artifacts are unacceptable, suitable for scenes where direct illumination attenuation is the primary shadowing component [S1].
*   **When not to use:** When accurate scattering effects (translucency, subsurface scattering) are required, as this method ignores them [S1].

## Background
The technique was described in *GPU Gems*, Chapter 39, "Volume Rendering Techniques," referencing Kniss et al. 2003 [S1]. It is presented as an improvement over volumetric shadow maps, which suffer from visual artifacts and high memory usage, and is part of the broader family of local illumination and shadowing techniques for participating media, specifically building upon the concepts of ray marching and shadow receiving for volumes [S1].

## Sources
[S1] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
