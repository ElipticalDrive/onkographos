# Volume Ray-Casting on Graphics Hardware

Also called: Stream model for volumetric ray-casting

## Overview
This method is a multi-pass algorithm for direct volume ray-casting on programmable graphics hardware (specifically DirectX 9-level architectures) that exploits intrinsic parallelism and efficient communication via a stream model [S1, unofficial]. It solves the problem of integrating acceleration techniques like early ray termination and empty-space skipping into 3D texture-based volume rendering by using texture render targets to accumulate state between passes [S1, unofficial]. It sets itself apart from single-pass hardware methods by explicitly managing the ray-marching state (accumulated color, opacity, and termination flags) across multiple rendering passes, allowing the GPU's early z-test to discard fragments that have already reached an opacity threshold or hit an iso-surface [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume texture (scalar field), a pre-computed ray direction texture (2D), and rendering parameters (opacity threshold, iso-surface value, step size).
*   **Output:** A final 2D color image.

**Data Structures:**
The method relies on a set of 2D textures aligned with the viewport to communicate state between passes. The exact layout is not specified in the sources, but the following logical textures are required [S1, unofficial]:
1.  **Ray Direction Texture:** Stores the ray direction vector for each pixel in local texture coordinates.
2.  **Accumulation Textures:** At least two 2D textures (one for color, one for opacity) to store the accumulated results of the ray marching. Floating-point textures are used to support negative values if necessary [S1, unofficial].
3.  **Iso-surface Intersection Texture:** Stores the local texture coordinates of iso-surface intersections, if applicable [S1, unofficial].

**Numbered Steps:**
1.  **Pre-computation:** For each pixel, compute the direction of the ray of sight through the volume in local texture coordinates. Store these directions in a 2D texture [S1, unofficial].
2.  **Initialization:** Initialize the accumulation textures (color and opacity) to zero. Initialize the iso-surface intersection texture if used [S1, unofficial].
3.  **Main Ray Marching Passes:** Perform a fixed number of rendering passes. In each pass:
    a.  Bind the accumulation textures from the previous pass as input textures.
    b.  Bind the ray direction texture.
    c.  Execute a fragment shader that:
        i.  Samples the volume at the current ray position.
        ii. Computes the color $C_i$ and opacity $\alpha_i$ for the sample.
        iii. Updates the accumulated color and opacity using the front-to-back compositing equations defined in the parent article [S1, S2, S3, S4, unofficial].
        iv. Advances the ray position along the direction vector.
        v.  Writes the updated accumulated color and opacity to the output accumulation textures [S1, unofficial].
4.  **Termination Check Pass:** Between main passes, perform an additional pass:
    a.  Read the accumulated opacity from the accumulation texture.
    b.  Test if the accumulated opacity exceeds a specified threshold or if an iso-surface has been hit (if iso-surface rendering is enabled).
    c.  If the test succeeds, set the fragment's depth value (z-value) to the maximum possible value.
    d.  If the test fails, set the fragment's depth value to zero [S1, unofficial].
5.  **Early Z-Test Filtering:** Enable the depth test with the function `GREATER`. Because fragments that have terminated (step 4c) have their depth set to the maximum, they will fail the `GREATER` test in subsequent main passes (step 3) and be discarded. Fragments that have not terminated (step 4d) have depth zero and will pass the test, allowing them to continue marching [S1, unofficial].
6.  **Final Pass:** After all main passes, perform a final pass to composite the accumulated color and opacity into the final image. If iso-surface intersections were stored, use them to restrict gradient and shading computations to those points [S1, unofficial].

**Formulas:**
The front-to-back compositing equations are used in the main passes [S1, S2, S3, S4, unofficial]:
$$
C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i
$$
$$
\alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
$$
Where $C'_{i}$ and $\alpha'_{i}$ are the accumulated color and opacity after step $i$, and $C_i$ and $\alpha_i$ are the color and opacity of the current sample.

**Parameters:**
*   **Opacity Threshold:** The value at which a ray is considered opaque enough to terminate. Not given in the sources.
*   **Iso-surface Value:** The scalar value defining the iso-surface. Not given in the sources.
*   **Number of Main Passes:** A fixed number of passes for ray traversal. Not given in the sources.
*   **Step Size:** The distance between samples along the ray. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point textures are required to store negative values and ensure precision in accumulation [S1, unofficial].
*   **Dependent Texture Fetches:** The number of dependent texture fetches is limited on the target hardware (ATI 9700), which may restrict the complexity of the shader [S1, unofficial].
*   **Early Z-Test Timing:** The early z-test is applied before the shader program is executed. This means the termination check pass must correctly set the depth values to effectively skip fragments in subsequent passes [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computation
ray_direction_texture = compute_ray_directions(volume_bounds, camera_params)

# Initialization
accum_color_texture = create_2d_texture(width, height, format=FLOAT)
accum_opacity_texture = create_2d_texture(width, height, format=FLOAT)
accum_color_texture.clear(0.0)
accum_opacity_texture.clear(0.0)

# Main Ray Marching Loop
for pass_index in range(number_of_main_passes):
    # Bind previous accumulation textures as inputs
    bind_texture(accum_color_texture, unit=0)
    bind_texture(accum_opacity_texture, unit=1)
    bind_texture(ray_direction_texture, unit=2)
    
    # Set up render target for current pass
    set_render_target(accum_color_texture, accum_opacity_texture)
    
    # Execute fragment shader for ray marching
    execute_shader(ray_marching_shader)
    
    # Termination Check Pass
    bind_texture(accum_opacity_texture, unit=0)
    set_render_target(depth_buffer_only=True)
    set_depth_test_function(GREATER)
    
    # Execute termination check shader
    execute_shader(termination_check_shader)
    
    # Reset render target for next main pass
    set_render_target(accum_color_texture, accum_opacity_texture)

# Final Pass
bind_texture(accum_color_texture, unit=0)
bind_texture(accum_opacity_texture, unit=1)
set_render_target(framebuffer)
execute_shader(final_composite_shader)
```

## Subtopics

### Back-face rasterized volume rendering
Uses back-face rasterization to generate rays. [Back-face rasterized volume rendering](1296-back-face-rasterized-volume-rendering.md)

### GPU Multi-Volume Ray Casting Mapper
Extends the concept to multiple volumes. [GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md)

### Hardware-Accelerated Volume Rendering via 3D Texturing
Uses 3D textures directly for sampling. [Hardware-Accelerated Volume Rendering via 3D Texturing](2799-hardware-accelerated-volume-rendering-via-3d-texturing.md)

### 3D Pixel Texture Classification
Uses pixel textures for classification. [3D Pixel Texture Classification](6055-3d-pixel-texture-classification.md)

### Hardware Shading via Pixel Texture
Uses pixel textures for shading. [Hardware Shading via Pixel Texture](6056-hardware-shading-via-pixel-texture.md)

### GPU Ray Casting for Light Field Display
Adapts ray casting for light fields. [GPU Ray Casting for Light Field Display](6503-gpu-ray-casting-for-light-field-display.md)

### VStudio Framework Volume Rendering Pipeline
A framework for volume rendering pipelines. [VStudio Framework Volume Rendering Pipeline](6916-vstudio-framework-volume-rendering-pipeline.md)

### Entry Point Adjustment
Adjusts ray entry points. [Entry Point Adjustment](7406-entry-point-adjustment.md)

### Custom Hardware Units
Uses custom hardware for rendering. [Custom Hardware Units](8284-custom-hardware-units.md)

## Operating characteristics
*   **Time Complexity:** The time complexity is proportional to the number of pixels multiplied by the number of main passes. The early z-test reduces the effective number of passes for opaque regions. Not given in the sources.
*   **Memory Complexity:** The memory complexity is dominated by the storage of the 2D textures for accumulation and ray directions. Not given in the sources.
*   **Measured Performance:** Performance gains up to a factor of 3 were demonstrated for typical renditions of volumetric data sets on the ATI 9700 graphics card [S1, unofficial].
*   **Strengths:** Exploits intrinsic parallelism of the GPU; uses efficient communication via texture render targets; effectively integrates early ray termination and empty-space skipping [S1, unofficial].
*   **Limitations:** Limited by the number of dependent texture fetches on the target hardware; requires multiple rendering passes which may introduce overhead [S1, unofficial].
*   **When to use:** When using DirectX 9-level hardware with programmable shaders and texture render target support [S1, unofficial].
*   **When not to use:** When hardware does not support texture render targets or has severe limitations on dependent texture fetches [S1, unofficial].

## Background
This method was proposed in a paper by the authors of [S1, unofficial] for implementation on DirectX 9-level hardware, specifically the ATI 9700 graphics card. It builds upon the Pixel Shader 2.0 API, which provides features such as per-fragment texture fetch operations, texture render targets, texture coordinate generation, per-fragment arithmetic, and depth replace [S1, unofficial]. The method was demonstrated on a variety of real-world data sets [S1, unofficial].

## Sources
[S1] Acceleration Techniques for GPU-based Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Acceleration_Techniques_for_GPU-based_Volume_Rendering/vis03-rc.pdf, fetched 2026-10-26 (unofficial)
