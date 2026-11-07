# Image-order Rendering Loop

Also called: Image order approach, Image-Order Volume Ray Casting, Main Rendering Loop, Octree Rendering Loop

## Overview
The Image-order Rendering Loop is the primary execution structure for direct volume ray casting, iterating over each pixel of the output image to generate a ray and compute its final color contribution [S1, S2, unofficial]. It solves the problem of mapping 3D volumetric data to a 2D screen by decoupling pixel processing, allowing for flexible shading modes and adaptive sampling [S1, S2, unofficial]. Unlike object-order methods (rasterization), this approach is described as more "CPU-like" but offers correct perspective projection, single-pass ray casting, and full 32-bit compositing precision without the need for multiple passes or early-z testing [S1, unofficial]. It builds on [Ray Traversal and Integration Schemes](8341-ray-traversal-and-integration-schemes.md) by providing the outer control flow that invokes the raymarcher for each pixel.

## How it works
The method implements a nested loop structure: an outer loop iterates over image pixels, and an inner loop (or function call) performs ray generation, traversal, and compositing.

### Inputs and Outputs
- **Inputs**: Camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$, projection mode), volume data structure (e.g., voxel grid or octree), shading mode enum, and rendering configuration (max depth, samples per pixel).
- **Outputs**: A 2D image buffer where each pixel contains the accumulated color $C_{final}$.

### Data Structures
The core data structure for configuration is the `Renderer` class [S3, unofficial]:
- `int maxDepth`: Maximum recursion depth for ray tracing. Default: 100 [S3, unofficial].
- `int samplesPerPixel`: Number of samples per pixel for anti-aliasing. Default: 500 [S3, unofficial].
- `RGB background`: Background color. Default: `{0, 0, 0}` [S3, unofficial].

The shading mode is an enum `ShadingMode` with values: `TRACE`, `DEPTH`, `DEPTH_HIT`, `DIFFUSE`, `NORMAL`, `INDEX` [S2, unofficial].
The projection mode is an enum `ProjectionMode` with values: `PERSPECTIVE`, `ORTHOGRAHIC` [S2, unofficial].

### Steps
1. **Initialization**: Set `CompletedSamples = 0`. Calculate light bounds $p_0, p_1$ [S2, unofficial].
2. **Camera Transformation**: Apply rotation to the camera position and direction [S2, unofficial].
3. **Pixel Loop**: Iterate over $y$ from $0$ to $vheight-1$ and $x$ from $0$ to $vwidth-1$ [S2, unofficial].
4. **Ray Generation**:
   - Calculate normalized coordinates $rx, ry$ from pixel coordinates [S2, unofficial].
   - Transform position `pos` [S2, unofficial].
   - If `ProjectionMode == PERSPECTIVE`:
     - $\mathbf{rayOrigin} = \text{cameraPos}$
     - $\mathbf{rayDirection} = \text{cameraDir} + \text{pos}$ [S2, unofficial]
   - If `ProjectionMode == ORTHOGRAPHIC`:
     - $\mathbf{rayOrigin} = \text{cameraPos} + \text{pos}$
     - $\mathbf{rayDirection} = \text{cameraDir}$ [S2, unofficial]
5. **Index Calculation**: Compute linear index $index = x + y \times vwidth$ [S2, unofficial].
6. **Shading/Rendering**:
   - If `ShadingMode == TRACE`:
     - Accumulate color by calling `ShadePathTrace` and averaging with previous samples [S2, unofficial].
   - Else:
     - Call `Raymarch` [S2, unofficial].
     - If a hit is detected, shade based on `ShadingMode`:
       - `DEPTH`: Color modulated by depth [S2, unofficial].
       - `DEPTH_HIT`: Color modulated by distance from camera to hit point [S2, unofficial].
       - `DIFFUSE`: Standard diffuse shading [S2, unofficial].
       - `NORMAL`: Visualize surface normals [S2, unofficial].
       - `INDEX`: Visualize material indices [S2, unofficial].
     - If no hit but ray intersects light AABB, write light color [S2, unofficial].
     - Else, write dark color $(25, 25, 25)$ [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_loop(scene, camera, image, shading_mode, projection_mode):
    """
    Main rendering loop for image-order volume ray casting.
    """
    completed_samples = 0
    light_bounds = calculate_light_bounds(scene) # p0, p1
    
    # Apply rotation to camera
    camera.apply_rotation()
    
    vheight = image.height
    vwidth = image.width
    
    for y in range(vheight):
        for x in range(vwidth):
            # Calculate normalized pixel coordinates
            rx, ry = calculate_pixel_coords(x, y, vwidth, vheight)
            
            # Transform position based on projection
            pos = transform_position(rx, ry, projection_mode)
            
            # Generate ray
            if projection_mode == PERSPECTIVE:
                ray_origin = camera.pos
                ray_direction = normalize(camera.dir + pos)
            else: # ORTHOGRAPHIC
                ray_origin = camera.pos + pos
                ray_direction = camera.dir
            
            index = x + y * vwidth
            
            if shading_mode == TRACE:
                # Path tracing with accumulation
                color = shade_path_trace(ray_origin, ray_direction, scene, depth=100)
                image[index] = average_samples(image[index], color)
                completed_samples += 1
            else:
                # Standard raymarching
                hit, hit_point, normal = raymarch(ray_origin, ray_direction, scene)
                
                if hit:
                    if shading_mode == DEPTH:
                        depth = distance(camera.pos, hit_point)
                        color = apply_depth_shading(hit_point.color, depth)
                    elif shading_mode == DEPTH_HIT:
                        depth = distance(camera.pos, hit_point)
                        color = apply_depth_hit_shading(hit_point.color, camera.pos, hit_point)
                    elif shading_mode == DIFFUSE:
                        color = apply_diffuse_shading(hit_point.color, hit_point.normal, scene.lights)
                    elif shading_mode == NORMAL:
                        color = apply_normal_shading(hit_point.normal)
                    elif shading_mode == INDEX:
                        color = apply_index_shading(hit_point.material_index)
                else:
                    # Check if ray intersects light AABB
                    if ray_intersects_aabb(ray_origin, ray_direction, light_bounds):
                        color = scene.light_color
                    else:
                        color = RGB(25, 25, 25)
                
                image[index] = color

def shade_path_trace(ro, rd, scene, depth):
    """
    Recursive path tracing for TRACE shading mode.
    """
    if depth <= 0:
        return RGB(0, 0, 0)
    
    intersection = False
    hit_point = None
    normal = None
    
    # Try raymarching first
    if raymarch(ro, rd, hit_point, normal, scene):
        intersection = True
    # Fallback to plane intersection if needed
    elif ray_plane_intersection(scene.plane_normal, scene.plane_distance, ro, rd, hit_point):
        normal = scene.plane_normal
        intersection = True
    
    if intersection:
        # Generate bounce direction
        bounce_target = hit_point + random_in_hemisphere(normal)
        return shade_path_trace(hit_point, bounce_target, scene, depth - 1)
    
    return RGB(0, 0, 0)

def apply_depth_shading(color, depth):
    """
    Modulates green channel by depth.
    """
    result = color
    result.g *= (depth / 2.0)
    return result

def apply_depth_hit_shading(color, camera_pos, hit):
    """
    Modulates green channel by distance from camera to hit.
    """
    depth = distance(camera_pos, hit)
    result = color
    result.g *= (depth / 2.0)
    return result

def apply_normal_shading(normal):
    """
    Visualizes normal components as RGB.
    """
    r = 1.0 if normal.x != 0 else 0.0
    g = 1.0 if normal.y != 0 else 0.0
    b = 1.0 if normal.z != 0 else 0.0
    return RGB(r, g, b)
```

### Parameters
- `Shader Model 3.0` or higher is required to enable conditional loops (early loop exit) in fragment shaders, which replaces multiple passes and early-z testing [S1, unofficial].
- `maxDepth`: Default 100 [S3, unofficial].
- `samplesPerPixel`: Default 500 [S3, unofficial].

### Edge Cases and Pitfalls
- **Precision**: The method relies on 32-bit compositing precision, which is maintained by avoiding intermediate buffer swaps [S1, unofficial].
- **Empty Space**: If the ray does not hit the volume and does not intersect the light AABB, it defaults to a dark color $(25, 25, 25)$ [S2, unofficial].
- **Plane Intersection**: When falling back to plane intersection, a small offset ($10^{-4}$) is added to the hit point along the normal to prevent self-intersection artifacts [S2, unofficial].

## Subtopics

### [Terminal Rendering Pipeline](3434-terminal-rendering-pipeline.md)
The Terminal Rendering Pipeline renders a 3D scene to a text-based terminal buffer by iterating over pixels, mapping them to Normalized Device Coordinates (NDC), casting rays via a raymarcher, and applying ASCII/ANSI styling based on depth and lighting. It differs from standard terminal rendering by explicitly accounting for character aspect ratio when converting pixel indices to NDC and using ANSI escape codes to modulate brightness based on depth percentage, effectively acting as a low-resolution, text-based raymarcher.

### [Diffuse Octree Rendering](4989-diffuse-octree-rendering.md)
Diffuse Octree Rendering is an image-order raycasting technique that traverses an octree to locate the first leaf node containing data for each pixel, then computes surface lighting using a diffuse and Phong specular model with distance attenuation. It differs from sibling methods by relying on a coarse-to-fine octree traversal to skip empty space entirely, rather than marching through a dense voxel grid.

### Renderer configuration
The Renderer configuration is a top-level rendering controller managing ray depth, sampling rate, and background color [S3, unofficial]. It defines the `Renderer` class with fields `int maxDepth = 100`, `int samplesPerPixel = 500`, and `RGB background{0, 0, 0}` [S3, unofficial]. It provides methods `RGB rayColor(const Ray &ray, const Scene &scene, int depth) const` and `void render(const Scene &scene, const Camera &camera, Image &image) const` [S3, unofficial].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of pixels ($W \times H$) and the number of samples per pixel. The inner raymarching cost depends on the volume density and traversal scheme (e.g., voxel stepping vs. octree traversal) [S1, S2, unofficial].
- **Memory Complexity**: Dominated by the storage of the volume data structure (voxel grid or octree) and the output image buffer [S1, S3, unofficial].
- **Performance**: The method is described as "CPU-like" in logic but is highly flexible, supporting adaptive sampling and correct perspective projection in a single pass [S1, unofficial]. It avoids the overhead of multiple passes required by object-order methods [S1, unofficial].
- **Strengths**: Correct perspective, single-pass ray casting, 32-bit compositing precision, flexibility for adaptive sampling [S1, unofficial].
- **Limitations**: Performance can be lower than rasterization for simple scenes due to the per-pixel ray traversal. Requires Shader Model 3.0+ for efficient conditional loops on GPU [S1, unofficial].
- **When to use**: Suitable for high-quality volume rendering, virtual endoscopy, fly-throughs, and integration into perspective scenes like games [S1, unofficial].
- **When not to use**: When real-time performance on older hardware without Shader Model 3.0 support is critical, or when object-order rasterization is sufficient for simple surfaces.

## Background
The image-order approach for volume ray casting has been widely adopted in GPU-based rendering since the early 2000s. Key implementations include rectilinear grid ray casting by Krüger and Westermann (2003), Röttger et al. (2003), Green (2004) in the NVIDIA SDK, Stegmaier et al. (2005), and Scharsach et al. (2006) [S1, unofficial]. It is enabled by the availability of conditional loops in fragment shaders (Shader Model 3.0 and higher), which allowed for single-pass ray casting with early loop exit, replacing multiple passes and early-z testing [S1, unofficial]. The NVIDIA SDK and CUDA SDK provided examples of computing ray intersections with bounding boxes, marching along rays, and compositing [S1, unofficial].

## Sources
[S1] GPU-Based Volume Ray-Casting with Advanced Illumination, https://doi.org/10.2312/egt.20091069, fetched 2026-10-26 (unofficial)
[S2] cianjinks/Sparse-Voxel-Octree, https://github.com/cianjinks/Sparse-Voxel-Octree, fetched 2026-10-26 (unofficial)
[S3] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
