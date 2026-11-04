# Two-pass volume rendering

Also called: Light/Volume two-pass, Slice-based two-pass volume rendering

## Overview
Two-pass volume rendering decouples light attenuation computation from color compositing by separating the volume rendering process into a light pass and an eye pass [S1, unofficial]. The first pass renders volume slices to accumulate light in a dedicated light buffer, while the second pass composites slices front-to-back using the attenuated light from the light buffer, applying an 'under' blend to accumulate color and opacity [S1, unofficial]. This method differs from single-pass ray marching by allowing efficient reuse of the light buffer and leveraging hardware 'under' blending for front-to-back compositing [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** 3D volume texture (density/value), 1D or 2D transfer function (value $\rightarrow$ color/opacity), camera matrices (eye and light view-projection), slice geometry, number of slices, density thresholds ($m_{densityMin}, m_{densityMax}$), and opacity correction scalar.
- **Outputs:** Final color buffer (eye buffer) containing the composited volume image.

**Data Structures:**
1.  **Light Buffer:** A texture of square resolution $k_{LightSize} \times k_{LightSize}$. Stores accumulated light intensity and alpha.
2.  **Eye Buffer:** The final render target.
3.  **Push Constants:**
    -   `LightPC`: Contains `lightMVP` (mat4), `opacityCorrection` (float), `m_densityMin` (float), `m_densityMax` (float). Size: `sizeof(LightPC)`.
    -   `EyePC`: Contains `eyeMVP` (mat4), `lightMVP` (mat4), `opacityCorrection` (float), `m_ambient` (float), `m_densityMin` (float), `m_densityMax` (float). Size: `sizeof(EyePC)`.
4.  **Descriptor Sets:**
    -   **Volume Set:** Binding 0: Volume texture; Binding 1: Transfer function.
    -   **Eye Set:** Binding 0: Volume texture; Binding 1: Transfer function; Binding 2: Light buffer.
    -   **Composite Set:** Binding 0: Eye buffer.

**Parameters:**
-   `opacityCorrection`: A scalar multiplier applied to slice opacity to maintain correct total opacity regardless of the number of slices rendered. Calculated as the ratio of a reference slice count to the actual slice count [S1, unofficial].
    $$ opacityCorrection = \frac{k_{ReferenceSlices}}{m_{numSlices}} $$
    Where $k_{ReferenceSlices}$ is typically 256 [S1, unofficial].
-   `m_densityMin`, `m_densityMax`: Density thresholds passed to shaders to control ray marching or slice sampling bounds [S1, unofficial].
-   `kLightSize`: The square resolution of the light buffer texture [S1, unofficial].

**Steps:**
1.  **Generate Slice Geometry:** Generate proxy geometry along the view direction spanning the volume bounds. The slices must be ordered back-to-front for the "over" operator in the light pass, but the iteration order must ensure front-to-back processing from the light's perspective in the eye pass. This is achieved by setting `reverse=true` when generating slice geometry, ensuring the light end is processed first [S1, unofficial].
2.  **Clear Buffers:**
    -   Clear the light buffer with the light color and alpha set to 0.0 [S1, unofficial].
    -   Clear the eye buffer with black (0,0,0,0) [S1, unofficial].
3.  **Iterate Slices:** For each slice, from the light end to the eye end:
    a.  **Light Pass:** Execute the light pass shader. Bind the Volume Set. Push `LightPC`. Draw the slice geometry. This pass writes to the light buffer using premultiplied over blending [S1, unofficial].
    b.  **Eye Pass:** Execute the eye pass shader. Bind the Eye Set. Push `EyePC`. Draw the slice geometry. This pass binds the light buffer as a sampler and composites the slice color and opacity onto the eye buffer using the 'under' blend mode [S1, unofficial].
4.  **Composite:** The final image is in the eye buffer after all slices have been processed [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_pass_volume_rendering(volume_texture, transfer_function, light_buffer, eye_buffer, camera, num_slices, reference_slices=256):
    # Calculate opacity correction
    opacity_correction = reference_slices / num_slices
    
    # Generate slice geometry ordered back-to-front for light pass, 
    # but iteration will be light-end to eye-end for eye pass
    slices = generate_slice_geometry(
        volume_bounds=camera.volume_bounds,
        view_direction=camera.forward,
        num_slices=num_slices,
        reverse=True  # Ensures light-end is first in iteration list
    )
    
    # Clear buffers
    clear_texture(light_buffer, color=light_color, alpha=0.0)
    clear_texture(eye_buffer, color=(0, 0, 0, 0))
    
    # Iterate slices from light end to eye end
    for slice in slices:
        # Light Pass
        bind_descriptor_set(volume_set, volume_texture, transfer_function)
        push_constants(light_pass_pipeline, LightPC(
            lightMVP=camera.light_view_projection,
            opacityCorrection=opacity_correction,
            m_densityMin=camera.density_min,
            m_densityMax=camera.density_max
        ))
        draw_slice(light_pass_pipeline, slice, blend_mode='premultiplied_over')
        
        # Eye Pass
        bind_descriptor_set(eye_set, volume_texture, transfer_function, light_buffer)
        push_constants(eye_pass_pipeline, EyePC(
            eyeMVP=camera.eye_view_projection,
            lightMVP=camera.light_view_projection,
            opacityCorrection=opacity_correction,
            m_ambient=camera.ambient_light,
            m_densityMin=camera.density_min,
            m_densityMax=camera.density_max
        ))
        draw_slice(eye_pass_pipeline, slice, blend_mode='under')

def generate_slice_geometry(volume_bounds, view_direction, num_slices, reverse):
    # Generate slice planes along view_direction
    # If reverse is True, the returned list is ordered such that the first
    # element is closest to the light source (farthest from eye)
    slices = []
    for i in range(num_slices):
        t = i / (num_slices - 1)
        position = volume_bounds.min + t * (volume_bounds.max - volume_bounds.min)
        normal = view_direction
        slices.append(Slice(position, normal))
    
    if reverse:
        slices.reverse()
    return slices

def draw_slice(pipeline, slice, blend_mode):
    # Bind slice geometry (quad)
    # Execute shader which samples volume and transfer function
    # Writes to bound target (light_buffer or eye_buffer)
    pass
```

**Edge Cases and Pitfalls:**
-   **Upside-down Volume:** Volume data uploaded from CPU stores rows top-to-bottom, causing the rendered volume to appear upside down. This is handled by a `flipY` function that mirrors the data along the Y axis before GPU upload [S1, unofficial].
-   **Slice Iteration Order:** Slice iteration order must strictly match the front-to-back requirement from the light. The implementation fixes `reverse=true` unconditionally, as no modern GPU configuration requires the opposite order [S1, unofficial].
-   **Opacity Correction:** Without opacity correction, the total opacity of the volume changes with the number of slices. The `opacityCorrection` parameter must be applied to maintain consistent image intensity [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of slices and $S$ is the number of fragments per slice. Each slice requires two passes (light and eye).
-   **Memory Complexity:** $O(k_{LightSize}^2)$ for the light buffer, plus $O(V)$ for the volume texture and transfer functions.
-   **Performance:** Performance scales linearly with the number of slices. The use of hardware 'under' blending in the eye pass can be more efficient than software compositing. The light buffer reuse allows for efficient lighting calculations across slices.
-   **Strengths:** Decouples lighting and compositing, allowing for efficient reuse of light information. Leverages hardware blending capabilities.
-   **Limitations:** Requires additional memory for the light buffer. The two-pass nature can increase overhead compared to single-pass methods if not optimized.
-   **When to Use:** Suitable for scenes where lighting effects are important and can be precomputed or approximated per slice. Useful when hardware supports efficient 'under' blending.
-   **When Not to Use:** Not ideal for real-time applications with strict memory constraints due to the light buffer. Single-pass methods may be preferred for simpler lighting scenarios.

## Background
Two-pass volume rendering is based on the design described in GPUGems Fig. 39-1 [S1, unofficial]. It was implemented in a Vulkan-based volume rendering demo, explicitly referencing this design [S1, unofficial]. The technique separates the volume rendering process into distinct stages to optimize lighting and compositing.

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
