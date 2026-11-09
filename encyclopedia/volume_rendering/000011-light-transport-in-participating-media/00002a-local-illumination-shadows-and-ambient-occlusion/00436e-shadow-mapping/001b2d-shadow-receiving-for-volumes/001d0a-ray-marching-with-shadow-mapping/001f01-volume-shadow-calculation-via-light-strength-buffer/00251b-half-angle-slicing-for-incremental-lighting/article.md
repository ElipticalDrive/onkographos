# Half-Angle Slicing for Incremental Lighting

Also called: Two-Pass Volumetric Shadow Setup, Image Space Shadow Attenuation, Pixel Buffer Shadowing, Render to Texture Shadowing

## Overview
Half-Angle Slicing for Incremental Lighting is a multipass technique that computes volumetric shadows by accumulating light attenuation in an off-screen pixel buffer, thereby avoiding the memory overhead and leakage artifacts associated with volumetric shadow maps [S1, S2]. It differs from its parent method, [Volume Shadow Calculation via Light Strength Buffer](1914-volume-shadow-calculation-via-light-strength-buffer.md), by decoupling the light transport sampling from the data volume resolution and using a single 2D buffer instead of a 3D volume [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume dataset, transfer function (color $C$, opacity $A$), light color $C_L$, observer view direction, light view direction, frame buffer resolution.
*   **Output:** Final image with volumetric shadows in the eye buffer.

**Data Structures:**
1.  **Eye Buffer:** Standard frame buffer storing the final composited color.
2.  **Light Buffer:** An off-screen render buffer (pixel buffer) or texture. It stores the accumulated light intensity arriving at each slice.
    *   **Layout:** 2D texture/buffer.
    *   **Initialization:** Cleared to the light color $C_L$ [S2]. Alternatively, initialized with an arbitrary image (e.g., for spotlights) [S1, S2].
    *   **Resolution:** Typically 30–50% higher than the frame buffer resolution to mitigate shadow boundary artifacts [S1].

**Parameters:**
*   $d_v$: Desired sampling distance along the observer's view direction [S2].
*   $\theta$: Angle between the observer's view direction and the light's view direction [S2].
*   $d_s$: Slice spacing, calculated to maintain a constant sampling rate relative to the view direction [S2].

**Formulas:**
The slice spacing $d_s$ is determined by the angle $\theta$ between the view and light directions [S1, S2]:
$$ d_s = d_v \cos\left(\frac{\theta}{2}\right) $$
*Note: Source [S1] presents this as $d_s = \cos(\theta/2) d_v$. Source [S2] presents it as Equation 7 with the same geometric relationship.*

**Algorithm Steps:**
1.  **Initialize Buffers:** Clear the eye buffer. Initialize the light buffer to $C_L$ [S2].
2.  **Determine Slice Axis:**
    *   Calculate the dot product of the light direction ($L$) and view direction ($V$).
    *   If $L \cdot V > 0$ (acute angle): Set the slice axis to the vector halfway between $L$ and $V$. Render the volume front-to-back for the eye using the **Under** operator [S2].
    *   If $L \cdot V \le 0$ (obtuse or perpendicular): Set the slice axis to the vector halfway between $L$ and $-V$ (inverted view). Render the volume back-to-front for the eye using the **Over** operator [S2].
    *   In both cases, render the volume front-to-back for the light using the **Over** operator [S2].
3.  **Iterate Slices:** For each slice defined by the calculated axis and spacing $d_s$:
    *   **Pass 1 (Eye Rendering):**
        *   Render the current slice geometry into the eye buffer.
        *   Bind the light buffer as a texture.
        *   In the fragment shader:
            1.  Evaluate the transfer function to get reflective color $C$ and opacity $A$ [S2].
            2.  Sample the light buffer at the corresponding 2D texture coordinates (derived from projecting slice vertices using the light's modelview and projection matrices) to get accumulated light color $C_L$ [S2].
            3.  Modulate the slice color: $C_{final} = C \times C_L$ [S2].
            4.  Blend $C_{final}$ and $A$ into the eye buffer using the operator determined in Step 2 (Over or Under) [S2].
    *   **Pass 2 (Light Buffer Update):**
        *   Render the same slice geometry into the light buffer.
        *   In the fragment shader:
            1.  Evaluate the transfer function for the alpha component $A$ [S2].
            2.  Set the fragment color to 0 (or black) [S2].
            3.  Blend into the light buffer using the **Over** operator. This accumulates the opacity, effectively reducing the light intensity for subsequent slices [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def half_angle_slicing(volume, transfer_function, light_color, view_dir, light_dir, frame_buffer, light_buffer):
    # 1. Initialize
    clear(frame_buffer)
    # Light buffer initialized to light_color or a spotlight texture
    set_texture(light_buffer, light_color) 
    
    # 2. Determine slicing parameters
    dot_product = dot(light_dir, view_dir)
    
    if dot_product > 0:
        # Acute angle: slice halfway between light and view
        slice_axis = normalize(light_dir + view_dir)
        eye_blend_op = UNDER_OPERATOR # Front-to-back for eye
    else:
        # Obtuse angle: slice halfway between light and -view
        slice_axis = normalize(light_dir - view_dir)
        eye_blend_op = OVER_OPERATOR # Back-to-front for eye
        
    # Calculate slice spacing to maintain constant sampling rate dv
    # theta is the angle between view_dir and light_dir
    theta = acos(dot(normalize(view_dir), normalize(light_dir)))
    ds = dv * cos(theta / 2.0)
    
    # 3. Render slices
    num_slices = compute_num_slices(volume_bounds, slice_axis, ds)
    
    for i in range(num_slices):
        # Get slice geometry at current depth along slice_axis
        slice_geometry = get_slice_geometry(volume, slice_axis, i * ds)
        
        # --- Pass 1: Eye Buffer ---
        bind_framebuffer(frame_buffer)
        bind_texture(light_buffer, unit=0)
        
        # Project slice vertices to light buffer space to get 2D coords
        light_view_proj = get_light_view_projection_matrix()
        light_tex_coords = project_to_light_buffer(slice_geometry, light_view_proj)
        
        for each fragment in slice_geometry:
            # Sample volume data
            vol_data = sample_volume(volume, fragment.world_pos)
            
            # Evaluate transfer function
            C, A = transfer_function(vol_data)
            
            # Get accumulated light attenuation from light buffer
            # Note: Light buffer stores remaining light intensity
            C_L = texture(light_buffer, light_tex_coords).rgb
            
            # Modulate color by light intensity
            final_color = C * C_L
            
            # Blend into eye buffer
            blend(fragment, final_color, A, eye_blend_op)
            
        # --- Pass 2: Light Buffer ---
        bind_framebuffer(light_buffer)
        
        for each fragment in slice_geometry:
            vol_data = sample_volume(volume, fragment.world_pos)
            A = transfer_function(vol_data).alpha
            
            # Set color to 0, blend alpha to accumulate opacity
            # This reduces the light intensity stored in the buffer for next slices
            blend(fragment, vec3(0.0), A, OVER_OPERATOR)
```

**Edge Cases and Pitfalls:**
*   **Attenuation Leakage:** When the view and light directions are perpendicular, naive 2D shadow buffers suffer from "attenuation leakage," where light intensity is stored at a coarse resolution and interpolated, causing blurry shadows and overly dark surfaces [S1]. Using a light buffer with 30–50% higher resolution than the frame buffer mitigates this [S1].
*   **Sharp Opacity Jumps:** Artifacts may appear at shadow boundaries if opacity jumps sharply from low to high due to image-space sampling [S1].
*   **Angle Dependency:** As the angle between observer and light changes, $d_s$ must be adjusted to maintain the constant sampling rate $d_v$ [S2].

## Subtopics

### Volumetric Shadow Maps
Pre-computed 3D textures storing shadow information. They are memory-intensive and suffer from interpolation artifacts compared to the image-space approach of Half-Angle Slicing [S1, S2].

### Shear Warp Rendering
A technique that slices along major volume axes. It is limited in the maximum angle between view and light directions it can handle effectively without reslicing artifacts [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of slices and the resolution of the buffers. Each slice requires two rendering passes (eye and light buffer) [S1, S2].
*   **Memory Complexity:** $O(W \times H)$ for the light buffer, where $W, H$ are the buffer dimensions. This is significantly less than the $O(N^3)$ required for volumetric shadow maps [S1].
*   **Performance:** Efficient for interactive rendering as it decouples light transport sampling from the data volume resolution [S1].
*   **Strengths:** Avoids volumetric shadow map artifacts; efficient memory usage; supports arbitrary view and light directions via the half-angle slice axis [S1, S2].
*   **Limitations:** Does not account for light scattering within the volume (translucency), resulting in dark/unrealistic images if scattering is significant [S2]. Sensitive to resolution mismatches between light buffer and frame buffer [S1].

## Background
The technique was introduced by Kniss et al. in 2003 as an improvement over volumetric shadow maps for interactive volume rendering [S2]. It utilizes an off-screen render buffer (pixel buffer) to accumulate light attenuation from the light's point of view, allowing the same slice to be rendered from both the eye and light points of view by modifying the slice axis to be halfway between the view and light directions [S1, S2].

## Sources
[S1] Multidimensional Transfer Functions for Interactive Volume ..., https://www.academia.edu/5403340/Multidimensional_Transfer_Functions_for_Interactive_Volume_Rendering, fetched 2026-11-01
[S2] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
