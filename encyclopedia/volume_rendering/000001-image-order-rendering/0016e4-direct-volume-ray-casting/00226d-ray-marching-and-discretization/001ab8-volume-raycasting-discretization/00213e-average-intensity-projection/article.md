# Average Intensity Projection

Also called: Additive Reprojection, Average Intensity, Mean Density, Mean Density Projection, Simple Additive Projections, X-ray projection (average)

## Overview
Average Intensity Projection (AIP) is a volume rendering technique that computes the image by averaging the scalar intensities of voxels along rays through the volume onto the image plane [S1, S4]. It serves as a simplified alternative to front-to-back compositing, producing an image equivalent to an X-ray projection through the data [S1, S4]. Unlike standard ray casting which accounts for opacity and absorption, AIP treats the volume as a transparent medium where every sample contributes equally to the final pixel value, making it computationally inexpensive but prone to low contrast [S1, S2].

## How it works
### Inputs and Outputs
**Inputs:**
*   **Volume Data:** A 3D grid of scalar values. Sources describe two common layouts:
    1.  **3D Texture:** Dimensions $X \times Y \times Z$ (e.g., $256 \times 256 \times 109$), stored in a `GL_TEXTURE_3D` with `GL_RGBA` internal format but `GL_RED` base format, using `GL_UNSIGNED_BYTE` type [S2].
    2.  **Raw Array:** 1D array of `unsigned char` values with dimensions $N_x \times N_y \times N_z$, stored in row-major order [S4].
*   **Camera/View:** Defined by a view position (`viewPos`) and a bounding box for the volume (e.g., min `vec3(-1.0)`, max `vec3(1.0)` or centered at volume origin) [S2, S4].
*   **Parameters:**
    *   `num_steps`: Fixed number of sampling steps along the ray (e.g., 256) [S2].
    *   `scale`: Scaling factor for output image dimensions ($ImageWidth = VolumeWidth \times scale$) [S4].
    *   `isoSurf`: Optional uniform float for isosurface extraction mode, processed as `iso = isoSurf - floor(isoSurf)` [S2].

**Outputs:**
*   **Image:** A 2D grid of pixel values.
    *   **Format:** `unsigned char` (0-255 range) [S2, S4].
    *   **Dimensions:** Scaled width and height [S4].
    *   **Value:** The arithmetic mean of the sampled voxel intensities along the ray [S3].

### Data Structures
*   **Ray:** Defined by origin `or` and direction `dir` [S2].
*   **Transfer Function (Optional):** A 1D texture (`GL_TEXTURE_1D`) mapping scalar values to color/opacity. In pure AIP, this is often bypassed or used only for final color mapping [S2, S3].
*   **Accumulators:**
    *   `totalVoxelValue` or `meanSum`: Accumulates the sum of sampled intensities [S4, S3].
    *   `cnt` or `validSamples`: Counts the number of valid samples taken [S2, S3].

### Steps
1.  **Initialize:** For each pixel, initialize `totalVoxelValue = 0.0` and `cnt = 1` (to avoid division by zero) [S2, S4].
2.  **Ray Setup:**
    *   Compute ray origin `fpos` (fragment position) and direction `direction = normalize(fpos - viewPos)` [S2].
    *   Alternatively, for orthographic projections, map pixel coordinates to volume space: `originalX = pixelX / scale`, `originalY = pixelY / scale`, with ray origin at `(originalX, originalY, 0)` and direction `(0, 0, 1)` [S4].
3.  **Intersection (Perspective):** Intersect the ray with the volume bounding box to find entry (`t_enter`) and exit (`t_exit`) distances using the slab method [S2].
4.  **Step Calculation:** Calculate ray length `dist = distance(entry, exit)` and step size `stepsize = dist / num_steps` [S2].
5.  **Sampling Loop:** Iterate `i` from 0 to `num_steps - 1`:
    *   Increment counter `cnt++` [S2].
    *   Determine sample position `curPos` along the ray [S2].
    *   **Coordinate Transformation:**
        *   Scale Z-coordinate if necessary: `tmp.z = curPos.z * 2.3486238532110093` (to handle non-isotropic data or texture normalization) [S2].
        *   Map to texture coordinates: `texPos = (1 + tmp) / 2` [S2].
        *   Invert Y-axis: `texPos.y = 1 - texPos.y` (due to image data inversion) [S2].
    *   **Sampling:** Fetch value `val = texture(texVol, texPos).x` [S2].
    *   **Accumulation:**
        *   If using raw array indexing (orthographic case): Translate sample to volume center, apply rotation matrix, translate back, check bounds, compute linear index, and fetch `voxelIntensity = vol.data[index] / 255.0f` [S4].
        *   Add to sum: `totalVoxelValue += val` (or `voxelIntensity`) [S2, S4].
    *   **Advance:** `curPos = curPos + direction * stepsize` [S2].
6.  **Finalize:** Compute the average: `meanValue = totalVoxelValue / cnt` [S2, S4].
7.  **Color Mapping (Optional):** Map `meanValue` to a final color using a transfer function or color map [S3]. If lighting is enabled, calculate normals from the mean position and apply lighting [S3].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def average_intensity_projection(volume_texture, view_pos, image_width, image_height, num_steps=256, scale=1.0):
    """
    Renders an Average Intensity Projection of a 3D volume.
    
    Args:
        volume_texture: 3D texture object or array of shape (Z, Y, X)
        view_pos: Camera position (vec3)
        image_width: Output image width
        image_height: Output image height
        num_steps: Number of samples per ray
        scale: Scaling factor for image dimensions relative to volume
        
    Returns:
        image_data: 2D array of floats (0.0 to 1.0) representing average intensity
    """
    image_data = np.zeros((image_height, image_width))
    
    # Define bounding box for intersection (example: unit cube centered at origin)
    # Note: Specific bounds depend on volume coordinate system. 
    # Source [S2] uses min=(-1,-1,-1), max=(1,1,1). Source [S4] uses volume indices.
    bbox_min = np.array([-1.0, -1.0, -1.0])
    bbox_max = np.array([1.0, 1.0, 1.0])
    
    for y in range(image_height):
        for x in range(image_width):
            # Map pixel to world space
            # Source [S4] uses orthographic mapping: originalX = pixelX / scale
            # Source [S2] uses perspective: fpos derived from fragment position on cube
            # Implementing perspective ray casting as per [S2]
            
            # Calculate fragment position on the bounding box face
            # Assuming a standard projection matrix inverse is applied to get fpos
            # For simplicity, assuming fpos is provided or calculated from screen coords
            fpos = calculate_fragment_position(x, y, image_width, image_height, bbox_min, bbox_max)
            
            direction = normalize(fpos - view_pos)
            
            # Ray-Bbox Intersection (Slab Method)
            t_enter, t_exit = intersect_box(view_pos, direction, bbox_min, bbox_max)
            
            if t_enter > t_exit:
                continue # Ray misses volume
                
            dist = t_exit - t_enter
            step_size = dist / num_steps
            
            total_value = 0.0
            count = 1 # Avoid division by zero
            
            current_pos = view_pos + direction * t_enter
            
            for i in range(num_steps):
                count += 1
                
                # Coordinate transformation specific to source [S2]
                # Scale Z to fit texture coordinate space [-1, 1] after mapping
                tmp = np.array([current_pos[0], current_pos[1], current_pos[2] * 2.3486238532110093])
                
                # Map to [0, 1] texture coordinates
                tex_pos = (1.0 + tmp) / 2.0
                
                # Invert Y axis
                tex_pos[1] = 1.0 - tex_pos[1]
                
                # Sample volume
                # Clamp tex_pos to [0, 1] to handle boundary conditions
                tex_pos_clamped = np.clip(tex_pos, 0.0, 1.0)
                val = sample_volume(volume_texture, tex_pos_clamped)
                
                total_value += val
                
                # Advance position
                current_pos = current_pos + direction * step_size
                
            # Compute average
            mean_value = total_value / count
            image_data[y, x] = mean_value
            
    return image_data

def sample_volume(texture, tex_coords):
    """
    Samples a 3D texture using trilinear interpolation.
    """
    # Implementation depends on specific texture API. 
    # Assuming standard GLSL-like texture3d behavior
    return texture3d(texture, tex_coords)

def intersect_box(origin, direction, min_b, max_b):
    """
    Slab method for ray-box intersection.
    Returns (t_enter, t_exit). Returns None if no intersection.
    """
    t_min = (min_b - origin) / direction
    t_max = (max_b - origin) / direction
    
    t1 = np.minimum(t_min, t_max)
    t2 = np.maximum(t_min, t_max)
    
    t_enter = np.maximum(np.maximum(t1[0], t1[1]), t1[2])
    t_exit = np.minimum(np.minimum(t2[0], t2[1]), t2[2])
    
    if t_enter > t_exit:
        return None
        
    # Ensure we start inside or at the entry point
    if t_enter < 0:
        t_enter = 0
        
    return t_enter, t_exit
```

### Edge Cases and Pitfalls
*   **Division by Zero:** The sample counter `cnt` is initialized to 1 and incremented before sampling to ensure `color /= cnt` never divides by zero [S2].
*   **Coordinate Inversion:** The texture Y-coordinate must be inverted (`1 - texPos.y`) because image data is often inverted on the Y-axis [S2].
*   **Non-Isotropic Data:** The Z-coordinate may require scaling by a specific constant (e.g., `2.3486238532110093`) before mapping to texture space to handle non-isotropic voxels or specific normalization requirements [S2].
*   **Low Contrast:** Additive projection techniques suffer from a confused or unclear appearance because all information is projected, resulting in very little contrast to distinguish gradual variations or small abrupt changes [S1].
*   **Occlusion:** Unlike front-to-back compositing, AIP does not handle obscuration, which can hide structural details [S1].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the number of samples per ray [S2].
*   **Memory Complexity:** $O(V)$ for the volume data, where $V$ is the number of voxels [S4].
*   **Performance:** Very fast due to the lack of complex compositing (opacity/alpha blending) and lighting calculations [S1, S4].
*   **Strengths:** Simple implementation, fast rendering, useful for quick overviews or X-ray-like visualizations [S1, S4].
*   **Limitations:** Low contrast, no depth cues from occlusion, sensitive to noise and background clutter [S1].
*   **When to Use:** When speed is critical and depth perception is not required, or for initial data exploration [S1].
*   **When Not to Use:** When detailed structural visualization, depth cues, or high contrast is needed [S1].

## Background
Average Intensity Projection, also known as additive reprojection, was developed by Hohne et al. for medical imaging applications, specifically stereotactic neurosurgery planning [S1]. It is one of their generalized projection images, designed to directly image the volume of data without surface shading [S1]. The technique is equivalent to an X-ray projection through the volume [S1]. It is often contrasted with techniques that use partial volume effects for surface shading or segmentation [S1].

## Sources
[S1] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26
[S2] ShashwatC/Ray, https://github.com/ShashwatC/Ray, fetched 2026-10-26 (unofficial)
[S3] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S4] indira654/implementing-basic-X-Ray-and-MIP-volume-rendering-using-ray-casting, https://github.com/indira654/implementing-basic-X-Ray-and-MIP-volume-rendering-using-ray-casting, fetched 2026-10-26 (unofficial)
[S5] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
