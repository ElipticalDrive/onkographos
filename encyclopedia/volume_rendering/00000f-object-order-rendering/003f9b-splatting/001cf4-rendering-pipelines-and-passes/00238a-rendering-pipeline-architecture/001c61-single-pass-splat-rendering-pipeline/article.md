# Single-Pass Splat Rendering Pipeline

Also called: Single-Pass High-Quality Splatting

## Overview
The Single-Pass Splat Rendering Pipeline is a real-time rendering technique that combines splat and mesh rendering components to enable high-quality shader effects (reflection, refraction) in a single rendering pass [S1, unofficial]. It solves the problem of multi-pass blending artifacts and high memory/workload associated with dense point sets by reorganizing input data into elevation map patches [S1, unofficial]. Unlike sibling methods that reconstruct smooth attributes by blending in screen space (requiring at least three passes and causing blurring), this method interpolates attributes at an earlier stage in object space using patch information, resulting in a single pass and improved normal/attribute quality [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D point set representing the model, optionally with per-point attributes (color, normal).
*   **Output:** A final image rendered in a single pass.

**Data Structures:**
The model is decomposed into patches. Each patch is defined by:
1.  **Plane Parameters:** A 3D point (origin), a normal vector, and a grid resolution [S1, unofficial].
2.  **Height Field:** A rectangular array of height values (elevation map) sampled on a regular grid over the planar domain [S1, unofficial]. Some grid points may be flagged with a special height value indicating no point exists above that grid location [S1, unofficial].
3.  **Attribute Maps:** Optional texture maps (e.g., color, normal) stored as texture objects for each patch individually [S1, unofficial].
4.  **Transformation Matrix:** One 4x4 matrix per patch [S1, unofficial].

**Storage:**
*   **VBO (Vertex Buffer Object):** Stores point positions and elevation data for each patch. Point positions on the patch plane are expressed by a single index value (combined row and column address in the sampling grid) [S1, unofficial].
*   **Index Buffer:** A linear list $1, 2, \dots, n$, where $n$ is the number of points in the patch. No connectivity encoding is needed [S1, unofficial].
*   **Texture Objects:** Store additional attribute maps [S1, unofficial].

**Algorithm Steps:**

1.  **Patch Construction (Pre-processing):**
    *   Partition the input 3D point set into patches. Each patch is parameterized as an elevation map over a planar domain [S1, unofficial].
    *   Resample the surface patch on a regular grid using Moving Least Squares (MLS) surface reconstruction [S1, unofficial].
    *   **Overlap Handling:** To prevent gaps due to MLS limitations, patches are made to slightly overlap. An input point near a boundary is included in a patch if its $\epsilon$-neighborhood intersects the point set of the region, where $\epsilon > 0$ [S1, unofficial].
    *   Resample attribute fields (normals, textures) onto the regular grid [S1, unofficial].

2.  **Upload to GPU:**
    *   Upload VBOs (elevation data) and Texture Objects (attributes) to graphics memory [S1, unofficial].

3.  **Rendering Loop:**
    *   Iterate over all patches.
    *   For each patch:
        *   Address the respective texture objects.
        *   Trigger the corresponding VBO for rendering using the linear index buffer [S1, unofficial].

4.  **Vertex Shader:**
    *   **Translation:** Translate point positions according to their elevation values from the VBO.
    *   **Rotation:** Rotate the translated points using the patch's 4x4 transformation matrix to place them on the surface of the original model [S1, unofficial].
    *   **Radius Calculation:** Set the radius of each point (point sprite) based on the sampling step size $\mu$ and elevation variance to prevent holes [S1, unofficial]. The base radius is $\frac{1}{2}\sqrt{2}\mu$. Radii are increased in areas with high variance between neighboring elevation values to account for spatial gaps [S1, unofficial].

5.  **Rasterization:**
    *   Rasterize the points as point sprites [S1, unofficial].

6.  **Fragment Shader:**
    *   **Splat Simulation:** Transform point-sprite fragments to simulate splatting, as point sprites are screen-aligned primitives [S1, unofficial].
    *   **Continuous Attribute Interpolation:**
        *   Determine per-fragment UV coordinates by projecting the fragment back onto the patch plane [S1, unofficial].
        *   Interpolate attributes (normals, textures) from nearby points using the patch's regular grid structure. This replaces screen-space blending with object-space interpolation [S1, unofficial].
    *   **Illumination:** Compute final fragment color by summing contributions from lighting, texture mapping, reflectance mapping (e.g., environment cube maps), and refraction mapping [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-processing (CPU)
def build_patches(point_set):
    patches = []
    for region in partition(point_set):
        # Resample using MLS
        grid, heights = resample_mls(region)
        # Handle overlap with epsilon
        epsilon = 0.01 # Example value
        boundary_points = get_boundary_points(region, epsilon)
        heights.extend(boundary_points)
        
        # Compute attributes
        normals = compute_normals(grid, heights)
        colors = sample_colors(grid, region)
        
        # Compute plane parameters
        plane_origin, plane_normal = fit_plane(grid)
        transform_matrix = compute_transform(plane_origin, plane_normal)
        
        patches.append({
            'grid': grid,
            'heights': heights,
            'normals': normals,
            'colors': colors,
            'transform': transform_matrix,
            'resolution': grid.resolution
        })
    return patches

# GPU Rendering (Shader Logic)
def render_patches(patches):
    for patch in patches:
        # Set up VBO and Texture Objects
        bind_vbo(patch.elevation_data)
        bind_texture(patch.color_map)
        bind_texture(patch.normal_map)
        
        # Draw using linear index buffer
        draw_indexed_linear(patch.num_points)

# Vertex Shader
void vertex_shader(in uint index, out vec3 world_pos, out float radius):
    # Get grid coordinates from index
    row = index / patch.resolution
    col = index % patch.resolution
    
    # Get base position on plane
    base_pos = patch.origin + row * patch.u_vec + col * patch.v_vec
    
    # Get elevation
    height = patch.heights[row][col]
    
    # Translate by elevation
    translated_pos = base_pos + height * patch.normal
    
    # Rotate/Transform to world space
    world_pos = patch.transform * translated_pos
    
    # Calculate radius
    # Base radius depends on sampling step size mu
    mu = patch.resolution / patch.physical_size
    base_radius = 0.5 * sqrt(2.0) * mu
    
    # Adjust for elevation variance to prevent holes
    h_center = height
    h_neighbors = get_neighbor_heights(row, col, patch.heights)
    variance = compute_variance(h_neighbors)
    radius = base_radius * (1.0 + variance * scale_factor)

# Fragment Shader
void fragment_shader(in vec3 world_pos, in vec2 uv, out vec4 color):
    # Transform fragment to simulate splatting
    # (Specific transformation depends on point sprite implementation)
    
    # Interpolate attributes from patch grid
    # Project fragment back to patch plane to get grid coordinates
    grid_uv = project_to_patch_plane(world_pos, patch)
    
    # Bilinear interpolation of normals and colors from texture objects
    normal = texture(patch.normal_map, grid_uv)
    diffuse = texture(patch.color_map, grid_uv)
    
    # Compute lighting
    light_color = compute_phong_shading(world_pos, normal)
    
    # Compute reflection/refraction
    view_dir = normalize(camera_pos - world_pos)
    reflect_color = texture(cube_map, reflect(view_dir, normal))
    refract_color = texture(cube_map, refract(view_dir, normal, ior))
    
    # Sum contributions
    final_color = light_color * diffuse + reflect_color + refract_color
    color = final_color
```

**Parameters:**
*   $\epsilon$ (Epsilon): A positive value used during patch construction to include input points near a patch boundary if their $\epsilon$-neighborhood intersects the point set of the region, ensuring patches slightly overlap to prevent gaps [S1, unofficial].
*   $\mu$ (Sampling step size): Determines the base splat radius [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Gaps between patches:** Moving Least Squares resampling cannot guarantee no gaps. Patches must overlap slightly using $\epsilon$ to ensure continuous error-free rendering [S1, unofficial].
*   **Holes due to elevation variance:** If elevation values vary significantly between neighbors, the base radius may be insufficient. Radii must be increased in these areas [S1, unofficial].

## Operating characteristics
*   **Complexity:** The pipeline consists of one single rendering pass, simplifying overall rendering complexity compared to current GPU-based splatting techniques which require at least three rendering passes [S1, unofficial].
*   **Performance:** Aggressively utilizes practical components of both splat and mesh rendering. Avoids performance-hungry GPU functionality such as multiple render targets and additional blending passes [S1, unofficial].
*   **Quality:** Improves visual quality of reconstructed attributes (normals, texture coordinates) by interpolating in object space rather than blending in screen space [S1, unofficial]. Allows for enhanced shader effects (reflection, refraction) without densely sampled models [S1, unofficial].
*   **Memory:** The resampled model is stored completely on graphics hardware. Connectivity is implicit and reconstructed in the fragment shader, requiring no additional memory for connectivity structures [S1, unofficial].

## Background
The Single-Pass Splat Rendering Pipeline was introduced by Tilo Ochotta, Stefan Hiller, and Dietmar Saupe in September 2006 [S1, unofficial]. It was published as Konstanzer Schriften in Mathematik und Informatik Nr. 219 [S1, unofficial]. The method builds on previous work regarding elevation map data structures for point-sampled surfaces [S1, unofficial].

## Sources
[S1] Single-Pass High-Quality Splatting, http://nbn-resolving.de/urn:nbn:de:bsz:352-opus-22395, fetched 2026-10-26 (unofficial)
