# Ray-marched Volumetric Lighting with Density Thresholding

Also called: Light Intensity Volume, Volumetric Light Map Precomputation

## Overview
This method precomputes light intensities and shadows within a 3D volume based on a light source's direction, color, decay, and a density mask, storing the result in a 3D texture for use during rendering [S1, unofficial]. It simulates light attenuation through a volumetric medium by ray-marching from each voxel in the light texture space towards the global light direction, accumulating density and applying Beer-Lambert-like exponential decay [S1, unofficial]. It differs from its sibling [Two-Pass Shadow Casting (Kajiya and Von Herzen)](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md) by precomputing the lighting into a static volume rather than casting dynamic shadow rays per pixel, and from [Ambient Light Approximation](2244-ambient-light-approximation.md) by accounting for directional attenuation and density-based occlusion rather than simple scalar occlusion factors.

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   `light_density_texture`: A 3D volume texture containing the density field of the participating medium.
    *   `noise_texture_3D`: A 3D texture containing noise values used to modulate density.
    *   `mask_texture_3D`: A 3D texture acting as a density mask.
    *   `global_light_direction`: A normalized 3D vector defining the direction of the light source.
    *   `global_light_color`: A 3D vector defining the color of the light.
    *   `light_decay`: A float controlling the rate of exponential light attenuation per unit density.
    *   `god_rays_intensity`: A float scalar for simulating god rays.
    *   `light_sample_count`: An integer defining the number of samples for the ray march.
    *   `density_threshold`: A float threshold for filtering noise and determining transparency.
    *   `light_map_resolution`: A 3D vector defining the resolution of the output light map.
*   **Outputs:**
    *   `light_map_3D`: A 3D `RenderTexture` (format `R8`, wrap `Clamp`, filter `Bilinear`) storing the computed light intensity for each voxel.

**Data Structures**
The core data structure is the 3D texture volume. The output `light_map_3D` has dimensions `light_map_resolution.x` $\times$ `light_map_resolution.y` $\times$ `light_map_resolution.z`. The input density and mask textures are accessed via indices mapped from the light map coordinates.

**Parameters**
*   `light_decay`: Range $[0, 100]$.
*   `god_rays_intensity`: Range $[0, 0.03]$.
*   `light_sample_count`: Range $[0, 128]$. Used to calculate `step_size` as $1.0 / \text{light\_sample\_count}$.
*   `density_threshold`: Range $[0, 1]$.
*   `step_size`: Calculated as $1.0 / \text{light\_sample\_count}$.
*   `scaled_step_size`: Calculated as $\text{light\_texture\_dims.x} \times \text{step\_size}$.

**Algorithm Steps**
1.  **Initialization:** For each voxel index `id` in the output `light_map_3D`:
    *   Set `current_coordinates` to `id`.
    *   Initialize `density = 0.0f`, `density_so_far = 0.0f`, and `light_intensity = 1.0f`.
    *   Calculate `normalized_light_direction = normalize(global_light_direction)`.
    *   Calculate `scaled_step_size = light_texture_dims.x * step_size`.
    *   Set `max_iteration = 256`.
    *   Check boundary conditions: If `current_coordinates` is out of bounds (x $\ge$ `light_texture_dims.x` or y $\ge$ `light_texture_dims.y` or z $\ge$ `light_texture_dims.z` or x < 0 or y < 0 or z < 0), write `light_intensity` (1.0) to `light_density_texture[id]` and terminate for this voxel.
2.  **Ray Marching Loop:** While `max_iteration > 0` and the ray is within bounds:
    *   Decrement `max_iteration`.
    *   Map `current_coordinates` to noise and mask indices using helper functions `light_id_to_noise_id` and `light_id_to_mask_id`.
    *   Sample `noise_density = noise_texture_3D[light_id_to_noise_id(current_coordinates)]`.
    *   Sample `mask = mask_texture_3D[light_id_to_mask_id(current_coordinates)]`.
    *   **Apply Mask and Threshold:**
        *   Calculate `density = min(1.0f, mask - abs(density_threshold)) * noise_density`.
    *   **Accumulate Density:**
        *   `density_so_far += density`.
    *   **Apply God Rays Logic:**
        *   If `density < density_threshold` AND `density_so_far < 0.5`:
            *   `density = god_rays_intensity`.
        *   Else if `density < density_threshold`:
            *   `density = 0.0f`.
    *   **Update Light Intensity:**
        *   Apply exponential decay: `light_intensity *= exp(-density * light_decay * step_size)`.
    *   **Advance Ray:**
        *   `current_coordinates += -normalized_light_direction * scaled_step_size`.
    *   **Check Boundary Conditions:**
        *   Determine `is_frag_out_boundries`: True if `current_coordinates.x >= light_texture_dims.x` or `current_coordinates.y >= light_texture_dims.y` or `current_coordinates.z >= light_texture_dims.z` or `current_coordinates.x < 0` or `current_coordinates.y < 0` or `current_coordinates.z < 0`.
3.  **Write Result:** Write the final `light_intensity` to `light_density_texture[id]`.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the compute kernel 'compute_light_intensities'
# Executed in parallel for each voxel id in light_map_3D

def compute_light_intensities(id: int3):
    # 1. Initialization
    current_coordinates = id
    density = 0.0f
    density_so_far = 0.0f
    light_intensity = 1.0f
    
    normalized_light_direction = normalize(global_light_direction)
    scaled_step_size = light_texture_dims.x * step_size
    max_iteration = 256
    
    # Boundary check at start
    if is_out_of_bounds(current_coordinates, light_texture_dims):
        light_density_texture[id] = light_intensity
        return

    # 2. Ray Marching Loop
    while max_iteration > 0 and not is_out_of_bounds(current_coordinates, light_texture_dims):
        max_iteration -= 1
        
        # Map coordinates to noise and mask textures
        noise_idx = light_id_to_noise_id(current_coordinates)
        mask_idx = light_id_to_mask_id(current_coordinates)
        
        # Sample textures
        noise_density = noise_texture_3D[noise_idx]
        mask = mask_texture_3D[mask_idx]
        
        # Apply mask and threshold filtering
        # Note: Source uses abs(density_threshold) in subtraction
        density = min(1.0f, mask - abs(density_threshold)) * noise_density
        
        # Accumulate density
        density_so_far += density
        
        # Apply god rays logic
        if density < density_threshold and density_so_far < 0.5f:
            density = god_rays_intensity
        elif density < density_threshold:
            density = 0.0f
            
        # Update light intensity using exponential decay
        # Formula: I_new = I_old * exp(-density * decay * step)
        light_intensity *= exp(-density * light_decay * step_size)
        
        # Advance ray
        current_coordinates += -normalized_light_direction * scaled_step_size

    # 3. Write result
    light_density_texture[id] = light_intensity

def is_out_of_bounds(coords: int3, dims: int3) -> bool:
    return (coords.x >= dims.x or coords.y >= dims.y or coords.z >= dims.z or
            coords.x < 0 or coords.y < 0 or coords.z < 0)
```

**Edge Cases and Pitfalls**
*   **Boundary Checking:** Boundary checks are performed at the start of the kernel and after each step. If the fragment is out of bounds initially, it returns early with the initial intensity (1.0) [S1, unofficial].
*   **Threshold Logic:** The threshold logic treats densities below `density_threshold` as transparent unless god rays conditions are met. This can lead to artifacts if `density_threshold` is set too high or too low relative to the noise density range [S1, unofficial].
*   **Precision:** The output format is `R8`, which may limit the precision of the stored light intensities, potentially causing banding in smooth gradients [S1, unofficial].

## Subtopics

### [Pseudo Volume Textures with Per-Voxel Lighting](7063-pseudo-volume-textures-with-per-voxel-lighting.md)
This method implements direct volume rendering using 2D textures containing tiles that represent low-resolution 2D slices of the volume, referred to as 'pseudo volume textures'. It solves the problem of rendering dynamic volumetric data by allowing the volume to be "painted" at runtime. It differs from true 3D volume textures by restricting effective maximum volume resolution further, but distinguishes itself from its parent [Ray-marched Volumetric Lighting with Density Thresholding](2456-ray-marched-volumetric-lighting-with-density-thresholding.md) by recomputing lighting for each frame via secondary ray marches for every voxel sample.

## Operating characteristics
*   **Time Complexity:** $O(V \cdot S)$, where $V$ is the number of voxels in the light map volume and $S$ is the number of ray march steps (`light_sample_count`). The maximum iteration count is capped at 256.
*   **Memory Complexity:** $O(V)$ for the output light map texture. Input textures (density, noise, mask) also require memory proportional to their resolution.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Precomputes lighting, allowing for real-time rendering of volumetric shadows and attenuation without per-pixel ray marching during the main render pass.
*   **Limitations:** Static lighting direction; changing the light direction requires recomputation. Resolution is limited by the 3D texture size and memory constraints. The `R8` format limits precision.
*   **When to use:** When the light direction is static or changes infrequently, and performance is critical for dynamic volumetric effects.
*   **When not to use:** When the light direction changes frequently, or when high-precision lighting is required.

## Background
This method is derived from a GPU-based implementation for volumetric smoke rendering, specifically utilizing compute shaders to precompute light intensities [S1, unofficial]. It is part of a broader effort to optimize volumetric rendering by separating the lighting calculation from the final shading pass.

## Sources
[S1] furkanunsal42/VolumetricSmoke, https://github.com/furkanunsal42/VolumetricSmoke, fetched 2026-10-26 (unofficial)
