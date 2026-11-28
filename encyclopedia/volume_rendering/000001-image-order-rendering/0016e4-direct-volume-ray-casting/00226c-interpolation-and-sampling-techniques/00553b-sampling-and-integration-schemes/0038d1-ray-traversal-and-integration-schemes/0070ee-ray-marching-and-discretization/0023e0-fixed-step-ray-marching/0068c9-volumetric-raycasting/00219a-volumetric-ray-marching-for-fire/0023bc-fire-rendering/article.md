# Fire Rendering

Also called: Fire Rendering via Reaction Coordinate, Fire Volume Rendering

## Overview
Fire rendering is a direct volume ray casting technique that renders fire by accumulating values determined by a reaction coordinate $Y$ rather than smoke density, using an artist-defined 1D texture to map these coordinates to fire colors [S1, unofficial]. Unlike smoke rendering, which blends values during the marching process, fire rendering accumulates these reaction-based values to simulate the emissive nature of flames [S1, unofficial].

## How it works
**Inputs and Outputs:** The method takes a 3D volume texture storing the scalar reaction coordinate $Y$ per voxel, an artist-defined 1D texture mapping $Y$ to RGBA fire colors, and camera parameters for ray generation [S1, unofficial][G]. It outputs a 2D image of the rendered fire, with optional lighting contributions for other scene objects [G].

**Data Structures:** The reaction coordinate volume is a 3D texture where each voxel stores a scalar value $Y$, with layout and resolution determined by the simulation source [S1, unofficial][G]. The color map is a 1D texture or array mapping $Y \in [0, 1]$ (or the simulation's range) to RGBA fire colors [S1, unofficial][G].

**Algorithm Steps:**
1. **Ray Generation:** For each pixel, generate a ray from the camera origin through the pixel's position in world space [G].
2. **Volume Intersection:** Compute the entry and exit distances ($t_{min}$, $t_{max}$) where the ray intersects the axis-aligned bounding box of the fire volume using the slab method [G].
3. **Ray Marching:** Step along the ray within the volume bounds $[t_{min}, t_{max}]$ [S1, unofficial]. At each step, sample the reaction coordinate $Y$ from the 3D volume texture [S1, unofficial]. Sample the color $C$ from the 1D color map using $Y$ as the index [S1, unofficial]. Accumulate the color additively, as fire rendering integrates emissive values rather than blending transmissive ones [S1, unofficial].
4. **Lighting (Optional):** To use the fire volume as a light source, sample the volume at several locations and treat each sample as a point light source, using $Y$ and the 1D color texture to determine intensity and color [S1, unofficial]. Alternatively, downsample the reaction coordinate texture to an extremely low resolution and use every voxel as a light source [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_fire(ray_origin, ray_direction, volume_min, volume_max, color_map, num_samples):
    # Ray-AABB intersection (slab method)
    t_min = -float('inf')
    t_max = float('inf')
    for i in range(3):
        if ray_direction[i] == 0.0:
            if ray_origin[i] < volume_min[i] or ray_origin[i] > volume_max[i]:
                return vec4(0.0, 0.0, 0.0, 0.0)
        else:
            t1 = (volume_min[i] - ray_origin[i]) / ray_direction[i]
            t2 = (volume_max[i] - ray_origin[i]) / ray_direction[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
    if t_max < 0.0 or t_min > t_max:
        return vec4(0.0, 0.0, 0.0, 0.0)

    step_size = (t_max - t_min) / num_samples
    accumulated_color = vec4(0.0, 0.0, 0.0, 0.0)

    for i in range(num_samples):
        t = t_min + (i + 0.5) * step_size
        position = ray_origin + t * ray_direction
        Y = sample_3d(volume_texture, position) # [G] Standard 3D texture lookup
        color = color_map.sample(Y) # [G] Standard 1D texture lookup
        accumulated_color += color

    return accumulated_color
```

**Parameters:** The reaction coordinate $Y$ is the primary variable driving the visual appearance and indexes into the 1D color texture [S1, unofficial]. The number of samples controls the smoothness of the ray marching, and using too few samples in the point light approach can lead to severe flickering [S1, unofficial].

**Edge Cases and Pitfalls:** Treating volume samples as point light sources can lead to severe flickering if not enough point samples are used [S1, unofficial]. The low-resolution voxel lighting approach is less prone to flickering but does not capture high-frequency lighting effects such as local lighting due to sparks [S1, unofficial]. Precision issues are not given in the sources [G].

## Operating characteristics
The time complexity is linear with respect to the number of samples per ray and the number of rays (pixels) [G]. Lighting calculations add complexity based on the number of light sources generated [S1, unofficial]. The memory complexity depends on the resolution of the 3D reaction coordinate volume and the 1D color map [G]. Strengths include simple implementation, artistic control via the 1D color texture, and the ability to function as a light source [S1, unofficial][G]. Limitations include potential flickering in the point light approach and loss of high-frequency details in the voxel light approach [S1, unofficial]. It should be used when a stylized or artist-controlled fire appearance is desired, or when the fire volume is already available as a simulation output [G]. It should not be used when physically accurate fire lighting with high-frequency details is required without significant performance cost [G].

## Background
Fire rendering is discussed in the context of real-time fluid simulation and rendering [S1, unofficial]. It is similar to smoke rendering but differs in the accumulation method and the use of the reaction coordinate [S1, unofficial]. A more physically based discussion of fire rendering can be found in Nguyen et al. 2002 [S1, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
