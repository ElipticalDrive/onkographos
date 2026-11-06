# Image-order Rendering Loop

Also called: Main Rendering Loop, Octree Rendering Loop

## Overview
The Image-order Rendering Loop is the top-level control structure that iterates over the output image pixels to generate rays and invoke the scene shading logic [S1, unofficial]. It solves the problem of mapping 2D screen coordinates to 3D scene geometry by managing camera parameters, projection modes, and shading strategies [S1, unofficial]. It differs from sibling traversal methods (e.g., [Ray-grid traversal](17-ray-grid-traversal.md)) by operating at the pixel level rather than the voxel level; it delegates the geometric intersection or volume integration to lower-level routines like `Raymarch` or `ShadePathTrace` [S1, unofficial].

## How it works
The method initializes rendering state, iterates over pixel coordinates and samples, generates rays based on the camera model, and accumulates color values according to the selected shading mode [S1, unofficial].

### Inputs and Outputs
- **Inputs**: Camera parameters (position, direction, projection mode), scene data (octree, lights), rendering configuration (shading mode, sample count).
- **Outputs**: A 2D image buffer (pixel colors).

### Data Structures
- **Renderer Configuration**:
  - `maxDepth`: Integer, maximum recursion depth for path tracing. Default: 100 [S2, unofficial].
  - `samplesPerPixel`: Integer, number of samples per pixel for anti-aliasing. Default: 500 [S2, unofficial].
  - `background`: RGB color, default `{0, 0, 0}` [S2, unofficial].
- **Shading Mode**: Enum determining the shading behavior. Values: `TRACE`, `DEPTH`, `DEPTH_HIT`, `DIFFUSE`, `NORMAL`, `INDEX` [S1, unofficial].
- **Projection Mode**: Enum determining ray generation. Values: `PERSPECTIVE`, `ORTHOGRAHIC` [S1, unofficial].

### Pseudocode
The following pseudocode implements the loop as described in [S1, unofficial]. It assumes helper functions `ShadePathTrace`, `Raymarch`, and `Shade*` are defined elsewhere.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ImageOrderRenderingLoop(scene, camera, image, shading_mode, projection_mode, num_samples):
    """
    Main rendering loop iterating over pixels and samples.
    """
    # Initialize accumulated color buffer
    accumulated_color = zeros_like(image)
    completed_samples = 0
    
    # Precompute light bounds if necessary for optimization
    # p0, p1 = calculate_light_bounds(scene)
    
    # Apply rotation to camera position and direction if needed
    # camera_pos = rotate(camera.position)
    # camera_dir = rotate(camera.direction)
    
    for s in range(num_samples):
        for y in range(image.height):
            for x in range(image.width):
                # Calculate ray origin and direction based on projection mode
                if projection_mode == PERSPECTIVE:
                    # Map pixel to normalized device coordinates or similar
                    # rx, ry = pixel_to_ndc(x, y)
                    # pos = transform(rx, ry)
                    ray_origin = camera.position
                    ray_direction = normalize(camera.direction + pos)
                elif projection_mode == ORTHOGRAPHIC:
                    # rx, ry = pixel_to_ndc(x, y)
                    # pos = transform(rx, ry)
                    ray_origin = camera.position + pos
                    ray_direction = camera.direction
                
                # Calculate pixel index
                index = x + y * image.width
                
                # Perform shading/ray marching
                if shading_mode == Shade::TRACE:
                    # Path tracing with recursion
                    color = ShadePathTrace(ray_origin, ray_direction, scene, depth=100)
                    # Average with previous samples
                    accumulated_color[index] = (accumulated_color[index] * completed_samples + color) / (completed_samples + 1)
                else:
                    # Direct raymarching or simple shading
                    hit = Raymarch(ray_origin, ray_direction, scene)
                    
                    if hit:
                        # Shade based on mode
                        if shading_mode == Shade::DEPTH:
                            depth = distance(camera.position, hit.point)
                            color = ShadeDepth(hit.color, depth)
                        elif shading_mode == Shade::DEPTH_HIT:
                            depth = distance(camera.position, hit.point)
                            color = ShadeDepthFromHit(hit.color, camera.position, hit.point)
                        elif shading_mode == Shade::DIFFUSE:
                            color = ShadeDiffuse(hit.color, hit.normal, ray_origin, ray_direction)
                        elif shading_mode == Shade::NORMAL:
                            color = ShadeNormal(hit.normal)
                        elif shading_mode == Shade::INDEX:
                            color = ShadeIndex(hit.material_index)
                    else:
                        # Check if ray intersects light AABB for background lighting
                        if ray_intersects_light_aabb(ray_origin, ray_direction, scene.lights):
                            color = scene.light_color
                        else:
                            color = RGB(25, 25, 25) # Dark color for no hit
                    
                    accumulated_color[index] += color
                
                completed_samples += 1
    
    # Finalize image
    for i in range(image.width * image.height):
        image[i] = accumulated_color[i] / num_samples
        
    return image
```

### Edge Cases and Pitfalls
- **Zero Direction Components**: If `ray_direction` has a zero component, division by zero may occur in normalization or ray-plane intersection checks. Guard against this by checking `abs(denom) > epsilon` [S1, unofficial].
- **Recursion Depth**: Path tracing requires a depth limit to prevent infinite loops. The default is 100 [S2, unofficial].
- **Floating Point Precision**: Small offsets (e.g., `1e-4f`) are added to hit points to prevent self-intersection artifacts [S1, unofficial].

## Subtopics

### [Terminal Rendering Pipeline](3434-terminal-rendering-pipeline.md)
The Terminal Rendering Pipeline renders a 3D scene to a text-based terminal buffer by iterating over pixels, mapping them to Normalized Device Coordinates (NDC), casting rays via a raymarcher, and applying ASCII/ANSI styling based on depth and lighting. It differs from standard terminal rendering by explicitly accounting for character aspect ratio when converting pixel indices to NDC and using ANSI escape codes to modulate brightness based on depth percentage, effectively acting as a low-resolution, text-based raymarcher.

### [Diffuse Octree Rendering](4989-diffuse-octree-rendering.md)
Diffuse Octree Rendering is an image-order raycasting technique that traverses an octree to locate the first leaf node containing data for each pixel, then computes surface lighting using a diffuse and Phong specular model with distance attenuation. It differs from sibling methods by relying on a coarse-to-fine octree traversal to skip empty space entirely, rather than marching through a dense voxel grid or using ray-marching steps within occupied nodes.

### Renderer configuration
Top-level rendering controller managing ray depth, sampling rate, and background color [S2, unofficial].
- **Parameters**: `maxDepth` (int, default 100), `samplesPerPixel` (int, default 500) [S2, unofficial].
- **Layout**: Class `Renderer` with fields `int maxDepth`, `int samplesPerPixel`, `RGB background` and methods `RGB rayColor(const Ray &ray, const Scene &scene, int depth) const`, `void render(const Scene &scene, const Camera &camera, Image &image) const` [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(P \cdot S \cdot C)$, where $P$ is the number of pixels, $S$ is the number of samples per pixel, and $C$ is the cost of the shading/raymarching routine per sample [S1, S2, unofficial].
- **Memory Complexity**: $O(P)$ for the image buffer and accumulated color buffer [S1, unofficial].
- **Strengths**: Simple control flow, easy to parallelize per-pixel, supports various shading modes [S1, unofficial].
- **Limitations**: Performance is dominated by the raymarching cost; high sample counts are required for noise reduction [S2, unofficial].

## Background
The Image-order Rendering Loop is a fundamental pattern in ray tracing and volume rendering, originating from early ray tracing implementations [S1, S2, unofficial]. It is widely used in educational ray tracers and real-time rendering engines that support path tracing or ray-marched volumes [S1, S2, unofficial].

## Sources
[S1] cianjinks/Sparse-Voxel-Octree, https://github.com/cianjinks/Sparse-Voxel-Octree, fetched 2026-10-26 (unofficial)
[S2] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
