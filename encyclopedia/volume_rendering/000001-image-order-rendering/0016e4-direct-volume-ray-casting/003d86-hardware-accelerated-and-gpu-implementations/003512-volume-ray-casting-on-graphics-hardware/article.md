# Volume Ray-Casting on Graphics Hardware
Also called: Stream model for volumetric ray-casting

## Overview
This method is a multi-pass approach for volume ray-casting on programmable graphics hardware (specifically DirectX 9-level hardware) that exploits intrinsic parallelism and efficient communication via a stream model [S1, unofficial]. It solves the challenge of achieving real-time or interactive rendering for large volumetric scalar fields by integrating early ray termination and empty-space skipping directly into the fragment processing pipeline [S1, unofficial]. It sets itself apart from CPU-based methods by leveraging hardware-supported per-fragment operations and texture render targets to accumulate state between passes, and from other GPU methods by its specific reliance on the early z-test driven by depth replacement to discard unnecessary fragment processing [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volumetric scalar field data, camera parameters (view direction, position), opacity threshold, and iso-value (if surface rendering is desired).
*   **Output:** Final 2D image (color and opacity) rendered to the screen or a texture.

**Data Structures:**
1.  **Volume Texture:** 3D texture containing the scalar field data.
2.  **Ray Direction Textures:** Two 2D textures storing the ray direction vectors in local texture coordinates for each pixel. These are computed prior to ray traversal and remain constant for all passes [S1, unofficial].
3.  **Accumulation Textures:** 2D textures (render targets) used to store accumulated color and opacity values between passes. Floating-point textures are required to support negative values if necessary [S1, unofficial].
4.  **Z-Buffer:** Used for the early z-test.

**Parameters:**
*   **Opacity Threshold:** A scalar value determining when ray casting terminates. Typical values are not explicitly given in the sources, but the method relies on a user-defined threshold [S1, unofficial].
*   **Iso-Value:** The scalar value defining the iso-surface. If hit, the process may terminate or switch to surface shading [S1, unofficial].
*   **Fixed Number of Steps:** Ray traversal is performed in a fixed number of rendering passes, corresponding to steps along the ray [S1, unofficial].

**Algorithm Steps:**
1.  **Pre-computation:** For each pixel, compute the direction of the ray through that pixel in local texture coordinates. Store these directions in two 2D textures [S1, unofficial].
2.  **Initialization:** Initialize the accumulation textures (color and opacity) to zero. Initialize the z-buffer appropriately for the first pass.
3.  **Main Passes (Ray Traversal Loop):**
    *   For each pass $i$ from 1 to $N$ (where $N$ is the fixed number of steps):
        *   Render a full-screen quad.
        *   In the pixel shader, retrieve the accumulated color and opacity from the accumulation textures using the current pass's index or texture coordinate logic.
        *   Retrieve the ray direction from the pre-computed direction textures.
        *   Compute the current sample position along the ray: $P_i = P_{start} + i \cdot \Delta d \cdot \vec{d}$, where $\vec{d}$ is the ray direction and $\Delta d$ is the step size [S1, unofficial].
        *   Sample the volume texture at $P_i$ to get the scalar value $s$.
        *   Compute the color $C$ and opacity $\alpha$ contribution for this step based on the transfer function applied to $s$ [S1, unofficial].
        *   Update the accumulated color and opacity:
            $$ C_{new} = C_{old} + C \cdot (1 - \alpha_{old}) $$
            $$ \alpha_{new} = \alpha_{old} + \alpha \cdot (1 - \alpha_{old}) $$
        *   Write the updated $C_{new}$ and $\alpha_{new}$ to the accumulation textures for the next pass.
        *   **Termination Check Pass:** Between main passes, perform an additional pass (or integrate into the main pass shader logic if hardware allows, but the source describes it as an additional pass) to test if the accumulated opacity $\alpha_{new}$ exceeds the threshold or if an iso-surface is hit.
            *   If the test succeeds: Modify the z-value of the fragment to the maximum possible depth value.
            *   If the test fails: Set the z-value to zero.
        *   Set the depth test to `GREATER`. This ensures that if the z-value was set to the maximum in the previous pass (indicating termination), all subsequent main passes for this fragment will be discarded by the early z-test before the shader executes [S1, unofficial].
4.  **Final Pass (Optional Surface Shading):** If an iso-surface was hit, use the stored intersection coordinates (written to a 2D texture during the hit detection) to perform gradient-based shading computations, restricting these expensive operations to the hit points [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computation
ray_directions = compute_ray_directions(view_params, volume_bounds)

# Initialization
accum_color = zeros(width, height, 3)
accum_opacity = zeros(width, height, 1)
z_buffer = initialize_z_buffer()

# Main Loop
for pass_index in range(num_steps):
    # Render accumulation textures for this pass
    render_pass(
        vertex_shader: full_screen_quad,
        fragment_shader: ray_cast_step(
            accum_color, accum_opacity, ray_directions, volume_texture, pass_index
        ),
        render_targets: [accum_color, accum_opacity]
    )
    
    # Termination Check Pass
    render_pass(
        vertex_shader: full_screen_quad,
        fragment_shader: check_termination(accum_opacity, threshold, iso_value),
        depth_test: GREATER,
        depth_write: True,
        z_replacement: True # Fragment shader sets z to max or 0
    )
    
    # Early z-test will skip subsequent passes if z was set to max

# Final rendering to screen
render_pass(
    vertex_shader: full_screen_quad,
    fragment_shader: composite(accum_color, accum_opacity),
    render_targets: [screen_buffer]
)
```

**Termination Check Shader Logic:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def check_termination(accum_opacity, threshold, iso_value):
    alpha = accum_opacity
    if alpha > threshold or iso_surface_hit:
        return max_depth # Discard subsequent passes
    else:
        return 0.0 # Allow subsequent passes
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point textures are necessary to store negative values and ensure precision in accumulation [S1, unofficial].
*   **Dependent Texture Fetches:** The ATI 9700 hardware has a limited number of dependent texture fetches. The algorithm must minimize these to avoid performance degradation [S1, unofficial].
*   **Early Z-Test Dependency:** The early z-test only works if all other per-fragment tests pass. The depth replace feature is critical for this mechanism [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method reduces the number of fragment shader executions by skipping rays that have already reached sufficient opacity or hit an iso-surface. This results in performance gains up to a factor of 3 for typical renditions of volumetric data sets on the ATI 9700 graphics card [S1, unofficial].
*   **Memory Complexity:** Requires additional 2D textures for accumulation and ray directions. The memory footprint is proportional to the screen resolution and the number of passes.
*   **Strengths:** Effective integration of early ray termination and empty-space skipping on hardware that supports texture render targets and early z-test. Exploits intrinsic parallelism of fragment processors [S1, unofficial].
*   **Limitations:** Dependent on specific hardware features (Pixel Shader 2.0, texture render targets, depth replace). Performance may vary on hardware with different limits on dependent texture fetches. The fixed number of steps may lead to over-sampling in some cases or under-sampling if not adjusted [S1, unofficial].
*   **When to use:** When targeting DirectX 9-level hardware (like ATI 9700) and seeking real-time performance for large volumetric datasets.
*   **When not to use:** On hardware lacking texture render targets or early z-test support.

## Background
This method was proposed in a paper by the authors of [S1, unofficial] in 2003. It builds upon the functionality provided by programmable graphics hardware, specifically the Pixel Shader 2.0 API in DirectX 9-level hardware [S1, unofficial]. The work addresses the challenge of real-time volume rendering for large datasets, which was difficult for CPU-based methods at the time. The ATI 9700 graphics card was the target architecture for demonstrating the effectiveness of the proposed algorithms [S1, unofficial].

## Sources
[S1] Acceleration Techniques for GPU-based Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Acceleration_Techniques_for_GPU-based_Volume_Rendering/vis03-rc.pdf, fetched 2026-10-26 (unofficial)
