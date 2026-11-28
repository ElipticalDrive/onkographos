# Two-Pass GPU Volume Ray Casting for Light Field Displays

Also called: Light field display aware volume rendering, Light field rendering pipeline, Multi-projector display rendering, Two-Pass GPU Volume Ray Casting, Two-Pass Light Field Rendering, Two-pass rendering for light field displays

## Overview
This method provides a specialized rendering pipeline for light field displays, enabling multiple freely moving naked-eye viewers to perceive virtual volumetric objects floating at fixed physical locations in the display workspace [S1, S2]. It sets itself apart from standard volume rendering by decoupling the 3D ray-casting computation from the 2D post-processing required to correct non-linear optical distortions inherent in multi-projector setups [S1, S2]. It builds on [Two-Pass Ray Marching Volume Rendering](5965-two-pass-ray-marching-volume-rendering.md) by extending the two-pass paradigm to handle multi-view light field geometry and per-projector distortion correction [G].

## How it works
The method consists of two distinct passes: a 3D rendering pass that generates per-projector views, and a 2D warping pass that corrects geometric and color distortions [S1, S2].

### Inputs and Outputs
*   **Inputs:**
    *   Volumetric dataset (stored in a 3D texture) [S1, S2].
    *   Light field geometry model defining projector positions and orientations [S1, S2].
    *   Precomputed per-pixel lookup tables (LUTs) for distortion correction and color mapping [S1, S2].
*   **Outputs:**
    *   Corrected images projected onto the physical light field display surface [S1, S2].

### Data Structures
*   **3D Texture:** Stores the entire volumetric dataset [S1]. Trilinear texture filtering is used for resampling during ray-casting [S1].
*   **Frame Buffer Object (FBO):** Used to render per-projector views off-screen during Pass 1 [S1, S2].
*   **Lookup Tables (LUTs):** Precomputed textures stored in GPU memory [S1, S2]. These contain per-pixel data used to warp geometry and modify colors in Pass 2 [S1, S2].
*   **Octree (Optional):** For very large datasets, the volume is decomposed into small cubical bricks organized into an octree structure maintained out-of-core [S2]. Leaves contain original data; inner nodes store filtered representations, value ranges, and optionally precomputed gradients [S2].

### Steps

**Pass 1: Per-Projector Volume Ray Casting** [S1, S2]
1.  **Initialize:** For each projector in the light field display array:
    a. Allocate and bind an FBO texture matching the projector's resolution [S1, S2].
    b. Set the view and projection matrices based on the idealized light field geometry model for this projector [S1, S2].
2.  **Render Proxy Geometry:** Render a proxy geometry (typically the volume’s bounding box) whose projection on the image plane covers all pixels emitting rays passing through the volume [S1].
3.  **Ray Casting Fragment Shader:** For each pixel covered by the proxy geometry:
    a. **Ray Generation:** Compute the ray origin and direction based on the projector's view and the pixel's position [S1].
    b. **Ray Marching:** Step from sample location to sample location along the ray [S1].
    c. **Resampling:** Fetch volume samples using trilinear texture filtering from the 3D texture [S1].
    d. **Compositing:** Perform front-to-back compositing of sample contributions (color and opacity) using an appropriate optical model [S1].
    e. **Termination:** Continue the loop until the volume is exited or full opacity is reached [S1].
4.  **Store Result:** Write the final composited color and opacity to the FBO texture for this projector [S1, S2].

**Pass 2: 2D Warping and Distortion Correction** [S1, S2]
1.  **Initialize:** For each projector view:
    a. Bind the FBO texture generated in Pass 1 as a source texture [S1, S2].
    b. Set up a full-screen quad or equivalent geometry to cover the output display area [S1, S2].
2.  **Warping Fragment Shader:** For each output pixel:
    a. **Lookup:** Retrieve the per-pixel lookup table data corresponding to the output pixel's position [S1, S2].
    b. **Geometry Warping:** Use the LUT to determine the source pixel coordinates in the Pass 1 texture, effectively warping the geometry to correct non-linear view distortions [S1, S2].
    c. **Color Correction:** Modify the color values using the LUT data to correct color distortions [S1, S2].
3.  **Display:** Output the corrected image to the projector [S1, S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1: Per-Projector Ray Casting
def pass1_volume_raycasting(projector_views, volume_texture, light_field_geometry):
    for view in projector_views:
        # Allocate and bind FBO for this projector
        fbo_texture = allocate_texture(view.resolution)
        bind_framebuffer(fbo_texture)
        set_view_projection(light_field_geometry.get_matrix(view))
        # Render proxy geometry (bounding box) to trigger fragment shader
        draw_bounding_box()
        unbind_framebuffer()
        # Store result
        view.pass1_result = fbo_texture

# Pass 2: 2D Warping and Distortion Correction
def pass2_distortion_correction(projector_views, distortion_luts):
    for view in projector_views:
        source_texture = view.pass1_result
        lut_texture = distortion_luts[view.id]
        bind_display_target(view.display_area)
        draw_fullscreen_quad()
        unbind_display_target()

# Fragment shader logic for Pass 1 (executed per pixel)
def fragment_shader_pass1(ray_origin, ray_direction, volume_texture):
    accumulated_color = vec3(0.0)
    accumulated_opacity = 0.0
    t = 0.0
    step = compute_step_size(ray_direction, volume_texture)  # Standard step calculation [G]
    while t < volume_texture.max_extent and accumulated_opacity < 1.0:
        sample_pos = ray_origin + t * ray_direction
        sample_value = texture_trilinear(volume_texture, sample_pos)
        color, opacity = apply_transfer_function(sample_value)
        alpha = opacity * (1.0 - accumulated_opacity)
        accumulated_color += color * alpha
        accumulated_opacity += alpha
        t += step
    return accumulated_color

# Fragment shader logic for Pass 2 (executed per output pixel)
def fragment_shader_pass2(output_pixel_coord, source_texture, lut_texture):
    source_uv = lookup_table_sample(lut_texture, output_pixel_coord, channel="warp")
    color_correction = lookup_table_sample(lut_texture, output_pixel_coord, channel="color")
    raw_color = texture_bilinear(source_texture, source_uv)
    corrected_color = raw_color * color_correction
    return corrected_color

# Helper definitions (standard operations)
def allocate_texture(resolution): return {}  # Allocates GPU texture memory
def bind_framebuffer(tex): pass  # Sets active render target
def set_view_projection(mat): pass  # Updates GPU projection matrix
def draw_bounding_box(): pass  # Triggers fragment shader for volume pixels
def unbind_framebuffer(): pass  # Resets render target
def bind_display_target(area): pass  # Sets output display surface
def draw_fullscreen_quad(): pass  # Triggers warping shader for output pixels
def texture_trilinear(tex, pos): return tex.sample(pos, filter="trilinear")  # Resamples volume
def apply_transfer_function(val): return val.color, val.opacity  # Applies optical model [G]
def compute_step_size(dir, tex): return tex.resolution / max(abs(dir))  # Standard step calculation [G]
def lookup_table_sample(lut, uv, channel): return lut.sample(uv, channel=channel)  # Reads LUT data
def texture_bilinear(tex, uv): return tex.sample(uv, filter="bilinear")  # Samples Pass 1 result
```

### Parameters
*   **Trilinear Texture Filtering:** Used for resampling during ray-casting [S1].
*   **Step Size:** Determined by volume resolution and view distance; not explicitly defined in sources [G].
*   **Optical Model:** Applied during compositing; specific model not detailed in sources [S1].

### Edge Cases and Pitfalls
*   **Large Datasets:** Standard 3D textures may not fit in GPU memory [S2]. The method supports an adaptive technique using an octree decomposition into cubical bricks maintained out-of-core [S2].
*   **Non-linear Distortions:** Small non-linear view and color distortions are common in multi-projector setups and must be corrected in Pass 2 [S1, S2].
*   **Full Opacity:** The ray marching loop terminates early if full opacity is reached, which is a standard optimization but must be handled correctly to avoid artifacts [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, S2].
*   **Memory Complexity:** Not given in the sources [S1, S2]. Additional memory for FBOs and LUTs is proportional to the display resolution [G]. For large datasets, the octree structure adds overhead but allows out-of-core processing [S2].
*   **Measured Performance:** Not given in the sources [S1, S2].
*   **Strengths:** Enables real-time volume rendering on light field displays [S1]. Separates 3D ray-casting from 2D distortion correction [S1, S2]. Handles large datasets via octree decomposition [S2].
*   **Limitations:** Requires precomputed LUTs for distortion correction. Assumes idealized light field geometry in Pass 1. Non-linear distortions must be small enough to be corrected by 2D warping [S1, S2].
*   **When to Use:** When rendering volumetric data on multi-projector light field displays with multiple viewers [G].
*   **When Not to Use:** When high-precision geometric alignment is not required or when the display is not a light field type [G].

## Background
The method was developed by Agus et al. in 2008 for the Eurographics conference [S1]. It follows the two-pass approach typical of contemporary multi-projector displays [BMY05] [S1, S2]. The technique was designed to provide multiple freely moving naked-eye viewers with the illusion of presence of virtual volumetric objects floating at fixed physical locations in the display workspace [S1, S2]. It is based on GPU ray-casting, which emerged as a flexible and efficient framework for real-time volume rendering [S1]. An adaptive technique based on octree decomposition was also designed to handle very large datasets [S2].

## Sources
[S1] GPU Accelerated Direct Volume Rendering on an Interactive Light Field Display, https://doi.org/10.1111/j.1467-8659.2008.01120.x, fetched 2026-10-29
[S2] Towards Advanced Volumetric Display of the Human Musculoskeletal System, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/021-025, fetched 2026-10-26
