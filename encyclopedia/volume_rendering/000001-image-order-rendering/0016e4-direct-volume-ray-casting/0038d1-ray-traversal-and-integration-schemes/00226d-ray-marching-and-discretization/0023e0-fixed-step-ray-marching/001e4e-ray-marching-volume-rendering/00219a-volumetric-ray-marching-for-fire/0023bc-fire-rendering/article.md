# Fire Rendering

Also called: Fire Rendering via Reaction Coordinate, Fire Volume Rendering

## Overview
Fire rendering is a direct volume ray casting technique that renders fire by accumulating values determined by a reaction coordinate $Y$ rather than smoke density, using an artist-defined 1D texture to map these coordinates to fire colors [S1, unofficial]. Unlike smoke rendering, which blends values during the marching process, fire rendering accumulates these reaction-based values to simulate the emissive nature of flames [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume texture containing the reaction coordinate $Y$ for each voxel. An artist-defined 1D texture mapping reaction coordinates to colors. Camera parameters for ray generation.
*   **Output:** A 2D image of the rendered fire. Optionally, lighting contributions for other scene objects.

**Data Structures:**
*   **Reaction Coordinate Volume:** A 3D texture where each voxel stores a scalar value $Y$. The layout and resolution depend on the simulation source.
*   **Color Map:** A 1D texture (or array) mapping $Y \in [0, 1]$ (or the simulation's range) to RGBA fire colors.

**Algorithm Steps:**
1.  **Ray Generation:** For each pixel, generate a ray from the camera origin through the pixel's position in world space.
2.  **Volume Intersection:** Determine the entry and exit points of the ray with the fire volume bounding box.
3.  **Ray Marching:** Step along the ray within the volume bounds.
    *   At each step, sample the reaction coordinate $Y$ from the 3D volume texture.
    *   Sample the color $C$ from the 1D color map using $Y$ as the index.
    *   Accumulate the color. Unlike smoke, where alpha blending is standard, fire rendering accumulates values based on the reaction coordinate [S1, unofficial]. The specific accumulation formula is not explicitly detailed in the source beyond "accumulating values," but typically this implies additive blending or integration of emissive intensity.
4.  **Lighting (Optional):**
    *   **Point Light Approach:** Sample the volume at several locations. Treat each sample as a point light source. Use $Y$ and the 1D color texture to determine the light's intensity and color.
    *   **Voxel Light Approach:** Downsample the reaction coordinate texture to an extremely low resolution. Use every voxel in this low-res texture as a light source.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_fire(ray_origin, ray_direction, volume_texture, color_map, num_samples):
    # Determine intersection with volume bounding box
    t_min, t_max = intersect_bounding_box(ray_origin, ray_direction, volume_bounds)
    
    if t_max < 0 or t_min > t_max:
        return background_color

    # Step along the ray
    step_size = (t_max - t_min) / num_samples
    accumulated_color = vec3(0.0)
    
    for i in range(num_samples):
        t = t_min + i * step_size
        position = ray_origin + t * ray_direction
        
        # Sample reaction coordinate Y from volume
        Y = volume_texture.sample(position)
        
        # Map Y to fire color using 1D texture
        color = color_map.sample(Y)
        
        # Accumulate color (additive accumulation for emissive fire)
        accumulated_color += color * step_size # [G] Standard integration approximation
        
    return accumulated_color

def render_fire_lighting_point(volume_texture, color_map, num_samples):
    """
    Alternative lighting approach: treat volume samples as point lights.
    """
    lights = []
    # Sample volume at several locations
    for _ in range(num_samples):
        position = random_point_in_volume()
        Y = volume_texture.sample(position)
        color = color_map.sample(Y)
        intensity = calculate_intensity(Y) # [G] Implementation detail not specified
        lights.append(PointLight(position, color, intensity))
    return lights

def render_fire_lighting_voxel(volume_texture, color_map, low_res_volume):
    """
    Alternative lighting approach: downsample volume and use voxels as lights.
    """
    lights = []
    # Downsample reaction coordinate texture to extremely low resolution
    low_res_vol = downsample(volume_texture, resolution=(1, 1, 1)) # [G] Specific downsampling method not specified
    
    # Use every voxel as a light source
    for voxel in low_res_vol:
        Y = voxel.value
        color = color_map.sample(Y)
        intensity = calculate_intensity(Y) # [G] Implementation detail not specified
        lights.append(PointLight(voxel.position, color, intensity))
    return lights
```

**Parameters:**
*   **Reaction Coordinate $Y$:** The primary variable driving the visual appearance. It indexes into the 1D color texture [S1, unofficial].
*   **Number of Samples:** Controls the smoothness of the ray marching. Too few samples in the point light approach can lead to severe flickering [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Flickering:** Treating volume samples as point light sources can lead to severe flickering if not enough point samples are used [S1, unofficial].
*   **Loss of High-Frequency Effects:** The low-resolution voxel lighting approach is less prone to flickering but does not capture high-frequency lighting effects such as local lighting due to sparks [S1, unofficial].
*   **Precision:** Not given in the sources.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per ray and the number of rays (pixels). Lighting calculations add complexity based on the number of light sources generated.
*   **Memory Complexity:** Depends on the resolution of the 3D reaction coordinate volume and the 1D color map.
*   **Strengths:** Simple to implement; allows for artistic control via the 1D color texture; can be used as a light source.
*   **Limitations:** The point light approach can flicker; the voxel light approach loses high-frequency details.
*   **When to use:** When a stylized or artist-controlled fire appearance is desired, or when the fire volume is already available as a simulation output.
*   **When not to use:** When physically accurate fire lighting with high-frequency details is required without significant performance cost.

## Background
Fire rendering is discussed in the context of real-time fluid simulation and rendering. It is similar to smoke rendering but differs in the accumulation method and the use of the reaction coordinate [S1, unofficial]. A more physically based discussion of fire rendering can be found in Nguyen et al. 2002 [S1, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
