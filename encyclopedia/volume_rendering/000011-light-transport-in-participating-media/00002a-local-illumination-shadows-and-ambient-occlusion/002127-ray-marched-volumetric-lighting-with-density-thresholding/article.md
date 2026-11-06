# Ray-marched Volumetric Lighting with Density Thresholding

Also called: Light Intensity Volume, Volumetric Light Map Precomputation

## Overview
This method precomputes light intensities and shadows within a 3D volume based on a light source's direction, color, decay, and a density mask, storing the result in a 3D texture for use during rendering [S1, unofficial]. It differs from its sibling [Lighting Models](2056-lighting-models.md) by precomputing the entire light transport field rather than evaluating it per-pixel, and from [Two-Pass Shadow Casting (Kajiya and Von Herzen)](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md) by using a single-pass GPU compute shader that ray-marches from each voxel towards the light direction to accumulate density and apply exponential decay [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `light_density_texture`: The output 3D volume texture (format `R8`, dimensions `light_map_resolution`).
    *   `noise_texture_3D`: A 3D texture containing noise density values.
    *   `mask_texture_3D`: A 3D texture acting as a density mask (dimensions `mask_texture_dims`).
    *   `global_light_direction`: Normalized `float3` vector.
    *   `global_light_color`: `float3` color of the light.
    *   `light_decay`: Float controlling attenuation rate.
    *   `god_rays_intensity`: Float for god rays effect.
    *   `light_sample_count`: Integer determining step size.
    *   `density_threshold`: Float threshold for filtering noise.
*   **Output:**
    *   `light_density_texture`: A 3D texture where each voxel stores the accumulated light intensity.

**Data Structures:**
*   `light_map_3D`: `RWTexture3D<float>` (or `RenderTexture` with `R8` format). Dimensions: `light_map_resolution.x` $\times$ `light_map_resolution.y` $\times$ `light_map_resolution.z`. Wrap mode: `Clamp`. Filter: `Bilinear`.
*   `noise_texture_3D`: `Texture3D<float>`.
*   `mask_texture_3D`: `Texture3D<float>`. Dimensions: `mask_texture_dims`.

**Parameters:**
*   `step_size`: Calculated as $1.0 / \text{light\_sample\_count}$.
*   `scaled_step_size`: $\text{light\_texture\_dims.x} \times \text{step\_size}$.
*   `max_iteration`: Fixed at 256.

**Algorithm Steps:**
1.  **Initialization:** For each voxel index `id` in the `light_map_3D` volume:
    *   Set `current_coordinates` = `id`.
    *   Initialize `density` = 0.0f, `density_so_far` = 0.0f, `light_intensity` = 1.0f.
    *   Calculate `normalized_light_direction` = normalize(`global_light_direction`).
    *   Calculate `scaled_step_size` = `light_texture_dims.x` $\times$ `step_size`.
    *   Set `max_iteration` = 256.
2.  **Ray Marching Loop:** While `!is_frag_out_boundries` and `max_iteration` $\ge$ 0:
    *   **Sample Density:**
        *   `noise_density` = `noise_texture_3D[light_id_to_noise_id(current_coordinates)]`.
        *   `mask` = `mask_texture_3D[light_id_to_mask_id(current_coordinates)]`.
    *   **Apply Mask and Threshold:**
        *   `density` = min(1.0f, mask - abs(`density_threshold`)) $\times$ `noise_density`.
    *   **Accumulate Density:**
        *   `density_so_far` += `density`.
    *   **God Rays Logic:**
        *   If `density` < `density_threshold` AND `density_so_far` < 0.5:
            *   `density` = `god_rays_intensity`.
        *   Else If `density` < `density_threshold`:
            *   `density` = 0.0f.
    *   **Update Light Intensity:**
        *   `light_intensity` *= exp(-`density` $\times$ `light_decay` $\times$ `step_size`).
    *   **Advance Ray:**
        *   `current_coordinates` += -`normalized_light_direction` $\times$ `scaled_step_size`.
    *   **Boundary Check:**
        *   `is_frag_out_boundries` = (`current_coordinates.x` $\ge$ `light_texture_dims.x`) OR (`current_coordinates.y` $\ge$ `light_texture_dims.y`) OR (`current_coordinates.z` $\ge$ `light_texture_dims.z`) OR (`current_coordinates.x` < 0) OR (`current_coordinates.y` < 0) OR (`current_coordinates.z` < 0).
    *   Decrement `max_iteration`.
3.  **Write Result:**
    *   `light_density_texture[id]` = `light_intensity`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Kernel: compute_light_intensities
# Thread ID: id (3D index)
# Global Params: light_texture_dims, global_light_direction, light_decay, god_rays_intensity, step_size, density_threshold
# Textures: noise_texture_3D, mask_texture_3D, light_density_texture

def light_id_to_noise_id(coord):
    # Mapping logic from source; assumed to be identity or simple scaling
    return coord

def light_id_to_mask_id(coord):
    # Mapping logic from source; assumed to be identity or simple scaling
    return coord

def main(id):
    # 1. Initialize
    current_coordinates = id
    density = 0.0f
    density_so_far = 0.0f
    light_intensity = 1.0f
    
    normalized_light_direction = normalize(global_light_direction)
    scaled_step_size = light_texture_dims.x * step_size
    max_iteration = 256
    
    # Check initial boundary
    is_frag_out_boundries = (current_coordinates.x >= light_texture_dims.x or
                             current_coordinates.y >= light_texture_dims.y or
                             current_coordinates.z >= light_texture_dims.z or
                             current_coordinates.x < 0 or
                             current_coordinates.y < 0 or
                             current_coordinates.z < 0)
    
    # 2. Loop
    while not is_frag_out_boundries and max_iteration >= 0:
        # Sample
        noise_density = noise_texture_3D[light_id_to_noise_id(current_coordinates)]
        mask = mask_texture_3D[light_id_to_mask_id(current_coordinates)]
        
        # Apply mask and threshold
        density = min(1.0f, mask - abs(density_threshold)) * noise_density
        
        # Accumulate
        density_so_far += density
        
        # God rays logic
        if density < density_threshold and density_so_far < 0.5:
            density = god_rays_intensity
        elif density < density_threshold:
            density = 0.0f
            
        # Update intensity
        light_intensity *= exp(-density * light_decay * step_size)
        
        # Advance
        current_coordinates += -normalized_light_direction * scaled_step_size
        
        # Boundary check
        is_frag_out_boundries = (current_coordinates.x >= light_texture_dims.x or
                                 current_coordinates.y >= light_texture_dims.y or
                                 current_coordinates.z >= light_texture_dims.z or
                                 current_coordinates.x < 0 or
                                 current_coordinates.y < 0 or
                                 current_coordinates.z < 0)
        
        max_iteration -= 1
        
    # 3. Write
    light_density_texture[id] = light_intensity
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Boundary checking is performed at the start of the kernel and after each step. If the fragment is out of bounds initially, it returns early (implicitly by not writing or writing 0 if initialized, though the source implies early exit logic) [S1, unofficial].
*   **Precision:** The use of `R8` format for the output texture may limit precision for light intensity values [S1, unofficial].
*   **Thresholding:** The `density_threshold` parameter is critical for filtering noise; densities below this value are treated as transparent unless god rays conditions are met [S1, unofficial].

## Subtopics

### [Pseudo Volume Textures with Per-Voxel Lighting](7063-pseudo-volume-textures-with-per-voxel-lighting.md)
This method implements direct volume rendering using 2D textures containing tiles that represent low-resolution 2D slices of the volume, referred to as 'pseudo volume textures'. It solves the problem of rendering dynamic volumetric data by allowing the volume to be "painted" at runtime. It differs from true 3D volume textures by restricting effective maximum volume resolution further, but distinguishes itself from its parent [Ray-marched Volumetric Lighting with Density Thresholding](2456-ray-marched-volumetric-lighting-with-density-thresholding.md) by recomputing lighting for each frame via secondary ray marches for every voxel sample.

## Operating characteristics
*   **Time Complexity:** $O(V \cdot S)$, where $V$ is the number of voxels in the light map volume and $S$ is the number of samples per ray (`light_sample_count` or `max_iteration`).
*   **Memory Complexity:** $O(V)$ for the output light map texture, plus input textures.
*   **Performance:** The method is implemented as a GPU compute shader, allowing parallel processing of voxels. Performance depends on `light_map_resolution` and `light_sample_count` [S1, unofficial].
*   **Strengths:** Precomputes lighting for efficient runtime rendering; handles volumetric shadows and attenuation.
*   **Limitations:** Static light direction (requires recomputation if light moves); resolution limited by texture size; `R8` format may cause banding.
*   **When to Use:** Static or slowly moving light sources in volumetric scenes where precomputation is feasible.
*   **When Not to Use:** Dynamic light sources requiring real-time updates without precomputation; high-precision lighting requirements.

## Background
This method originates from the project `VolumetricSmoke` by furkanunsal42, which implements a GPU-based volumetric smoke simulation and lighting system [S1, unofficial]. It is part of a broader effort to simulate realistic volumetric effects in real-time graphics.

## Sources
[S1] furkanunsal42/VolumetricSmoke, https://github.com/furkanunsal42/VolumetricSmoke, fetched 2026-10-26 (unofficial)
