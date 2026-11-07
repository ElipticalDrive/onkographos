# Volume Ray-Casting on Graphics Hardware

Also called: Stream model for volumetric ray-casting

## Overview
This method is a multi-pass approach for volume ray-casting on programmable graphics hardware (specifically DirectX 9-level hardware) that exploits intrinsic parallelism and efficient communication via a stream model [S1, unofficial]. It solves the problem of integrating acceleration techniques, such as early ray termination and empty-space skipping, into 3D texture-based volume rendering by leveraging the early z-test and texture render targets [S1, unofficial]. It sets itself apart from single-pass fragment shader implementations by using intermediate 2D textures to store accumulated state and a dedicated termination pass to discard fragments early, achieving performance gains up to a factor of 3 on typical datasets [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volumetric scalar data stored in 3D texture memory; camera parameters; opacity threshold; iso-surface value (optional).
*   **Output:** Final rendered image stored in a 2D texture or frame buffer.

**Data Structures:**
1.  **Volume Texture:** 3D texture containing scalar voxel values.
2.  **Direction Textures:** Two 2D textures storing the ray direction in local texture coordinates for each pixel, computed prior to ray traversal [S1, unofficial].
3.  **Accumulation Textures:** Two 2D textures (one for color, one for opacity) used as render targets in each pass to store accumulated values. These are accessed in consecutive passes [S1, unofficial].
4.  **Z-Buffer:** Used for the early z-test to discard fragments.

**Numbered Steps:**
1.  **Ray Direction Pre-computation:** For each pixel, compute the direction of the ray of sight through the volume in local texture coordinates. Store these directions in two 2D textures [S1, unofficial].
2.  **Multi-Pass Ray Traversal:** Execute a fixed number of rendering passes. In each pass:
    a.  Bind the accumulation textures from the previous pass as input textures.
    b.  Bind the volume texture and direction textures.
    c.  Execute the fragment shader to sample the volume along the ray step, classify the value, and update the accumulated color and opacity using the front-to-back compositing equations defined in [Direct volume ray casting](19-direct-volume-ray-casting.md).
    d.  Render the result to the accumulation textures for the next pass [S1, unofficial].
3.  **Termination Check Pass:** Between main traversal passes, execute an additional pass:
    a.  Read the accumulated opacity from the accumulation texture.
    b.  Test if the opacity exceeds a specified threshold or if an iso-surface is hit.
    c.  If the test succeeds, modify the fragment's z-value to the maximum possible depth value.
    d.  If the test fails, set the z-value to zero [S1, unofficial].
4.  **Early Z-Test Filtering:** Set the depth test function to GREATER. Because fragments that passed the termination check have their z-value set to the maximum, they will fail the GREATER test in subsequent passes and be discarded. Fragments with z-value zero will pass the test (assuming standard depth range) and continue processing [S1, unofficial].
5.  **Final Rendering:** After all traversal passes, render the final accumulated color and opacity to the screen or output texture. If an iso-surface was hit, use the stored intersection coordinates to perform gradient-based shading [S1, unofficial].

**Formulas:**
*   **Compositing (per step):**
    $$
    C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i
    $$
    $$
    \alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
    $$
    Where $C_i$ and $\alpha_i$ are the color and opacity of the current sample, and $C'_{i-1}, \alpha'_{i-1}$ are the accumulated values from the previous step [S1, unofficial].
*   **Z-Value Modification:**
    $$
    z_{new} = \begin{cases} 
    z_{max} & \text{if } \alpha' \geq \text{threshold} \lor \text{iso-surface hit} \\
    0 & \text{otherwise}
    \end{cases}
    $$
    Where $z_{max}$ is the maximum representable depth value [S1, unofficial].

**Parameters:**
*   **Opacity Threshold:** Not explicitly given, but typically used to terminate rays [S1, unofficial].
*   **Iso-surface Value:** User-defined for surface extraction [S1, unofficial].
*   **Number of Passes:** Fixed number of rendering passes for ray traversal [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Dependent Texture Fetches:** On ATI hardware, the number of dependent texture fetches is limited. Care must be taken to structure the shader to avoid exceeding this limit [S1, unofficial].
*   **Floating Point Textures:** Accumulation textures must support floating point values to handle negative values if required, though standard compositing uses non-negative values [S1, unofficial].
*   **Early Z-Test Enablement:** The early z-test is only enabled if all other per-fragment tests pass. Ensure depth testing is configured correctly to leverage this optimization [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computation
ray_directions = compute_ray_directions(camera_matrix, volume_bounds)
store_in_texture(ray_directions, direction_texture)

# Multi-pass loop
for pass_index in range(num_passes):
    # Bind previous accumulation textures as inputs
    bind_texture(accum_color_texture, texture_unit_0)
    bind_texture(accum_opacity_texture, texture_unit_1)
    bind_texture(volume_texture, texture_unit_2)
    bind_texture(direction_texture, texture_unit_3)
    
    # Set render targets for current pass
    set_render_target(accum_color_texture_next)
    set_render_target(accum_opacity_texture_next)
    
    # Execute fragment shader for ray step
    # Shader reads accumulated values, samples volume, updates accumulation
    render_fullscreen_quad()
    
    # Swap textures for next iteration
    swap(accum_color_texture, accum_color_texture_next)
    swap(accum_opacity_texture, accum_opacity_texture_next)
    
    # Termination check pass
    bind_texture(accum_opacity_texture, texture_unit_0)
    set_render_target(z_buffer_texture) # Or use depth buffer directly if supported
    set_depth_test(GREATER)
    
    # Fragment shader for termination check
    # Reads opacity, sets depth to max if threshold exceeded
    render_fullscreen_quad()
    
    # Reset depth test for next main pass
    set_depth_test(GREATER) # Ensure GREATER is set for subsequent passes
```

## Subtopics

### Early Ray Termination
A technique to stop ray marching when accumulated opacity exceeds a threshold, reducing computation. This method implements it via the termination check pass and early z-test [S1, unofficial].

### Empty-Space Skipping
A technique to skip regions of the volume with no relevant data. This method integrates it by discarding fragments that have already terminated or passed through empty space, using the z-test to skip shader execution [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of passes and the number of fragments that survive the early z-test. The multi-pass approach adds overhead but reduces per-fragment work for terminated rays.
*   **Memory Complexity:** Requires additional 2D textures for accumulation and direction storage. The size is proportional to the screen resolution [S1, unofficial].
*   **Measured Performance:** Performance gains up to a factor of 3 for typical renditions of volumetric data sets on the ATI 9700 graphics card [S1, unofficial].
*   **Strengths:** Effective integration of acceleration techniques; exploits hardware parallelism; efficient communication between passes via texture render targets [S1, unofficial].
*   **Limitations:** Requires hardware support for texture render targets, dependent texture fetches, and early z-test; limited to DirectX 9-level hardware features [S1, unofficial].
*   **When to Use:** When targeting older programmable graphics hardware (e.g., ATI 9700) and needing to integrate early ray termination and empty-space skipping efficiently [S1, unofficial].
*   **When Not to Use:** On modern hardware with more advanced shading capabilities and memory management where single-pass or other acceleration techniques might be more efficient [S1, unofficial].

## Background
This method was proposed in a paper by the authors of [S1, unofficial] in 2003. It builds upon the Pixel Shader 2.0 API and features available on DirectX 9-level hardware, such as the ATI 9700. The work addresses the challenge of achieving real-time volume rendering for large datasets by leveraging the intrinsic parallelism of graphics hardware [S1, unofficial].

## Sources
[S1] Acceleration Techniques for GPU-based Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Acceleration_Techniques_for_GPU-based_Volume_Rendering/vis03-rc.pdf, fetched 2026-10-26 (unofficial)
