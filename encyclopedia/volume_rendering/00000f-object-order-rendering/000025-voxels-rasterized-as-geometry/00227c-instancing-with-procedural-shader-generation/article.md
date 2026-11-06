# Instancing with Procedural/Shader Generation

Also called: Instanced Voxel Rendering with Palette Textures, Palette-based Voxel Rendering, Voxel Instancing

## Overview
Instancing with Procedural/Shader Generation renders a voxel model by submitting a single draw call that instances a base geometric primitive (typically a unit cube) for each occupied voxel [S1, unofficial]. It solves the CPU bottleneck of issuing individual draw calls for thousands of voxels by packing per-voxel data (position, color) into a single buffer and using the graphics pipeline's instancing mechanism to transform and color them procedurally [S1, unofficial]. It differs from [Naive Voxel Rendering](1103-naive-voxel-rendering.md) by reducing API overhead to a single call, and from [Voxel Fragment Shader Rendering](5849-voxel-fragment-shader-rendering.md) by computing the voxel's spatial position in the vertex/geometry stage rather than reconstructing it purely from fragment shader texture lookups [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of occupied voxel indices $(i, j, k)$, a 2D palette texture mapping UV coordinates to base colors, and a Model-View-Projection (MVP) matrix.
*   **Output:** A rasterized image of the voxel model with Phong lighting.

**Data Structures:**
1.  **Instance Buffer:** A contiguous array of floats bound to `GL_ARRAY_BUFFER`. Each instance contains exactly 5 floats in the following order:
    *   `offsetX` (float): X-coordinate of the voxel center.
    *   `offsetY` (float): Y-coordinate of the voxel center.
    *   `offsetZ` (float): Z-coordinate of the voxel center.
    *   `u` (float): S-coordinate for palette texture sampling.
    *   `v` (float): T-coordinate for palette texture sampling.
    *   Total size: $N_{voxels} \times 5 \times 4$ bytes [S1, unofficial].
2.  **Palette Texture:** A 2D texture (e.g., `GL_RGBA8`) storing the color map. Wrap modes must be set to `GL_CLAMP_TO_EDGE` to prevent sampling artifacts at boundaries [S1, unofficial].
3.  **Base Geometry:** A single unit cube mesh (6 faces, 24 vertices for flat shading or 8 vertices for shared shading) [S1, unofficial].

**Numbered Steps:**
1.  **Data Preparation:** For each occupied voxel at grid index $(i, j, k)$, calculate its world position. If the source format uses Z-up (e.g., .vly) and the renderer uses Y-up (OpenGL), swap the Y and Z components of the offset [S1, unofficial].
2.  **Buffer Upload:** Pack the offsets and UV coordinates into the instance buffer and upload it to GPU memory with `GL_STATIC_DRAW` hint [S1, unofficial].
3.  **Vertex Shader Execution:** For each vertex of the base cube instance:
    *   Retrieve the local vertex position $P_{local}$.
    *   Retrieve the instance offset $O_{inst} = (offsetX, offsetY, offsetZ)$.
    *   Compute the combined world position: $P_{world} = P_{local} + O_{inst}$ [S1, unofficial].
    *   Transform to clip space: $gl\_Position = MVP \times [P_{world}, 1]^T$ [S1, unofficial].
    *   Pass the instance UVs $(u, v)$ to the fragment shader [S1, unofficial].
4.  **Fragment Shader Execution:** For each rasterized fragment:
    *   Sample the base color from the palette texture: $C_{base} = \text{texture}(uPaletteTexture, vUV).rgb$ [S1, unofficial].
    *   Compute lighting using a Phong model:
        *   Ambient: $C_{amb} = C_{base} \times I_{ambient}$
        *   Diffuse: $C_{diff} = C_{base} \times \max(\mathbf{N} \cdot \mathbf{L}, 0.0)$
        *   Specular: $C_{spec} = \mathbf{1} \times \text{spec} \times I_{specular}$
        *   Total: $C_{final} = C_{amb} + C_{diff} + C_{spec}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader Logic (per vertex per instance)
def vertex_shader(local_position: vec3, instance_offset: vec3, instance_uv: vec2, mvp_matrix: mat4) -> vec4:
    # Combine local geometry with instance position
    # Note: Coordinate system swap (Y/Z) is handled during instance buffer population
    world_position = local_position + instance_offset
    
    # Transform to clip space
    clip_position = mvp_matrix * vec4(world_position, 1.0)
    
    # Pass UVs to fragment shader
    gl_Position = clip_position
    return instance_uv

# Fragment Shader Logic (per fragment)
def fragment_shader(instance_uv: vec2, palette_texture: texture2d, light_pos: vec3, camera_pos: vec3, 
                    ambient_intensity: float, specular_intensity: float, shininess: float) -> vec4:
    # Sample base color from palette
    base_color = texture(palette_texture, instance_uv).rgb
    
    # Calculate normals (assuming unit cube, normal is face normal)
    # In a real implementation, normals are passed from vertex shader or computed from geometry
    normal = normalize(get_normal_from_geometry()) 
    
    # Lighting vectors
    to_light = normalize(light_pos - get_world_position())
    to_camera = normalize(camera_pos - get_world_position())
    reflect_dir = reflect(-to_light, normal)
    
    # Phong Lighting
    ambient = base_color * ambient_intensity
    diffuse = base_color * max(dot(normal, to_light), 0.0)
    
    # Specular component
    spec_factor = pow(max(dot(to_camera, reflect_dir), 0.0), shininess)
    specular = vec3(1.0) * spec_factor * specular_intensity
    
    final_color = ambient + diffuse + specular
    return vec4(final_color, 1.0)
```

**Parameters:**
*   `ambient_intensity`: 0.35 [S1, unofficial].
*   `specular_intensity`: 0.5 [S1, unofficial].
*   `shininess`: 32 [S1, unofficial].
*   `light_pos_world`: (10, 15, 10) [S1, unofficial].
*   `camera_distance`: 5.0, modulated by zoom [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Coordinate System Mismatch:** Source formats may use Z-up while OpenGL uses Y-up. This is handled by swapping Y and Z during offset calculation [S1, unofficial].
*   **Texture Artifacts:** Palette texture sampling at boundaries can cause bleeding. Mitigated by setting wrap modes to `GL_CLAMP_TO_EDGE` [S1, unofficial].

## Subtopics

### Instanced Voxelization
A technique to draw all voxelization slices in a single draw call using instancing. The instance ID is used to specify the target slice, allowing efficient rendering of deforming meshes or obstacles without separate draw calls per slice [S2, unofficial].

### Sphere Impostors for DEM Pebbles
Rendering of Discrete Element Method (DEM) pebble beds using instanced sphere impostors. Each pebble is drawn as a quad with an exact orthographic ray-sphere hit test in the fragment shader, providing sphere shading and depth. This allows rendering 27,000 pebbles at interactive rates with only 1.2-1.3 ms of CPU overhead per frame. If no wgpu device is available, egui discs are used as a fallback [S3, unofficial].

### Instanced Voxelization
Uses instancing to draw all slices in a single draw call. The instance ID specifies the target slice, enabling efficient voxelization of deforming meshes or obstacles [S2, unofficial].

### Sphere Impostors for DEM Pebbles
Rendering of Discrete Element Method (DEM) pebble beds using instanced sphere impostors. Each pebble is drawn as a quad with an exact orthographic ray-sphere hit test in the fragment shader, providing sphere shading and depth. This allows rendering 27,000 pebbles at interactive rates with only 1.2-1.3 ms of CPU overhead per frame. If no wgpu device is available, egui discs are used as a fallback [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Setup time is $O(V)$ to compute offsets and pack data. Draw time is $O(1)$ draw call, with fragment shader cost proportional to visible voxels [S1, unofficial].
*   **Memory Complexity:** $O(V)$ for the instance buffer (5 floats per voxel) and $O(P)$ for the palette texture [S1, unofficial].
*   **Performance:** Implemented in the VoxelRenderer Android application for rendering .vly voxel models using OpenGL ES 3.0 [S1, unofficial].
*   **Strengths:** Reduces CPU-GPU synchronization overhead by batching all voxels into a single draw call. Simple implementation using standard OpenGL features.
*   **Limitations:** Requires a palette texture, which may limit color depth or require dithering. Lighting is simple Phong, which may not suffice for complex materials. Visibility culling is not inherent to the instancing mechanism itself and must be handled separately (e.g., frustum culling) [S1, unofficial].

## Background
This method is implemented in the VoxelRenderer Android application for rendering .vly voxel models using OpenGL ES 3.0 [S1, unofficial]. It builds on the principles of [Voxels rasterized as geometry](27-voxels-rasterized-as-geometry.md), specifically leveraging the instancing variant to optimize draw call overhead.

## Sources
[S1] Il-castor/VoxelRenderer, https://github.com/Il-castor/VoxelRenderer, fetched 2026-10-26 (unofficial)
[S2] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S3] dhoby-ghaut: GPU rendering for the 3D and 2D views (sphere impostors for pebbles; GPU or accelerated CSG ray tracing), https://github.com/theodoreOnzGit/outram-park-backend/issues/587, fetched 2026-10-26 (unofficial)
