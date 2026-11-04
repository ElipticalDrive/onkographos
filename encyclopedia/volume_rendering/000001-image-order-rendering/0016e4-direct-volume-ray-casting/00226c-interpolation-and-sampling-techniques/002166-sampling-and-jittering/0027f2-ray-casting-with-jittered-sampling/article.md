# Ray Casting with Jittered Sampling

Also called: Jittered Sampling Volume Rendering

## Overview
Ray Casting with Jittered Sampling is a direct volume rendering technique that casts rays through a 3D voxel grid to accumulate color and opacity, using jittered sampling offsets to reduce aliasing artifacts [S1, unofficial]. It distinguishes itself from uniform ray sampling by perturbing sample positions to break up regular grid patterns, and from stochastic methods by using a deterministic, pre-computed jitter texture rather than runtime random number generation [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   3D Voxel Data Texture: Stores voxel colors/opacities [S1, unofficial].
*   Camera parameters: Projection matrix, ModelView matrix, Camera position [S1, unofficial].
*   Volume parameters: Bounding box (min/max), translation, scale [S1, unofficial].
*   Rendering parameters: Number of samples per ray, ray step size, lighting flag [S1, unofficial].
*   Jitter Texture: 2D texture storing pre-computed jitter offsets [S1, unofficial].

**Outputs:**
*   Color buffer: Accumulated RGBA color per pixel [S1, unofficial].

### Data Structures

**Voxel Data Texture (`textureIDs[0]`)**
*   **Target:** `GL_TEXTURE_3D` [S1, unofficial].
*   **Format:**
    *   `GL_RGBA32F` for floating-point data [S1, unofficial].
    *   `GL_RGBA8` for unsigned byte data [S1, unofficial].
    *   `GL_COMPRESSED_RGBA_S3TC_DXT5_EXT` if uncompressed size > 1GB [S1, unofficial].
*   **Content:** Voxel colors and opacities [S1, unofficial].

**Jitter Texture (`textureIDs[1]`)**
*   **Target:** `GL_TEXTURE_2D` [S1, unofficial].
*   **Size:** 32x32 pixels [S1, unofficial].
*   **Content:** 2D jitter offset vectors [S1, unofficial].

**Geometry**
*   **Structure:** Bounding box composed of 6 faces (front, back, left, right, top, bottom) [S1, unofficial].
*   **Primitive:** `GL_QUADS` [S1, unofficial].
*   **Vertices:** 4 vertices per face, defined by bounding box corners ($x_{min/max}, y_{min/max}, z_{min/max}$) [S1, unofficial].

### Parameters

*   **$N_{samples}$ (m_numSamples):** Number of samples per ray.
    *   If voxel count is 0: $N_{samples} = \sqrt{dim_X^2 + dim_Y^2 + dim_Z^2} + 2$ [S1, unofficial].
    *   Clamped to minimum 32 [S1, unofficial].
*   **$JitterTexSize$:** Resolution of jitter texture. Fixed at 32x32 [S1, unofficial].
*   **$RayStepSize$:** Distance between samples along the ray.
    *   Formula: $RayStepSize = \frac{\sqrt{3.0}}{N_{samples}}$ [S1, unofficial].
    *   Normalizes step size relative to the diagonal of the unit cube [S1, unofficial].
*   **$VolTranslation$:** Bounding box center [S1, unofficial].
*   **$VolScale$:** Bounding box dimensions [S1, unofficial].
*   **$VolExtentMin/Max$:** Bounding box min/max coordinates [S1, unofficial].

### Algorithm Steps

1.  **Coordinate Transformation Setup:**
    *   Define transformation to normalize volume bounding box to a unit cube centered at the origin (or [0,1] range depending on implementation, note: source says "centered at origin" but formula implies [0,1] range after +0.5).
    *   Transform $VolExtentMin$, $VolExtentMax$, $CameraPosition$, and $LightPosition$ into this local space using:
        $$coord_{local} = \frac{coord_{world} - VolTranslation}{VolScale} + 0.5$$
        [S1, unofficial]

2.  **Near Plane Pass:**
    *   Draw a near plane quad using a separate shader program (`s_pCameraNearPlaneProg`) [S1, unofficial].
    *   Pass transformation matrices and parameters [S1, unofficial].

3.  **Stencil Test Setup:**
    *   Enable stencil test [S1, unofficial].
    *   Set function to `GL_EQUAL` with reference value 0 [S1, unofficial].
    *   Set operation on pass to `GL_INCR` [S1, unofficial].
    *   *Purpose:* Ensures only one pixel modifies the color buffer per fragment [S1, unofficial].

4.  **Blending Setup:**
    *   Enable alpha blending [S1, unofficial].
    *   Set blend factors to `GL_ONE` and `GL_ONE_MINUS_SRC_ALPHA` [S1, unofficial].

5.  **Ray Casting Shader Execution:**
    *   For each pixel, cast a ray through the volume [S1, unofficial].
    *   Sample the 3D voxel texture along the ray path [S1, unofficial].
    *   Apply jitter offsets from the 2D jitter texture to sample positions to reduce aliasing [S1, unofficial].
    *   Accumulate color and opacity along the ray [S1, unofficial].
    *   Optionally compute lighting if `ComputeLighting` flag is set [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader / Geometry Setup
def setup_geometry(bounding_box_min, bounding_box_max):
    """
    Creates a bounding box geometry with 6 faces (quads).
    """
    vertices = []
    # Define 6 faces: front, back, left, right, top, bottom
    # Each face has 4 vertices
    faces = [
        # Front (z = max)
        [(bounding_box_min.x, bounding_box_min.y, bounding_box_max.z),
         (bounding_box_max.x, bounding_box_min.y, bounding_box_max.z),
         (bounding_box_max.x, bounding_box_max.y, bounding_box_max.z),
         (bounding_box_min.x, bounding_box_max.y, bounding_box_max.z)],
        # Back (z = min)
        [(bounding_box_min.x, bounding_box_min.y, bounding_box_min.z),
         (bounding_box_max.x, bounding_box_min.y, bounding_box_min.z),
         (bounding_box_max.x, bounding_box_max.y, bounding_box_min.z),
         (bounding_box_min.x, bounding_box_max.y, bounding_box_min.z)],
        # Left (x = min)
        [(bounding_box_min.x, bounding_box_min.y, bounding_box_min.z),
         (bounding_box_min.x, bounding_box_max.y, bounding_box_min.z),
         (bounding_box_min.x, bounding_box_max.y, bounding_box_max.z),
         (bounding_box_min.x, bounding_box_min.y, bounding_box_max.z)],
        # Right (x = max)
        [(bounding_box_max.x, bounding_box_min.y, bounding_box_min.z),
         (bounding_box_max.x, bounding_box_max.y, bounding_box_min.z),
         (bounding_box_max.x, bounding_box_max.y, bounding_box_max.z),
         (bounding_box_max.x, bounding_box_min.y, bounding_box_max.z)],
        # Top (y = max)
        [(bounding_box_min.x, bounding_box_max.y, bounding_box_min.z),
         (bounding_box_max.x, bounding_box_max.y, bounding_box_min.z),
         (bounding_box_max.x, bounding_box_max.y, bounding_box_max.z),
         (bounding_box_min.x, bounding_box_max.y, bounding_box_max.z)],
        # Bottom (y = min)
        [(bounding_box_min.x, bounding_box_min.y, bounding_box_min.z),
         (bounding_box_max.x, bounding_box_min.y, bounding_box_min.z),
         (bounding_box_max.x, bounding_box_min.y, bounding_box_max.z),
         (bounding_box_min.x, bounding_box_min.y, bounding_box_max.z)]
    ]
    return faces

def transform_to_local_space(coord, vol_translation, vol_scale):
    """
    Transforms world coordinates to local unit cube space.
    Formula: (coord - translation) / scale + 0.5
    """
    return (coord - vol_translation) / vol_scale + 0.5

def calculate_num_samples(dim_x, dim_y, dim_z):
    """
    Calculates number of samples per ray.
    If voxel count is 0: sqrt(dimX^2 + dimY^2 + dimZ^2) + 2
    Clamped to minimum 32.
    """
    if dim_x == 0 and dim_y == 0 and dim_z == 0:
        num_samples = math.sqrt(dim_x**2 + dim_y**2 + dim_z**2) + 2
    else:
        # Assuming dim_x, dim_y, dim_z are non-zero for typical case
        num_samples = math.sqrt(dim_x**2 + dim_y**2 + dim_z**2) + 2
    
    return max(num_samples, 32)

def calculate_ray_step_size(num_samples):
    """
    Calculates ray step size.
    Formula: sqrt(3.0) / num_samples
    """
    return math.sqrt(3.0) / num_samples

def jittered_volume_ray_casting(camera_position, projection_matrix, model_view_matrix,
                                voxel_texture, jitter_texture, jitter_tex_size,
                                vol_translation, vol_scale, vol_extent_min, vol_extent_max,
                                light_position, compute_lighting, num_samples):
    """
    Main ray casting function.
    """
    # 1. Coordinate Transformation
    # Transform volume extents and camera/light positions to local space
    local_extent_min = transform_to_local_space(vol_extent_min, vol_translation, vol_scale)
    local_extent_max = transform_to_local_space(vol_extent_max, vol_translation, vol_scale)
    local_camera_pos = transform_to_local_space(camera_position, vol_translation, vol_scale)
    local_light_pos = transform_to_local_space(light_position, vol_translation, vol_scale)
    
    # 2. Setup Rendering State
    enable_stencil_test()
    set_stencil_function(GL_EQUAL, ref=0)
    set_stencil_op(GL_INCR, GL_KEEP, GL_KEEP)
    
    enable_alpha_blending()
    set_blend_factors(GL_ONE, GL_ONE_MINUS_SRC_ALPHA)
    
    # 3. Near Plane Pass
    draw_near_plane_quad(projection_matrix, model_view_matrix, local_camera_pos, local_light_pos)
    
    # 4. Ray Casting Loop
    ray_step_size = calculate_ray_step_size(num_samples)
    
    for each pixel (u, v):
        # Generate ray direction from camera to pixel
        ray_dir = generate_ray_direction(u, v, projection_matrix, model_view_matrix, local_camera_pos)
        
        # Initialize accumulation
        accumulated_color = (0.0, 0.0, 0.0, 0.0)
        accumulated_opacity = 0.0
        
        # Determine ray start and end points in local space
        ray_start = local_camera_pos
        ray_end = ray_start + ray_dir * distance_to_far_plane
        
        # Calculate number of steps
        ray_length = length(ray_end - ray_start)
        num_steps = int(ray_length / ray_step_size)
        
        # Get jitter offset from texture
        jitter_u = (u % jitter_tex_size) / jitter_tex_size
        jitter_v = (v % jitter_tex_size) / jitter_tex_size
        jitter_offset = sample_2d_texture(jitter_texture, jitter_u, jitter_v)
        
        # Sample along ray
        for i in range(num_steps):
            # Calculate sample position with jitter
            t = (i + jitter_offset) * ray_step_size
            sample_pos = ray_start + ray_dir * t
            
            # Check if sample is within volume bounds
            if is_inside_volume(sample_pos, local_extent_min, local_extent_max):
                # Sample voxel texture
                voxel_color = sample_3d_texture(voxel_texture, sample_pos)
                voxel_opacity = voxel_color.a
                
                # Apply transfer function if needed (not specified in source, assume identity)
                # color = transfer_function(voxel_color)
                
                # Composite
                # Alpha blending: C_out = C_in + C_new * (1 - A_in)
                # A_out = A_in + A_new * (1 - A_in)
                weight = voxel_opacity * (1.0 - accumulated_opacity)
                accumulated_color += voxel_color.rgb * weight
                accumulated_opacity += weight
                
                # Early termination if fully opaque
                if accumulated_opacity >= 1.0:
                    break
        
        # Write to color buffer
        write_color_buffer(u, v, accumulated_color)
```

### Edge Cases and Pitfalls

*   **Memory Limit:** If the uncompressed voxel texture size exceeds 1GB, the format is automatically switched to `GL_COMPRESSED_RGBA_S3TC_DXT5_EXT` to fit within GPU memory limits [S1, unofficial].
*   **Empty Volume:** If the voxel data pointer is NULL and no subvolumes exist, a default color LUT is generated with 4 entries (transparent black, red, green, blue) and applied to the volume [S1, unofficial].
*   **Precision:** The coordinate transformation formula $(coord - translation) / scale + 0.5$ assumes consistent units between world coordinates and volume dimensions [S1, unofficial].

## Subtopics

### Stochastic Jittering
Uses runtime random number generation for jitter offsets instead of a pre-computed texture. Offers more randomness but may be slower due to RNG overhead. [Link](924-stochastic-jittering.md)

### Hierarchical Sampling (Inverse CDF Sampling)
Uses hierarchical data structures to sample based on probability distributions, useful for importance sampling. [Link](973-hierarchical-sampling-inverse-cdf-sampling.md)

### Uniform Ray Sampling
Samples at fixed intervals without jitter, leading to aliasing artifacts. Simpler but lower quality. [Link](1261-uniform-ray-sampling.md)

### Monte Carlo Volume Rendering
Uses stochastic sampling with many samples to reduce noise, often for complex lighting. [Link](2060-monte-carlo-volume-rendering.md)

### Random Jitter with Reprojection
Uses jittered sampling with reprojection techniques for temporal stability in animations. [Link](2782-random-jitter-with-reprojection.md)

### Importance Sampling for SPH Rendering
Tailored for Smoothed Particle Hydrodynamics data, sampling based on particle density. [Link](3624-importance-sampling-for-sph-rendering.md)

### Warp-based Sampling
Uses warping techniques to adjust sampling patterns, possibly for adaptive resolution. [Link](4298-warp-based-sampling.md)

## Operating characteristics

*   **Time Complexity:** $O(N_{pixels} \times N_{samples})$ per frame [S1, unofficial].
*   **Memory Complexity:** $O(VoxelVolume + JitterTextureSize)$ [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with resolution and number of samples [S1, unofficial].
*   **Strengths:** Reduces aliasing artifacts compared to uniform sampling; deterministic jitter pattern ensures reproducibility [S1, unofficial].
*   **Limitations:** Requires pre-computed jitter texture; may still exhibit some aliasing if jitter pattern is not well-designed [S1, unofficial].
*   **When to use:** When aliasing is a concern and deterministic results are preferred [S1, unofficial].
*   **When not to use:** When memory is extremely constrained (due to jitter texture) or when stochastic methods are preferred for other reasons [S1, unofficial].

## Background

*   **Origin:** The specific implementation details (e.g., `VoxVizThesis`, `GigaVoxels`) are from a thesis by Apecoraro [S1, unofficial].
*   **History:** The technique is part of the GigaVoxels framework for large-scale volume rendering [S1, unofficial].
*   **Usage:** Used in the VoxViz visualization tool [S1, unofficial].

## Sources

[S1] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
