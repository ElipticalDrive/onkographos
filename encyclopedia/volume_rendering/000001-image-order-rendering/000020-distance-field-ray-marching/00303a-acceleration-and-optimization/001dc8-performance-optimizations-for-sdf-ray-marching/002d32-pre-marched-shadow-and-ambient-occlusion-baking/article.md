# Pre-marched Shadow and Ambient Occlusion Baking

Also called: Baked shadow ray, Pre-marched ambient occlusion

## Overview
A technique to improve visual fidelity and performance by pre-calculating shadow and ambient occlusion data into the volume texture, leveraging the static nature of the cloud volume [S1, unofficial]. It sets itself apart from real-time ray marching by trading runtime compute for a one-time or event-driven baking cost, avoiding expensive per-frame camera renders [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A static signed distance field (SDF) volume representing the cloud geometry.
*   **Output:** A 3D texture (volume texture) containing baked shadow and ambient occlusion values.

**Data Structures:**
*   **Baked Volume Texture:** A 3D texture where each voxel stores pre-computed lighting attenuation factors. The resolution is typically low, matching the base cloud volume resolution.

**Steps:**
1.  **Bake Shadow:** Pre-march a shadow ray through the volume for each voxel. This involves casting rays in the direction of the light source(s) to determine occlusion.
    *   *Detail:* Due to the low volume resolution, this can be done in real-time during the baking phase. A typical implementation uses 4 shadow samples per cloud sample [S1, unofficial].
    *   *Dynamic Time-of-Day:* Because the baking is computationally feasible in real-time (during the bake step, not runtime), the shadow data can be regenerated to allow for dynamic changes in the sun's position (time-of-day) without runtime cost [S1, unofficial].
2.  **Bake Ambient Occlusion:** Pre-march ambient occlusion rays into the volume texture. This adds shading based on local geometry density without runtime cost [S1, unofficial].
3.  **Runtime Lookup:** During rendering, retrieve the volumetric cloud shadow using a single `tex3D` lookup on the baked texture. This ensures the shadow perfectly matches the cloud geometry [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Baking Phase (Offline or Real-time Bake)
def bake_shadow_and_ao(sdf_volume, light_direction, resolution):
    """
    Pre-compute shadow and ambient occlusion for a static SDF volume.
    """
    # Initialize output textures
    shadow_texture = create_3d_texture(resolution)
    ao_texture = create_3d_texture(resolution)
    
    # Iterate over all voxels in the volume
    for x in range(resolution.x):
        for y in range(resolution.y):
            for z in range(resolution.z):
                # Get position in world space
                pos = voxel_to_world(x, y, z, resolution)
                
                # --- Bake Shadow ---
                # Cast shadow ray in light direction
                # Use 4 samples per cloud sample as per source [S1, unofficial]
                shadow_factor = 1.0
                num_shadow_samples = 4
                step_size = get_sdf_step(pos, sdf_volume) # From parent method
                
                for i in range(num_shadow_samples):
                    sample_pos = pos + light_direction * (i + 1) * step_size
                    # Check if sample is inside cloud (SDF < 0)
                    if sdf_volume.sample(sample_pos) < 0:
                        shadow_factor = 0.0
                        break
                
                shadow_texture.set(x, y, z, shadow_factor)
                
                # --- Bake Ambient Occlusion ---
                # Pre-march ambient occlusion rays
                # This adds shading without runtime cost [S1, unofficial]
                ao_factor = 1.0
                # Implementation details for AO marching are not specified in source [S1, unofficial]
                # Assume a similar ray marching approach in multiple directions
                ao_factor = compute_ao(pos, sdf_volume) 
                
                ao_texture.set(x, y, z, ao_factor)
                
    return shadow_texture, ao_texture

# Runtime Phase
def render_voxel(x, y, z, shadow_texture, ao_texture):
    """
    Retrieve baked lighting data for a voxel.
    """
    # Single tex3D lookup for shadow [S1, unofficial]
    shadow = shadow_texture.sample(x, y, z)
    
    # Retrieve AO
    ao = ao_texture.sample(x, y, z)
    
    # Combine with base cloud color
    base_color = get_cloud_color(x, y, z)
    final_color = base_color * shadow * ao
    
    return final_color
```

**Parameters:**
*   **Shadow Samples per Cloud Sample:** 4 [S1, unofficial].
*   **Volume Resolution:** Low, matching the base cloud volume [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Static Assumption:** This method relies on the volume being static. If the cloud geometry changes, the baked textures must be regenerated [S1, unofficial].
*   **Memory:** Storing separate textures for shadow and AO increases memory usage, though the low resolution mitigates this [S1, unofficial].

## Subtopics

### [Occlusion Caching](14832-occlusion-caching.md)
Occlusion caching is a performance optimization for static volumetric lighting that pre-calculates shadow and ambient occlusion data into volume textures, avoiding expensive per-frame camera renders. It sets itself apart from real-time ray marching by trading runtime compute for a one-time or event-driven baking cost, leveraging the static nature of most cloud volumes and occluders.

## Operating characteristics
*   **Time Complexity:** Baking time is proportional to the number of voxels times the number of shadow/AO samples. Since volume resolution is low, baking can be done in real-time (during the bake phase) [S1, unofficial]. Runtime cost is O(1) per voxel for texture lookup.
*   **Memory Complexity:** O(N) where N is the number of voxels, for storing shadow and AO textures.
*   **Performance:** Significantly reduces runtime cost by eliminating per-frame shadow and AO ray marching [S1, unofficial].
*   **Strengths:** Perfect shadow alignment with cloud geometry, low runtime cost, supports dynamic time-of-day if baked dynamically [S1, unofficial].
*   **Limitations:** Requires static geometry, increased memory for baked textures, baking cost (though feasible in real-time for low res) [S1, unofficial].
*   **When to Use:** Static volumetric clouds where high-quality shadows and AO are desired without runtime cost.
*   **When Not to Use:** Dynamic volumes where geometry changes frequently.

## Background
Originated in the context of rendering volumetric clouds using signed distance fields, as described in a blog post by uhawkvr [S1, unofficial]. The technique leverages the low resolution of cloud volumes to make pre-marching feasible.

## Sources
[S1] Rendering volumetric clouds using signed distance fields, https://blog.uhawkvr.com/rendering/rendering-volumetric-clouds-using-signed-distance-fields/, fetched 2026-10-26 (unofficial)
