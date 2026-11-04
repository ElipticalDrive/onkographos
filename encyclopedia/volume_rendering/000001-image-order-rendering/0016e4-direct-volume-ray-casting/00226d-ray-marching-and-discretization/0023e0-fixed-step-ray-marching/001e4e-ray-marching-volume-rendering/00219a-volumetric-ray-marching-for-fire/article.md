# Volumetric Ray Marching for Fire

Also called: Fire Simulation Volume Rendering, Fire Volume Rendering

## Overview
Volumetric Ray Marching for Fire is a GPU-friendly technique that samples a 3D volume along a ray to render fire simulation results stored in 3D textures [S1, unofficial]. It distinguishes itself from smoke rendering by accumulating values based on a reaction coordinate (fuel) rather than blending density values, using an artist-defined color gradient to map these coordinates to flame colors [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two 3D textures from the simulation result containing density and reaction coordinate (fuel) values [S1, unofficial]. A bounding volume defining the fire region [S1, unofficial].
*   **Outputs:** Final pixel color [S1, unofficial].

**Data Structures:**
*   **3D Textures:** Store density and reaction coordinate values per voxel [S1, unofficial].
*   **Color Gradient:** An artist-defined mapping specified by three colors: core, border, and smoke [S1, unofficial].
*   **1D Texture (Optional):** Maps reaction coordinates to colors for fire rendering [S2, unofficial].

**Steps:**
1.  Inject a shader into the post-processing stage of the main camera [S1, unofficial].
2.  Generate rays starting at the center of the camera in the direction of each pixel [S1, unofficial].
3.  Calculate whether the ray hits the target volume [S1, unofficial].
4.  If the ray does not hit the volume, the sampled value is set to 0 [S1, unofficial].
5.  If the ray hits, march through the volume in small steps and sample the volume at each step [S1, unofficial].
6.  Sample the density and reaction coordinate textures from the simulation result at each step [S1, unofficial].
7.  Use the density value to determine the thickness of the smoke, and use the reaction coordinate to determine the color of the flame by mapping its value to the core, border, and smoke gradient [S1, unofficial].
8.  Calculate the average of the samples weighted by the length of the ray inside the target volume to determine the pixel color [S1, unofficial].

**Formulas:**
The pixel color is determined by the average of samples weighted by the length of the ray inside the target volume [S1, unofficial]. For smoke rendering, the pipeline borrows from existing literature on volumetric clouds [S1, unofficial]. The equation uses $d$ for density, $\theta$ for the angle between the camera ray and the sun, and $g$ for eccentricity which defaults to $0.2$ [S1, unofficial]. The exact mathematical formula for the eccentricity function is not given in the sources [S1, unofficial]. The end result is multiplied by the smoke color for a more realistic look [S1, unofficial].

**Parameters:**
*   **Voxel Grid Size:** $128 \times 128 \times 128$ [S1, unofficial].
*   **GPU Blocks:** $16 \times 16 \times 16$ blocks, each containing 512 threads [S1, unofficial].
*   **Eccentricity ($g$):** $0.2$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   If the ray does not hit the volume, the sampled value is 0 [S1, unofficial].
*   Debugging attribute propagation across timestamps is difficult due to HLSL shader limitations [S1, unofficial].
*   Treating volume samples as point light sources can lead to severe flickering if not enough point samples are used [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_fire(camera_center, camera_direction, pixel_uv, volume_density, volume_reaction, volume_bounds, step_size):
    # Generate ray
    ray_origin = camera_center
    dir_x = camera_direction[0] + pixel_uv[0]
    dir_y = camera_direction[1] + pixel_uv[1]
    dir_z = camera_direction[2] + pixel_uv[2]
    length = math.sqrt(dir_x**2 + dir_y**2 + dir_z**2)
    ray_direction = (dir_x / length, dir_y / length, dir_z / length)

    # Check intersection with volume bounds (AABB)
    # volume_bounds = [min_x, max_x, min_y, max_y, min_z, max_z]
    t_min = 0.0
    t_max = float('inf')
    for i in range(3):
        if ray_direction[i] == 0.0:
            if ray_origin[i] < volume_bounds[2 * i] or ray_origin[i] > volume_bounds[2 * i + 1]:
                return [0.0, 0.0, 0.0]
        else:
            t1 = (volume_bounds[2 * i] - ray_origin[i]) / ray_direction[i]
            t2 = (volume_bounds[2 * i + 1] - ray_origin[i]) / ray_direction[i]
            t_enter = min(t1, t2)
            t_exit = max(t1, t2)
            t_min = max(t_min, t_enter)
            t_max = min(t_max, t_exit)
    if t_min > t_max:
        return [0.0, 0.0, 0.0]

    # March through volume
    total_weighted_color = [0.0, 0.0, 0.0]
    total_weight = 0.0
    current_t = t_min

    while current_t <= t_max:
        current_pos = (ray_origin[0] + ray_direction[0] * current_t,
                       ray_origin[1] + ray_direction[1] * current_t,
                       ray_origin[2] + ray_direction[2] * current_t)

        # Sample textures
        density = volume_density[current_pos]  # Not given in the sources: exact texture sampling implementation
        reaction = volume_reaction[current_pos]  # Not given in the sources: exact texture sampling implementation

        # Determine color based on reaction coordinate
        # Map reaction to gradient: core, border, smoke
        flame_color = map_reaction_to_color(reaction)  # Not given in the sources: exact gradient mapping algorithm

        # Weight by step length (ray length inside volume)
        total_weighted_color = [total_weighted_color[i] + flame_color[i] * step_size for i in range(3)]
        total_weight += step_size

        # Move along ray
        current_t += step_size

    # Calculate average
    if total_weight > 0:
        final_color = [total_weighted_color[i] / total_weight for i in range(3)]
        # Apply smoke color multiplication for realism
        final_color = [final_color[i] * smoke_color[i] for i in range(3)]
        return final_color
    else:
        return [0.0, 0.0, 0.0]
```

## Subtopics

### Ember Effect
Distinct spurts of burning particles caused by incompletely combusted fuel advected into the air [S1, unofficial]. It is integrated by simulating an additional Voxel space of ember particles [S1, unofficial]. Randomly generate ember particles at the bottom of the fire with Perlin noise [S1, unofficial]. Propel particles upwards through advection [S1, unofficial]. Raymarching procedure detects these particles and adds a small glowing aura around them to make them appear realistic [S1, unofficial].

### Fire Rendering
Rendering fire by accumulating values determined by the reaction coordinate Y rather than smoke density, using an artist-defined 1D texture that maps reaction coordinates to colors to give the appearance of fire [S2, unofficial]. March through the volume and accumulate values determined by the reaction coordinate Y, using the 1D texture [S2, unofficial]. To use the fire volume as a light source, sample the volume at several locations and treat each sample as a point light source, using the reaction coordinate and 1D color texture to determine intensity and color [S2, unofficial]. Alternative lighting approach: downsample the texture of reaction coordinates to an extremely low resolution and use every voxel as a light source [S2, unofficial]. Unlike smoke rendering which blends values during marching, fire rendering accumulates values based on the reaction coordinate [S2, unofficial]. The low-resolution voxel lighting approach is less prone to flickering but does not capture high-frequency lighting effects such as local lighting due to sparks [S2, unofficial].

## Operating characteristics
*   **Performance:** Real-time (60+ fps) on an NVIDIA RTX 2060 graphics card [S1, unofficial].
*   **Complexity:** Linear with respect to the number of steps along the ray within the volume [S1, unofficial].
*   **Strengths:** GPU-friendly, real-time performance, realistic appearance with ember effects [S1, unofficial].
*   **Limitations:** Severe flickering if treating volume samples as point light sources without enough samples [S2, unofficial]. May not capture high-frequency lighting effects with low-resolution voxel lighting [S2, unofficial]. Debugging is difficult due to HLSL limitations [S1, unofficial].

## Background
Originated from a project by CS184 students, implementing a fire simulation and rendering pipeline [S1, unofficial]. The rendering technique borrows from existing literature on volumetric clouds [S1, unofficial]. The fire rendering approach is similar to smoke rendering but uses reaction coordinate accumulation [S2, unofficial]. A more physically based discussion of fire rendering can be found in Nguyen et al. 2002 [S2, unofficial].

## Sources
[S1] Fire Simulation and Rendering - GitHub Pages, https://cs184-firesim.github.io/final-report/, fetched 2026-10-27 (unofficial)
[S2] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
