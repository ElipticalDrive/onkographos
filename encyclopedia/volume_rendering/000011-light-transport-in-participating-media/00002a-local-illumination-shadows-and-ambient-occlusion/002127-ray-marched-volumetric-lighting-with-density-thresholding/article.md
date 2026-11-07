# Ray-marched Volumetric Lighting with Density Thresholding

Also called: Light Intensity Volume, Volumetric Light Map Precomputation

## Overview
This method precomputes light intensities and shadows within a 3D volume based on a light source direction, color, decay, and a density mask, storing the result in a 3D texture for use during rendering [S1, unofficial]. It differs from its sibling [Lighting Models](2056-lighting-models.md) by explicitly simulating light attenuation through a volumetric medium via ray-marching rather than using static shading approximations, and from [Two-Pass Shadow Casting (Kajiya and Von Herzen)](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md) by operating as a single-pass precomputation step that generates a lookup volume rather than dynamic shadow maps.

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `light_density_texture`: The input volume containing density values (e.g., from a fluid simulation or noise field).
    *   `mask_texture_3D`: A binary or grayscale mask defining where light computation is active.
    *   `noise_texture_3D`: A 3D texture providing noise density values for the ray-march.
    *   `global_light_direction`: A normalized 3D vector defining the light source direction.
    *   `global_light_color`: A 3D vector defining the light color.
    *   `light_decay`: A float controlling the rate of exponential light attenuation per unit density.
    *   `light_sample_count`: An integer defining the number of samples for light computation.
    *   `density_threshold`: A float threshold for filtering noise and determining transparency.
    *   `god_rays_intensity`: A float scalar for simulating god rays when density is low.
    *   `light_map_resolution`: A 3D vector defining the resolution of the output light map.
*   **Outputs:**
    *   `light_map_3D`: A 3D RenderTexture (Format `R8`, Wrap `Clamp`, Filter `Bilinear`) storing the computed light intensity for each voxel.

**Data Structures:**
The core data structure is a 3D texture (`RWTexture3D<float>` or equivalent in compute shaders) indexed by voxel coordinates $(x, y, z)$. The input density and mask textures are accessed via coordinate mapping functions `light_id_to_noise_id` and `light_id_to_mask_id`, which map the light map resolution indices to the respective input texture dimensions.

**Parameters:**
*   `step_size`: Calculated as $1.0 / \text{light\_sample\_count}$.
*   `scaled_step_size`: Calculated as $\text{light\_texture\_dims}.x \times \text{step\_size}$.
*   `max_iteration`: Fixed at 256.
*   `light_decay`: Range [0, 100].
*   `god_rays_intensity`: Range [0, 0.03].
*   `density_threshold`: Range [0, 1].

**Algorithm Steps:**
1.  **Initialization:** For each voxel `id` in the `light_map_3D` volume:
    *   Initialize `current_coordinates` to `id`.
    *   Initialize `density` to $0.0$.
    *   Initialize `density_so_far` to $0.0$.
    *   Initialize `light_intensity` to $1.0$.
    *   Calculate `normalized_light_direction` = normalize(`global_light_direction`).
    *   Calculate `scaled_step_size` = `light_texture_dims.x` * `step_size`.
    *   Set `max_iteration` = 256.
2.  **Ray Marching Loop:** While `!is_frag_out_boundries` and `max_iteration >= 0`:
    *   Decrement `max_iteration`.
    *   Sample `noise_density` = `noise_texture_3D[light_id_to_noise_id(current_coordinates)]`.
    *   Sample `mask` = `mask_texture_3D[light_id_to_mask_id(current_coordinates)]`.
    *   **Apply Mask and Threshold:**
        *   `density` = $\min(1, \text{mask} - |\text{density\_threshold}|) \times \text{noise\_density}$.
    *   **Accumulate Density:**
        *   `density_so_far` += `density`.
    *   **Apply God Rays Logic:**
        *   If `density` < `density_threshold` AND `density_so_far` < 0.5:
            *   `density` = `god_rays_intensity`.
        *   Else If `density` < `density_threshold`:
            *   `density` = 0.0.
    *   **Update Light Intensity:**
        *   `light_intensity` *= $\exp(-\text{density} \times \text{light\_decay} \times \text{step\_size})$.
    *   **Advance Ray:**
        *   `current_coordinates` += `-normalized_light_direction` * `scaled_step_size`.
    *   **Check Boundaries:**
        *   `is_frag_out_boundries` = (`current_coordinates.x` >= `light_texture_dims.x`) OR (`current_coordinates.y` >= `light_texture_dims.y`) OR (`current_coordinates.z` >= `light_texture_dims.z`) OR (`current_coordinates.x` < 0) OR (`current_coordinates.y` < 0) OR (`current_coordinates.z` < 0).
3.  **Write Result:**
    *   `light_density_texture[id]` = `light_intensity`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the compute shader kernel 'compute_light_intensities'
# Thread ID 'id' corresponds to a voxel in the output light map

def compute_light_intensities(id: int3, 
                              light_map_resolution: int3,
                              light_texture_dims: float3,
                              global_light_direction: float3,
                              light_decay: float,
                              light_sample_count: int,
                              density_threshold: float,
                              god_rays_intensity: float,
                              noise_texture_3D: Texture3D,
                              mask_texture_3D: Texture3D,
                              light_density_texture: RWTexture3D<float>):
    
    # 1. Initialization
    current_coordinates = id
    density = 0.0f
    density_so_far = 0.0f
    light_intensity = 1.0f
    
    normalized_light_direction = normalize(global_light_direction)
    step_size = 1.0f / light_sample_count
    scaled_step_size = light_texture_dims.x * step_size
    max_iteration = 256
    
    # Boundary check at start
    if (current_coordinates.x >= light_texture_dims.x or 
        current_coordinates.y >= light_texture_dims.y or 
        current_coordinates.z >= light_texture_dims.z or 
        current_coordinates.x < 0 or 
        current_coordinates.y < 0 or 
        current_coordinates.z < 0):
        light_density_texture[id] = 0.0f
        return

    # 2. Ray Marching Loop
    while (not is_out_of_bounds(current_coordinates, light_texture_dims) and max_iteration >= 0):
        max_iteration -= 1
        
        # Map coordinates to input textures
        noise_id = light_id_to_noise_id(current_coordinates)
        mask_id = light_id_to_mask_id(current_coordinates)
        
        noise_density = noise_texture_3D[noise_id]
        mask = mask_texture_3D[mask_id]
        
        # Apply mask and threshold filtering
        density = min(1.0f, mask - abs(density_threshold)) * noise_density
        
        # Accumulate density
        density_so_far += density
        
        # Apply god rays logic
        if density < density_threshold and density_so_far < 0.5f:
            density = god_rays_intensity
        elif density < density_threshold:
            density = 0.0f
            
        # Update light intensity using exponential decay
        light_intensity *= exp(-density * light_decay * step_size)
        
        # Advance ray
        current_coordinates += -normalized_light_direction * scaled_step_size
        
        # Check boundary conditions
        if is_out_of_bounds(current_coordinates, light_texture_dims):
            break
            
    # 3. Write result
    light_density_texture[id] = light_intensity

def is_out_of_bounds(coords: int3, dims: float3) -> bool:
    return (coords.x >= dims.x or coords.y >= dims.y or coords.z >= dims.z or
            coords.x < 0 or coords.y < 0 or coords.z < 0)

def light_id_to_noise_id(coords: int3) -> int3:
    # Implementation detail: maps light map resolution to noise texture resolution
    # Not given in the sources. Typically involves scaling based on texture dimensions.
    pass

def light_id_to_mask_id(coords: int3) -> int3:
    # Implementation detail: maps light map resolution to mask texture resolution
    # Not given in the sources.
    pass
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Boundary checking is performed at the start of the kernel and after each step. If the fragment is out of bounds initially, it returns early [S1, unofficial]. The boundary check logic uses `>=` for max dimensions and `< 0` for min dimensions [S1, unofficial].
*   **Precision:** The output format is `R8`, which may limit the precision of the stored light intensities [S1, unofficial].
*   **Noise Filtering:** The `density_threshold` parameter is critical for filtering noise; densities below this value are treated as transparent unless god rays conditions are met [S1, unofficial].

## Subtopics

### [Pseudo Volume Textures with Per-Voxel Lighting](7063-pseudo-volume-textures-with-per-voxel-lighting.md)
This method implements direct volume rendering using 2D textures containing tiles that represent low-resolution 2D slices of the volume, referred to as 'pseudo volume textures'. It solves the problem of rendering dynamic volumetric data by allowing the volume to be "painted" at runtime. It differs from true 3D volume textures by restricting effective maximum volume resolution further, but distinguishes itself from its parent [Ray-marched Volumetric Lighting with Density Thresholding](2456-ray-marched-volumetric-lighting-with-density-thresholding.md) by recomputing lighting for each frame via secondary ray marches for every voxel sample.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of voxels in the light map volume and $S$ is the number of samples per ray (`light_sample_count`).
*   **Memory Complexity:** $O(N)$ for the output light map texture, plus input textures for density, mask, and noise.
*   **Performance:** The method is implemented as a GPU compute shader, allowing for parallel processing of voxels. Performance depends on the resolution of the light map and the number of samples.
*   **Strengths:** Precomputes lighting for efficient runtime rendering; handles volumetric shadows and attenuation; supports god rays effects.
*   **Limitations:** Precomputation may not adapt well to dynamic light sources or moving volumes without recomputation; limited by the `R8` output format precision.
*   **When to Use:** Suitable for static or slowly changing volumetric scenes where lighting precomputation is feasible.
*   **When Not to Use:** Unsuitable for dynamic light sources or rapidly changing volumes where real-time recomputation is required.

## Background
This method originates from the work of furkanunsal42 in the `VolumetricSmoke` project [S1, unofficial]. It is a GPU-based technique designed to simulate light attenuation through a volumetric medium by ray-marching from each voxel in the light texture space towards the global light direction, accumulating density and applying Beer-Lambert-like exponential decay [S1, unofficial].

## Sources
[S1] furkanunsal42/VolumetricSmoke, https://github.com/furkanunsal42/VolumetricSmoke, fetched 2026-10-26 (unofficial)
