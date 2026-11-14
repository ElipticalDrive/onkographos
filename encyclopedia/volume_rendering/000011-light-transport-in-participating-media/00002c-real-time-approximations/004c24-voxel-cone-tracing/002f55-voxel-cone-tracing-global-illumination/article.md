# Voxel Cone Tracing Global Illumination

Also called: Crassin's voxel cone tracing, Voxel Cone Traced Global Illumination, VCTGI

## Overview
Voxel Cone Tracing Global Illumination (VCTGI) is a real-time global illumination technique that approximates indirect lighting by storing scene radiance in a voxel grid and sampling it via cone tracing [S3, unofficial]. It differs from diffuse-only voxel GI by supporting specular and glossy reflections with significantly lower performance costs, requiring only a single cone trace per glossy material [S1, unofficial]. Unlike sparse voxel octrees, this implementation typically uses a plain 3D texture to support moving cameras without expensive reconstruction [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scene geometry (triangles), surface properties (color, normal, emission), light sources, and camera position.
*   **Output:** A final rendered image with indirect diffuse and specular lighting.

**Data Structures:**
1.  **Voxel Grid:** A plain 3D texture (not a sparse octree) storing view-independent radiance. The resolution is chosen to balance memory and accuracy.
2.  **Mipmap Chain:** Generated from the radiance texture using trilinear interpolation. Anisotropic mipmaps may be used to reduce light bleeding [S3, unofficial].

**Steps:**

1.  **Voxelization:**
    *   Project triangles into the voxel grid. To maximize covered area and minimize holes, project triangles in the geometry shader based on the greatest component of the surface normal [S3, unofficial].
        *   If the Z component is greatest, project onto the XY plane.
        *   If the X component is greatest, project onto the YZ plane.
        *   If the Y component is greatest, project onto the XZ plane.
    *   Use **Conservative Rasterization** to ensure triangles cover the center of pixels, preventing holes in the voxelized mesh [S3, unofficial].

2.  **Radiance Volume Construction:**
    *   Combine surface color, normal, emission, and direct lighting (including shadows) into the voxel grid [S3, unofficial].
    *   **Concurrency Handling:** Since shaders execute concurrently in warps, unsynchronized writes to the voxel texture can cause race conditions. Mitigation strategies include:
        *   Using a Compare-And-Swap (CAS) loop with a moving average, converting RGBA8 to R32UI before accumulation [S3, unofficial].
        *   Using `NV_shader_atomic_float` extensions to perform `imageAtomicAdd` on floats [S3, unofficial].

3.  **Mipmap Generation:**
    *   Generate mipmaps of the radiance texture. Anisotropic mipmaps are preferred to reduce light bleeding artifacts [S3, unofficial].

4.  **Lighting (Cone Tracing):**
    *   For each fragment, shoot cones around the hemisphere oriented along the surface normal [S3, unofficial].
    *   **Cone Types:**
        *   **Diffuse:** Large aperture, samples lower mipmaps (higher resolution) to capture broad indirect lighting.
        *   **Specular:** Smaller aperture, samples lower mipmaps to capture glossy reflections. Only one cone is typically needed for glossy materials [S1, unofficial].
        *   **Ambient Occlusion:** Used for occlusion calculations.
    *   **Aperture:** The cone aperture determines the rate at which the diameter increases with distance from the surface. The diameter selects the appropriate mipmap level [S3, unofficial].
    *   **Accumulation:** Traverse the voxel structure using front-to-back accumulation. For each step, update color and opacity using the following formulas [S3, unofficial]:
        $$
        \text{color} = \text{opacity} \cdot \text{color} + (1 - \text{opacity}) \cdot \text{read\_opacity} \cdot \text{read\_color}
        $$
        $$
        \text{opacity} = \text{opacity} + (1 - \text{opacity}) \cdot \text{read\_opacity}
        $$
    *   **Weighting:** Average the results from multiple cones (Monte Carlo style), weighting each partial result by the cosine similarity between the cone direction and the surface normal [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Voxelization (Geometry Shader)
def voxelize_triangle(triangle, voxel_grid, conservative_rasterization=True):
    # Determine projection plane based on normal
    normal = triangle.normal
    if abs(normal.z) > abs(normal.x) and abs(normal.z) > abs(normal.y):
        projection_plane = 'XY'
    elif abs(normal.x) > abs(normal.y):
        projection_plane = 'YZ'
    else:
        projection_plane = 'XZ'
    
    # Project vertices to 2D
    projected_vertices = project_to_plane(triangle.vertices, projection_plane)
    
    # Rasterize with conservative rasterization to fill gaps
    if conservative_rasterization:
        fragments = conservative_rasterize(projected_vertices, voxel_grid.resolution)
    else:
        fragments = standard_rasterize(projected_vertices, voxel_grid.resolution)
        
    for frag in fragments:
        # Accumulate radiance into voxel
        # Handle concurrency via CAS or atomic floats
        voxel_grid.atomic_add(frag.position, frag.color, frag.emission)

# Step 2: Lighting (Fragment Shader)
def trace_cone(origin, direction, aperture, voxel_grid):
    opacity = 0.0
    accumulated_color = vec3(0.0)
    
    # Traverse along the cone
    # Step size depends on aperture and desired precision
    step_size = calculate_step_size(aperture)
    
    for distance in range(0, max_distance, step_size):
        current_pos = origin + direction * distance
        current_aperture = calculate_aperture(distance, aperture)
        
        # Select mipmap level based on cone diameter
        mip_level = select_mipmap(current_aperture, voxel_grid.resolution)
        
        # Sample voxel
        read_color, read_opacity = voxel_grid.sample(current_pos, mip_level)
        
        # Front-to-back accumulation
        # color = opacity*color + (1 - opacity) * read_opacity * read_color
        # opacity = opacity + (1 - opacity) * read_opacity
        accumulated_color = opacity * accumulated_color + (1 - opacity) * read_opacity * read_color
        opacity = opacity + (1 - opacity) * read_opacity
        
        if opacity >= 1.0:
            break
            
    return accumulated_color, opacity

def compute_global_illumination(fragment, voxel_grid):
    normal = fragment.normal
    diffuse_color = vec3(0.0)
    specular_color = vec3(0.0)
    
    # Diffuse GI: Multiple cones
    num_diffuse_cones = 16
    for i in range(num_diffuse_cones):
        direction = sample_hemisphere(normal)
        # Large aperture for diffuse
        aperture = calculate_diffuse_aperture(i, num_diffuse_cones)
        color, _ = trace_cone(fragment.position, direction, aperture, voxel_grid)
        weight = dot(direction, normal) # Cosine weighting
        diffuse_color += color * weight
        
    diffuse_color /= num_diffuse_cones
    
    # Specular GI: Single cone for glossy
    # Small aperture for specular
    specular_aperture = calculate_specular_aperture()
    specular_direction = reflect(-fragment.view_direction, normal)
    specular_color, _ = trace_cone(fragment.position, specular_direction, specular_aperture, voxel_grid)
    
    return diffuse_color, specular_color
```

## Operating characteristics
*   **Performance:** In a test with 1024x768 resolution and ~70,000 triangles, frame times were: Voxelization 0.97ms, Radiance volume 4.96ms, Isotropic mipmaps 0.09ms, Lighting 5.1ms [S3, unofficial].
*   **Complexity:** Lighting complexity is proportional to the number of cones traced per fragment. Diffuse GI requires many cones (e.g., 16), while specular GI requires only one [S1, unofficial].
*   **Strengths:** Real-time performance; supports specular reflections; simple implementation with plain 3D textures; no complex octree reconstruction for moving cameras [S3, unofficial].
*   **Limitations:** Voxel artifacts may appear on highly polished reflectors [S1, unofficial]; light bleeding can occur if anisotropic mipmaps are not used [S3, unofficial]; requires conservative rasterization to avoid holes [S3, unofficial].
*   **When to use:** Real-time applications requiring both diffuse and specular global illumination, especially with moving cameras [S3, unofficial].
*   **When not to use:** Scenes with extremely high-frequency indirect lighting details that require higher resolution than the voxel grid can provide, or where voxel artifacts are unacceptable [S1, unofficial].

## Background
The Voxel Cone Tracing global illumination technique was proposed by Cyril Crassin et al. in 2011 [S2, unofficial][S3, unofficial]. It was introduced to bring global illumination into the realm of real-time rendering by leveraging the efficiency of voxel grids and cone tracing [S3, unofficial]. The technique has been used in commercial video games, such as *The Tomorrow Children* [S3, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] domme/VoxelConeTracing, https://github.com/domme/VoxelConeTracing, fetched 2026-10-26 (unofficial)
[S3] KKarol01/VoxelConeTracingGI, https://github.com/KKarol01/VoxelConeTracingGI, fetched 2026-10-26 (unofficial)
