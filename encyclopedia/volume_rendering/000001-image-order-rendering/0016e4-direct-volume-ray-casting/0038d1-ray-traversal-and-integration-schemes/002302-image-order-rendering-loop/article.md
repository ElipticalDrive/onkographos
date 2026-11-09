# Image-order Rendering Loop

Also called: Image order approach, Image-Order Volume Ray Casting, Main Rendering Loop, Octree Rendering Loop

## Overview
The Image-order Rendering Loop is the primary control structure for direct volume ray casting that iterates over output pixels to generate camera rays and invoke a raymarcher for shading [S2, S1, unofficial]. It solves the problem of mapping 3D volumetric data to a 2D image by handling ray generation, traversal, and compositing in a single pass per pixel [S1, unofficial]. Unlike object-order methods (rasterization), this approach is described as more "CPU-like" in logic but offers correct perspective projection, adaptive sampling capabilities, and full 32-bit compositing precision without requiring intermediate buffers [S1, unofficial]. It builds on [Ray Traversal and Integration Schemes](8341-ray-traversal-and-integration-schemes.md) by providing the outer iteration context for those schemes.

## How it works
The method operates as a nested loop structure: an outer loop over image pixels, an optional loop for multi-sampling anti-aliasing, and an inner call to a raymarching routine that traverses the volume data structure.

### Inputs and Outputs
- **Inputs**:
  - `vwidth`, `vheight`: Dimensions of the output image buffer [S2, unofficial].
  - `cameraPos`: Camera position in world space [S2, unofficial].
  - `cameraDir`: Camera forward direction vector [S2, unofficial].
  - `ProjectionMode`: Enum, either `PERSPECTIVE` or `ORTHOGRAPHIC` [S2, unofficial].
  - `ShadingMode`: Enum determining shading behavior (`TRACE`, `DEPTH`, `DEPTH_HIT`, `DIFFUSE`, `NORMAL`, `INDEX`) [S2, unofficial].
  - `NumSamples`: Number of samples per pixel for anti-aliasing [S2, unofficial].
  - `ShadingMode == Shade::TRACE`: Boolean flag enabling path tracing accumulation [S2, unofficial].
  - Volume data structure (e.g., Octree, Voxel Grid) and Transfer Function (implicit in `Raymarch`) [S2, S1, unofficial].
- **Outputs**:
  - `Image`: A buffer of size `vwidth * vheight` containing final pixel colors [S2, unofficial].

### Data Structures
- **Pixel Index**: Calculated as `index = x + y * vwidth` [S2, unofficial].
- **Ray Parameters**:
  - `rayOrigin`: Starting point of the ray [S2, unofficial].
  - `rayDirection`: Normalized direction vector of the ray [S2, unofficial].
- **Shading State**:
  - `CompletedSamples`: Counter for accumulated samples in path tracing mode [S2, unofficial].
  - `hit`: Intersection point vector [S2, unofficial].
  - `normal`: Surface normal vector at intersection [S2, unofficial].

### Numbered Steps
1. **Initialization**: Set `CompletedSamples = 0` [S2, unofficial]. Calculate light bounds `p0, p1` if required for light intersection checks [S2, unofficial]. Apply any necessary rotation to `cameraPos` and `cameraDir` [S2, unofficial].
2. **Sample Loop**: Loop over sample index `s` from `0` to `NumSamples - 1` [S2, unofficial].
   - If `ShadingMode == Shade::TRACE`, initialize accumulation buffer for the pixel to zero [S2, unofficial].
3. **Pixel Loop**: Loop over `y` from `0` to `vheight - 1` [S2, unofficial].
   - Loop over `x` from `0` to `vwidth - 1` [S2, unofficial].
4. **Ray Generation**:
   - Calculate normalized pixel coordinates `rx, ry` from `x, y` [S2, unofficial].
   - Transform `pos` based on projection mode [S2, unofficial].
   - If `ProjectionMode == PERSPECTIVE`:
     - `rayOrigin = cameraPos` [S2, unofficial].
     - `rayDirection = normalize(cameraDir + pos)` [S2, unofficial].
   - If `ProjectionMode == ORTHOGRAPHIC`:
     - `rayOrigin = cameraPos + pos` [S2, unofficial].
     - `rayDirection = cameraDir` [S2, unofficial].
5. **Shading Invocation**:
   - If `ShadingMode == Shade::TRACE`:
     - Call `ShadePathTrace(rayOrigin, rayDirection, hit, normal, depth)` [S2, unofficial].
     - Accumulate the returned color into the pixel buffer, averaging with previous samples [S2, unofficial].
   - Else:
     - Call `Raymarch(rayOrigin, rayDirection, hit, normal, ...)` [S2, unofficial].
     - If a hit is detected (`hit` is valid):
       - Shade based on `ShadingMode`:
         - `DEPTH`: Color green channel scaled by depth (`depth / 2.0f`) [S2, unofficial].
         - `DEPTH_HIT`: Color green channel scaled by distance from camera to hit (`distance(cameraPos, hit) / 2.0f`) [S2, unofficial].
         - `DIFFUSE`: Compute diffuse lighting using ambient and diffuse terms [S2, unofficial].
         - `NORMAL`: Map normal components to RGB (1.0 if component != 0, else 0.0) [S2, unofficial].
         - `INDEX`: Return object color [S2, unofficial].
     - If no hit but ray intersects light AABB:
       - Write light color to pixel [S2, unofficial].
     - Else:
       - Write dark color `(25, 25, 25)` to pixel [S2, unofficial].
6. **Completion**: After all pixels and samples are processed, the `Image` buffer contains the final rendered output [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def image_order_rendering_loop(vwidth, vheight, camera_pos, camera_dir, projection_mode, shading_mode, num_samples, scene):
    """
    Main rendering loop for image-order volume ray casting.
    """
    image = create_image_buffer(vwidth, vheight)
    
    # Pre-calculate light bounds if needed for light intersection checks
    light_bounds = calculate_light_bounds(scene)
    
    # Apply rotation to camera parameters if necessary [S2, unofficial]
    camera_pos = apply_rotation(camera_pos)
    camera_dir = apply_rotation(camera_dir)
    
    for s in range(num_samples):
        for y in range(vheight):
            for x in range(vwidth):
                # Calculate normalized pixel position [S2, unofficial]
                rx, ry = calculate_pixel_coordinates(x, y, vwidth, vheight)
                pos = transform_position(rx, ry, projection_mode)
                
                # Generate ray based on projection mode [S2, unofficial]
                if projection_mode == PERSPECTIVE:
                    ray_origin = camera_pos
                    ray_direction = normalize(camera_dir + pos)
                else:  # ORTHOGRAPHIC
                    ray_origin = camera_pos + pos
                    ray_direction = camera_dir
                
                index = x + y * vwidth
                
                if shading_mode == Shade::TRACE:
                    # Path tracing accumulation [S2, unofficial]
                    accumulated_color = vec3(0.0, 0.0, 0.0)
                    hit = vec3(0.0, 0.0, 0.0)
                    normal = vec3(0.0, 0.0, 0.0)
                    
                    # Call path tracer with recursion depth limit [S2, unofficial]
                    # Note: Depth parameter handling depends on specific implementation [S2, unofficial]
                    color = shade_path_trace(ray_origin, ray_direction, hit, normal, depth=100)
                    accumulated_color += color
                    CompletedSamples += 1
                    
                    # Average with previous samples [S2, unofficial]
                    if CompletedSamples > 1:
                        image[index] = (image[index] * (CompletedSamples - 1) + color) / CompletedSamples
                    else:
                        image[index] = color
                        
                else:
                    # Standard raymarching [S2, unofficial]
                    hit = vec3(0.0, 0.0, 0.0)
                    normal = vec3(0.0, 0.0, 0.0)
                    hit_detected = raymarch(ray_origin, ray_direction, hit, normal, scene)
                    
                    if hit_detected:
                        if shading_mode == DEPTH:
                            depth = length(hit - camera_pos)
                            image[index] = vec3(0.0, depth / 2.0, 0.0)
                        elif shading_mode == DEPTH_HIT:
                            depth = length(hit - camera_pos)
                            image[index] = vec3(0.0, depth / 2.0, 0.0)
                        elif shading_mode == DIFFUSE:
                            # Compute diffuse lighting [S2, unofficial]
                            ambient = vec3(0.1, 0.1, 0.1)
                            light_color = vec3(1.0, 1.0, 1.0)
                            light_dir = normalize(vec3(0.0, 1.0, 0.0)) # Example light direction
                            diffuse = max(dot(normal, light_dir), 0.0) * light_color
                            object_color = get_object_color_at_hit(hit, scene)
                            image[index] = (ambient + diffuse) * object_color
                        elif shading_mode == NORMAL:
                            # Map normal to RGB [S2, unofficial]
                            image[index] = vec3(
                                1.0 if normal.x != 0 else 0.0,
                                1.0 if normal.y != 0 else 0.0,
                                1.0 if normal.z != 0 else 0.0
                            )
                        elif shading_mode == INDEX:
                            image[index] = get_object_color_at_hit(hit, scene)
                    else:
                        # Check for light intersection [S2, unofficial]
                        if ray_intersects_aabb(ray_origin, ray_direction, light_bounds):
                            image[index] = get_light_color()
                        else:
                            image[index] = vec3(25.0, 25.0, 25.0) / 255.0 # Dark color [S2, unofficial]
                            
    return image
```

### Parameters and Typical Values
- `ShadingMode`: Enum. Values: `TRACE`, `DEPTH`, `DEPTH_HIT`, `DIFFUSE`, `NORMAL`, `INDEX` [S2, unofficial].
- `ProjectionMode`: Enum. Values: `PERSPECTIVE`, `ORTHOGRAPHIC` [S2, unofficial].
- `NumSamples`: Integer. Default not specified, but typically >1 for anti-aliasing [S2, unofficial].
- `maxDepth`: Integer. Default: 100 (used in path tracing recursion) [S3, unofficial].
- `samplesPerPixel`: Integer. Default: 500 (used in advanced path tracing variants) [S3, unofficial].

### Edge Cases and Pitfalls
- **Shader Model Requirement**: Requires Shader Model 3.0 or higher to enable conditional loops and early loop exit in fragment shaders, which is critical for performance and correctness in GPU implementations [S1, unofficial].
- **Precision**: The method supports full 32-bit compositing precision, avoiding the need for a separate compositing buffer [S1, unofficial].
- **Light Intersection**: If a ray does not hit the volume but intersects the light's AABB, the light color is written; otherwise, a dark color is used [S2, unofficial]. Failure to correctly handle the light AABB check may result in incorrect lighting in transparent regions.
- **Plane Intersection**: In some implementations, rays may intersect a floor plane if they miss the volume geometry; this requires explicit checking against floor boundaries to avoid incorrect hits [S2, unofficial].

## Subtopics

### [Terminal Rendering Pipeline](3434-terminal-rendering-pipeline.md)
The Terminal Rendering Pipeline renders a 3D scene to a text-based terminal buffer by iterating over pixels, mapping them to Normalized Device Coordinates (NDC), casting rays via a raymarcher, and applying ASCII/ANSI styling based on depth and lighting. It differs from standard terminal rendering by explicitly accounting for character aspect ratio when converting pixel indices to NDC and using ANSI escape codes to modulate brightness based on depth percentage, effectively acting as a low-resolution, text-based raymarcher.

### [Diffuse Octree Rendering](4989-diffuse-octree-rendering.md)
Diffuse Octree Rendering is an image-order raycasting technique that traverses an octree to locate the first leaf node containing data for each pixel, then computes surface lighting using a diffuse and Phong specular model with distance attenuation. It differs from sibling methods by relying on a coarse-to-fine octree traversal to skip empty space entirely, rather than marching through a dense voxel grid.

### Renderer configuration
A top-level rendering controller managing ray depth, sampling rate, and background color [S3, unofficial]. It provides a structured interface for configuring the image-order loop's parameters.

- **Definition**: Top-level rendering controller managing ray depth, sampling rate, and background color [S3, unofficial].
- **Parameters**:
  - `maxDepth`: Maximum recursion depth for ray tracing. Default: 100 [S3, unofficial].
  - `samplesPerPixel`: Number of samples per pixel for anti-aliasing/noise reduction. Default: 500 [S3, unofficial].
- **Layout**:
  - Class `Renderer`.
  - Fields: `int maxDepth = 100`, `int samplesPerPixel = 500`, `RGB background{0, 0, 0}` [S3, unofficial].
  - Methods: `RGB rayColor(const Ray &ray, const Scene &scene, int depth) const`, `void render(const Scene &scene, const Camera &camera, Image &image) const` [S3, unofficial].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of pixels ($O(W \times H)$) and the number of samples per pixel ($O(S)$). The cost per sample depends on the ray traversal complexity (e.g., $O(\log N)$ for octrees, $O(1)$ per step for regular grids) [S1, S2, unofficial].
- **Memory Complexity**: $O(W \times H)$ for the image buffer. Additional memory depends on the volume data structure (e.g., octree nodes) [S1, S2, unofficial].
- **Measured Performance**: Enables single-pass ray casting with 32-bit compositing precision, avoiding the overhead of multiple passes and early-z testing required in object-order approaches [S1, unofficial].
- **Scaling**: Scales well with resolution and sample count, but performance is heavily dependent on the efficiency of the underlying raymarcher and data structure traversal [S1, unofficial].
- **Strengths**: Correct perspective projection, flexible adaptive sampling, simpler implementation compared to object-order methods, full 32-bit compositing precision [S1, unofficial].
- **Limitations**: Can be slower than rasterization for simple scenes due to the per-pixel ray traversal; requires Shader Model 3.0+ for optimal GPU performance [S1, unofficial].
- **When to use**: Ideal for high-quality volume rendering, virtual endoscopy, fly-throughs, and integration into perspective scenes where correct lighting and compositing are critical [S1, unofficial].
- **When not to use**: Not suitable for real-time applications requiring extremely high frame rates with simple geometry, where object-order rasterization might be more efficient [S1, unofficial].

## Background
The image-order approach has been widely adopted in GPU-based volume raycasting since the early 2000s. Key implementations include rectilinear grid raycasting by Krüger and Westermann (2003), Röttger et al. (2003), Green (2004) in the NVIDIA SDK, Stegmaier et al. (2005), Scharsach et al. (2006), and Gobbetti et al. (2008) [S1, unofficial]. It has also been applied to unstructured tetrahedral grids by Weiler et al. (2002, 2003, 2004), Bernardon et al. (2004), Callahan et al. (2006), and Muigg et al. (2007) [S1, unofficial]. The approach is enabled by conditional loops in fragment shaders (Shader Model 3.0 and higher), which allow for early loop exit and replace multiple passes [S1, unofficial].

## Sources
[S1] GPU-Based Volume Ray-Casting with Advanced Illumination, https://doi.org/10.2312/egt.20091069, fetched 2026-10-26 (unofficial)
[S2] cianjinks/Sparse-Voxel-Octree, https://github.com/cianjinks/Sparse-Voxel-Octree, fetched 2026-10-26 (unofficial)
[S3] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
