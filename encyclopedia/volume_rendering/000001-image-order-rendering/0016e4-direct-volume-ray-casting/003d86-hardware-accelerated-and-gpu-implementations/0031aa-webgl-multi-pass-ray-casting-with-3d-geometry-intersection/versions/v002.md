# WebGL Multi-pass Ray-casting with 3D Geometry Intersection

Also called: Five-pass ray-casting pipeline, WebGL volume rendering with intersected 3D geometry

## Overview
This method implements direct volume ray casting within the WebGL API constraints, specifically addressing the coexistence of volumetric datasets and 3D triangular meshes [S1]. It solves the problem of rendering opaque and transparent 3D geometry alongside semi-transparent volume data by using a five-pass rendering pipeline to handle occlusion, blending, and ray intersection [S1]. Unlike standard WebGL volume rendering which often ignores external geometry or relies on unsupported 3D textures, this approach encodes depth and coordinate data into 2D RGBA textures to facilitate ray-geometry intersection checks during the final compositing pass [S1].

## How it works
The method extends the direct volume ray casting pipeline [19-direct-volume-ray-casting.md] by introducing explicit handling of 3D polygonal meshes. It requires five distinct rendering passes to gather necessary depth and color information for the final ray-casting step, which combines volume traversal with 3D geometry intersection.

### Inputs and Outputs
**Inputs:**
*   Volumetric dataset: A set of 2D slices stacked along the Z-axis, arranged in a 2D texture atlas [S1].
*   3D Geometry: A set of triangular meshes representing opaque or semi-transparent objects [S1].
*   Camera parameters: Origin $\mathbf{o}$, direction $\mathbf{d}$, and view frustum [S1].
*   Transfer function: Maps scalar values to color $C$ and opacity $\alpha$ [S1].

**Outputs:**
*   Final image in the screen buffer [S1].

### Data Structures and Layouts
Due to WebGL 1.0 limitations, specific data structures are used to bypass the lack of 3D and Float texture support [S1]:

1.  **Volume Data (2D Texture Atlas):**
    *   The 3D volume is decomposed into 2D slices.
    *   These slices are arranged in a matrix configuration within a single 2D texture [S1].
    *   Sampling requires decoding the 2D texture coordinates back into 3D volume coordinates [S1].

2.  **Depth Buffers (RGBA Textures):**
    *   Since WebGL lacks native Float texture support for depth buffers in FBOs (and depth texture extensions are not universally implemented), depth values are encoded into RGBA color channels [S1].
    *   **Pass 1 Output:** Depth of all 3D meshes, encoded as RGBA.
    *   **Pass 3 Output:** Depth of the back side of the volume bounding box, encoded as RGBA [S1].
    *   **Encoding Formula (GLSL):**
        ```glsl
        vec4 packFloat(float value) {
            const vec4 bitSh = vec4(256.0*256.0*256.0, 256.0*256.0, 256.0, 1.0);
            const vec4 bitMsk = vec4(0.0, 1.0/256.0, 1.0/256.0, 1.0/256.0);
            return fract(value * bitSh) - bitMsk * floor(value * bitSh);
        }
        ```
    *   **Decoding Formula (GLSL):**
        ```glsl
        float unpackFloat(vec4 rgba) {
            const vec4 bitSh = vec4(1.0/(256.0*256.0*256.0), 1.0/(256.0*256.0), 1.0/256.0, 1.0);
            return dot(rgba, bitSh);
        }
        ```

3.  **Frame Buffer Objects (FBOs):**
    *   Multiple FBOs are created to store intermediate results:
        *   FBO 1: 3D Mesh Depth (RGBA) [S1].
        *   FBO 2: 3D Mesh Color (RGBA) [S1].
        *   FBO 3: Back Cube Depth (RGBA) [S1].
        *   FBO 4: Back Cube Coordinates (RGBA) [S1].

### Numbered Steps

**Pass 1: Depth Pass**
1.  Render all 3D triangular meshes into FBO 1.
2.  The fragment shader outputs the current depth value encoded as RGBA using `packFloat` [S1].

**Pass 2: Color of Surface Objects**
1.  Render all 3D triangular meshes into FBO 2.
2.  Store the final color of the surfaces [S1].

**Pass 3: Back Cube Depth**
1.  Render the back faces of the volume bounding box into FBO 3.
2.  Use the same shader as Pass 1 to output the depth of the back cube faces encoded as RGBA [S1].

**Pass 4: Back Coordinates of the Cube**
1.  Render the back faces of the volume bounding box into FBO 4.
2.  Assign 3D texture coordinates to each vertex of the cube.
3.  The rasterization process interpolates these per-vertex colors, which represent the volume data 3D texture coordinates [S1].

**Pass 5: Ray Traversal and 3D Surfaces**
1.  Render the scene (volume and 3D objects) to the screen buffer.
2.  Objects must be sorted in Z-depth order, rendered from back to front [S1].
3.  The fragment shader performs the ray casting, using textures from FBOs 1-4 as inputs [S1].

### Ray Casting Algorithm (Fragment Shader)
For each pixel, the following logic is executed [S1]:

1.  **Initialization:**
    *   Initialize number of steps $S$.
    *   Compute ray position $\mathbf{ray\_pos}$ and ray direction $\mathbf{ray\_direction}$.
    *   Compute maximum distance $D$.
    *   Compute interval steps $s$ [S1].

2.  **Traversal Loop:**
    ```python
    # Pseudocode for the ray traversal loop in the fragment shader
    # Inputs: ray_pos, ray_direction, s, S, volume_atlas, fbo1_depth, fbo3_depth, transfer_function
    # Outputs: final_color, final_alpha

    accumulated_color = vec3(0.0)
    accumulated_alpha = 0.0

    for i from 0 to S:
        # 1. Sample Volume
        # Decode 3D coordinates from the current ray_pos or use interpolated coords from FBO4
        # Sample scalar value V from volume_atlas
        V = sample_volume(volume_atlas, ray_pos)
        
        # 2. Classification
        # Apply transfer function to get color C and opacity alpha
        C, alpha = transfer_function(V)
        
        # 3. 3D Geometry Intersection Check
        # Access depth data from FBO1 (mesh depth) and FBO3 (back cube depth)
        # Unpack depth values using unpackFloat
        mesh_depth = unpackFloat(sample(fbo1_depth, screen_coords))
        back_cube_depth = unpackFloat(sample(fbo3_depth, screen_coords))
        
        # Determine if a 3D object is closer than the current ray_pos
        # If mesh_depth < ray_pos.z (in view space) or similar depth comparison,
        # intersect with the 3D geometry color from FBO2
        if is_intersected_with_3d_geometry(mesh_depth, ray_pos):
            # Use color from FBO2 for the intersection point
            # This effectively stops volume sampling at the geometry surface
            # or blends the geometry color with the volume
            geometry_color = sample(fbo2_color, screen_coords)
            # Blend geometry with accumulated volume
            accumulated_color = blend(accumulated_color, geometry_color, alpha)
            accumulated_alpha = blend_alpha(accumulated_alpha, alpha)
            break # Stop volume traversal if geometry is opaque or fully occludes
        
        # 4. Accumulate Color (Front-to-Back)
        # Standard over operator
        accumulated_color = accumulated_color + (1.0 - accumulated_alpha) * alpha * C
        accumulated_alpha = accumulated_alpha + (1.0 - accumulated_alpha) * alpha
        
        # 5. Termination Check
        if accumulated_alpha >= 1.0 or ray_pos > bounding_box:
            break
            
        # 6. Increment Ray Position
        ray_pos = ray_pos + s * ray_direction

    return accumulated_color, accumulated_alpha
    ```

### Parameters
*   **Rendering Order:** Objects (volumes and 3D opaque meshes) must be sorted in z-depth order and rendered from back to front to correctly handle transparency and occlusion [S1].
*   **Blending Mode:** Back-to-front blending composition is used for the ray-casting shaders in the final rendering pass [S1].
*   **Step Size:** Determined by the maximum distance $D$ and number of steps $S$ [S1].

### Edge Cases and Pitfalls
*   **Lack of 3D Textures:** WebGL lacks support for 3D textures, requiring the use of a 2D texture atlas workaround to sample volumetric data [S1].
*   **Lack of Float Textures:** WebGL lacks support for Float textures, requiring depth data to be encoded into RGBA color channels for storage in FBOs [S1].
*   **Precision Loss:** Encoding 24-bit depth into 8-bit RGBA channels may introduce precision issues, particularly for large depth ranges [S1].
*   **Performance Overhead:** The multi-pass approach requires five rendering passes and multiple Frame Buffer Objects (FBOs), which increases computational overhead and memory usage [S1].

## Operating characteristics
*   **Time Complexity:** The method requires five rendering passes. The final pass involves ray marching for each pixel, with complexity proportional to the number of steps $S$ and the number of pixels [S1].
*   **Memory Complexity:** Requires storage for multiple FBOs (Depth 1, Color 1, Depth 2, Coordinates 2) plus the volume texture atlas. Memory usage is higher than single-pass methods due to the intermediate buffers [S1].
*   **Strengths:** Enables the coexistence of 3D triangular meshes and volumetric data in WebGL, which is not natively supported by standard volume rendering techniques in this environment [S1].
*   **Limitations:** High computational overhead due to multiple passes; potential precision loss from depth encoding; dependency on 2D texture atlas for volume sampling [S1].
*   **When to use:** When rendering interactive volume visualizations in web browsers where 3D geometry must interact with volume data, and hardware supports WebGL 1.0/2.0 [S1].
*   **When not to use:** When performance is critical and 3D geometry interaction is not required, or when native 3D/Float texture support is available (e.g., desktop OpenGL/Vulkan) [S1].

## Background
This method was introduced by Arbelaiz, Moreno, and Kabongo in 2016 [S1]. It extends previous WebGL ray-casting efforts by Congote et al. [CSK* 11], Mobeen et al. [MF12], and Noguera and Jiménez [NJ12] by exploring the possibilities of the coexistence of 3D triangular meshes along volume data within the constraints of the Web platform [S1]. The work addresses challenges such as the lack of 3D and depth textures in WebGL, which are common restrictions when targeting a wide range of devices from mobile to desktop [S1].

## Sources
[S1] Deployment of Volume Rendering Interactive Visualizations in Web Platforms With Intersected 3D Geometry, https://doi.org/10.2312/ceig.20161312, fetched 2026-10-26
