# Gradientless Shading via Pixel Textures

Also called: Forward Difference Shading, Frame Buffer Arithmetic Shading

## Overview
Gradientless Shading via Pixel Textures is an image-based shading technique that approximates the diffuse lighting gradient by computing forward differences of scalar material values using OpenGL blending and pixel textures [S1, unofficial]. It solves the problem of calculating surface normals for shading without storing or copying the entire frame buffer for gradient calculation, which is required by sibling methods like [Depth-Gradient Shading](6618-depth-gradient-shading.md) [S1, unofficial]. It sets itself apart by exploiting the rasterization pipeline's texture coordinate interpolation and pixel texturing to perform arithmetic on the frame buffer directly, rather than relying on explicit depth-difference calculations or texture lookups for gradients [S1, unofficial].

## How it works
The method approximates the directional derivative of the scalar field $X$ towards the light source $\vec{L}$ to estimate diffuse intensity.

**Inputs and Outputs:**
- Input: Scalar volume data mapped to a slicing plane, light direction $\vec{L}$, viewport dimensions.
- Output: Frame buffer containing the approximated diffuse lighting intensity $I_d$.

**Data Structures:**
- **Frame Buffer:** Stores RGB color values. Initially used to store interpolated texture coordinates (surface positions), then used for arithmetic results.
- **Texture Lookup Table (LUT):** A 1D or 2D table mapping scalar values to RGB colors. The alpha channel is used for alpha testing.
- **Pixel Texture:** Enabled during the shading passes to re-sample the material values from the frame buffer content.

**Parameters:**
- $\Delta$: Step size for the forward difference. Not given in the sources how this is varied per-point; it is a global constant for the pass [S1, unofficial].
- OpenGL Pixel Bias: Used to shift texture coordinates in the second pass [S1, unofficial].

**Formulas:**
The diffuse intensity is approximated by the forward difference:
$$I_d \approx \frac{\partial X}{\partial L} = X(\vec{p}_0) - X(\vec{p}_0 + \Delta \cdot \vec{L})$$
where:
- $X$ is the scalar material value.
- $\vec{p}_0$ is the surface point.
- $\vec{L}$ is the light direction.
- $\Delta$ is the step size.

**Steps:**
1. **Vertex Attribute Assignment:** For each vertex of the slicing plane geometry, assign the computed texture coordinates $(u, v, w)$ as the vertex color $(R, G, B)$. The texture coordinates are in the range $[0, 1]$ [S1, unofficial].
2. **Disable Modulation:** Temporarily set all RGB values in the RGB$\alpha$ lookup table to 1. This ensures that during rasterization, the interpolated color values (which correspond to the texture space coordinates/positions of points on the slicing plane) are not modulated by texture samples [S1, unofficial].
3. **Render Positions:** Render the slicing plane. The frame buffer now contains the position of surface points encoded in the RGB channels [S1, unofficial].
4. **Restore Material Values:** Reset the RGB entries of the texture lookup table to produce the original scalar values. Read the pixel data from the frame buffer into main memory [S1, unofficial].
5. **First Shading Pass:** Draw the pixel data into the frame buffer with pixel texture enabled. Render the original surface points. The frame buffer now contains the scalar values $X(\vec{p}_0)$ [S1, unofficial].
6. **Second Shading Pass:** Draw the pixel data into the frame buffer again with pixel texture enabled. Shift the pixel values (texture coordinates) towards the light source using the OpenGL pixel bias. This effectively samples $X(\vec{p}_0 + \Delta \cdot \vec{L})$ [S1, unofficial].
7. **Blending:** Change the blending equation to subtract the second pass values from the first pass values already in the frame buffer. The result is the approximated diffuse lighting $I_d$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradientless_shading(volume_data, light_direction, viewport_size, step_size):
    """
    Implements Gradientless Shading via Pixel Textures.
    Assumes an OpenGL-like environment with pixel textures and blending.
    """
    # Step 1: Prepare geometry with texture coordinates as colors
    # For each vertex v of the slicing plane:
    #   v.color = v.texture_coordinates  # (u, v, w) mapped to (R, G, B)
    
    # Step 2: Configure LUT to disable modulation
    # Set all RGB entries in the RGBa lookup table to 1.0
    set_lut_rgb_to_one()
    
    # Step 3: Render positions to frame buffer
    # The frame buffer now holds the interpolated texture coordinates (positions)
    render_slicing_plane()
    
    # Step 4: Restore LUT and read back
    # Reset RGB entries of the texture lookup table to produce original scalar values
    restore_lut_to_scalar_values()
    
    # Read pixel data from frame buffer into main memory
    frame_buffer_data = read_frame_buffer()
    
    # Step 5: First pass - Render original values
    enable_pixel_texture()
    set_blend_function(GL_ONE, GL_ONE) # Or appropriate setup for accumulation
    draw_pixels(frame_buffer_data)
    
    # Step 6: Second pass - Render shifted values
    # Shift pixel values towards the light source using OpenGL pixel bias
    set_pixel_bias(light_direction * step_size)
    
    # Step 7: Subtract second pass from first pass
    # Change blending equation to subtract current from existing
    set_blend_function(GL_ONE, GL_ONE_MINUS_SRC_ALPHA) # Or GL_ONE, GL_MINUS_SRC_COLOR depending on exact GL state
    # Note: The source describes "subtract the second pass values from the first pass values already in the frame buffer"
    # This implies: FrameBuffer = FrameBuffer - NewSample
    set_blend_function(GL_ONE, GL_ONE_MINUS_SRC_COLOR) # Approximation of subtraction logic described
    draw_pixels(frame_buffer_data)
    
    disable_pixel_texture()
    reset_pixel_bias()
```

**Edge Cases and Pitfalls:**
- **Low Contrast:** Surfaces rendered by this method exhibit low contrast [S1, unofficial].
- **Incorrect Results:** Incorrect results are produced in regions where the variation of the gradient magnitude across the surface is high [S1, unofficial].
- **Fixed Step Size:** The step size $\Delta$ cannot be easily varied per-point with this approach, which limits accuracy in areas requiring finer gradient estimation [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The overhead of reading/writing pixel data does not depend on data resolution but on the size of the viewport [S1, unofficial].
- **Memory Complexity:** Uses one fourth of the memory needed in gradient shading [S1, unofficial].
- **Performance:** Rendering time differs insignificantly from gradient shading [S1, unofficial]. However, it requires reading and writing pixel data twice with enabled pixel texturing, whereas gradient shading copies the frame buffer once [S1, unofficial].
- **Scaling:** The relative contribution to overall rendering time decreases rapidly with increasing data size because the overhead is viewport-dependent, not data-dependent [S1, unofficial].
- **Strengths:** Lower memory footprint compared to gradient shading; avoids explicit frame buffer copying for gradient calculation [S1, unofficial].
- **Limitations:** Low contrast; inaccurate in regions of high gradient magnitude variation; fixed step size [S1, unofficial].

## Background
This technique is described in the context of volume rendering via 2D textures using register combiners and pixel textures [S1, unofficial]. It is presented as an alternative to gradient shading, exploiting OpenGL's pixel texturing capabilities to perform frame buffer arithmetic [S1, unofficial].

## Sources
[S1] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
