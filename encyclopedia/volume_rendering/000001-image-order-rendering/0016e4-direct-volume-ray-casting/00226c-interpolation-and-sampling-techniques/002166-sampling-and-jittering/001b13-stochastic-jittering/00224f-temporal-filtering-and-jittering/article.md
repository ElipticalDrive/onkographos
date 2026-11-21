# Temporal Filtering and Jittering

Also called: Temporal Filtering, Temporal Supersampling

## Overview
Temporal Filtering and Jittering is a technique that smooths out noise and improves perceived volume resolution without extra raymarching steps [S1, unofficial]. It addresses spatial banding artifacts caused by coarse slice resolution by combining sub-voxel depth jittering with temporal reprojection [S1, unofficial]. It distinguishes itself from static spatial filters by leveraging camera motion vectors to blend frames [S1, unofficial], and from pure spatial sampling methods by accumulating jittered samples over time to improve perceived resolution without additional raymarching steps [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current frame’s 3D texture (or accumulated volume data), previous frame’s 3D texture, camera motion vectors, and a pseudo-random sequence (e.g., blue noise or Halton sequence) [S1, unofficial].
*   **Outputs:** A final filtered 3D texture [S1, unofficial].

**Data Structures:**
*   **3D Texture:** Stores volumetric data (e.g., transmittance, in-scattered light) [S1, unofficial]. Layout is not explicitly defined in sources, but standard hardware trilinear interpolation (`GL_LINEAR`) is used for sampling [S1, unofficial].
*   **Motion Vectors:** 2D vectors per pixel representing the displacement of scene points between frames, used for reprojection [S1, unofficial].
*   **Jitter Offset:** A sub-voxel offset derived from the pseudo-random sequence [S1, unofficial].

**Steps:**
1.  **Depth Jittering:** Each frame, apply a sub-voxel offset to ray positions [S1, unofficial]. The offset is derived from a pseudo-random sequence such as blue noise or a Halton sequence [S1, unofficial]. This offsets the sampling positions to break up spatial aliasing patterns [S1, unofficial].
2.  **Raymarching/Accumulation:** Execute the volume rendering pass (raymarching) using the jittered ray positions to generate the current frame’s 3D texture data [S1, unofficial].
3.  **Temporal Reprojection Compute Pass:** Blend the current frame’s 3D texture with the previous frame's 3D texture using camera motion vectors [S1, unofficial]. This step effectively smooths out noise by accumulating information over time [S1, unofficial].
4.  **Final Composition (Post-processing):** The accumulated volumetric data is applied to the scene during post-processing or forward-lighting [S1, unofficial]:
    *   Retrieve the linear depth of the opaque scene fragment from the scene's depth buffer [S1, unofficial].
    *   Convert the fragment's screen coordinates and linear depth into 3D texture coordinates $(u, v, w)$ using the exponential depth equation [S1, unofficial].
    *   Sample the integrated 3D texture using hardware trilinear interpolation [S1, unofficial].
    *   Add the sampled in-scattered light to the scene color and attenuate the scene color by the sampled transmittance [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def temporal_filtering_and_jittering(current_frame_texture, previous_frame_texture, motion_vectors, frame_index):
    """
    Implements Temporal Filtering and Jittering for volume rendering.
    
    Args:
        current_frame_texture (ndarray): Raw accumulated volume data for the current frame.
        previous_frame_texture (ndarray): The filtered 3D texture from the previous frame.
        motion_vectors (ndarray): 2D motion vectors for the current frame.
        frame_index (int): Index of the current frame for jitter sequence lookup.
        
    Returns:
        ndarray: The final filtered 3D texture for the current frame.
    """
    height, width = current_frame_texture.shape[:2]
    filtered_texture = np.zeros_like(current_frame_texture)
    
    # Step 1: Generate jitter offset for the current frame
    # [G] Standard pseudo-random sequence lookup (e.g., blue noise or Halton sequence)
    jitter_offset = np.array([0.0, 0.0]) 
    
    # Step 2 & 3: Temporal Reprojection and Blending
    for y in range(height):
        for x in range(width):
            # Reproject previous frame's pixel to current screen space
            mv = motion_vectors[y, x]
            prev_x = int(x + mv[0])
            prev_y = int(y + mv[1])
            
            # Clamp to bounds
            prev_x = max(0, min(width - 1, prev_x))
            prev_y = max(0, min(height - 1, prev_y))
            
            prev_value = previous_frame_texture[prev_y, prev_x]
            curr_value = current_frame_texture[y, x]
            
            # Blend current and previous data
            # [G] Simple temporal accumulation blend; exact weights depend on implementation
            filtered_value = 0.5 * curr_value + 0.5 * prev_value
            filtered_texture[y, x] = filtered_value
            
    return filtered_texture

def final_composition(filtered_texture, scene_depth_buffer, screen_coordinates, scene_color):
    """
    Applies the filtered volumetric data to the final scene.
    
    Args:
        filtered_texture (ndarray): The output from temporal_filtering_and_jittering.
        scene_depth_buffer (ndarray): Linear depth of the opaque scene.
        screen_coordinates (tuple): Fragment's screen coordinates (u, v).
        scene_color (tuple): Opaque scene color.
        
    Returns:
        tuple: The final scene color with volumetric effects.
    """
    # Retrieve linear depth of the opaque scene fragment
    linear_depth = scene_depth_buffer[screen_coordinates]
    
    # Convert screen coordinates and linear depth to 3D texture coordinates (u, v, w)
    # using the exponential depth equation [S1, unofficial]
    # [G] Standard exponential depth mapping: w = exp(-depth * scale) normalized to [0,1]
    u, v = screen_coordinates
    w = np.exp(-linear_depth * 0.1) # [G] Placeholder scale factor
    
    # Sample the integrated 3D texture using hardware trilinear interpolation
    # [G] Standard trilinear interpolation over the 3D texture
    sampled_light = np.zeros(3) # [G] Placeholder for trilinear sample
    
    # Sample transmittance (assumed to be stored in the same 3D texture or a companion texture)
    # [G] Assuming transmittance is stored in the same 3D texture or a companion texture
    transmittance = 1.0 # [G] Placeholder for trilinear transmittance sample
    
    # Add in-scattered light to scene color and attenuate by transmittance
    final_color = tuple(c * transmittance + s for c, s in zip(scene_color, sampled_light))
    
    return final_color
```

**Parameters:**
*   **Jitter Sequence:** Blue noise or Halton sequence [S1, unofficial].
*   **Interpolation:** Hardware trilinear interpolation (`GL_LINEAR`) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Motion Vectors:** Incorrect or missing motion vectors will cause ghosting or blurring in the temporal reprojection step [S1, unofficial].
*   **Precision:** Not given in the sources [S3, book].
*   **Boundaries:** Not given in the sources [S3, book].

## Subtopics

### [Edge-Preserving Kawase Blur](7026-edge-preserving-kawase-blur.md)
Edge-Preserving Kawase Blur is a post-processing technique designed to remove dither patterns introduced by blue noise jittering in volume rendering while preserving cloud edge details. It distinguishes itself from standard box blurs by using edge-aware weighting to prevent the smoothing of sharp volumetric boundaries, and from temporal accumulation methods by operating on a single frame without requiring reprojection of previous frames.

### Bicubic Filtering Upscaling
A post-processing technique that smooths noise artifacts from blue noise dithering by calculating a new pixel value using a cubic polynomial over 16 neighboring pixels [S2, unofficial]. It is applied as a separate post-processing pass using Frame Buffer Objects (FBOs) to render the main scene into a texture, which is then sampled by a bicubic filtering material on a fullscreen triangle [S2, unofficial].

### Stochastic Filtering
A technique to replace weighted volume lookups (trilinear interpolation) with a single stochastically jittered sample to speed up rendering while maintaining unbiased results [S3, book]. It provides a roughly 3.5x speedup compared to trilinear filtering [S3, book]. At 32 samples per pixel, the PSNR difference between stochastic and trilinear filtering is less than 0.1 dB compared to a converged reference [S3, book]. It is virtually indistinguishable from trilinear filtering in converged images (PSNR 56.23 dB) but is significantly faster [S3, book].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S3, book].
*   **Memory Complexity:** Not given in the sources [S3, book].
*   **Measured Performance:** Stochastic filtering (a variant) provides a roughly 3.5x speedup compared to trilinear filtering [S3, book]. Timings were measured on an NVIDIA RTX 3090 GPU at 1920 × 1080 resolution with three bounces of multiple (isotropic) scattering, using a Moana cloud at 1000 × 680 × 1224 voxel resolution [S3, book].
*   **Scaling:** Not given in the sources [S3, book].
*   **Strengths:** Smooths out noise and improves perceived volume resolution without extra raymarching steps [S1, unofficial]. Stochastic filtering variant is significantly faster [S3, book].
*   **Limitations:** Requires motion vectors for temporal reprojection [S1, unofficial]. Stochastic filtering may introduce noise at low sample counts, though the difference is negligible at 32 samples per pixel [S3, book].
*   **When to use:** When spatial banding artifacts are visible due to coarse slice resolution and extra raymarching steps are undesirable [S1, unofficial].
*   **When not to use:** When motion vectors are unavailable or unreliable [S1, unofficial].

## Background
*   **Origin:** The technique is described in the context of volumetric fog using 3D textures in OpenGL [S1, unofficial] and efficient unbiased volume path tracing on the GPU [S3, book].
*   **History and Development:** The use of depth jittering with blue noise or Halton sequences to hide spatial banding artifacts is a known technique in volume rendering [S1, unofficial]. The integration of temporal reprojection for noise smoothing is an extension of this approach [S1, unofficial]. Stochastic filtering as a replacement for trilinear interpolation was introduced to improve performance in volume path tracing [S3, book].
*   **Where it is used:** Real-time volumetric rendering, particularly for clouds and fog [S1, unofficial][S2, unofficial].

## Sources
[S1] Volumetric Fog Using 3D Textures in OpenGL, https://salivity.github.io/opengl/article/volumetric-fog-using-3-d-textures-in-open-gl, fetched 2026-10-27 (unofficial)
[S2] Real-time dreamy Cloudscapes with Volumetric Raymarching, https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/, fetched 2026-10-28 (unofficial)
[S3] Efficient Unbiased Volume Path Tracing on the GPU, https://doi.org/10.1007/978-1-4842-7185-8_43, fetched 2026-10-24 (book)
