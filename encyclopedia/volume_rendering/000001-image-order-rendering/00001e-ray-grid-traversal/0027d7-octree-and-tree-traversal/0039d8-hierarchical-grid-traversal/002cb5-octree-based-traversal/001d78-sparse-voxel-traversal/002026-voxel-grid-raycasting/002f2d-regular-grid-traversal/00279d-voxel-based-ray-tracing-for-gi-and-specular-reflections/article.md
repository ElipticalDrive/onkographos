# Voxel-based Ray Tracing for GI and Specular Reflections

Also called: ApplyVoxelRT, Voxel RT, VoxelDDA, VoxelReflectionFallback

## Overview
A voxel-based ray tracing system that uses 3-D Digital Differential Analyzer (3D-DDA) traversal through a 128³ block-occupancy grid to provide global illumination (GI) and specular reflection fallbacks [S1, unofficial]. It serves as a drop-in replacement for Screen-Space Ray Tracing (SSRT) for GI and a fallback for Screen-Space Reflections (SSR) when coverage is incomplete [S1, unofficial]. Unlike [Regular Grid Traversal](5939-regular-grid-traversal.md) which typically renders volumetric density, this method samples a pre-computed Light Propagation Volume (LPV) buffer at ray hit positions to compute indirect lighting and reflection color [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin ($O$), ray direction ($D$), current surface position, LPV texture (indirect lighting data), voxel mask grid (`imgVoxelMask`).
*   **Output:** Indirect lighting color (for GI) or reflected color (for specular fallback).

**Data Structures:**
*   **Voxel Grid:** A 128³ block-occupancy grid stored in `imgVoxelMask` [S1, unofficial]. Populated each frame in the shadow pass [S1, unofficial].
*   **LPV Buffer:** Light Propagation Volumes data, accessed via image load/store [S1, unofficial].

**Parameters:**
*   `VOXEL_GI_SAMPLES`: Number of cosine-weighted hemisphere rays for GI sampling. Default: 4 [S1, unofficial].
*   `VOXEL_GI_STEPS`: Configurable number of steps in voxel GI ray traversal [S1, unofficial].
*   `VOXEL_REFL_STEPS`: Configurable number of steps in voxel reflection ray traversal [S1, unofficial].
*   `VOXEL_RT_REFLECTIONS`: Feature flag to enable specular reflection fallback. Commented out by default [S1, unofficial].
*   `VOXEL_GI`: Indirect effect mode identifier, value 6 [S1, unofficial].

**Algorithm Steps:**

1.  **VoxelDDA Traversal:** Perform 3-D DDA traversal over the voxel grid.
    *   Handle grid bounds and sign-safe boundary distances [S1, unofficial].
    *   Include ray bias to avoid self-intersection and handle near-parallel rays gracefully [S1, unofficial].
    *   Return the face-centre position and the axis-aligned normal of the hit voxel [S1, unofficial].

2.  **SampleVoxelLighting:**
    *   Read the LPV data half a block in front of the hit face [S1, unofficial].
    *   This obtains the indirect color at the surface from propagated light [S1, unofficial].

3.  **ApplyVoxelRT (GI Mode):**
    *   Trace `VOXEL_GI_SAMPLES` cosine-weighted hemisphere rays from the surface [S1, unofficial].
    *   For each ray, use `VoxelDDA` to find a hit.
    *   If a hit occurs, use `SampleVoxelLighting` to get the LPV color.
    *   Blend the sky contribution with the LPV color bounce at the hit surfaces [S1, unofficial].

4.  **VoxelReflectionFallback (Specular Mode):**
    *   Perform a single specular DDA ray trace [S1, unofficial].
    *   Activated only when screen-space reflection misses (indicated by `environmentReflection.a < 1`) [S1, unofficial].
    *   Scale the voxel lighting to match typical sky-reflection brightness ranges to avoid visual mismatch [S1, unofficial].
    *   Fill in the LPV-sampled voxel color [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
VOXEL_GRID_SIZE = 128
VOXEL_GI_SAMPLES = 4
VOXEL_GI_STEPS = 16 # Example default, configurable
VOXEL_REFL_STEPS = 16 # Example default, configurable
RAY_BIAS = 1e-4 # Small epsilon for self-intersection avoidance

def VoxelDDA(origin, direction, max_steps):
    """
    Performs 3D DDA traversal on the voxel grid.
    Returns (hit_position, hit_normal) or (None, None) if no hit.
    """
    # Map origin to initial voxel coordinates
    voxel_x = floor(origin.x * VOXEL_GRID_SIZE)
    voxel_y = floor(origin.y * VOXEL_GRID_SIZE)
    voxel_z = floor(origin.z * VOXEL_GRID_SIZE)
    
    # Check if origin is inside the grid bounds
    if not (0 <= voxel_x < VOXEL_GRID_SIZE and 
            0 <= voxel_y < VOXEL_GRID_SIZE and 
            0 <= voxel_z < VOXEL_GRID_SIZE):
        return None, None

    # Calculate tMax and tDelta for each axis
    # Note: Exact implementation depends on grid spacing and ray direction sign
    # tDeltaX = dx / |dir_x|, etc.
    # tMaxX = (boundary_x - origin_x) / dir_x
    # Adjust tMax by RAY_BIAS to avoid self-intersection
    
    for step in range(max_steps):
        # Check occupancy of current voxel
        if is_voxel_occupied(voxel_x, voxel_y, voxel_z):
            # Calculate hit position on the face
            # This requires tracking the specific tMax that caused the step
            hit_pos = origin + direction * current_t_max
            hit_normal = get_axis_normal(voxel_x, voxel_y, voxel_z, current_t_max)
            return hit_pos, hit_normal
        
        # Advance to nearest boundary
        # Determine which axis has the smallest tMax
        if tMaxX < tMaxY and tMaxX < tMaxZ:
            voxel_x += step_x
            tMaxX += tDeltaX
        elif tMaxY < tMaxZ:
            voxel_y += step_y
            tMaxY += tDeltaY
        else:
            voxel_z += step_z
            tMaxZ += tDeltaZ
            
        # Check bounds
        if not (0 <= voxel_x < VOXEL_GRID_SIZE and 
                0 <= voxel_y < VOXEL_GRID_SIZE and 
                0 <= voxel_z < VOXEL_GRID_SIZE):
            break
            
    return None, None

def SampleVoxelLighting(hit_position, hit_normal):
    """
    Reads LPV data half a block in front of the hit face.
    """
    # Offset position by half a voxel in the direction of the normal
    voxel_size = 1.0 / VOXEL_GRID_SIZE
    sample_pos = hit_position + hit_normal * (voxel_size * 0.5)
    
    # Sample LPV texture at sample_pos
    # LPV coordinates are typically normalized [0,1]
    lpv_uv = sample_pos # Assuming LPV covers the same world space
    indirect_color = texture(LPV_texture, lpv_uv).rgb
    return indirect_color

def ApplyVoxelRT(surface_position, surface_normal, view_direction):
    """
    Computes GI using cosine-weighted hemisphere sampling.
    """
    total_indirect = vec3(0.0)
    
    # Generate cosine-weighted hemisphere samples
    for i in range(VOXEL_GI_SAMPLES):
        sample_dir = cosine_weighted_hemisphere_sample(surface_normal)
        
        # Trace ray
        ray_origin = surface_position + surface_normal * RAY_BIAS
        hit_pos, hit_normal = VoxelDDA(ray_origin, sample_dir, VOXEL_GI_STEPS)
        
        if hit_pos is not None:
            # Get lighting from LPV at hit point
            lpv_color = SampleVoxelLighting(hit_pos, hit_normal)
            total_indirect += lpv_color
        else:
            # Sky contribution if ray escapes grid
            # Not fully specified in sources, assumed to be sky color or ambient
            total_indirect += sky_color 
            
    # Average samples
    return total_indirect / VOXEL_GI_SAMPLES

def VoxelReflectionFallback(reflection_direction, surface_position):
    """
    Fallback for SSR when coverage is incomplete.
    """
    ray_origin = surface_position + reflection_direction * RAY_BIAS
    hit_pos, hit_normal = VoxelDDA(ray_origin, reflection_direction, VOXEL_REFL_STEPS)
    
    if hit_pos is not None:
        lpv_color = SampleVoxelLighting(hit_pos, hit_normal)
        # Scale to match sky-reflection brightness
        scaled_color = lpv_color * brightness_scale_factor
        return scaled_color
    else:
        return vec3(0.0)
```

**Edge Cases and Pitfalls:**
*   **Hardware Support:** Requires LPV and `GL_ARB_shader_image_load_store` support. Gracefully degrades to SSRT if these are not met [S1, unofficial].
*   **Precision:** Division-by-zero/NaN sources in TAA resolve pass can produce black pixel artifacts, especially on hardware with aggressive FP16 FTZ behavior (e.g., NVIDIA RTX 5000 series). Epsilon guards are required in `invTonemap()` and other functions [S1, unofficial].
*   **Self-Intersection:** Ray bias is necessary to avoid self-intersection artifacts [S1, unofficial].
*   **Near-Parallel Rays:** DDA traversal must handle near-parallel rays gracefully to avoid infinite loops or incorrect hits [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Proportional to `VOXEL_GI_SAMPLES` * `VOXEL_GI_STEPS` for GI, and `VOXEL_REFL_STEPS` for reflections. The DDA traversal is efficient, O(N) where N is the number of steps.
*   **Memory Complexity:** Depends on the LPV buffer size and the voxel mask grid (128³). No additional large buffers are needed as it reuses existing shadow pass data [S1, unofficial].
*   **Strengths:** Provides global illumination and specular reflections in scenes where screen-space methods fail (e.g., occluded areas, off-screen objects). Drop-in replacement for SSRT [S1, unofficial].
*   **Limitations:** Relies on the accuracy of the LPV propagation. Lower resolution grid (128³) may result in blocky lighting. Requires specific hardware features (`GL_ARB_shader_image_load_store`) [S1, unofficial].
*   **When to Use:** When screen-space GI/Reflections are insufficient or unavailable. Useful for Minecraft-style voxel worlds where LPV is already computed [S1, unofficial].
*   **When Not to Use:** On hardware lacking required OpenGL extensions. If LPV is not enabled or populated.

## Background
This method was implemented by nyzxor as a pull request to the Eclipse-Shader (an edit of Chocapic13 V9 / Bliss) for Minecraft, integrated via Iris [S1, unofficial]. It was merged into the Unstable branch on May 22, 2026 [S1, unofficial]. The implementation aims to provide an alternative global illumination method and a fallback for specular reflections when screen-space ray tracing misses [S1, unofficial].

## Sources
[S1] Add voxel-based ray tracing for GI and specular reflections, https://github.com/nyzxor/Eclipse-Shader/pull/1, fetched 2026-10-25 (unofficial)
