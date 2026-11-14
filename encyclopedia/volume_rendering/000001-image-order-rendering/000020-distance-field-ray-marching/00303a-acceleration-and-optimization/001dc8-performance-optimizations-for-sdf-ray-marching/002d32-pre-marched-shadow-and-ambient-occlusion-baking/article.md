# Pre-marched Shadow and Ambient Occlusion Baking

Also called: Baked shadow ray, Pre-marched ambient occlusion

## Overview
This method improves visual fidelity and performance in static volumetric rendering by pre-calculating shadow and ambient occlusion data into the volume texture. It leverages the static nature of the cloud volume to bake directional shadow rays and ambient attenuation, allowing for dynamic lighting changes (e.g., time-of-day) without runtime ray-marching costs for shadows. It sets itself apart from runtime shadow calculation by using a single texture lookup to match cloud geometry perfectly, and from static lighting by preserving volumetric shadow depth.

## How it works
**Inputs and Outputs:**
*   **Input:** A static signed distance field (SDF) volume texture representing the cloud geometry. The volume resolution is typically low.
*   **Output:** An augmented volume texture containing the SDF values, pre-marched shadow data, and pre-marched ambient occlusion data.

**Data Structures:**
The volume texture stores multiple channels per voxel. While the exact bit-packing is not specified, it typically includes:
1.  SDF value (distance to surface).
2.  Shadow factor (accumulated shadow from a directional light source).
3.  Ambient occlusion factor (accumulated occlusion from surrounding geometry).

**Steps:**

1.  **Initialize Volume:** Load or generate the base SDF volume texture for the static cloud geometry.
2.  **Pre-march Shadow Ray:**
    *   For each voxel in the volume, cast a shadow ray in the direction of the light source.
    *   The ray marches through the volume using the SDF for step sizes.
    *   Accumulate shadow contribution. The source notes a ratio of "4 shadow samples per cloud sample," implying a lower resolution or coarser sampling for the shadow bake compared to the final render, or a specific subsampling strategy.
    *   Store the accumulated shadow factor in the shadow channel of the volume texture.
    *   *Note:* Because the volume resolution is low, this pre-marching can be done in real-time during initialization or offline.
3.  **Pre-march Ambient Occlusion Ray:**
    *   For each voxel, calculate ambient occlusion by sampling the surrounding volume.
    *   This involves marching rays in various directions (or using a hemisphere sample) to determine how much of the sky is visible.
    *   Store the ambient occlusion factor in the AO channel of the volume texture.
4.  **Runtime Rendering:**
    *   During rendering, for each pixel, perform the standard SDF ray march to find the cloud surface.
    *   Perform a single `tex3D` lookup at the hit position to retrieve the pre-baked shadow and ambient occlusion values.
    *   Apply these values to the lighting calculation. This allows for dynamic time-of-day changes by simply updating the light direction used in the bake (if baked dynamically) or by using the baked data as a base multiplier.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bake_shadow_and_ao(volume_sdf, volume_resolution, light_direction, shadow_samples_per_cloud_sample=4):
    """
    Pre-marches shadow and ambient occlusion into the volume texture.
    
    Args:
        volume_sdf: 3D array representing the SDF of the cloud.
        volume_resolution: Dimensions of the volume.
        light_direction: Normalized vector for the light source.
        shadow_samples_per_cloud_sample: Ratio of shadow samples to cloud samples.
        
    Returns:
        augmented_volume: 3D array with SDF, shadow, and AO channels.
    """
    # Initialize augmented volume with SDF
    # Assuming channels: [SDF, Shadow, AO]
    augmented_volume = np.zeros((volume_resolution, volume_resolution, volume_resolution, 3))
    augmented_volume[:, :, :, 0] = volume_sdf
    
    # Pre-march Shadow
    # Iterate over all voxels
    for x in range(volume_resolution):
        for y in range(volume_resolution):
            for z in range(volume_resolution):
                # Get starting position in world space
                pos = voxel_to_world(x, y, z)
                
                # March shadow ray
                shadow_accum = 0.0
                current_pos = pos
                # Determine step size based on shadow_samples_per_cloud_sample
                # This implies a coarser march or specific sampling pattern
                step_size = calculate_step_size(volume_resolution, shadow_samples_per_cloud_sample)
                
                # March until out of bounds or fully shadowed
                while is_inside_volume(current_pos):
                    # Sample SDF at current position
                    sdf_val = volume_sdf[world_to_voxel(current_pos)]
                    
                    # If inside cloud (SDF < 0), accumulate shadow
                    if sdf_val < 0:
                        shadow_accum += 1.0 # Simplified accumulation
                    
                    current_pos += light_direction * step_size
                
                # Normalize or clamp shadow value
                shadow_accum = clamp(shadow_accum, 0.0, 1.0)
                augmented_volume[x, y, z, 1] = shadow_accum

    # Pre-march Ambient Occlusion
    # Iterate over all voxels again
    for x in range(volume_resolution):
        for y in range(volume_resolution):
            for z in range(volume_resolution):
                pos = voxel_to_world(x, y, z)
                
                # Sample surrounding voxels for AO
                # This is a simplified approximation; real AO might use hemisphere sampling
                ao_accum = 0.0
                sample_count = 0
                
                # Example: Sample neighbors in a small radius
                for dx in [-1, 0, 1]:
                    for dy in [-1, 0, 1]:
                        for dz in [-1, 0, 1]:
                            if dx == 0 and dy == 0 and dz == 0:
                                continue
                            neighbor_pos = pos + vec3(dx, dy, dz) * step_size
                            if is_inside_volume(neighbor_pos):
                                neighbor_sdf = volume_sdf[world_to_voxel(neighbor_pos)]
                                if neighbor_sdf < 0:
                                    ao_accum += 1.0
                                sample_count += 1
                
                # Calculate AO factor (0 = fully occluded, 1 = fully open)
                if sample_count > 0:
                    ao_factor = 1.0 - (ao_accum / sample_count)
                else:
                    ao_factor = 1.0
                
                augmented_volume[x, y, z, 2] = clamp(ao_factor, 0.0, 1.0)

    return augmented_volume

def render_cloud_with_baked_data(ray_origin, ray_direction, baked_volume):
    """
    Renders a cloud using pre-baked shadow and AO.
    """
    # Standard SDF ray march
    hit_pos = march_sdf(ray_origin, ray_direction, baked_volume)
    
    if hit_pos is None:
        return background_color
    
    # Single texture lookup for baked data
    shadow = baked_volume[world_to_voxel(hit_pos), 1]
    ao = baked_volume[world_to_voxel(hit_pos), 2]
    
    # Lighting calculation
    diffuse_light = calculate_diffuse(light_direction, normal) * shadow
    ambient_light = calculate_ambient() * ao
    
    return diffuse_light + ambient_light
```

**Parameters:**
*   `shadow_samples_per_cloud_sample`: Typically 4, as noted in the source. This controls the resolution of the shadow bake relative to the cloud geometry.
*   `volume_resolution`: Low resolution is key to making the pre-marching feasible in real-time or offline.

**Edge Cases and Pitfalls:**
*   **Static Geometry:** This method only works for static volumes. Dynamic clouds require real-time shadow calculation.
*   **Light Direction:** If the light direction changes significantly, the baked shadows may become inaccurate. The source notes that real-time pre-marching allows for dynamic time-of-day, implying the bake is updated when the light direction changes.
*   **Precision:** Low volume resolution may lead to aliasing or blocky shadows.

## Subtopics

### [Occlusion Caching](14832-occlusion-caching.md)
A related technique for caching occlusion data.

## Operating characteristics
*   **Time Complexity:** Baking is $O(N)$ where $N$ is the number of voxels, multiplied by the number of samples per voxel for shadow and AO. Runtime is $O(1)$ per pixel for shadow/AO lookup, dominated by the SDF ray march.
*   **Memory Complexity:** $O(N)$ for the augmented volume texture.
*   **Performance:** Significantly reduces runtime cost for shadows and AO. The source notes that shadow baking can be done in real-time due to low volume resolution.
*   **Strengths:** Perfect volumetric shadow matching, dynamic lighting support (if baked dynamically), reduced runtime cost.
*   **Limitations:** Only for static geometry, memory overhead for augmented texture, potential aliasing at low resolutions.

## Background
This technique is described in the context of rendering volumetric clouds using signed distance fields. It is presented as an optimization for static cloud volumes.

## Sources
[S1] Rendering volumetric clouds using signed distance fields, https://blog.uhawkvr.com/rendering/rendering-volumetric-clouds-using-signed-distance-fields/, fetched 2026-10-26 (unofficial)
