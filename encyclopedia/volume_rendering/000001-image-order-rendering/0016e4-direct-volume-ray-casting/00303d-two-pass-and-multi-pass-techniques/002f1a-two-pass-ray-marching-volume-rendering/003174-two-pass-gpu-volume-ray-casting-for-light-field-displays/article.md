# Two-Pass GPU Volume Ray Casting for Light Field Displays

Also called: Light field display aware volume rendering, Light field rendering pipeline, Multi-projector display rendering, Two-Pass GPU Volume Ray Casting, Two-Pass Light Field Rendering, Two-pass rendering for light field displays

## Overview
This method provides a specialized rendering pipeline for light field displays, enabling multiple freely moving naked-eye viewers to perceive virtual volumetric objects floating at fixed physical locations in the display workspace [S1, S2]. It addresses the challenge of rendering volumetric data across multiple projectors with precise geometric alignment and color consistency. It sets itself apart from standard volume rendering by decoupling the 3D ray-casting computation from the 2D post-processing required to correct non-linear optical distortions inherent in multi-projector setups [S1, S2]. It builds on [Two-Pass Ray Marching Volume Rendering](5965-two-pass-ray-marching-volume-rendering.md) by extending the two-pass paradigm to handle multi-view light field geometry and per-projector distortion correction.

## How it works
The method consists of two distinct passes: a 3D rendering pass that generates per-projector views, and a 2D warping pass that corrects geometric and color distortions.

### Inputs and Outputs
*   **Inputs:**
    *   Volumetric dataset (stored in a 3D texture) [S1, S2].
    *   Light field geometry model defining projector positions and orientations [S1, S2].
    *   Precomputed per-pixel lookup tables (LUTs) for distortion correction and color mapping [S1, S2].
*   **Outputs:**
    *   Corrected images projected onto the physical light field display surface [S1, S2].

### Data Structures
*   **3D Texture:** Stores the entire volumetric dataset. Trilinear texture filtering is used for resampling during ray-casting [S1].
*   **Frame Buffer Object (FBO):** Used to render per-projector views off-screen during Pass 1 [S1, S2].
*   **Lookup Tables (LUTs):** Precomputed textures stored in GPU memory. These contain per-pixel data used to warp geometry and modify colors in Pass 2 [S1, S2].
*   **Octree (Optional):** For very large datasets, the volume is decomposed into small cubical bricks organized into an octree structure maintained out-of-core. Leaves contain original data; inner nodes store filtered representations, value ranges, and optionally precomputed gradients [S2].

### Steps

**Pass 1: Per-Projector Volume Ray Casting**
1.  **Initialize:** For each projector in the light field display array:
    a. Bind the FBO corresponding to the current projector's resolution.
    b. Set up the projection matrix based on the idealized light field geometry model for this projector [S1, S2].
2.  **Render Proxy Geometry:** Render a proxy geometry (e.g., the volume’s bounding box) whose projection on the image plane covers all pixels emitting rays passing through the volume [S1].
3.  **Ray Casting Fragment Shader:** For each pixel covered by the proxy geometry:
    a. **Ray Generation:** Compute the ray origin and direction based on the projector's view and the pixel's position [S1].
    b. **Ray Marching:** Step from sample location to sample location along the ray [S1].
    c. **Resampling:** Fetch volume samples using trilinear texture filtering from the 3D texture [S1].
    d. **Compositing:** Perform front-to-back compositing of sample contributions (color and opacity) using an appropriate optical model [S1].
    e. **Termination:** Continue the loop until the volume is exited or full opacity is reached [S1].
4.  **Store Result:** Write the final composited color and opacity to the FBO texture for this projector [S1, S2].

**Pass 2: 2D Warping and Distortion Correction**
1.  **Initialize:** For each projector view:
    a. Bind the FBO texture generated in Pass 1 as a source texture.
    b. Set up a full-screen quad or equivalent geometry to cover the output display area [S1, S2].
2.  **Warping Fragment Shader:** For each output pixel:
    a. **Lookup:** Retrieve the per-pixel lookup table data corresponding to the output pixel's position [S1, S2].
    b. **Geometry Warping:** Use the LUT to determine the source pixel coordinates in the Pass 1 texture, effectively warping the geometry to correct non-linear view distortions [S1, S2].
    c. **Color Correction:** Modify the color values using the LUT data to correct color distortions [S1, S2].
    d. **Sampling:** Sample the Pass 1 texture at the warped coordinates [S1, S2].
3.  **Display:** Output the corrected image to the projector [S1, S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1: Per-Projector Ray Casting
def pass1_volume_raycasting(projector_views, volume_texture, light_field_geometry):
    for view in projector_views:
        # Bind FBO for this projector
        fbo = create_fbo(view.resolution)
        bind_fbo(fbo)
        
        # Set up projection based on light field geometry
        projection_matrix = light_field_geometry.get_projection_matrix(view)
        set_projection(projection_matrix)
        
        # Render proxy geometry (e.g., bounding box) to activate fragment shader
        render_proxy_geometry()
        
        # Fragment shader logic (executed per pixel)
        # ray_origin, ray_direction computed from view and pixel coordinates
        accumulated_color = vec3(0.0)
        accumulated_opacity = 0.0
        
        # Ray marching loop
        t = 0.0
        while t < max_distance and accumulated_opacity < 1.0:
            sample_position = ray_origin + t * ray_direction
            sample_value = texture_trilinear(volume_texture, sample_position)
            
            # Optical model: compute color and opacity from sample_value
            color, opacity = transfer_function(sample_value)
            
            # Front-to-back compositing
            alpha = opacity * (1.0 - accumulated_opacity)
            accumulated_color += color * alpha
            accumulated_opacity += alpha
            
            t += step_size # Step size determined by volume resolution and view distance
            
        write_to_fbo(fbo, accumulated_color)
        unbind_fbo()

# Pass 2: 2D Warping and Distortion Correction
def pass2_distortion_correction(projector_views, pass1_textures, distortion_luts):
    for view in projector_views:
        # Bind output display target
        bind_display_target(view.display_area)
        
        # Bind Pass 1 texture and corresponding LUT
        source_texture = pass1_textures[view.id]
        lut_texture = distortion_luts[view.id]
        
        # Render full-screen quad to activate warping shader
        render_fullscreen_quad()
        
        # Fragment shader logic (executed per output pixel)
        # output_pixel_coord is the current pixel in the display area
        source_uv = lookup_warp(lut_texture, output_pixel_coord)
        color_correction = lookup_color(lut_texture, output_pixel_coord)
        
        # Sample from Pass 1 result
        raw_color = texture(source_texture, source_uv)
        
        # Apply color correction
        final_color = apply_color_correction(raw_color, color_correction)
        
        write_to_display(final_color)
```

### Parameters
*   **Trilinear Texture Filtering:** Used for resampling during ray-casting [S1].
*   **Step Size:** Determined by volume resolution and view distance; not explicitly defined in sources [G].
*   **Optical Model:** Applied during compositing; specific model not detailed in sources [S1].

### Edge Cases and Pitfalls
*   **Large Datasets:** Standard 3D textures may not fit in GPU memory. The method supports an adaptive technique using an octree decomposition into cubical bricks maintained out-of-core [S2].
*   **Non-linear Distortions:** Small non-linear view and color distortions are common in multi-projector setups and must be corrected in Pass 2 [S1, S2].
*   **Full Opacity:** The ray marching loop terminates early if full opacity is reached, which is a standard optimization but must be handled correctly to avoid artifacts [S1].

## Operating characteristics
*   **Time Complexity:** Pass 1 is $O(N \cdot R)$, where $N$ is the number of pixels across all projectors and $R$ is the average number of ray steps. Pass 2 is $O(M)$, where $M$ is the total number of output pixels.
*   **Memory Complexity:** $O(V)$ for the volume texture, where $V$ is the number of voxels. Additional memory for FBOs and LUTs is proportional to the display resolution. For large datasets, the octree structure adds overhead but allows out-of-core processing [S2].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Enables real-time volume rendering on light field displays; separates complex 3D computation from 2D correction; handles large datasets via octree decomposition [S1, S2].
*   **Limitations:** Requires precomputed LUTs for distortion correction; assumes idealized light field geometry in Pass 1; non-linear distortions must be small enough to be corrected by 2D warping [S1, S2].
*   **When to Use:** When rendering volumetric data on multi-projector light field displays with multiple viewers.
*   **When Not to Use:** When high-precision geometric alignment is not required or when the display is not a light field type.

## Background
The method was developed by Agus et al. in 2008 for the Eurographics conference [S1]. It follows the two-pass approach typical of contemporary multi-projector displays [BMY05] [S1, S2]. The technique was designed to provide multiple freely moving naked-eye viewers with the illusion of presence of virtual volumetric objects floating at fixed physical locations in the display workspace [S1, S2]. It is based on GPU ray-casting, which emerged as a flexible and efficient framework for real-time volume rendering [S1]. An adaptive technique based on octree decomposition was also designed to handle very large datasets [S2].

## Sources
[S1] GPU Accelerated Direct Volume Rendering on an Interactive Light Field Display, https://doi.org/10.1111/j.1467-8659.2008.01120.x, fetched 2026-10-29
[S2] Towards Advanced Volumetric Display of the Human Musculoskeletal System, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/021-025, fetched 2026-10-26
