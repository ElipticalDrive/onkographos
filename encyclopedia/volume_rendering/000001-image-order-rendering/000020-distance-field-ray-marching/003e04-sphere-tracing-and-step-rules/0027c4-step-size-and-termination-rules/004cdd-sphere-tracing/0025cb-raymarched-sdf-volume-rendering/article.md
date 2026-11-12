# Raymarched SDF Volume Rendering

Also called: Ray Marching with Signed Distance Fields, Raymarching Signed Distance Fields, SDF Raycasting, SDF Raymarching

## Overview
Raymarched SDF Volume Rendering is a technique for rendering 3D scenes defined by implicit surfaces (Signed Distance Fields) without rasterizing triangle meshes [S1, S2, S3, S4, unofficial]. It solves the problem of displaying complex, procedural, or fractal geometry by casting rays from the camera and stepping along them using distance estimates provided by the SDF [S1, S2, unofficial]. Unlike traditional rasterization or even standard ray tracing with explicit geometry, this method relies entirely on mathematical functions to define shape, offering infinite sharpness and precision for primitive-based SDFs, which is crucial for CAD and professional graphic design [S3, unofficial]. It differs from its siblings by focusing on the specific implementation of the marching loop and shading within the SDF framework, building on the principles of [Sphere Tracing with SDFs](6223-sphere-tracing-with-sdfs.md).

## How it works
The method renders a pixel by casting a ray into the scene and iteratively advancing it until it intersects a surface or escapes the scene.

**Inputs and Outputs**
*   **Inputs**: Camera origin $\mathbf{o}$, normalized ray direction $\mathbf{d}$, maximum ray distance $t_{max}$, convergence threshold $\epsilon$, and the scene SDF function $f(\mathbf{p})$ which returns the signed distance to the nearest surface.
*   **Outputs**: Pixel color (and optionally alpha) based on the hit surface's material and lighting, or background color if no hit occurs.

**Data Structures**
*   **Ray**: Defined as $r(t) = \mathbf{o} + t\mathbf{d}$, where $t$ is the distance along the ray [S1, unofficial].
*   **SDF Function**: A procedural function $f: \mathbb{R}^3 \to \mathbb{R}$ [S3, unofficial]. For primitives, it returns the Euclidean distance. For example:
    *   Sphere: $f(\mathbf{p}) = \|\mathbf{p} - \mathbf{c}\| - r$ [S4, unofficial].
    *   Box: $f(\mathbf{p}) = \|\max(|\mathbf{p} - \mathbf{c}| - \mathbf{s}, 0)\|$ [S4, unofficial].
*   **Scene Composition**: Complex shapes are built using Boolean operations on primitive SDFs [S3, S4, unofficial]:
    *   Union: $f_{union}(\mathbf{p}) = \min(f_1(\mathbf{p}), f_2(\mathbf{p}))$ [S3, S4, unofficial].
    *   Intersection: $f_{intersect}(\mathbf{p}) = \max(f_1(\mathbf{p}), f_2(\mathbf{p}))$ [S3, S4, unofficial].
    *   Subtraction: $f_{sub}(\mathbf{p}) = \max(f_1(\mathbf{p}), -f_2(\mathbf{p}))$ [S3, S4, unofficial].
*   **Material ID (Optional)**: Some implementations return a vector where the x-component is the distance and the y-component is a material ID [S1, unofficial].

**Numbered Steps**
1.  **Ray Construction**: For each pixel, construct the ray direction $\mathbf{d}$ from the camera basis vectors and pixel UV coordinates [S4, unofficial]. The ray is defined as $p(s) = o + s\,\hat{r}$ [S1, unofficial].
2.  **Initialization**: Initialize the accumulated distance $t = 0$ (or near-plane value) and current position $\mathbf{p} = \mathbf{o}$ [S1, S4, unofficial].
3.  **Marching Loop**: Iterate up to a maximum number of steps $N_{max}$ (e.g., 64 [S4, unofficial] or 130 [S1, unofficial]):
    *   **Evaluate SDF**: Compute $d = f(\mathbf{p})$ [S1, S4, unofficial].
    *   **Termination Check**:
        *   If $d < \epsilon$ (e.g., $0.005$ [S4, unofficial] or $0.001$ [S1, unofficial]), the ray has hit the surface. Break loop.
        *   If $t > t_{max}$ (e.g., $400.0$ [S4, unofficial]), the ray has escaped. Break loop.
    *   **Step**: Advance the ray. The standard safe step is $t \leftarrow t + d$ [S1, S33, unofficial]. However, some implementations use a conservative step size, such as $t \leftarrow t + d \times 0.5$, to prevent tunnelling through thin structures or due to non-rigid transforms [S4, S1, unofficial].
    *   **Update Position**: $\mathbf{p} \leftarrow \mathbf{o} + t\mathbf{d}$ [S4, unofficial].
4.  **Shading**: If a hit occurred ($d < \epsilon$):
    *   **Normal Estimation**: Compute the surface normal $\mathbf{n}$ using finite differences (gradient of the SDF) [S2055, S4619, S4]:
        $$ \mathbf{n} = \frac{\nabla f(\mathbf{p})}{\|\nabla f(\mathbf{p})\|} $$
        Approximated numerically with epsilon $\epsilon_{grad}$ (e.g., $0.02$ [S4, unofficial]):
        $$ \mathbf{n} \approx \text{normalize}\left( \begin{bmatrix} f(\mathbf{p} + \epsilon_{grad}\hat{x}) - f(\mathbf{p} - \epsilon_{grad}\hat{x}) \\ f(\mathbf{p} + \epsilon_{grad}\hat{y}) - f(\mathbf{p} - \epsilon_{grad}\hat{y}) \\ f(\mathbf{p} + \epsilon_{grad}\hat{z}) - f(\mathbf{p} - \epsilon_{grad}\hat{z}) \end{bmatrix} \right) $$
    *   **Lighting**: Compute diffuse lighting using the light direction $\mathbf{l}$ and ambient light $\mathbf{a}$ [S4, unofficial]:
        $$ \mathbf{C}_{diffuse} = \mathbf{C}_{light} \cdot \max(\mathbf{l} \cdot \mathbf{n}, 0) $$
        $$ \mathbf{C}_{final} = (\mathbf{C}_{diffuse} + \mathbf{a} \cdot 2.0) \cdot \mathbf{C}_{base} $$
    *   **Fog**: Apply exponential fog based on distance from camera $dist = \|\mathbf{p} - \mathbf{o}\|$ [S4, unofficial]:
        $$ f_{fog} = e^{-dist \cdot \rho} $$
        $$ \mathbf{C}_{out} = \mathbf{C}_{fog} \cdot (1 - f_{fog}) + \mathbf{C}_{final} \cdot f_{fog} $$
    *   **Alpha**: Set alpha based on lighting or material properties (e.g., shadow transparency) [S4, unofficial].
5.  **Background**: If no hit occurred, return the background color.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raymarch_sdf(camera_origin, camera_direction, max_steps, max_dist, epsilon, sdf_func):
    """
    Renders a single pixel using raymarched SDF.
    
    Args:
        camera_origin (vec3): The position of the camera.
        camera_direction (vec3): The normalized direction vector for the pixel.
        max_steps (int): Maximum number of marching iterations (e.g., 64 or 130).
        max_dist (float): Maximum ray distance (e.g., 400.0).
        epsilon (float): Convergence threshold for surface hit (e.g., 0.005).
        sdf_func (function): Function f(p) returning signed distance to nearest surface.
        
    Returns:
        vec4: Final pixel color with alpha.
    """
    t = 0.0
    pos = camera_origin
    
    for i in range(max_steps):
        # Evaluate SDF at current position
        d = sdf_func(pos)
        
        # Check for surface hit
        if d < epsilon:
            # Hit detected, proceed to shading
            break
        
        # Check for escape
        if t > max_dist:
            # Ray escaped, return background
            return vec4(0.0, 0.0, 0.0, 0.0) # Placeholder for background color
        
        # Conservative step to prevent tunnelling (factor 0.5 from [S4, unofficial])
        # Note: Standard sphere tracing uses step = d. Conservative uses step = d * 0.5.
        step_size = d * 0.5 
        t += step_size
        pos = camera_origin + t * camera_direction
    
    # If loop finished without break, check if we hit or escaped
    # Re-evaluate d at final pos to confirm hit status if needed, 
    # but typically the break condition handles it.
    if d < epsilon:
        # Shading
        # 1. Compute Normal via Finite Differences
        eps_grad = 0.02
        dx = vec3(eps_grad, 0.0, 0.0)
        dy = vec3(0.0, eps_grad, 0.0)
        dz = vec3(0.0, 0.0, eps_grad)
        
        grad_x = sdf_func(pos + dx) - sdf_func(pos - dx)
        grad_y = sdf_func(pos + dy) - sdf_func(pos - dy)
        grad_z = sdf_func(pos + dz) - sdf_func(pos - dz)
        
        normal = normalize(vec3(grad_x, grad_y, grad_z))
        
        # 2. Lighting
        light_dir = normalize(vec3(0.0, 1.0, 0.0)) # Placeholder light direction
        light_color = vec3(0.7, 0.7, 0.7) # Placeholder light color
        ambient_color = vec3(0.2, 0.2, 0.2) # Placeholder ambient
        
        diffuse = light_color * max(dot(light_dir, normal), 0.0)
        final_light = diffuse + ambient_color * 2.0
        
        base_color = vec3(0.7, 0.7, 0.7) # Placeholder base material color
        color = base_color * final_light
        
        # 3. Fog
        distance_from_cam = length(pos - camera_origin)
        fog_density = 0.1
        fog_color = vec3(0.8, 0.8, 0.8)
        fog_factor = exp(-distance_from_cam * fog_density)
        
        out_color = fog_color * (1.0 - fog_factor) + color * fog_factor
        
        # 4. Alpha
        alpha = max(dot(light_dir, normal), 0.0)
        
        return vec4(out_color, alpha)
    else:
        # No hit
        return vec4(0.0, 0.0, 0.0, 0.0)
```

**Parameters**
*   `MARCH_STEPS`: Maximum iterations. Typical values: 64 [S4, unofficial], 130 [S1, unofficial].
*   `epsilon`: Convergence threshold. Typical values: $0.005$ [S4, unofficial], $0.001$ [S1, unofficial].
*   `max_dist`: Maximum ray distance. Typical values: $400.0$ [S4, unofficial].
*   `focal_length`: Camera parameter. Typical value: $0.62$ [S4, unofficial].
*   `step_scale`: Multiplier for step size. Typical value: $0.5$ [S4, unofficial] (conservative), $1.0$ (standard).

**Edge Cases and Pitfalls**
*   **Tunnelling**: If the step size is too large or the SDF is not a true Euclidean distance (e.g., due to non-rigid transforms like wing flapping), rays may skip over thin geometry [S1, unofficial]. Solution: Use conservative step sizes (e.g., $0.5 \times d$) or increase iteration count [S1, S4, unofficial].
*   **Precision**: Primitive-based SDFs offer infinite sharpness, but numerical precision issues can arise with very small distances or large coordinate values [S3, unofficial].
*   **Clipping**: The rendering quad must be positioned beyond the camera's near clipping plane (e.g., $z=0.32$ if near clip is $0.3$) [S4, unofficial].
*   **Layer Culling**: In engine implementations, use separate cameras or layer masks to ensure the volume renders in the correct depth order relative to other scene elements [S4, unofficial].

## Subtopics

### [Mandelbulb Sphere Tracing](4527-mandelbulb-sphere-tracing.md)
Renders fractals using iterative SDFs. Chosen when infinite self-similar detail is required, but suffers from normal noise and distance estimator inaccuracy at high zoom.

### [Zeno Implementation](1544-zeno-implementation.md)
Specific implementation details for the Zeno framework.

### [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md)
Enables gradient-based optimization of SDF parameters.

### [Sphere Tracing with Composite SDF and Adaptive Sampling](3867-sphere-tracing-with-composite-sdf-and-adaptive-sampling.md)
Uses adaptive sampling for composite SDFs to improve performance.

### [Uniform Sphere Tracing](4406-uniform-sphere-tracing.md)
Uses fixed step sizes instead of SDF distances, simpler but slower.

### [Menger Sponge Raymarching](4875-menger-sponge-raymarching.md)
Renders the Menger sponge fractal using recursive SDFs.

### [Synchronized-tracing](8552-synchronized-tracing.md)
Synchronizes ray marching across threads/pixels.

### [Cone-Traced Supersampling](9466-cone-traced-supersampling.md)
Uses cone tracing for anti-aliasing in SDF rendering.

### [Kernel-Reflection Sequences](10186-kernel-reflection-sequences.md)
Handles reflections using kernel sequences in SDF space.

### [Trace-based SDF Rendering](10785-trace-based-sdf-rendering.md)
General trace-based approach for SDFs.

## Operating characteristics
*   **Time Complexity**: $O(N \cdot C)$, where $N$ is the number of pixels and $C$ is the average number of marching steps per pixel. The cost is dominated by the SDF evaluation per step [S3, unofficial].
*   **Memory Complexity**: $O(1)$ auxiliary memory per pixel, as the scene is defined procedurally [S3, unofficial]. This results in a reduced memory footprint compared to mesh-based rendering [S3, unofficial].
*   **Measured Performance**: Performance depends heavily on GPU ALU capabilities rather than memory bandwidth, as pure mathematical SDFs are compute-bound [S3, unofficial]. Modern GPUs have evolved to support this efficiently [S3, unofficial].
*   **Strengths**: Infinite sharpness and precision for primitives; small memory footprint; easy to create complex procedural geometry and fractals; no need for mesh generation or UV unwrapping [S3, unofficial].
*   **Limitations**: Cannot easily represent non-SDF geometry (e.g., scanned meshes) without conversion; thin structures may require conservative stepping, reducing performance; normal estimation via finite differences can be noisy for complex SDFs [S1, S3, unofficial].
*   **When to Use**: Demoscene, Shadertoy art, procedural content generation, fractal rendering, CAD visualization where precision is key [S3, unofficial].
*   **When Not to Use**: Scenes with large amounts of explicit mesh data, real-time applications with extremely limited ALU power, or when exact physical accuracy of complex non-procedural geometry is required.

## Background
*   **Origin**: The concept of implicit surfaces using boolean operations (min/max) dates back to 1972 by A. Ricci and later B. Wyvill and G. Wyvill in 1989 [S3, unofficial].
*   **History**: The first mention of raymarching SDFs for rendering 3D fractals was in a paper by Sandin, Hart, and Kauffman, though they did not coin the term "SDF" [S3, unofficial]. C. Hart documented the technique in 1995, incorrectly calling it "Sphere Tracing" [S3, unofficial].
*   **Modern SDFs**: Modern SDFs, characterized by constraining the field to be a Euclidean Distance, were explored by Inigo Quilez starting in 2007, influenced by Alex Evan's 2006 work and Keenan Crane's 2005 work [S3, unofficial].
*   **Usage**: The technique became popular in the demoscene and Shadertoy communities for art-directed content with sophisticated modeling, shading, and animation [S3, unofficial]. It is also used at Pixar for procedural content and reduced memory footprint [S3, unofficial].

## Sources
[S1] el-bakkali/manim-glsl-butterflies, https://github.com/el-bakkali/manim-glsl-butterflies, fetched 2026-10-26 (unofficial)
[S2] AJKKinney/Mandelbulb, https://github.com/AJKKinney/Mandelbulb, fetched 2026-10-27 (unofficial)
[S3] Raymarching Distance Fields, https://iquilezles.org/articles/raymarchingdf/, fetched 2026-10-27 (unofficial)
[S4] Advanced shaders in Unity - Volume render a death star | Habrador, https://www.habrador.com/tutorials/shaders/1-volume-rendering-death-star/, fetched 2026-10-28 (unofficial)
