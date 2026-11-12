# Voxel-based Ray Tracing for GI and Specular Reflections

Also called: ApplyVoxelRT, Voxel RT, VoxelDDA, VoxelReflectionFallback

## Overview
Voxel-based Ray Tracing is a global illumination (GI) and specular reflection fallback method that uses Digital Differential Analyzer (3D-DDA) traversal through a uniform block-occupancy voxel grid to sample indirect lighting and reflections [S1, unofficial]. It serves as a drop-in replacement for Screen-Space Ray Tracing (SSRT) for GI and a fallback for Screen-Space Reflections (SSR) when coverage is incomplete, combining sky occlusion with Light Propagation Volumes (LPV) colour bleeding [S1, unofficial]. Unlike sibling methods that may rely on screen-space data alone, this method leverages a pre-computed 3D voxel grid populated in the shadow pass to provide scene-wide illumination data [S1, unofficial].

## How it works
The method operates on a uniform 3D voxel grid representing the scene's Axis-Aligned Bounding Box (AABB). It consists of four core functions: `VoxelDDA` for traversal, `SampleVoxelLighting` for data retrieval, `ApplyVoxelRT` for GI, and `VoxelReflectionFallback` for specular reflections.

### Inputs and Outputs
*   **Inputs**:
    *   `ray_origin`: World-space ray origin (feet-relative).
    *   `ray_direction`: Normalized world-space ray direction.
    *   `voxel_grid`: A 128³ block-occupancy grid stored in `imgVoxelMask` [S1, unofficial].
    *   `lpv_data`: Light Propagation Volumes texture containing indirect colour data [S1, unofficial].
    *   `sky_color`: Sky contribution colour [S1, unofficial].
    *   `VOXEL_GI_SAMPLES`: Number of hemisphere rays (default 4) [S1, unofficial].
    *   `VOXEL_GI_STEPS`: Max steps for GI traversal [S1, unofficial].
    *   `VOXEL_REFL_STEPS`: Max steps for reflection traversal [S1, unofficial].
*   **Outputs**:
    *   `indirect_color`: Blended indirect lighting colour (GI).
    *   `reflection_color`: Specular reflection colour (Fallback).

### Data Structures
*   **Voxel Grid**: A 128³ grid of occupancy masks. Each cell indicates whether a block exists at that location. Populated each frame in the shadow pass [S1, unofficial].
*   **LPV Data**: Texture storing propagated light information. Accessed at coordinates relative to the hit face [S1, unofficial].

### Algorithm Steps

1.  **VoxelDDA Traversal**:
    Perform 3D-DDA traversal through the voxel grid.
    *   Calculate distances to the next cell boundary along each axis ($t_x, t_y, t_z$).
    *   Advance the coordinate with the smallest distance.
    *   Handle grid bounds and sign-safe boundary distances [S1, unofficial].
    *   Include a **ray bias** to avoid self-intersection and handle near-parallel rays gracefully [S1, unofficial].
    *   Return the face-centre position and the axis-aligned normal of the hit face [S1, unofficial].
    *   Stop when the ray exits the grid bounds or reaches the step limit.

2.  **SampleVoxelLighting**:
    *   Given the hit face position from `VoxelDDA`, sample the LPV data.
    *   Offset the sampling coordinate by half a block in front of the hit face to obtain the indirect colour at the surface from propagated light [S1, unofficial].

3.  **ApplyVoxelRT (GI)**:
    *   Trace `VOXEL_GI_SAMPLES` (default 4) cosine-weighted hemisphere rays from the surface point [S1, unofficial].
    *   For each ray, call `VoxelDDA` to find a hit.
    *   If a hit is found, call `SampleVoxelLighting` to get the LPV colour bounce.
    *   Blend the sky contribution with the LPV colour bounce at hit surfaces [S1, unofficial].
    *   Return the accumulated indirect colour.

4.  **VoxelReflectionFallback (Specular)**:
    *   Activated only when screen-space reflection misses (indicated by `environmentReflection.a < 1`) [S1, unofficial].
    *   Perform a single specular DDA ray trace using `VoxelDDA` with `VOXEL_REFL_STEPS` [S1, unofficial].
    *   If a hit is found, sample the voxel colour using `SampleVoxelLighting`.
    *   Scale the voxel lighting to match typical sky-reflection brightness ranges to avoid visual mismatch [S1, unofficial].
    *   Return the scaled colour.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
VOXEL_GRID_SIZE = 128
VOXEL_GI_SAMPLES_DEFAULT = 4
VOXEL_GI_STEPS_DEFAULT = 64 # Example default, configurable
VOXEL_REFL_STEPS_DEFAULT = 64 # Example default, configurable

def VoxelDDA(ray_origin, ray_direction, max_steps):
    """
    Perform 3D-DDA traversal over the voxel grid.
    Returns (hit_position, hit_normal) or None if no hit.
    """
    # Initialize DDA variables
    # Calculate initial cell coordinates
    current_cell = floor(ray_origin / voxel_size)
    
    # Calculate step directions
    step_x = 1 if ray_direction.x > 0 else -1
    step_y = 1 if ray_direction.y > 0 else -1
    step_z = 1 if ray_direction.z > 0 else -1
    
    # Calculate distances to next boundary
    t_delta_x = abs(voxel_size / ray_direction.x)
    t_delta_y = abs(voxel_size / ray_direction.y)
    t_delta_z = abs(voxel_size / ray_direction.z)
    
    # Initial t_max values
    if ray_direction.x > 0:
        t_max_x = (voxel_size - (ray_origin.x - current_cell.x * voxel_size)) / ray_direction.x
    else:
        t_max_x = (current_cell.x * voxel_size - ray_origin.x) / ray_direction.x
        
    if ray_direction.y > 0:
        t_max_y = (voxel_size - (ray_origin.y - current_cell.y * voxel_size)) / ray_direction.y
    else:
        t_max_y = (current_cell.y * voxel_size - ray_origin.y) / ray_direction.y
        
    if ray_direction.z > 0:
        t_max_z = (voxel_size - (ray_origin.z - current_cell.z * voxel_size)) / ray_direction.z
    else:
        t_max_z = (current_cell.z * voxel_size - ray_origin.z) / ray_direction.z
        
    # Ray bias to avoid self-intersection [S1, unofficial]
    epsilon = 1e-4
    t_max_x += epsilon
    t_max_y += epsilon
    t_max_z += epsilon
    
    for step in range(max_steps):
        # Check grid bounds
        if not is_inside_grid(current_cell):
            return None
            
        # Check occupancy
        if is_occupied(current_cell):
            # Calculate hit position (face centre)
            hit_pos = ray_origin + ray_direction * min(t_max_x, t_max_y, t_max_z)
            
            # Determine normal based on which axis hit first
            if t_max_x < t_max_y and t_max_x < t_max_z:
                normal = vec3(-step_x, 0, 0)
            elif t_max_y < t_max_x and t_max_y < t_max_z:
                normal = vec3(0, -step_y, 0)
            else:
                normal = vec3(0, 0, -step_z)
                
            return (hit_pos, normal)
            
        # Advance DDA
        if t_max_x < t_max_y:
            if t_max_x < t_max_z:
                current_cell.x += step_x
                t_max_x += t_delta_x
            else:
                current_cell.z += step_z
                t_max_z += t_delta_z
        else:
            if t_max_y < t_max_z:
                current_cell.y += step_y
                t_max_y += t_delta_y
            else:
                current_cell.z += step_z
                t_max_z += t_delta_z
                
    return None

def SampleVoxelLighting(hit_position, hit_normal):
    """
    Read LPV data half a block in front of the hit face.
    """
    # Offset by half a block in the direction of the normal
    sample_pos = hit_position + hit_normal * (voxel_size * 0.5)
    
    # Sample LPV texture
    lpv_color = texture(lpv_texture, sample_pos)
    return lpv_color

def ApplyVoxelRT(surface_position, surface_normal, sky_color):
    """
    Drop-in replacement for ApplySSRT.
    Traces hemisphere rays for GI.
    """
    total_indirect = vec3(0.0)
    num_samples = VOXEL_GI_SAMPLES_DEFAULT
    
    for i in range(num_samples):
        # Generate cosine-weighted hemisphere ray
        ray_dir = cosine_weighted_hemisphere_sample(surface_normal)
        
        # Trace ray
        hit = VoxelDDA(surface_position, ray_dir, VOXEL_GI_STEPS_DEFAULT)
        
        if hit is not None:
            hit_pos, hit_normal = hit
            lpv_color = SampleVoxelLighting(hit_pos, hit_normal)
            total_indirect += lpv_color
        else:
            # No hit, use sky contribution
            total_indirect += sky_color
            
    # Average samples
    return total_indirect / num_samples

def VoxelReflectionFallback(view_ray_origin, view_ray_direction, environment_reflection):
    """
    Specular reflection fallback when SSR misses.
    """
    # Check if SSR missed
    if environment_reflection.a >= 1.0:
        return environment_reflection.rgb
        
    # Perform single specular DDA trace
    hit = VoxelDDA(view_ray_origin, view_ray_direction, VOXEL_REFL_STEPS_DEFAULT)
    
    if hit is not None:
        hit_pos, hit_normal = hit
        voxel_color = SampleVoxelLighting(hit_pos, hit_normal)
        
        # Scale to match sky-reflection brightness
        scaled_color = voxel_color * brightness_scale_factor
        return scaled_color
    else:
        return environment_reflection.rgb
```

### Parameters
*   `VOXEL_GI_SAMPLES`: Number of cosine-weighted hemisphere rays for GI sampling. Default: 4 [S1, unofficial].
*   `VOXEL_GI_STEPS`: Configurable parameter for the number of steps in voxel GI ray traversal [S1, unofficial].
*   `VOXEL_REFL_STEPS`: Configurable parameter for the number of steps in voxel reflection ray traversal [S1, unofficial].
*   `VOXEL_RT_REFLECTIONS`: Feature flag to enable specular reflection fallback. Commented out by default [S1, unofficial].
*   `VOXEL_GI`: Indirect effect mode identifier, value 6, added to the `indirect_effect` enum [S1, unofficial].

### Edge Cases and Pitfalls
*   **Hardware Requirements**: Requires LPV (Light Propagation Volumes) and `GL_ARB_shader_image_load_store` support. Gracefully degrades to SSRT if these are not met [S1, unofficial].
*   **Precision Issues**: Division-by-zero or NaN sources in TAA resolve pass (e.g., `invTonemap()`) can produce black pixel artifacts, especially on hardware with aggressive FP16 FTZ behaviour (e.g., NVIDIA RTX 5000 series). Epsilon guards are required to prevent division by zero [S1, unofficial].
*   **Self-Intersection**: Ray bias must be included in DDA traversal to avoid self-intersection and handle near-parallel rays gracefully [S1, unofficial].
*   **Visual Mismatch**: Reflection fallback must scale voxel lighting to match typical sky-reflection brightness ranges to avoid visual mismatch [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: O(N) where N is the number of steps per ray. For GI, this is multiplied by `VOXEL_GI_SAMPLES`. For reflections, it is a single trace.
*   **Memory Complexity**: O(1) additional memory per pixel, as it relies on the pre-computed voxel grid and LPV texture.
*   **Performance**: Depends on grid resolution (128³) and step limits. The method is efficient as it avoids complex scene traversal, relying on the uniform grid structure.
*   **Strengths**: Provides scene-wide GI and reflections independent of camera position or screen-space coverage. Drop-in replacement for SSRT.
*   **Limitations**: Requires LPV and specific OpenGL extensions. Lower resolution than path tracing. Artifacts may occur if LPV data is not properly propagated.
*   **When to Use**: When screen-space methods (SSRT/SSR) fail due to occlusion or incomplete coverage, and LPV is available.
*   **When Not to Use**: When hardware does not support required extensions or LPV is not enabled.

## Background
The method was implemented by nyzxor as a pull request to the Eclipse-Shader (an edit of Chocapic13 V9 / Bliss) for Minecraft, integrated via Iris [S1, unofficial]. It was merged into the Unstable branch on May 22, 2026 [S1, unofficial]. The implementation builds on the principles of 3D-DDA traversal shared by the Regular Grid Traversal family [4091-voxel-based-ray-tracing-for-gi-and-specular-reflections, 4240-optimised-voxel-ray-marching].

## Sources
[S1] Add voxel-based ray tracing for GI and specular reflections, https://github.com/nyzxor/Eclipse-Shader/pull/1, fetched 2026-10-25 (unofficial)
