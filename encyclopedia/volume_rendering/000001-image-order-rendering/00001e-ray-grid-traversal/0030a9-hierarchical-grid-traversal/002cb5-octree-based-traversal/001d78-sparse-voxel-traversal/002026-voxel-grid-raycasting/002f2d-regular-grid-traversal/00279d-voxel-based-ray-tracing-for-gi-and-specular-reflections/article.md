# Voxel-based Ray Tracing for GI and Specular Reflections

Also called: ApplyVoxelRT, Voxel RT, VoxelDDA, VoxelReflectionFallback

## Overview
A voxel-based ray tracing system using Digital Differential Analyzer (DDA) traversal through a 128³ block-occupancy grid to provide global illumination (GI) and specular reflection fallbacks [S1, unofficial]. It serves as a drop-in replacement for Screen-Space Ray Tracing (SSRT) for GI and a fallback for Screen-Space Reflections (SSR) when coverage is incomplete, combining sky occlusion with Light Propagation Volume (LPV) colour bleeding [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 128³ block-occupancy voxel grid stored in `imgVoxelMask` (populated each frame in the shadow pass) [S1, unofficial]. LPV data for indirect lighting [S1, unofficial]. Camera position and view vectors [S1, unofficial].
*   **Output:** Indirect lighting contribution (GI) and reflected colour (Specular) [S1, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 128³ block-occupancy grid. The occupancy status is stored in `imgVoxelMask` [S1, unofficial].
*   **LPV Data:** Accessed via image load/store operations [S1, unofficial].

**Parameters:**
*   `VOXEL_GI_SAMPLES`: Number of cosine-weighted hemisphere rays for GI sampling. Default value is 4 [S1, unofficial].
*   `VOXEL_GI_STEPS`: Configurable parameter for the number of steps in voxel GI ray traversal [S1, unofficial].
*   `VOXEL_REFL_STEPS`: Configurable parameter for the number of steps in voxel reflection ray traversal [S1, unofficial].
*   `VOXEL_RT_REFLECTIONS`: Feature flag to enable specular reflection fallback. Commented out by default [S1, unofficial].
*   `VOXEL_GI`: Indirect effect mode identifier, value 6, added to the `indirect_effect` enum [S1, unofficial].

**Algorithm Steps:**

1.  **VoxelDDA():** Perform 3-D DDA traversal over the voxel grid.
    *   Handle grid bounds and sign-safe boundary distances [S1, unofficial].
    *   Include ray bias in DDA traversal to avoid self-intersection and handle near-parallel rays gracefully [S1, unofficial].
    *   Return the face-centre position and axis-aligned normal of the hit voxel [S1, unofficial].

2.  **SampleVoxelLighting():**
    *   Read the LPV data half a block in front of the hit face [S1, unofficial].
    *   Obtain the indirect colour at the surface from propagated light [S1, unofficial].

3.  **ApplyVoxelRT():**
    *   Trace `VOXEL_GI_SAMPLES` cosine-weighted hemisphere rays [S1, unofficial].
    *   Blend sky contribution with LPV colour bounce at hit surfaces [S1, unofficial].
    *   This function acts as a drop-in replacement for `ApplySSRT()` [S1, unofficial].

4.  **VoxelReflectionFallback():**
    *   Perform a single specular DDA ray trace [S1, unofficial].
    *   Activate only when screen-space reflection misses (i.e., `environmentReflection.a < 1`) [S1, unofficial].
    *   Scale voxel lighting to match typical sky-reflection brightness ranges to avoid visual mismatch [S1, unofficial].
    *   Fill in LPV-sampled voxel colour [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Voxel-based Ray Tracing
# Assumes standard GLSL-like vector operations and image load/store access

def VoxelDDA(origin, direction, grid_size=128.0):
    """
    Perform 3-D DDA traversal over the voxel grid.
    Returns: (hit_position, hit_normal, hit_distance) or None if no hit.
    """
    # Ray bias to avoid self-intersection
    ray_bias = 1e-4
    
    # Normalize direction for DDA
    inv_dir = 1.0 / direction
    step = sign(direction)
    
    # Initial voxel coordinates
    current_voxel = floor(origin * grid_size)
    
    # Calculate initial t values for crossing voxel boundaries
    # t_max_x, t_max_y, t_max_z represent the distance to the next boundary
    # t_delta_x, t_delta_y, t_delta_z represent the distance between boundaries
    t_max = (current_voxel + 0.5 + step * 0.5 - origin) * inv_dir
    t_delta = step * inv_dir
    
    # Initialize t_max with ray bias to start inside the first voxel if needed
    # Note: Exact sign-safe boundary distance logic is implementation-specific
    # but generally involves handling negative directions correctly.
    
    for i in range(1000): # Max steps to prevent infinite loops
        # Determine which axis we are crossing next
        if t_max.x < t_max.y:
            if t_max.x < t_max.z:
                # Cross X boundary
                current_voxel.x += step.x
                t_max.x += t_delta.x
                hit_normal = vec3(-step.x, 0.0, 0.0)
            else:
                # Cross Z boundary
                current_voxel.z += step.z
                t_max.z += t_delta.z
                hit_normal = vec3(0.0, 0.0, -step.z)
        else:
            if t_max.y < t_max.z:
                # Cross Y boundary
                current_voxel.y += step.y
                t_max.y += t_delta.y
                hit_normal = vec3(0.0, -step.y, 0.0)
            else:
                # Cross Z boundary
                current_voxel.z += step.z
                t_max.z += t_delta.z
                hit_normal = vec3(0.0, 0.0, -step.z)
        
        # Check bounds
        if not all(0 <= current_voxel < grid_size):
            return None
            
        # Check occupancy in imgVoxelMask
        # Note: imgVoxelMask access is hardware-dependent (GL_ARB_shader_image_load_store)
        if is_occupied(current_voxel, imgVoxelMask):
            hit_position = origin + direction * t_max.x # Approximate hit pos
            # Refine hit position to face centre
            hit_position = current_voxel + 0.5
            return (hit_position, hit_normal, length(hit_position - origin))
            
    return None

def SampleVoxelLighting(hit_position, hit_normal):
    """
    Read LPV data half a block in front of the hit face.
    """
    # Offset by half a block in the direction of the normal
    offset = hit_normal * 0.5
    sample_position = hit_position + offset
    
    # Sample LPV data at sample_position
    # Note: LPV sampling logic is specific to the LPV implementation
    lpv_colour = sample_lpv(sample_position)
    return lpv_colour

def ApplyVoxelRT(origin, normal, view_vector, sky_colour):
    """
    Drop-in replacement for ApplySSRT().
    Traces VOXEL_GI_SAMPLES cosine-weighted hemisphere rays.
    """
    num_samples = 4 # VOXEL_GI_SAMPLES default
    total_indirect = vec3(0.0)
    
    # Generate cosine-weighted hemisphere samples
    for i in range(num_samples):
        # Generate random direction on hemisphere oriented by normal
        # Note: Specific hemisphere sampling algorithm is not detailed in sources
        # [G] Standard cosine-weighted hemisphere sampling:
        # u1, u2 = random()
        # theta = 2 * PI * u1
        # phi = acos(sqrt(u2))
        # local_dir = vec3(sin(phi)*cos(theta), sin(phi)*sin(theta), cos(phi))
        # world_dir = transform_to_world(local_dir, normal)
        
        # For pseudocode completeness, assume a helper exists
        ray_dir = cosine_weighted_hemisphere_sample(normal)
        
        # Trace ray
        hit = VoxelDDA(origin, ray_dir)
        
        if hit is not None:
            hit_pos, hit_norm, dist = hit
            # Sample lighting at hit
            lpv_col = SampleVoxelLighting(hit_pos, hit_norm)
            total_indirect += lpv_col
        else:
            # No hit, use sky contribution
            total_indirect += sky_colour
            
    # Average samples
    indirect_colour = total_indirect / num_samples
    return indirect_colour

def VoxelReflectionFallback(origin, view_vector, environment_reflection):
    """
    Single specular DDA ray trace.
    Activated only when environment_reflection.a < 1.
    """
    if environment_reflection.a >= 1.0:
        return environment_reflection.rgb
        
    # Reflect view vector
    reflect_dir = reflect(-view_vector, normal_at_origin) # Normal assumed from context
    
    # Trace ray
    hit = VoxelDDA(origin, reflect_dir)
    
    if hit is not None:
        hit_pos, hit_norm, dist = hit
        # Sample lighting at hit
        lpv_col = SampleVoxelLighting(hit_pos, hit_norm)
        
        # Scale voxel lighting to match typical sky-reflection brightness
        # Note: Specific scaling factor is not detailed in sources
        # [G] Assume a scaling factor based on sky brightness or empirical value
        scaled_colour = lpv_col * 0.5 # Placeholder scaling
        return scaled_colour
    else:
        return environment_reflection.rgb

# Integration Logic (Pseudocode for Shader Entry Points)

def composite1_fsh():
    if indirect_effect == VOXEL_GI:
        # Check hardware support
        if has_lp_support() and has_image_load_store():
            voxel_gi = ApplyVoxelRT(...)
            apply_lighting(voxel_gi)
        else:
            # Fallback to SSRT
            ssrt = ApplySSRT(...)
            apply_lighting(ssrt)

def specular_glsl():
    env_reflection = get_environment_reflections()
    
    if defined(VOXEL_RT_REFLECTIONS):
        if env_reflection.a < 1.0:
            voxel_reflection = VoxelReflectionFallback(...)
            env_reflection.rgb = voxel_reflection
            
    return env_reflection
```

**Edge Cases and Pitfalls:**
*   **Hardware Support:** Requires LPV (Light Propagation Volumes) and `GL_ARB_shader_image_load_store` support. Gracefully degrades to SSRT if these are not met [S1, unofficial].
*   **Division by Zero:** Epsilon guards are required to prevent division by zero in `invTonemap()`, `FastCatmulRom()`, and TAA blending calculations, especially on hardware with aggressive FP16 FTZ behaviour (e.g., NVIDIA RTX 5000 series) [S1, unofficial].
*   **Self-Intersection:** Ray bias must be included in DDA traversal to avoid self-intersection [S1, unofficial].
*   **Near-Parallel Rays:** DDA traversal must handle near-parallel rays gracefully [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of steps (`VOXEL_GI_STEPS`, `VOXEL_REFL_STEPS`) and the number of samples (`VOXEL_GI_SAMPLES`). The DDA traversal is efficient, typically O(N) where N is the number of voxels traversed.
*   **Memory Complexity:** Depends on the size of the voxel grid (128³) and LPV data. No new buffers are needed beyond the existing `imgVoxelMask` [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales with the number of GI samples and steps. Reflection fallback is a single trace, so it is less expensive than full GI sampling.
*   **Strengths:** Provides GI and specular reflections in areas where screen-space methods fail (e.g., off-screen objects, low coverage). Drop-in replacement for SSRT.
*   **Limitations:** Requires LPV and specific GPU features. Quality depends on the resolution of the voxel grid (128³) and LPV accuracy.
*   **When to use:** When screen-space GI/Reflections are insufficient or unavailable. Useful for Minecraft-style voxel environments.
*   **When not to use:** When hardware lacks LPV or image load/store support. When performance is critical and SSRT is sufficient.

## Background
*   **Origin:** Implemented by nyzxor as a pull request to the Eclipse-Shader (an edit of Chocapic13 V9 / Bliss) for Minecraft, integrated via Iris [S1, unofficial].
*   **History:** Added in May 2026 [S1, unofficial].
*   **Usage:** Used in the Eclipse-Shader for Minecraft [S1, unofficial].

## Sources
[S1] Add voxel-based ray tracing for GI and specular reflections, https://github.com/nyzxor/Eclipse-Shader/pull/1, fetched 2026-10-25 (unofficial)
