# Image-order Rendering Loop
Also called: Image order approach, Image-Order Volume Ray Casting, Main Rendering Loop, Octree Rendering Loop

## Overview
The image-order rendering loop is the fundamental control structure for direct volume ray casting, iterating over each pixel of the output image to generate a ray, traverse the volume, and composite the result [S1, S2, S3, unofficial]. It solves the problem of mapping 3D volumetric data to a 2D image plane by decoupling the sampling process from the data structure, allowing for adaptive sampling and correct perspective projection in a single pass [S1, unofficial]. Unlike object-order methods (rasterization), which process data primitives first, this approach is described as "CPU-like" but offers flexibility for advanced illumination and 32-bit compositing precision [S1, unofficial]. It builds on the shared principles of [Image-order rendering](16-image-order-rendering.md) and [Ray Traversal and Integration Schemes](8341-ray-traversal-and-integration-schemes.md).

## How it works
The method executes a nested loop structure: an outer loop iterates over image pixels, and an inner loop (the raymarcher) iterates along the ray path within the volume.

**Inputs and Outputs:**
*   **Inputs:** Image dimensions ($W \times H$), camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$, projection mode), volume data structure, transfer function $T(\cdot)$, and shading mode.
*   **Outputs:** Final color buffer $C_{final}[x][y]$ for each pixel $(x, y)$.

**Data Structures:**
*   **Pixel Coordinates:** Integer indices $x \in [0, W-1]$, $y \in [0, H-1]$.
*   **Ray Parameters:** Origin $\mathbf{o}$, direction $\mathbf{d}$ (normalized), step size $\Delta t$.
*   **Accumulators:** Color $C_{acc}$ (RGB), Opacity $\alpha_{acc}$ (scalar), Transmittance $T_{acc} = 1 - \alpha_{acc}$.

**Numbered Steps:**

1.  **Initialization:** For each pixel $(x, y)$, initialize accumulators: $C_{acc} = (0, 0, 0)$, $\alpha_{acc} = 0$, $T_{acc} = 1$. Calculate the ray origin $\mathbf{o}$ and direction $\mathbf{d}$ based on the projection mode (Perspective or Orthographic) [S2, unofficial].
2.  **Ray Discretization:** Determine the entry ($t_{start}$) and exit ($t_{end}$) points of the ray intersecting the volume bounding box. Calculate the number of samples $N$ and step size $\Delta t = (t_{end} - t_{start}) / N$ [S5, S10, S17, unofficial].
3.  **Marching Loop:** For $i = 0$ to $N-1$:
    a.  **Sample Point:** Calculate position $\mathbf{p}_i = \mathbf{o} + (t_{start} + i \cdot \Delta t) \mathbf{d}$.
    b.  **Sampling:** Retrieve scalar value $d_i$ from the volume at $\mathbf{p}_i$ using interpolation (e.g., trilinear) [S7, S10, S43, unofficial].
    c.  **Classification:** Map $d_i$ to color $C_i$ and opacity $\alpha_i$ using the transfer function $T(d_i) \rightarrow (C_i, \alpha_i)$ [S2, S2789]. Correct opacity for step size: $\alpha'_i = 1 - (1 - \alpha_i)^{\Delta t}$ [S11, S32, unofficial].
    d.  **Compositing:** Update accumulators:
        $$
        C_{acc} = C_{acc} + T_{acc} \cdot \alpha'_i \cdot C_i
        $$
        $$
        \alpha_{acc} = \alpha_{acc} + T_{acc} \cdot \alpha'_i
        $$
        $$
        T_{acc} = T_{acc} \cdot (1 - \alpha'_i)
        $$
    e.  **Early Termination:** If $\alpha_{acc} \ge 1.0$ (or a threshold close to 1.0), break the loop [S10, S12, unofficial].
4.  **Final Compositing:** Composite the accumulated color over the background color $C_{bg}$:
    $$
    C_{final}[x][y] = C_{acc} + T_{acc} \cdot C_{bg}
    $$
    [S1, S14, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def image_order_rendering_loop(image_width, image_height, camera, volume, transfer_function, background_color):
    # Initialize output buffer
    output_buffer = [[(0.0, 0.0, 0.0) for _ in range(image_width)] for _ in range(image_height)]
    
    # Pre-calculate ray bounds for the volume bounding box
    # Assuming volume has a method to get intersection t-values for a ray
    # This step determines t_start and t_end for each ray
    # Optimization: Precompute bounding box intersections if possible
    
    for y in range(image_height):
        for x in range(image_width):
            # 1. Ray Generation
            ray_origin, ray_direction = camera.generate_ray(x, y)
            
            # 2. Ray-Volume Intersection
            t_start, t_end = volume.get_ray_intersection(ray_origin, ray_direction)
            
            if t_start > t_end:
                # Ray does not intersect volume
                output_buffer[y][x] = background_color
                continue
            
            # 3. Discretization
            num_samples = calculate_num_samples(t_end - t_start)
            delta_t = (t_end - t_start) / num_samples
            
            # 4. Accumulators
            acc_color = (0.0, 0.0, 0.0)
            acc_alpha = 0.0
            transmittance = 1.0
            
            # 5. Marching Loop
            for i in range(num_samples):
                # Current position along ray
                t = t_start + i * delta_t
                pos = ray_origin + ray_direction * t
                
                # Check if pos is within volume bounds (redundant if t_start/t_end are exact, but safe)
                if not volume.contains(pos):
                    continue
                
                # Sampling
                scalar_value = volume.sample(pos)
                
                # Classification
                color, alpha = transfer_function(scalar_value)
                
                # Opacity correction for step size
                # alpha_corrected = 1 - (1 - alpha) ** delta_t
                # Using approximation for small delta_t or exact formula
                alpha_corrected = 1.0 - (1.0 - alpha) ** delta_t
                
                # Compositing
                contribution = transmittance * alpha_corrected * color
                acc_color = tuple(acc_color[j] + contribution[j] for j in range(3))
                acc_alpha += transmittance * alpha_corrected
                transmittance *= (1.0 - alpha_corrected)
                
                # Early Ray Termination
                if acc_alpha >= 0.99: # Threshold close to 1.0
                    break
            
            # 6. Final Compositing
            final_color = tuple(acc_color[j] + transmittance * background_color[j] for j in range(3))
            output_buffer[y][x] = final_color
            
    return output_buffer
```

**Parameters:**
*   `num_samples`: Number of steps per ray. Typical values depend on resolution and volume density. Not given in the sources.
*   `threshold`: Early termination threshold. Typical value: 0.99 or 1.0 [S10, S12, unofficial].
*   `ShadingMode`: Enum determining shading behavior (TRACE, DEPTH, DIFFUSE, etc.) [S2, unofficial].
*   `ProjectionMode`: Enum determining ray generation (PERSPECTIVE, ORTHOGRAPHIC) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point errors in ray-position calculation can cause gaps or overlaps in sampling. Use robust ray-box intersection algorithms [S1, unofficial].
*   **Boundary Conditions:** Rays missing the volume entirely must be handled to avoid invalid memory access [S2, unofficial].
*   **Shader Model:** Requires Shader Model 3.0 or higher for conditional loops (early exit) in fragment shaders to replace multiple passes [S1, unofficial].

## Subtopics

### [Terminal Rendering Pipeline](3434-terminal-rendering-pipeline.md)
The Terminal Rendering Pipeline renders a 3D scene to a text-based terminal buffer by iterating over pixels, mapping them to Normalized Device Coordinates (NDC), casting rays via a raymarcher, and applying ASCII/ANSI styling based on depth and lighting. It differs from standard terminal rendering by explicitly accounting for character aspect ratio when converting pixel indices to NDC and using ANSI escape codes to modulate brightness based on depth percentage, effectively acting as a low-resolution, text-based raymarcher. See [Terminal Rendering Pipeline](3434-terminal-rendering-pipeline.md).

### [Diffuse Octree Rendering](4989-diffuse-octree-rendering.md)
Diffuse Octree Rendering is an image-order raycasting technique that traverses an octree to locate the first leaf node containing data for each pixel, then computes surface lighting using a diffuse and Phong specular model with distance attenuation. It differs from sibling methods by relying on a coarse-to-fine octree traversal to skip empty space entirely, rather than marching through a dense voxel grid. See [Diffuse Octree Rendering](4989-diffuse-octree-rendering.md).

### Renderer configuration
A top-level rendering controller managing ray depth, sampling rate, and background color [S3, unofficial].
*   **Definition:** Manages global rendering parameters for the image-order loop [S3, unofficial].
*   **Parameters:** `maxDepth` (maximum recursion depth for ray tracing, default: 100), `samplesPerPixel` (number of samples per pixel for anti-aliasing/noise reduction, default: 500) [S3, unofficial].
*   **Layout:** Class `Renderer` with fields `int maxDepth = 100`, `int samplesPerPixel = 500`, `RGB background{0, 0, 0}`. Methods: `RGB rayColor(const Ray &ray, const Scene &scene, int depth) const`, `void render(const Scene &scene, const Camera &camera, Image &image) const` [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(W \cdot H \cdot N)$, where $W, H$ are image dimensions and $N$ is the number of samples per ray. Early ray termination can reduce the average $N$.
*   **Memory Complexity:** $O(W \cdot H)$ for the output buffer. The volume data structure size depends on the representation (e.g., dense grid vs. sparse octree).
*   **Performance:** On GPU, it benefits from parallel pixel processing. Requires Shader Model 3.0+ for efficient conditional loops [S1, unofficial].
*   **Strengths:** Correct perspective projection, single-pass rendering, flexible shading models, 32-bit compositing precision [S1, unofficial].
*   **Limitations:** Can be computationally expensive for large volumes or high sample counts. Performance depends heavily on the efficiency of the ray-volume intersection and sampling routines.
*   **When to Use:** When high-quality perspective rendering and flexible shading are required. Suitable for GPU-based implementations.
*   **When Not to Use:** When real-time performance is critical and the volume is sparse (consider object-order or octree-based methods).

## Background
The image-order approach is a fundamental technique in volume rendering, contrasting with object-order (rasterization) methods [S1, unofficial]. It has been widely adopted in GPU-based volume ray casting, enabled by the availability of conditional loops in fragment shaders (Shader Model 3.0+) [S1, unofficial]. Key implementations and optimizations have been presented in various works, including those by Krüger and Westermann (2003), Röttger et al. (2003), Green (2004), Stegmaier et al. (2005), and others [S1, unofficial]. It is particularly useful for applications like virtual endoscopy, fly-throughs, and integration into perspective scenes such as games [S1, unofficial].

## Sources
[S1] GPU-Based Volume Ray-Casting with Advanced Illumination, https://doi.org/10.2312/egt.20091069, fetched 2026-10-26 (unofficial)
[S2] cianjinks/Sparse-Voxel-Octree, https://github.com/cianjinks/Sparse-Voxel-Octree, fetched 2026-10-26 (unofficial)
[S3] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
