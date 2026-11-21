# WebGL Multi-pass Ray-casting with 3D Geometry Intersection

Also called: Five-pass ray-casting pipeline, VMesh Real-Time Rendering, WebGL volume rendering with intersected 3D geometry

## Overview
This method enables the interactive coexistence of volumetric datasets and 3D triangular meshes in WebGL by using a multi-pass rendering pipeline to handle occlusion, blending, and ray intersection [S1, S2]. It solves the problem of rendering opaque and transparent 3D geometry within or alongside volume data, a key requirement for medical and engineering visualizations, while adhering to WebGL constraints such as the lack of native 3D and float texture support [S1]. Unlike single-pass ray-casting approaches [S2], this method explicitly separates the computation of depth, surface color, and volume traversal into distinct passes to correctly manage back-to-front blending and geometry intersections [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D triangular meshes (opaque), a volumetric dataset (represented as a 2D texture atlas), camera matrices, and transfer functions.
*   **Outputs:** A final 2D image composited from volume and mesh data.

**Data Structures:**
*   **Volume Data:** Stored as a 2D texture atlas where 2D slices are stacked in the Z-axis arranged in a matrix configuration on a single 2D plane [S1].
*   **Depth Data:** Since WebGL 1.0 lacks native float texture support, depth values from intermediate passes are encoded into RGBA color channels of Frame Buffer Objects (FBOs) [S1].
*   **FBOs:** Multiple Frame Buffer Objects are used to store intermediate results: depth maps, surface colors, and volume coordinates [S1].

**Numbered Steps:**
The pipeline consists of five passes (or four in the VMesh variant [S2]), executed in the following order:

1.  **Depth Pass (Meshes):** Render the depth of all 3D meshes in the scene into an FBO. The fragment shader outputs the current depth encoded in RGBA color channels [S1].
2.  **Surface Color Pass (Meshes):** Render all 3D meshes into a separate FBO to store their color information [S1].
3.  **Back Cube Depth Pass (Volume):** Render the depth of the back side of the volume bounding box into an FBO, encoded in RGBA, using the same shader logic as Pass 1 [S1].
4.  **Back Coordinates Pass (Volume):** Render the back faces of the volume bounding box into a separate FBO. The 3D texture coordinates are assigned as color on each vertex, so rasterization interpolates these per-vertex colors to represent volume data 3D texture coordinates [S1].
5.  **Ray Traversal and Composition Pass:** Render 3D objects and volume data to the screen buffer. Objects must be sorted in Z-depth order and rendered from back to front [S1]. In the fragment shader, rays are cast towards the volume. All outputs from previous passes are sampled as 2D textures. The shader performs ray marching, checks for 3D geometry intersections using depth data from Passes 1 and 3, and composites the results [S1].

**Pseudocode:**
The following pseudocode represents the core ray-casting logic executed in the fragment shader during the final pass (Pass 5), adapted from [S1]. It assumes access to textures generated in previous passes.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs:
# depth_mesh_tex: Texture from Pass 1 (Mesh Depth)
# color_mesh_tex: Texture from Pass 2 (Mesh Color)
# depth_vol_tex: Texture from Pass 3 (Volume Back Depth)
# coord_vol_tex: Texture from Pass 4 (Volume Back Coordinates/3D TexCoords)
# volume_atlas_tex: 2D Texture Atlas containing volume slices
# camera_pos: vec3
# screen_pos: vec2 (current fragment coordinate)
# projection_matrix, view_matrix: Matrices

# Helper: Unpack float from RGBA (GLSL-style logic implemented in Python-like pseudocode)
def unpack_rgba_to_float(r, g, b, a):
    # Corresponds to the GLSL function in [S1]
    # value = r * 256^3 + g * 256^2 + b * 256 + a
    # Note: The source uses bit shifting logic. 
    # bitSh = [256^3, 256^2, 256, 1]
    # bitMsk = [0, 1/256, 1/256, 1/256]
    # The unpacking reverses the packing.
    # Standard packing: val = r*256^3 + g*256^2 + b*256 + a
    return r * (256.0**3) + g * (256.0**2) + b * 256.0 + a

# 1. Initialize Ray Parameters
# Compute ray direction based on camera and screen position
ray_direction = compute_ray_direction(camera_pos, screen_pos, projection_matrix, view_matrix)
ray_pos = camera_pos

# Determine ray bounds
# Front face position is typically known or computed via ray-box intersection
# Back face depth is sampled from depth_vol_tex
back_depth_rgba = sample_texture(depth_vol_tex, screen_pos)
back_depth_float = unpack_rgba_to_float(back_depth_rgba.r, back_depth_rgba.g, back_depth_rgba.b, back_depth_rgba.a)

# Calculate maximum distance D to the back of the volume bounding box
# Assuming ray_pos is at camera, and back_depth_float is the depth along the ray
max_distance = back_depth_float 

# Compute number of steps S and step size s
# S is determined by resolution and max_distance
step_size = max_distance / S

# 2. Ray Traversal Loop
accumulated_color = vec3(0.0)
accumulated_alpha = 0.0

for i from 0 to S:
    # Current sample position
    current_sample_pos = ray_pos + (i * step_size) * ray_direction
    
    # Check for 3D Geometry Intersection
    # Sample depth from mesh depth texture
    mesh_depth_rgba = sample_texture(depth_mesh_tex, screen_pos)
    mesh_depth_float = unpack_rgba_to_float(mesh_depth_rgba.r, mesh_depth_rgba.g, mesh_depth_rgba.b, mesh_depth_rgba.a)
    
    # Determine if a mesh exists at this depth closer than current ray position
    # Note: Depth comparison depends on coordinate system (NDC vs View Space)
    # If mesh_depth_float < current_depth_along_ray, a mesh occludes the volume
    if mesh_exists_at_depth(mesh_depth_float, current_sample_pos):
        # Get mesh color
        mesh_color = sample_texture(color_mesh_tex, screen_pos)
        
        # Composite mesh color with accumulated volume color
        # Back-to-front blending
        # C_out = C_vol + (1 - alpha_vol) * C_mesh
        # alpha_out = alpha_vol + (1 - alpha_vol) * alpha_mesh (assuming opaque mesh, alpha_mesh=1)
        
        # If mesh is opaque, it blocks further volume sampling
        accumulated_color = accumulated_color + (1.0 - accumulated_alpha) * mesh_color
        accumulated_alpha = 1.0
        break # Terminate ray due to occlusion
    
    # Sample Volume Data
    # Use coord_vol_tex to get 3D texture coordinates for the current ray position
    # The 3D coordinates are interpolated from the back face rendering in Pass 4
    # However, standard ray casting uses ray_pos to index into the volume atlas
    # The source [S1] implies using the 3D texture coordinates from the back face 
    # to map the ray position to the volume atlas.
    
    # Interpolate sample at current ray_pos from volume atlas
    # This requires mapping ray_pos to the 2D atlas coordinates
    volume_value = sample_volume_atlas(volume_atlas_tex, ray_pos)
    
    # Compute color and opacity using transfer function
    color, alpha = apply_transfer_function(volume_value)
    
    # Accumulate
    # Standard front-to-back compositing for volume
    accumulated_color = accumulated_color + (1.0 - accumulated_alpha) * alpha * color
    accumulated_alpha = accumulated_alpha + (1.0 - accumulated_alpha) * alpha
    
    # Early Ray Termination
    if accumulated_alpha >= 1.0:
        break
        
    # Increment ray position
    ray_pos = ray_pos + step_size * ray_direction

# Final Output
output_color = accumulated_color
```

**Parameters:**
*   **Rendering Order:** Objects (volumes and 3D opaque meshes) must be sorted in z-depth order and rendered from back to front [S1].
*   **Blending Mode:** Back-to-front blending composition is used for the ray-casting shaders in the final rendering pass [S1].
*   **Threshold:** Early ray termination occurs when accumulated opacity $\alpha \ge 1.0$ [S1].

**Edge Cases and Pitfalls:**
*   **Lack of 3D Textures:** WebGL lacks support for 3D textures, requiring the use of a 2D texture atlas workaround to sample volumetric data [S1].
*   **Lack of Float Textures:** WebGL lacks support for Float textures, requiring depth data to be encoded into RGBA color channels for storage in FBOs. Precision loss may occur during packing/unpacking [S1].
*   **Occlusion Handling:** Correctly determining if a 3D object occludes the volume requires comparing the depth of the mesh (from Pass 1) with the current ray position. This must be done at each step or at least at the entry point [S1].

## Operating characteristics
*   **Complexity:** The multi-pass approach requires five rendering passes and multiple Frame Buffer Objects (FBOs), which is a downside in terms of computational overhead and memory usage [S1].
*   **Performance:** The method is designed for interactive visualization on web platforms, balancing quality with the constraints of WebGL [S1].
*   **Strengths:** Supports the coexistence of 3D triangular meshes and volumetric datasets, which is critical for medical and engineering applications [S1].
*   **Limitations:** High memory usage due to multiple FBOs; precision issues due to RGBA depth encoding; complexity of implementation due to multiple passes and sorting requirements [S1].

## Background
This method extends previous WebGL ray-casting efforts by Congote et al. [CSK* 11], Mobeen et al. [MF12], and Noguera and Jiménez [NJ12] by exploring the coexistence of 3D triangular meshes along volume data within the constraints of the Web platform [S1]. It addresses challenges such as the lack of 3D and depth textures in WebGL [S1]. The VMesh variant [S2] implements a similar four-pass process using three.js for VMesh assets.

## Sources
[S1] Deployment of Volume Rendering Interactive Visualizations in Web Platforms With Intersected 3D Geometry, https://doi.org/10.2312/ceig.20161312, fetched 2026-10-26
[S2] VMesh: Hybrid Volume-Mesh Representation for Efficient View Synthesis, https://core.ac.uk/download/560406650.pdf, fetched 2026-10-27
