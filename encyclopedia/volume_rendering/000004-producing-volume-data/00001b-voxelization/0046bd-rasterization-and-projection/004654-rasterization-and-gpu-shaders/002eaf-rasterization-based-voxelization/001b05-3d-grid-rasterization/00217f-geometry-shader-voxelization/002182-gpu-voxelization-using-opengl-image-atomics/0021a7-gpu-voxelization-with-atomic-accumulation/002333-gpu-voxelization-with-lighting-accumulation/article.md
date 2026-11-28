# GPU Voxelization with Lighting Accumulation

Also called: Light Injection & Opacity Voxelization, Voxelization and storing direct lighting

## Overview
GPU Voxelization with Lighting Accumulation is a rasterization-based technique that converts scene triangle meshes into a 3D voxel grid to store direct lighting information and opacity for subsequent indirect lighting calculations, such as cone tracing [S1, unofficial][S2, unofficial]. It differs from standard geometry voxelization by leveraging cube map depth maps for occlusion-aware radiance injection relative to each point light [S1, unofficial]. Unlike [GPU Voxelization with Atomic Accumulation](2559-gpu-voxelization-with-atomic-accumulation.md), which uses hardware atomics to resolve race conditions and compute moving averages, this method relies on conservative rasterization and specific projection strategies to ensure complete coverage and correct lighting accumulation without explicit atomic operations for every sample [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene triangle meshes, light sources (spotlights, point lights, emissive materials), and a target 3D voxel grid.
*   **Outputs:** A sparse 3D texture where each voxel stores direct lighting radiance and opacity.

**Data Structures:**
*   **Voxel Grid:** Implemented as a sparse 3D texture using the `GL_ARB_sparse_texture` extension [S1, unofficial].
    *   **Layout:** RGBA format.
        *   Channels 0-2 (RGB): Encode the light radiance coming from the voxel [S2, unofficial].
        *   Channel 3 (A): Encodes voxel opacity (1.0 for opaque, 0.0 for empty) [S2, unofficial].
    *   **Memory Management:** Chunks of the texture can be enabled or disabled. Enabled chunks consume memory; disabled chunks do not. Only areas containing voxels are enabled to optimize memory usage [S1, unofficial].
*   **Light Data:** For point lights, a cube depth map is used to determine occlusion [S1, unofficial].

**Parameters:**
*   **Voxel Type:**
    *   *Isotropic:* Standard voxels sharing data for all faces.
    *   *Anisotropic:* Voxels with different data for each face. Increases memory usage sixfold compared to isotropic voxels but allows accurate illumination where light hits only one side of thin objects [S1, unofficial].
*   **Viewport Resolution:** Must exactly match the voxel grid scale so that fragment size matches voxel size [S1, unofficial].

**Steps:**

1.  **Vertex Transformation:**
    *   Use a vertex shader to transform vertex positions into the voxel grid coordinate system [S1, unofficial].
    *   Apply an orthographic projection matrix that captures the entire scene in visible clip-space [S1, unofficial].

2.  **Depth Testing Disable:**
    *   Disable all depth testing before rasterization. This ensures that all primitives, even occluded ones, are processed in the fragment shader [S1, unofficial].

3.  **Geometry Shader Projection:**
    *   Use a geometry shader to project every primitive onto a 2D surface [S1, unofficial].
    *   Determine the axis with the largest face area (greatest "axis shadow") when looking through the camera perspective along one of the axes [S1, unofficial].
    *   Swizzle the X, Y, and Z components and substitute the depth to zero to project the primitive triangle on the 2D surface [S1, unofficial].
    *   This projection maximizes the area and prevents cracks between voxels [S1, unofficial].

4.  **Rasterization:**
    *   Rasterize the transformed primitives.
    *   Ensure the viewport resolution exactly matches the voxel grid scale [S1, unofficial].
    *   Enable hardware conservative rasterization via the `GL_CONSERVATIVE_RASTERIZATION_NV` extension [S1, unofficial]. This expands the outer edges of triangle primitives so a fragment is considered covered if any part of it is touched by a primitive, fixing gaps where rendered geometry does not fill voxel fragments fully [S1, unofficial].

5.  **Fragment Shader Processing:**
    *   For every surface, set all intersecting voxels to fully opaque [S2, unofficial].
    *   Calculate direct light information using the Lambertian model [S1, unofficial].
    *   For spotlights, calculate diffuse light using the Lambertian model, light falloff, and shadow maps [S2, unofficial].
    *   For point lights, use the cube depth map to inject lighting data with occlusion awareness [S1, unofficial].
    *   Add light emitted by emissive materials into the 3D texture [S2, unofficial].
    *   Store the computed RGBA value into the 3D texture using OpenGL's `imageStore()` command [S2, unofficial].
    *   Discard fragments (do not write to a conventional 2D render target) [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
def vertex_shader(position, model_matrix, voxel_grid_matrix):
    # Transform vertex positions into the voxel grid coordinate system
    # using an orthographic projection matrix that captures the entire scene
    transformed_position = voxel_grid_matrix * model_matrix * position
    return transformed_position

# Geometry Shader
def geometry_shader(triangle_vertices):
    # Calculate face areas for each axis projection
    # Axis X, Y, Z projections
    area_x = calculate_face_area_projection(triangle_vertices, axis='x')
    area_y = calculate_face_area_projection(triangle_vertices, axis='y')
    area_z = calculate_face_area_projection(triangle_vertices, axis='z')
    
    # Select axis with largest face area (greatest 'axis shadow')
    if area_x >= area_y and area_x >= area_z:
        axis = 'x'
    elif area_y >= area_x and area_y >= area_z:
        axis = 'y'
    else:
        axis = 'z'
        
    # Swizzle components and substitute depth to zero
    # Project primitive on 2D surface
    projected_vertices = []
    for v in triangle_vertices:
        if axis == 'x':
            projected_vertices.append(vec4(v.y, v.z, 0.0, 1.0))
        elif axis == 'y':
            projected_vertices.append(vec4(v.x, v.z, 0.0, 1.0))
        else:
            projected_vertices.append(vec4(v.x, v.y, 0.0, 1.0))
            
    return projected_vertices

# Fragment Shader
def fragment_shader(fragment_coords, voxel_grid_texture, light_sources):
    # fragment_coords match voxel grid coordinates due to viewport resolution match
    
    # Initialize output
    light_radiance = vec3(0.0)
    opacity = 0.0
    
    # Set voxel to fully opaque
    opacity = 1.0
    
    # Calculate direct lighting
    for light in light_sources:
        if light.type == 'spotlight':
            # Calculate diffuse light using Lambertian model
            # Include light falloff and shadow map lookup
            diffuse = calculate_lambertian_diffuse(light, fragment_coords)
            falloff = calculate_light_falloff(light, fragment_coords)
            shadow = check_shadow_map(light, fragment_coords)
            light_radiance += diffuse * falloff * shadow
            
        elif light.type == 'point':
            # Use cube depth map for occlusion-aware radiance injection
            radiance = inject_point_light_radiance(light, fragment_coords)
            light_radiance += radiance
            
        elif light.type == 'emissive':
            # Add light emitted by emissive materials
            light_radiance += light.emission_color
            
    # Store RGBA value into 3D texture
    # RGB: light radiance, A: opacity
    output_color = vec4(light_radiance, opacity)
    
    # Discard fragment (do not write to screen)
    discard_fragment()
    
    return output_color

def calculate_face_area_projection(vertices, axis):
    # Helper to calculate projected area of triangle on plane perpendicular to axis
    # Returns scalar area value
    pass

def calculate_lambertian_diffuse(light, coords):
    # Standard Lambertian reflection: max(0, dot(N, L)) * light_intensity
    pass

def calculate_light_falloff(light, coords):
    # Inverse square or linear falloff based on distance
    pass

def check_shadow_map(light, coords):
    # Sample shadow map to determine if voxel is in shadow
    pass

def inject_point_light_radiance(light, coords):
    # Sample cube depth map to determine visibility
    # Return radiance if visible
    pass
```

## Subtopics

### [Point Light Voxel Injection](21109-point-light-voxel-injection.md)
Point Light Voxel Injection accumulates point light radiance into a 3D voxel grid by rendering scene geometry to a depth buffer, then dispatching a compute shader that reads voxel albedo/normal/radiance textures and the light's cube depth map to inject lighting data. It differs from standard geometry voxelization by leveraging cube map depth maps for occlusion-aware radiance injection relative to each point light.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Depends on the number of enabled chunks in the sparse 3D texture. Memory is only consumed for areas with voxels [S1, unofficial]. Anisotropic voxels increase memory usage sixfold compared to isotropic voxels [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:**
    *   Efficient memory usage via sparse textures [S1, unofficial].
    *   Accurate illumination for thin objects using anisotropic voxels [S1, unofficial].
    *   Prevents cracks and gaps through conservative rasterization and smart projection [S1, unofficial].
*   **Limitations:**
    *   Approximates only indirect diffuse lighting, not specular [S2, unofficial].
    *   Sparse textures are not optimized for very small chunks allocated, so enabling only areas with voxels might be slow [S1, unofficial].
    *   Requires specific OpenGL extensions (`GL_ARB_sparse_texture`, `GL_CONSERVATIVE_RASTERIZATION_NV`) [S1, unofficial].
*   **When to use:** When direct lighting needs to be precomputed and stored for later indirect lighting calculations (e.g., cone tracing), and memory efficiency is important [S1, unofficial].
*   **When not to use:** When specular lighting or directional light directionality within the voxel is required [S2, unofficial].

## Background
The technique was developed as part of research into real-time voxel cone tracing. It builds on the concept of voxelizing scene geometry to store lighting information, as described in works evaluating voxel cone tracing for real-time applications [S1, unofficial]. The specific implementation details, including the use of sparse textures and conservative rasterization, are derived from NVIDIA Developer articles and practical implementations in projects like GIDemo [S1, unofficial][S2, unofficial].

## Sources
[S1] Voxel Cone Tracing Evaluation for Real-Time Applications, https://kth.diva-portal.org/smash/get/diva2:1886204/FULLTEXT01.pdf, fetched 2026-10-27 (unofficial)
[S2] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
