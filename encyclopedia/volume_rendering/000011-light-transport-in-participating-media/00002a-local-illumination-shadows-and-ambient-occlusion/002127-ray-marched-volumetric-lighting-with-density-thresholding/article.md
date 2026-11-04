# Ray-marched Volumetric Lighting with Density Thresholding

Also called: Light Intensity Volume, Volumetric Light Map Precomputation

## Overview
This method precomputes light intensities and shadows within a 3D volume based on a light source direction, color, decay, and a density mask, storing the result in a 3D texture for use during rendering [S1, unofficial]. It differs from real-time ray marching by performing the light transport calculation offline (or in a dedicated compute pass) to decouple lighting cost from scene complexity, using density thresholding to filter noise and simulate god rays [S1, unofficial].

## How it works
**Inputs:**
*   `light_map_resolution`: Vector3 defining the resolution of the output 3D light map volume.
*   `light_density_texture`: Input 3D texture containing the scene density.
*   `mask_texture_3D`: Input 3D texture acting as a density mask.
*   `noise_texture_3D`: Input 3D texture containing noise values (e.g., Worley noise) used to modulate density.
*   `global_light_direction`: Normalized Vector3 defining the direction of the light source.
*   `global_light_color`: Vector3 defining the color of the light.
*   `light_decay`: Float controlling the rate of exponential light attenuation per unit density. Range [0, 100] [S1, unofficial].
*   `god_rays_intensity`: Float scalar applied to density values below the threshold when accumulated density is low to simulate god rays. Range [0, 0.03] [S1, unofficial].
*   `light_sample_count`: Integer, number of samples for light computation. Range [0, 128] [S1, unofficial].
*   `density_threshold`: Float, density threshold for light computation. Range [0, 1] [S1, unofficial].

**Outputs:**
*   `light_map_3D`: A 3D RenderTexture (Format `R8`, Wrap `Clamp`, Filter `Bilinear`) storing the final light intensity for each voxel.

**Data Structures:**
The core data structure is the `light_map_3D` volume. Each voxel stores a single `R8` (8-bit red) value representing the accumulated light intensity [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:**
    *   Create `light_map_3D` with dimensions `light_map_resolution`.
    *   Calculate `step_size` as $1.0 / \text{light\_sample\_count}$.
    *   Calculate `scaled_step_size` as $\text{light\_texture\_dims}.x \times \text{step\_size}$.
    *   Set `max_iteration` to 256.
    *   Dispatch the compute kernel `compute_light_intensities` with thread groups sized $\lceil \text{light\_map\_resolution}.x / 8 \rceil$, $\lceil \text{light\_map\_resolution}.y / 8 \rceil$, $\lceil \text{light\_map\_resolution}.z / 8 \rceil$.

2.  **Per-Voxel Ray Marching (Compute Kernel):**
    For each voxel identified by `id` (current coordinates):
    *   Initialize `density_so_far = 0.0f`, `light_intensity = 1.0f`.
    *   Normalize `global_light_direction`.
    *   Loop while `max_iteration > 0` and the ray is within bounds:
        1.  **Sample Density:**
            *   Map current coordinates to noise space: `noise_id = light_id_to_noise_id(current_coordinates)`.
            *   Map current coordinates to mask space: `mask_id = light_id_to_mask_id(current_coordinates)`.
            *   `noise_density = noise_texture_3D[noise_id]`.
            *   `mask_value = mask_texture_3D[mask_id]`.
        2.  **Apply Mask and Threshold:**
            *   `density = min(1.0f, mask_value - abs(density_threshold)) * noise_density`.
            *   `density_so_far += density`.
        3.  **God Rays Logic:**
            *   If `density < density_threshold` AND `density_so_far < 0.5`:
                *   `density = god_rays_intensity`.
            *   Else if `density < density_threshold`:
                *   `density = 0.0f`.
        4.  **Attenuation:**
            *   Update light intensity using Beer-Lambert-like exponential decay:
                $$ \text{light\_intensity} = \text{light\_intensity} \times \exp(-\text{density} \times \text{light\_decay} \times \text{step\_size}) $$
        5.  **Advance Ray:**
            *   `current_coordinates += -global_light_direction * scaled_step_size`.
            *   Check boundary conditions: `is_out_of_bounds = current_coordinates.x >= light_texture_dims.x || current_coordinates.x < 0 || ... || current_coordinates.z < 0`.
            *   Decrement `max_iteration`.
    *   **Write Result:**
        *   `light_map_3D[id] = light_intensity`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the compute kernel 'compute_light_intensities'
# Assumes access to global variables: light_map_resolution, light_texture_dims, 
# global_light_direction, light_decay, god_rays_intensity, step_size, density_threshold,
# noise_texture_3D, mask_texture_3D, light_map_3D (RWTexture3D)

def compute_light_intensities(id: int3):
    # 1. Initialize loop variables
    current_coordinates = id
    density_so_far = 0.0
    light_intensity = 1.0
    max_iteration = 256
    
    # Pre-calculate constants
    normalized_light_direction = normalize(global_light_direction)
    scaled_step_size = light_texture_dims.x * step_size
    
    # 2. Boundary check at start
    if is_out_of_bounds(current_coordinates, light_texture_dims):
        return

    # 3. Ray marching loop
    while max_iteration > 0:
        # Sample textures
        noise_id = light_id_to_noise_id(current_coordinates)
        mask_id = light_id_to_mask_id(current_coordinates)
        
        noise_density = noise_texture_3D[noise_id]
        mask_value = mask_texture_3D[mask_id]
        
        # Apply mask and threshold filtering
        # Note: Source uses 'min(1, mask - abs(density_threashold)) * (density)'
        # Assuming 'density' in source refers to noise_density
        effective_density = min(1.0, mask_value - abs(density_threshold)) * noise_density
        density_so_far += effective_density
        
        # Apply god rays logic
        if effective_density < density_threshold and density_so_far < 0.5:
            effective_density = god_rays_intensity
        elif effective_density < density_threshold:
            effective_density = 0.0
            
        # Update light intensity using exponential decay
        # Formula: light_intensity *= exp(-density * light_decay * step_size)
        light_intensity *= exp(-effective_density * light_decay * step_size)
        
        # Advance ray
        current_coordinates += -normalized_light_direction * scaled_step_size
        
        # Check boundary conditions
        if is_out_of_bounds(current_coordinates, light_texture_dims):
            break
            
        max_iteration -= 1
        
    # 4. Write result
    light_map_3D[id] = light_intensity

def is_out_of_bounds(coords: int3, dims: int3) -> bool:
    return (coords.x >= dims.x or coords.x < 0 or
            coords.y >= dims.y or coords.y < 0 or
            coords.z >= dims.z or coords.z < 0)
```

## Operating characteristics
*   **Time Complexity:** $O(V \times S)$, where $V$ is the number of voxels in the light map volume and $S$ is the number of ray march steps (`light_sample_count` or `max_iteration`).
*   **Memory Complexity:** $O(V)$ for the output 3D texture. Input textures also consume memory.
*   **Performance:** Performance depends heavily on `light_map_resolution` and `light_sample_count`. The method is executed on the GPU via compute shaders.
*   **Strengths:** Precomputes complex volumetric lighting and shadows, allowing real-time rendering to simply sample the 3D texture. Efficient for static scenes or scenes with static lighting.
*   **Limitations:** Requires precomputation time. Cannot handle dynamic lights or dynamic geometry changes without recomputation. Resolution is limited by the 3D texture size.
*   **When to use:** Static volumetric scenes where lighting does not change frequently.
*   **When not to use:** Dynamic scenes with moving lights or geometry.

## Background
This method is described in the context of a GPU-based volumetric smoke simulation project [S1, unofficial]. It utilizes a compute shader technique to simulate light attenuation through a volumetric medium. The approach combines ray marching with density thresholding to filter noise and enhance visual quality (god rays).

## Sources
[S1] furkanunsal42/VolumetricSmoke, https://github.com/furkanunsal42/VolumetricSmoke, fetched 2026-10-26 (unofficial)
