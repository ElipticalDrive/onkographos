# Hardware-Z Cull Depth Peeling

Also called: Modified depth peeling with z-buffer culling, OIT approach with hardware depth test

## Overview
Hardware-Z Cull Depth Peeling is an Order-Independent Transparency (OIT) technique designed for rendering semi-transparent layers on animated volumetric surfaces where pre-sorting geometry is computationally prohibitive [S1]. It solves the visual artifacts of standard alpha blending by extracting transparent layers sequentially, but distinguishes itself from standard depth peeling by offloading fragment culling to the hardware z-buffer rather than performing it via shader-based discard operations [S1]. This separation minimizes wave divergence and reduces unnecessary fragment shader invocations, making it particularly suitable for scenarios requiring the stable resolution of the first $K$ semi-transparent layers [S1].

## How it works
The method renders transparent layers one by one until the last visible layer is reached or a user-defined maximum number of layers is exceeded [S1]. Each rendering pass consists of four distinct steps. The algorithm assumes an encoded surface texture is available to determine valid intersections, preventing transparent parts from incorrectly occupying every layer [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Previous layer's depth buffer (read-only).
    *   Encoded surface texture (for intersection testing).
    *   Scene geometry and lighting parameters.
    *   Accumulated opacity buffer (to track total coverage).
*   **Outputs:**
    *   Final composited color buffer.
    *   Updated hardware z-buffer.
    *   Updated accumulated opacity buffer.

### Data Structures
*   **Depth Buffer:** Standard hardware z-buffer.
*   **Accumulated Opacity Buffer:** A render target storing the sum of opacities of all rendered layers up to the current pass.
*   **Layer Depth Texture:** A temporary texture storing the nearest depth found for the current layer during Step 1.

### Parameters
*   **Maximum Layers ($N_{max}$):** User-defined limit on the number of rendering passes. Typical value: 4 [S1].
*   **Opacity Threshold ($T_{op}$):** Threshold for accumulated opaqueness. If accumulated opacity exceeds this value, subsequent layers are treated as invisible. Typical value: 0.95 [S1].
*   **Far Culling Plane ($z_{max}$):** The depth value written to the z-buffer to cull fragments behind the current layer or when opacity is sufficient.

### Numbered Steps
Each pass $i$ (from 1 to $N_{max}$) executes the following sequence:

1.  **Layer Depth Finding:**
    *   Set the z-buffer to the previous layer's depth in read-only mode.
    *   Render the geometry. For each fragment, check for intersection with the encoded surface. If no intersection exists, discard the fragment [S1].
    *   If an intersection exists, calculate the depth of the new layer.
    *   Write the nearest depth to the output color using a 'min' blending function. This effectively finds the front-most intersection of the new layer relative to the previous layer's depth [S1].

2.  **Depth Conversion #1:**
    *   Perform a full-screen pass (or a pass covering the bounding volume of the geometry) to update the hardware z-buffer.
    *   Read the depth found in Step 1.
    *   Compare the new layer's depth ($d_{new}$) with the previous scene's depth ($d_{prev}$) stored in the z-buffer.
    *   If $d_{new} > d_{prev}$, the fragment is behind the previous layer's visible surface. Cull this fragment by writing $z_{max}$ to the z-buffer [S1].
    *   Otherwise, write $d_{new}$ to the z-buffer. This step ensures that only pixels potentially visible in the new layer are processed in subsequent steps [S1].

3.  **Layer Shading:**
    *   Render the geometry again for the current layer.
    *   Set the z-function to 'equal'. This ensures that lighting computations are performed *only* for pixels that passed the depth test in Step 2, eliminating overdraw [S1].
    *   Use hardware alpha-blending to blend the newly shaded layer with the previously rendered layers in the color buffer [S1].

4.  **Depth Conversion #2:**
    *   Perform another full-screen pass similar to Step 2.
    *   Check the accumulated opaqueness ($\alpha_{acc}$) for each pixel.
    *   If $\alpha_{acc} > T_{op}$, write $z_{max}$ to the z-buffer. This culls all subsequent layers for this pixel, as the surface is considered effectively opaque [S1].
    *   Otherwise, preserve the current z-buffer state or update it based on the current layer's depth if necessary for subsequent passes [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
Z_MAX = 1000.0  # Far culling plane value
OPACITY_THRESHOLD = 0.95
MAX_LAYERS = 4

# State
accumulated_opacity = 0.0
previous_layer_depth = 0.0 # Initialized to near plane or previous frame's last layer depth

for layer_index in range(1, MAX_LAYERS + 1):
    # Step 1: Layer Depth Finding
    # Render geometry with z-buffer set to previous_layer_depth (read-only)
    # Fragment Shader:
    #   if not intersect_encoded_surface():
    #       discard()
    #   depth = calculate_layer_depth()
    #   output_color = min(depth, current_output_color) # Min blending
    
    # Read the resulting depth texture from Step 1
    layer_depth_texture = read_output_color()
    
    # Step 2: Depth Conversion #1
    # Full-screen pass to update hardware z-buffer
    # Fragment Shader:
    #   d_new = layer_depth_texture.sample(coord)
    #   d_prev = hardware_z_buffer.read(coord)
    #   if d_new > d_prev:
    #       hardware_z_buffer.write(coord, Z_MAX)
    #   else:
    #       hardware_z_buffer.write(coord, d_new)
        
    # Step 3: Layer Shading
    # Render geometry with z-function set to EQUAL
    # Fragment Shader:
    #   color = compute_lighting()
    #   alpha = get_layer_alpha()
    #   hardware_color_buffer.blend(color, alpha)
    
    # Update accumulated opacity
    # Note: In practice, this might require a separate read or accumulation buffer update
    accumulated_opacity += get_current_layer_alpha()
    
    # Step 4: Depth Conversion #2
    # Full-screen pass to check opacity threshold
    # Fragment Shader:
    #   if accumulated_opacity > OPACITY_THRESHOLD:
    #       hardware_z_buffer.write(coord, Z_MAX)
    #   # Else, leave z-buffer as is from Step 2
    
    # Update previous layer depth for next iteration
    previous_layer_depth = layer_depth_texture.sample(coord) # Conceptual update
    
    # Optimization: If accumulated_opacity >= 1.0, break early
    if accumulated_opacity >= 1.0:
        break
```

### Edge Cases and Pitfalls
*   **Wave Divergence:** Standard depth peeling performs lighting conditionally in the same draw call as depth finding, causing high wave divergence due to shader discard operations. This method avoids this by separating depth finding (Step 1) from shading (Step 3) [S1].
*   **Surface Intersection Check:** In Step 1, it is necessary to search for intersection with the encoded surface to determine whether the fragment should be discarded. Without this, transparent parts would occupy each layer, requiring more passes for a correct image [S1].
*   **Performance Scaling:** Empirical results show that 3-4 passes are enough in many cases. After the second pass, there are almost no changes in the resulting image [S1].

## Operating characteristics
*   **Time Complexity:** The method requires $N$ passes, where $N$ is the number of layers rendered (up to $N_{max}$). Each pass involves 4 steps, including two full-screen passes for depth conversion. However, the cost per pass decreases as more layers are rendered because the z-buffer culls more pixels in subsequent passes [S1].
*   **Memory Complexity:** Requires storage for the previous layer's depth, the accumulated opacity buffer, and temporary textures for depth conversion. The memory overhead is constant relative to the number of layers [S1].
*   **Measured Performance:** Tests on an NVIDIA GeForce RTX 2080 GPU at resolutions of 800x800 and 2560x1600 showed that the method maintains good performance even at higher resolutions, outperforming basic shell mapping without LODs at larger texture sizes [S1].
*   **Strengths:**
    *   Reduces wave divergence by separating depth finding and shading.
    *   Utilizes hardware z-buffer for culling, avoiding expensive shader-based discard operations.
    *   Provides stable and high-quality results for the first $K$ semi-transparent layers [S1].
*   **Limitations:**
    *   Requires a user-defined maximum number of layers; layers beyond this limit are not rendered, potentially causing visual artifacts (e.g., darkening on parts where many prisms meet) [S1].
    *   The full-screen depth conversion passes add a constant cost, though this is compensated by reduced rendering time in subsequent passes [S1].
    *   Not suitable for scenarios requiring accurate rendering of all layers regardless of count, unlike linked-list or transmittance-based OIT methods [S1].

## Background
Hardware-Z Cull Depth Peeling was introduced in the paper "Real-time Rendering of Small-scale Volumetric Structure on Animated Surfaces" [S1]. It addresses the limitations of standard depth peeling, which suffers from high computational cost due to shader-based culling and wave divergence [S1]. The method is particularly relevant for rendering semi-transparent parts on the edges of animated volumetric surfaces, where sorting tetrahedrons is too expensive [S1]. It builds on the concept of depth peeling introduced in [15] and improves upon variants like those in [19, 20] by leveraging hardware depth tests for culling [S1].

## Sources
[S1] Real-time Rendering of Small-scale Volumetric Structure on Animated Surfaces, https://doi.org/10.20948/graphicon-2021-3027-64-73, fetched 2026-10-26
