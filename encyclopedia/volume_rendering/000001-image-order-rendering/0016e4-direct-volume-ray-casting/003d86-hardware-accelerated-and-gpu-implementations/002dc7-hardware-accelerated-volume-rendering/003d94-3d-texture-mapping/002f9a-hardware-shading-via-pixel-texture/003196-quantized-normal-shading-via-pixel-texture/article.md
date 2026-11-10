# Quantized Normal Shading via Pixel Texture

Also called: Quantized Normal Shading

## Overview
Quantized Normal Shading via Pixel Texture is a hardware-accelerated volume rendering technique that encodes surface normals into a 2D pixel texture to provide shading while avoiding the redundancy and truncation errors associated with standard hardware interpolation [S1]. It distinguishes itself from nearest-neighbor volume texture sampling by using a dedicated 2D texture lookup for lighting calculations, ensuring that the quantized normal data is retrieved without interpolation artifacts [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (scalar values $V, G, H$ or similar multi-dimensional components) and a current view vector.
*   **Output:** A shaded 2D image composited into the frame buffer.

**Data Structures:**
*   **Shading Pixel Texture:** A 2D texture where the red channel stores the first 8 bits of the quantized normal, and the green channel stores the second 8 bits [S1]. The blue and alpha channels are unused or reserved [S1].
*   **Scratch/Auxiliary Buffer:** A temporary buffer used to render individual slices before they are composited into the final frame buffer [S1].

**Quantization Scheme:**
Normals are quantized using a 16-bit scheme [S1]. The normal vector $\mathbf{n}$ is mapped to a 16-bit integer index. This index is then split into two 8-bit values:
1.  First 8 bits $\rightarrow$ Red channel ($R$)
2.  Second 8 bits $\rightarrow$ Green channel ($G$) [S1]

**Algorithm Steps:**
1.  **Generate Shading Texture:** For the current view direction, compute the quantized normals for all relevant data points. Encode these normals into the 2D pixel texture as described above. This is typically done in software on a per-view basis [S1].
    *   *Alternative Hardware Path:* Render the quantized normals as points with hardware lighting, then copy the result from the frame buffer to the pixel texture. Note that some hardware implementations do not support this flexible operation [S1].
2.  **Render Slice to Scratch Buffer:** Render the current slice of the volume data (using the appropriate transfer function components) into the scratch/auxiliary buffer [S1].
3.  **Apply Shading via Pixel Texture:** Use a `glCopyPixels()` operation (or equivalent) to render the slice from the scratch buffer while applying the shading pixel texture [S1].
    *   *Constraint:* On hardware like the SGI Octane 2 with V-series cards, pixel texture lookup does not occur directly on a per-fragment basis during standard rasterization. Therefore, the slice must be rendered to a buffer first, and then copied/pixel-textured [S1].
    *   *Interpolation:* Nearest-neighbor interpolation must be used for the pixel texture lookup to preserve the quantized normal values [S1].
4.  **Composite:** Blend the shaded slice from the scratch buffer into the main frame buffer, combining it with previously rendered slices [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volume_slice(volume_data, view_vector, frame_buffer, scratch_buffer, shading_texture):
    """
    Renders a single slice of the volume with quantized normal shading.
    
    Args:
        volume_data: 3D array of scalar values (V, G, H).
        view_vector: 3D vector representing the current camera view direction.
        frame_buffer: The final image buffer.
        scratch_buffer: Temporary buffer for the current slice.
        shading_texture: 2D texture containing quantized normals (R=bits 0-7, G=bits 8-15).
    """
    
    # 1. Generate or update the shading pixel texture for the current view
    # This step is performed per-view. In software:
    shading_texture = generate_shading_texture(volume_data, view_vector)
    
    # 2. Render the current slice to the scratch buffer
    # This involves sampling the volume data and applying the transfer function
    render_slice_to_buffer(volume_data, scratch_buffer)
    
    # 3. Apply shading using the pixel texture
    # On hardware like SGI Octane 2, this requires glCopyPixels() because
    # pixel texture lookup is not available during standard per-fragment rasterization.
    # The texture uses nearest-neighbor interpolation.
    apply_pixel_texture_shading(scratch_buffer, shading_texture)
    
    # 4. Composite the shaded slice into the frame buffer
    # Blend the result from the scratch buffer with the existing frame buffer
    composite_slice_to_frame(scratch_buffer, frame_buffer)

def generate_shading_texture(volume_data, view_vector):
    """
    Encodes quantized normals into a 2D texture.
    """
    texture_width = ... # Determined by data layout
    texture_height = ...
    shading_texture = create_2d_texture(texture_width, texture_height)
    
    for each data point in volume_data:
        normal = compute_normal(data_point, view_vector)
        quantized_index = quantize_normal(normal) # 16-bit integer
        
        # Split 16-bit index into two 8-bit channels
        red_channel = quantized_index & 0xFF
        green_channel = (quantized_index >> 8) & 0xFF
        
        shading_texture.set_pixel(data_point.u, data_point.v, 
                                  color=(red_channel, green_channel, 0, 0))
    return shading_texture
```

**Parameters:**
*   **Quantization Bits:** 16 bits total (8 bits per channel for R and G) [S1].
*   **Interpolation Mode:** Nearest-neighbor [S1].

**Edge Cases and Pitfalls:**
*   **Hardware Flexibility:** Some hardware implementations are not flexible enough to support rendering quantized normals as points with hardware lighting and copying them to a pixel texture [S1]. In such cases, the software-based generation of the shading texture is required [S1].
*   **Rasterization Limitations:** On certain architectures (e.g., SGI Octane 2), pixel texture lookup is not available during standard per-fragment rasterization, necessitating the use of an auxiliary buffer and `glCopyPixels()` [S1].
*   **Performance Cost:** Generating the shading pixel texture on a per-view basis in software has a minimal performance cost [S1].

## Subtopics

### RGB Normal Shading via Pixel Texture
A lighting method using texture-shader features (e.g., Nvidia GeForce 3) where the normal vector at each point is encoded in the three color values of an RGB texture [S2]. The GeForce 3 interpolates these values for each pixel, computes the dot product with the current view vector, and uses the result as a lookup into a texture that encodes a reflection map [S2]. This method provides better precision than the normal index method but costs more memory [S2].

### Hardware Lighting via Normal Index Textures
A hardware lighting method encoding normal information in a separate set of textures. It encodes the normal at each data point as an 8-bit index indicating one of 256 direction vectors uniformly distributed in 3-space [S2]. These textures are drawn in a separate pass using a palette to map each normal index to a specular light value for the current view vector [S2]. This method has lower precision than the RGB texture method but uses less memory [S2].

## Operating characteristics
*   **Time Complexity:** The generation of the shading pixel texture is performed on a per-view basis. The cost is described as "minimal" when done in software [S1].
*   **Memory Complexity:** The method requires a scratch/auxiliary buffer for each slice [S1]. The shading texture size depends on the quantization scheme (16-bit per normal) [S1].
*   **Performance:** The method is implemented on SGI Octane 2 with V-series graphics cards and nVidia GeForce3 series [S1]. The performance cost of generating the shading texture is minimal [S1].
*   **Strengths:** Avoids hardware interpolation redundancy and truncation errors associated with standard normal interpolation [S1].
*   **Limitations:** Requires an auxiliary buffer on some hardware due to rasterization limitations [S1]. Some hardware does not support the alternative hardware lighting path [S1].
*   **When to Use:** When high-quality shading is required in volume rendering and the hardware supports pixel texture operations (or can be emulated via copy operations) [S1].
*   **When Not to Use:** When hardware flexibility is limited and the software-based texture generation becomes a bottleneck, or when memory constraints are severe (compared to normal index methods) [S1, S2].

## Background
The method was developed for interactive volume rendering using multi-dimensional transfer functions [S1]. It was implemented on the SGI Octane 2 with V-series graphics cards and the nVidia GeForce3 series graphics adaptor [S1]. The approach leverages the texture-shader features of hardware like the GeForce 3 to perform lighting calculations in hardware, enhancing the geometry and structure of the rendered data [S2]. The key observation in the implementation is that pixel texture operations only need to be applied to the portion of the volume currently changing (e.g., when the transfer function is manipulated) [S1].

## Sources
[S1] Interactive Volume Rendering Using Multi-Dimensional Transfer Functions, https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
[S2] Interactive Multi-volume Visualization, https://doi.org/10.1007/3-540-46080-2_11, fetched 2026-10-26
