# Voxel Volume Rendering

Also called: Block model rendering, Shader-driven voxel volume

## Overview
Voxel volume rendering visualizes a 3D scalar field by projecting light absorption and emission through the volume, revealing internal density variations without requiring surface extraction [S1, unofficial]. It differs from isosurface methods (like Marching Cubes) by preserving continuous volumetric data rather than extracting discrete boundaries, and from point-cloud rendering by operating on a structured grid [S1, S2, unofficial]. It is distinct from analytic volume rendering (e.g., ray marching through signed distance fields) by relying on pre-computed or interpolated discrete samples stored in a texture [S2, unofficial].

## How it works
**Inputs:**
1.  **Voxel Grid:** A 3D array of scalar values (e.g., density, concentration). In implementation, this is often a `Float32Array` uploaded as a `Data3DTexture` [S2, unofficial].
2.  **Transfer Function:** A mapping from scalar values to color and opacity. This can be a preset (e.g., "Viridis") or a custom two-stop function defined by display min/max and thresholds [S1, S2, unofficial].
3.  **Camera Parameters:** Position, focal point, and view-up vector [S1, unofficial].

**Outputs:**
*   A 2D image where pixel intensity represents the accumulated light through the volume [S1, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 1D array of size $N_x \times N_y \times N_z$. For a $64^3$ grid, this is approximately 1 MB; for $256^3$, approximately 64 MB [S2, unofficial].
*   **Texture:** The grid is uploaded to GPU memory as a 3D texture (e.g., `Data3DTexture` in WebGL/Three.js) [S2, unofficial].

**Steps:**
1.  **Data Preparation:** Load the 3D dataset and associate the scalar array with the rendering proxy [S1, unofficial]. If generating from sparse samples, interpolate onto a regular grid (e.g., using Inverse Distance Weighting) [S2, unofficial].
2.  **Texture Upload:** Upload the voxel grid to the GPU as a 3D texture [S2, unofficial].
3.  **Transfer Function Setup:** Apply a color and opacity mapping. This can be a preset or a user-defined range (displayMin/displayMax) [S1, S2, unofficial].
4.  **Ray Marching:** Cast rays from the camera through the volume. For each ray step:
    *   Sample the 3D texture at the current ray position [S2, unofficial].
    *   Check for valid data (skip sentinel values like $< -1e29$) [S2, unofficial].
    *   Apply the transfer function to get color ($C$) and opacity ($\alpha$) [S2, unofficial].
    *   Perform front-to-back compositing:
        $$ C_{out} = C_{in} + C_{sample} \cdot \alpha_{sample} \cdot (1 - \alpha_{in}) $$
        $$ \alpha_{out} = \alpha_{in} + \alpha_{sample} \cdot (1 - \alpha_{in}) $$
5.  **Rendering Mode Selection:**
    *   **Smooth Compositing:** Standard ray marching with linear interpolation between voxel centers [S2, unofficial].
    *   **Block Mode:** Snap ray steps to voxel centers for a crisp, blocky appearance [S2, unofficial].
6.  **Display Tweaks:** Adjust opacity, threshold, and color gradient. These are uniform updates and do not require re-uploading the texture [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assuming a WebGL/Three.js-like environment with a 3D texture 'volumeTexture'
# and a fragment shader performing the compositing.

def render_voxel_volume(camera, volume_texture, transfer_function, dims, block_mode=False):
    """
    Renders a voxel volume using ray marching.
    
    Args:
        camera: Camera object with position, focal point, and view-up.
        volume_texture: 3D texture containing the scalar field.
        transfer_function: Function mapping scalar value -> (color, opacity).
        dims: Tuple (width, height, depth) of the voxel grid.
        block_mode: Boolean, if True, snap ray steps to voxel centers.
    """
    # 1. Set up camera
    set_camera_position(camera.position)
    set_camera_focal_point(camera.focal_point)
    set_camera_view_up(camera.view_up)
    
    # 2. Setup rendering context
    set_texture_3d(volume_texture)
    set_transfer_function(transfer_function)
    
    # 3. Ray Marching Loop (conceptual, executed in shader)
    for each pixel (x, y) in image:
        ray_origin = camera.position
        ray_direction = compute_ray_direction(x, y, camera)
        
        accumulated_color = (0, 0, 0, 0)
        accumulated_alpha = 0.0
        
        # Determine step size and range based on dims and camera
        step_size = compute_step_size(dims, ray_direction)
        t_min, t_max = compute_ray_bounds(ray_origin, ray_direction, dims)
        
        t = t_min
        while t < t_max:
            # Sample position in voxel space
            sample_pos = ray_origin + ray_direction * t
            
            if block_mode:
                # Snap to voxel center
                sample_pos = snap_to_voxel_center(sample_pos, dims)
            
            # Sample scalar value from texture
            scalar_value = sample_3d_texture(volume_texture, sample_pos)
            
            # Skip invalid data (sentinel value)
            if scalar_value < -1e29:
                t += step_size
                continue
            
            # Apply transfer function
            color, opacity = transfer_function(scalar_value)
            
            # Front-to-back compositing
            alpha_contribution = opacity * (1.0 - accumulated_alpha)
            accumulated_color = accumulated_color + color * alpha_contribution
            accumulated_alpha = accumulated_alpha + opacity * (1.0 - accumulated_alpha)
            
            # Early exit if fully opaque
            if accumulated_alpha >= 1.0:
                break
                
            t += step_size
        
        # Apply global opacity and threshold
        final_color = accumulated_color * global_opacity
        if accumulated_alpha < threshold:
            final_color = (0, 0, 0, 0)
            
        set_pixel_color(x, y, final_color)

    # 4. Render and save
    render()
    save_screenshot(output_path, image_resolution=(1600, 1200))
```

**Parameters:**
*   **dims:** Voxel grid dimensions. Typical: $[64, 64, 64]$ [S2, unofficial].
*   **displayMin/Max:** Range for the transfer function. Typical: $0$ to $500$ [S2, unofficial].
*   **opacity:** Global volume opacity. Typical: $0.4$ [S2, unofficial].
*   **threshold:** Alpha cutoff for rendering. Typical: $0.1$ [S2, unofficial].
*   **ImageResolution:** Output image size. Typical: $[1600, 1200]$ for production, $[1000, 800]$ for animation [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Sentinel Values:** Invalid data points (e.g., $< -1e29$) must be skipped to prevent artifacts [S2, unofficial].
*   **Memory Usage:** High-resolution grids consume significant memory ($256^3 \approx 64$ MB) [S2, unofficial].
*   **Uniform Updates:** Display parameters (min/max, opacity, threshold) are uniform updates and do not require texture re-upload, allowing real-time tweaking [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of ray steps and pixels. Complexity is $O(P \cdot S)$, where $P$ is the number of pixels and $S$ is the number of steps per ray [G].
*   **Memory Complexity:** Linear with respect to the number of voxels, $O(N_x \cdot N_y \cdot N_z)$ [S2, unofficial].
*   **Performance:** Real-time rendering is possible with lower resolutions (e.g., $64^3$) and optimized ray marching [S2, unofficial]. Higher resolutions may require down-sampling or hierarchical structures.
*   **Strengths:** Preserves internal structure; supports real-time parameter tweaking; suitable for sparse data interpolated onto a grid [S2, unofficial].
*   **Limitations:** Memory intensive for high-resolution volumes; block mode sacrifices visual smoothness [S2, unofficial].

## Background
Voxel volume rendering is a standard technique in scientific visualization, often implemented in libraries like VTK and ParaView [S1, unofficial]. It is used to visualize scalar fields such as concentration, density, or interpolated geological data [S1, S2, unofficial]. The method has been adapted for web-based rendering using WebGL and libraries like Three.js [S2, unofficial].

## Sources
[S1] cjcamp16/sciviz-paraview, https://github.com/cjcamp16/sciviz-paraview, fetched 2026-10-26 (unofficial)
[S2] feat: 3D IDW interpolation volume and voxel visualisation, https://github.com/darkmine-oss/baselode/pull/50, fetched 2026-10-27 (unofficial)
