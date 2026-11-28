# Texture-Based Volume Rendering

Also called: Geometry-based volume visualization, Proxy Geometry Rendering, Slice-based rendering, TBVR, Texture Based Ray Casting, Texture-based visualization

## Overview
Texture-Based Volume Rendering (TBVR) is a direct volume rendering technique that accelerates image-order ray casting by decomposing the 3D volume into a set of 2D planar slices [S1, unofficial][S2, unofficial][S7, unofficial]. These slices are treated as geometric primitives (proxy geometry) that are rasterized by the GPU, with the volumetric data stored in 3D or 2D texture memory [S2, unofficial][S7, unofficial]. Unlike [View-Aligned Slicing](6530-view-aligned-slicing.md), which dynamically computes slice geometry to maintain constant sampling distance, TBVR typically relies on fixed object-aligned slices or view-aligned slices that may vary in sampling density, trading memory efficiency and artifact susceptibility for rendering speed [S2, unofficial][S3, book].

## How it works
The method operates in three stages: Initialize, Update, and Draw [S3, book].

### Inputs and Outputs
*   **Input:** Volumetric scalar field data (regular grid), transfer function (TF), camera parameters (view/projection matrices), and rendering mode (transparent/solid) [S3, book][S5, unofficial].
*   **Output:** 2D color buffer with composited volume image.

### Data Structures
1.  **Volume Texture:**
    *   **3D Texture:** Stores the entire volume in GPU memory [S6, unofficial]. Format: `GL_RGBA` or `GL_RED` (for scalar data). Type: `GL_UNSIGNED_BYTE` or `GL_FLOAT`. Filters: `GL_LINEAR` (trilinear interpolation) [S3, book][S5, unofficial].
    *   **2D Texture Stack:** Stores slices as a stack of 2D textures [S7, unofficial]. Requires replication along X, Y, and Z axes for object-aligned rendering, consuming 3x memory [S2, unofficial][S3, book].
2.  **Transfer Function Texture:**
    *   1D texture mapping scalar values to RGBA color/opacity [S3, book].
3.  **Proxy Geometry:**
    *   Array of vertices defining planar polygons (quads or triangle fans) intersecting the volume bounding box [S2, S6, unofficial].
    *   **Object-Aligned:** Rectangles aligned with principal axes (X, Y, Z) [S2, unofficial][S3, book].
    *   **View-Aligned:** Polygons orthogonal to the viewing direction, constructed by intersecting slicing planes with the volume bounding box [S2, S6, unofficial].
4.  **Uniforms/Constants:**
    *   `u_slices`: `vec3` containing texture dimensions $(D_x, D_y, D_z)$ for coordinate normalization [S5, unofficial].
    *   `u_thresh`: `float` threshold for fragment discard [S5, unofficial].
    *   `u_alphaScale`: `float` multiplier for opacity [S5, unofficial].
    *   `u_clipPlaneX/Y/Z`, `u_clipCubeMax/Min`: Clipping parameters [S5, unofficial].

### Numbered Steps

1.  **Initialize:**
    *   Load volume data into CPU memory.
    *   Process data (e.g., compute gradients, downsample) if required [S3, book].
    *   Create transfer function lookup tables (1D textures) [S3, book].
    *   Download volume data to GPU as 3D texture or 2D texture stack [S2, unofficial][S3, book].
    *   Compile fragment shaders [S3, book].

2.  **Update (Per Frame):**
    *   Compute proxy geometry:
        *   For **View-Aligned**: Intersect slicing planes with the volume bounding box [S6, unofficial]. Sort resulting vertices in counter-clockwise order [S6, unofficial]. Construct polygon primitives (e.g., triangle fan from centroid) [S2, S6, unofficial].
        *   For **Object-Aligned**: Select the texture stack (X, Y, or Z) closest to the camera viewpoint [S2, unofficial].
    *   Update vertex arrays with proxy geometry vertices and their corresponding 3D texture coordinates [S3, book]. Texture coordinates are assigned to vertices, either on CPU or via vertex shader [S2, unofficial][S3, book].
    *   Refresh textures if rendering mode or transfer function parameters change [S3, book].

3.  **Draw:**
    *   Set up rendering state:
        *   Disable lighting and culling [S3, book][S9, unofficial].
        *   Enable alpha blending. For Over operator (back-to-front): Source = 1, Destination = 1 - Source Alpha [S2, unofficial]. For Under operator (front-to-back): Source = 1 - Destination Alpha, Destination = 1 [S2, unofficial].
        *   Disable depth testing or disable depth buffer writing if blending with opaque geometry [S9, unofficial].
    *   Bind volume texture and transfer function texture [S3, book][S9, unofficial].
    *   Specify shader input parameters (matrices, uniforms) [S9, unofficial].
    *   Draw slice polygons in sorted order (back-to-front for Over operator) [S2, unofficial][S3, book].
    *   Restore rendering state [S3, book].

### Fragment Shader Execution
For each fragment of a proxy polygon:
1.  **Interpolate Texture Coordinates:** Receive interpolated 3D texture coordinates from vertex attributes [S3, book].
2.  **Data Lookup:** Sample the volume texture at the interpolated coordinates to retrieve the scalar value $v$ [S3, book].
    *   *Note:* If using 2D textures, bilinear interpolation is used within slices, causing artifacts at slice boundaries [S8, unofficial]. 3D textures use trilinear interpolation [S1, unofficial].
3.  **Transfer Function Lookup:** Use the scalar value $v$ as a texture coordinate for a dependent lookup into the 1D transfer function texture to retrieve color $C$ and opacity $\alpha$ [S3, book].
4.  **Illumination (Optional):**
    *   Compute surface normal using central differences (finite differences) between neighboring voxels [S2, S6, unofficial].
    *   Apply lighting model (e.g., Blinn-Phong) to modify color [S2, S5, unofficial].
5.  **Compositing:** The resulting color and alpha are sent to the hardware compositing stage (alpha blending) [S2, unofficial][S3, book].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vertex_shader(position, model_matrix, view_matrix, projection_matrix):
    # Transform vertex to world space for texture coordinate calculation
    tex_position = model_matrix @ vec4(position, 1.0)
    
    # Camera position in world space
    v_cam = inverse(view_matrix)[:3, 3]
    
    gl_Position = projection_matrix @ view_matrix @ model_matrix @ vec4(position, 1.0)
    return tex_position, v_cam

def fragment_shader(tex_position, v_cam, volume_texture, slices_dims, threshold, alpha_scale, 
                    clip_cube_enabled, clip_cube_max, clip_cube_min,
                    clip_plane_x_enabled, clip_plane_x, clip_plane_y_enabled, clip_plane_y,
                    clip_plane_z_enabled, clip_plane_z):
    # Normalize texture coordinates based on volume dimensions
    tex_coords = vec3(
        (tex_position.x / slices_dims.x) + 0.5,
        (tex_position.y / slices_dims.y) + 0.5,
        (tex_position.z / slices_dims.z) + 0.5
    )
    
    # Sample volume data
    color = texture3D(volume_texture, tex_coords)
    
    # Discard if below threshold
    if color.r < threshold:
        return None
    
    # Clipping logic
    if clip_cube_enabled:
        if not (clip_cube_min.x <= tex_position.x <= clip_cube_max.x and
                clip_cube_min.y <= tex_position.y <= clip_cube_max.y and
                clip_cube_min.z <= tex_position.z <= clip_cube_max.z):
            return None
    if clip_plane_x_enabled and tex_position.x > clip_plane_x:
        return None
    if clip_plane_y_enabled and tex_position.z > clip_plane_y:
        return None
    if clip_plane_z_enabled and tex_position.y < clip_plane_z:
        return None
        
    # Apply alpha scale
    color.a *= alpha_scale
    return color

def shading(normal, view_dir, light_dir, Ka, Kd, Ks, shininess, light_colour, ambient_light):
    half_vector = normalize(light_dir + view_dir)
    ambient = Ka * ambient_light
    diffuse_light = max(dot(light_dir, normal), 0.0)
    diffuse = Kd * light_colour * diffuse_light
    specular_light = pow(max(dot(half_vector, normal), 0.0), shininess)
    if diffuse_light <= 0.0:
        specular_light = 0.0
    specular = Ks * light_colour * specular_light
    return ambient + diffuse + specular

def fragment_shader_solid(tex_position, v_cam, volume_texture, slices_dims, threshold, alpha_scale,
                          light_pos1, light_pos2):
    tex_coords = vec3(
        (tex_position.x / slices_dims.x) + 0.5,
        (tex_position.y / slices_dims.y) + 0.5,
        (tex_position.z / slices_dims.z) + 0.5
    )
    
    color = texture3D(volume_texture, tex_coords)
    if color.r < threshold:
        return None
        
    # Compute Normal via Central Differences
    delta = 1.0 / 256.0
    sample1 = texture3D(volume_texture, tex_coords - vec3(delta, 0.0, 0.0))
    sample2 = texture3D(volume_texture, tex_coords + vec3(delta, 0.0, 0.0))
    N_x = sample2.x - sample1.x
    
    sample1 = texture3D(volume_texture, tex_coords - vec3(0.0, delta, 0.0))
    sample2 = texture3D(volume_texture, tex_coords + vec3(0.0, delta, 0.0))
    N_y = sample2.x - sample1.x
    
    sample1 = texture3D(volume_texture, tex_coords - vec3(0.0, 0.0, delta))
    sample2 = texture3D(volume_texture, tex_coords + vec3(0.0, 0.0, delta))
    N_z = sample2.x - sample1.x
    
    normal = normalize(vec3(N_x, N_y, N_z))
    
    L1 = normalize(light_pos1 - vec3(tex_position))
    L2 = normalize(light_pos2 - vec3(tex_position))
    V = normalize(v_cam - vec3(tex_position))
    
    # Lighting constants from [S5, unofficial]
    Ka = vec3(0.1, 0.1, 0.1)
    Kd = vec3(0.6, 0.6, 0.6)
    Ks = vec3(0.2, 0.2, 0.2)
    shininess = 100.0
    light_colour = vec3(1.0, 1.0, 1.0)
    ambient_light = vec3(0.3, 0.3, 0.3)
    
    shade1 = shading(normal, V, L1, Ka, Kd, Ks, shininess, light_colour, ambient_light)
    shade2 = shading(normal, V, L2, Ka, Kd, Ks, shininess, light_colour, ambient_light)
    
    color.rgb = shade1 + shade2
    return color
```

### Parameters
*   **PLANE_INC:** Increment value for plane slicing, typically initialized as 1.0 (voxel spacing) [S5, unofficial].
*   **u_slices:** Uniform `vec3` containing texture dimensions $(TEXTURE\_SIZE\_X, TEXTURE\_SIZE\_Y, TEXTURE\_SIZE\_Z)$ [S5, unofficial].
*   **u_thresh:** Uniform `float` threshold for discarding fragments [S5, unofficial].
*   **u_alphaScale:** Uniform `float` multiplier for fragment alpha [S5, unofficial].
*   **Lighting Constants (Phong):** $K_a=(0.1,0.1,0.1)$, $K_d=(0.6,0.6,0.6)$, $K_s=(0.2,0.2,0.2)$, shininess=100.0, ambientLight=(0.3,0.3,0.3), lightColour=(1.0,1.0,1.0) [S5, unofficial].

### Edge Cases and Pitfalls
*   **Precision:** Trilinear interpolation in 3D textures is slightly slower than bilinear in 2D but provides better quality [S1, S8, unofficial].
*   **Boundaries:** Object-aligned slices suffer from "pop-up" artifacts when switching between texture stacks (X, Y, Z) due to changes in sampling points [S2, S8, unofficial].
*   **Intensity Variations:** As the camera moves, the sampling distance changes in object-aligned rendering, causing intensity variations [S2, unofficial][S3, book].
*   **Viewing Angle:** 2D texture stacks suffer from sampling artifacts when the viewing angle approaches 45 degrees relative to the slices, as the viewer sees between slice gaps [S1, unofficial].
*   **Memory:** Object-aligned slices require 3x memory consumption due to data replication along principal directions [S2, unofficial][S3, book].

## Subtopics

### Object-Aligned Slices
Uses fixed proxy geometry aligned with the principal axes (X, Y, Z) of the volume [S2, unofficial][S3, book]. Requires storing the volume as three stacks of 2D textures (replicated data), consuming 3x memory [S6, unofficial]. Fast and simple to implement but suffers from memory overhead, image popping artifacts during stack swaps, and intensity variations as the camera moves [S2, unofficial][S3, book][S6, unofficial].

### View-Aligned Slices
Uses proxy geometry that is orthogonal to the viewing direction [S7, unofficial]. Requires 3D textures (or on-the-fly reconstruction from 2D stacks) [S7, unofficial]. Slower to compute geometry but more memory efficient (no replication) and provides consistent sampling distance, eliminating 45-degree viewing artifacts [S2, unofficial][S3, book][S7, unofficial].

### 2D Slice Replication Approach
A variant of object-aligned rendering where data is stored as a set of 2D texture slices [S6, unofficial]. Proxy polygons are simple rectangles aligned with slices [S3, book]. To avoid data replication, slices can be reconstructed on the fly, incurring performance overhead [S7, unofficial]. Sampling rate depends on volume resolution, which can be mitigated by adding intermediate slices and using trilinear interpolation [S3, book].

## Operating characteristics
*   **Time Complexity:** Rendering time is proportional to the number of slices and the resolution of the image [G]. Fragment shader execution is parallelized on the GPU [S4, unofficial].
*   **Memory Complexity:**
    *   **3D Textures:** $O(N)$ where $N$ is the number of voxels [G].
    *   **2D Textures (Object-Aligned):** $O(3N)$ due to replication along X, Y, Z axes [S2, unofficial][S3, book].
*   **Measured Performance:** Achieves interactive frame rates on modern GPUs by leveraging hardware texture sampling and alpha blending [S1, S4, unofficial]. 2D textures are faster than 3D textures due to simpler bilinear interpolation, but 3D textures provide better image quality [S1, S8, unofficial].
*   **Strengths:** High performance on GPUs; simple implementation; leverages standard graphics pipeline [S1, S2, unofficial].
*   **Limitations:** Artifacts at slice boundaries (2D); memory usage (2D replication); sampling artifacts at oblique angles (2D); dependency on sampling rate [S1, S2, unofficial][S3, book][S7, unofficial].
*   **When to Use:** Interactive visualization where performance is critical and artifacts are acceptable or mitigated (e.g., view-aligned slices) [G].
*   **When Not to Use:** High-quality offline rendering requiring precise sampling (use ray casting); memory-constrained environments with large volumes using 2D textures [G].

## Background
Texture-Based Volume Rendering was developed to accelerate volume rendering on graphics hardware [S4, unofficial]. Early implementations used 2D textures due to hardware limitations, leading to object-aligned slicing techniques [S7, unofficial]. The advent of 3D texture support in GPUs allowed for view-aligned slicing and trilinear interpolation, significantly improving image quality and eliminating viewing angle artifacts [S1, S7, unofficial]. The technique is widely used in medical imaging and scientific visualization for interactive exploration of volumetric data [S3, book][S5, unofficial].

## Sources
[S1] Direct Volume Rendering Data, https://www.scribd.com/document/1050958395/Direct-Volume-Rendering-Data, fetched 2026-10-25 (unofficial)
[S2] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
[S3] Chapter 39. Volume Rendering Techniques | NVIDIA Developer, https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-39-volume-rendering-techniques, fetched 2026-10-26 (book)
[S4] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
[S5] DoctorTone/MedicalViz, https://github.com/DoctorTone/MedicalViz, fetched 2026-10-27 (unofficial)
[S6] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S7] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S8] A Survey of GPU-Based Volume Rendering of ..., https://www.sci.utah.edu/~csilva/papers/rita2005.pdf, fetched 2026-10-29 (unofficial)
[S9] Volume Rendering Lecture Nov-30-2005 (Spanish), https://www.ciroduran.com/personal2/wp-content/uploads/2009/07/clase-volumerendering.pdf, fetched 2026-10-29 (unofficial)
