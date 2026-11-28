# Lambertian Shading with Gradient Normals

Also called: Gradient Normal Lambertian Shading, Lambertian Lighting with SDF, Diffuse Lighting with Gradient Normals, Lambertian Shading

## Overview
Lambertian Shading with Gradient Normals is a surface shading technique for ray-marched signed distance fields (SDFs) or scalar volumes that approximates the surface normal using the numerical gradient of the field, then applies the Lambertian reflectance model to compute diffuse illumination [S1, unofficial][S4, unofficial][S6, unofficial]. It solves the problem of shading implicit surfaces that lack explicit mesh normals, allowing for realistic diffuse lighting without polygonization artifacts [S4, unofficial][S5, unofficial]. Unlike sibling methods that may use analytic normals, tetrahedral interpolation, or regularization, this method relies on finite-difference or analytic gradient estimation of the scalar field $f$ (or $\phi$) at the hit point [S3, unofficial].

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{r}$ (normalized) [S6, unofficial].
- Signed distance field function $f(\mathbf{p})$ returning the distance to the nearest surface [S1, unofficial][S6, unofficial].
- Light direction $\mathbf{l}$ (normalized, pointing from surface to light) [S1, unofficial][S3, unofficial]. Default: $(1.0, 1.0, 1.0)$ if not provided [S3, unofficial].
- Surface color $\mathbf{c}_{surf}$ (RGB, range $[0, 1]$) [S3, unofficial]. Default: $(0.8, 0.5, 0.3)$ [S3, unofficial].
- Ambient light intensity $a$ (range $[0, 1]$) [S3, unofficial]. Default: $0.1$ [S3, unofficial].
- Background color $\mathbf{c}_{bg}$ (RGB, range $[0, 1]$) [S3, unofficial]. Default: $(0.2, 0.2, 0.2)$ [S3, unofficial].
- Hit threshold $\epsilon$ (distance tolerance) [S6, unofficial]. Default: $0.001$ [S6, unofficial].

**Outputs:**
- Shaded pixel color $\mathbf{c}_{out}$ (RGB) [S3, unofficial].

**Data Structures:**
- Point $\mathbf{p} \in \mathbb{R}^3$: Current ray position [S6, unofficial].
- Normal $\mathbf{n} \in \mathbb{R}^3$: Estimated surface normal [S1, unofficial].
- Gradient $\nabla f \in \mathbb{R}^3$: Numerical gradient of the SDF [S1, unofficial][S4, unofficial].

**Steps:**

1. **Ray Marching:** March along the ray $\mathbf{p}(t) = \mathbf{o} + t\mathbf{r}$ using the SDF $f(\mathbf{p})$ to find the first intersection where $f(\mathbf{p}) < \epsilon$ [S5, unofficial][S6, unofficial]. If no intersection is found within the maximum steps, return $\mathbf{c}_{bg}$ [G].

2. **Gradient Estimation:** Compute the gradient $\nabla f$ at the hit point $\mathbf{p}$ using central finite differences [S3, unofficial][S4, unofficial]. For a step size $h$ (typically the SDF's resolution or a small constant like $10^{-4}$):
   $$
   \nabla f(\mathbf{p}) \approx \left( \frac{f(\mathbf{p} + h\mathbf{e}_x) - f(\mathbf{p} - h\mathbf{e}_x)}{2h}, \frac{f(\mathbf{p} + h\mathbf{e}_y) - f(\mathbf{p} - h\mathbf{e}_y)}{2h}, \frac{f(\mathbf{p} + h\mathbf{e}_z) - f(\mathbf{p} - h\mathbf{e}_z)}{2h} \right)
   $$
   where $\mathbf{e}_x, \mathbf{e}_y, \mathbf{e}_z$ are unit vectors along the axes [G].

3. **Normal Normalization:** Normalize the gradient to obtain the surface normal $\mathbf{n}$:
   $$
   \mathbf{n} = \frac{\nabla f}{\|\nabla f\|}
   $$
   If $\|\nabla f\| < 10^{-5}$, the gradient is degenerate; return base RGB without shading [S7, unofficial].

4. **View Direction:** Compute the view direction $\mathbf{v}$ as the negative ray direction:
   $$
   \mathbf{v} = -\mathbf{r}
   $$
   [S7, unofficial].

5. **Normal Orientation:** Ensure the normal faces the viewer. If $\mathbf{n} \cdot \mathbf{v} < 0$, flip the normal:
   $$
   \text{if } \mathbf{n} \cdot \mathbf{v} < 0, \quad \mathbf{n} = -\mathbf{n}
   $$
   [S7, unofficial].

6. **Lambertian Diffuse Calculation:** Compute the dot product between the normal and the light direction:
   $$
   \text{ndotl} = \mathbf{n} \cdot \mathbf{l}
   $$
   Clamp to $[0, 1]$ to handle back-facing surfaces:
   $$
   \text{ndotl} = \max(0.0, \text{ndotl})
   $$
   [S2, unofficial][S3, unofficial].

7. **Lighting Intensity:** Combine ambient and diffuse lighting:
   $$
   I = a + (1.0 - a) \times \text{ndotl}
   $$
   Clip $I$ to $[0.0, 1.0]$ [S3, unofficial].

8. **Final Color:** Multiply the lighting intensity by the surface color:
   $$
   \mathbf{c}_{out} = I \times \mathbf{c}_{surf}
   $$
   [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def lambertian_shading_with_gradient_normals(ray_origin, ray_direction, sdf_func, light_dir=None, 
                                             surface_color=(0.8, 0.5, 0.3), ambient=0.1, 
                                             background=(0.2, 0.2, 0.2), epsilon=0.001, 
                                             max_steps=64, step_size=0.01, grad_h=1e-4):
    """
    Renders a single ray using Lambertian shading with gradient normals.
    
    Args:
        ray_origin: (float, float, float) Ray origin.
        ray_direction: (float, float, float) Normalized ray direction.
        sdf_func: Callable returning float distance for a (float, float, float) point.
        light_dir: (float, float, float) Normalized light direction. Defaults to (1,1,1).
        surface_color: (float, float, float) Surface RGB color.
        ambient: float Ambient light intensity.
        background: (float, float, float) Background RGB color.
        epsilon: float Hit threshold.
        max_steps: int Maximum ray march steps.
        step_size: float Step size for ray marching.
        grad_h: float Step size for gradient finite differences.
        
    Returns:
        (float, float, float) Shaded color.
    """
    if light_dir is None:
        light_dir = (1.0, 1.0, 1.0)
    
    # Normalize light direction
    l_len = math.sqrt(light_dir[0]**2 + light_dir[1]**2 + light_dir[2]**2)
    if l_len > 0:
        light_dir = (light_dir[0]/l_len, light_dir[1]/l_len, light_dir[2]/l_len)
    else:
        light_dir = (0.0, 0.0, 0.0) # Fallback

    # Ray Marching
    t = 0.0
    hit_point = None
    for _ in range(max_steps):
        p = (ray_origin[0] + ray_direction[0] * t,
             ray_origin[1] + ray_direction[1] * t,
             ray_origin[2] + ray_direction[2] * t)
        
        d = sdf_func(p)
        
        if d < epsilon:
            hit_point = p
            break
        
        t += step_size
    
    if hit_point is None:
        return background

    # Gradient Estimation
    px, py, pz = hit_point
    
    # Central differences for x
    f_plus_x = sdf_func((px + grad_h, py, pz))
    f_minus_x = sdf_func((px - grad_h, py, pz))
    grad_x = (f_plus_x - f_minus_x) / (2.0 * grad_h)
    
    # Central differences for y
    f_plus_y = sdf_func((px, py + grad_h, pz))
    f_minus_y = sdf_func((px, py - grad_h, pz))
    grad_y = (f_plus_y - f_minus_y) / (2.0 * grad_h)
    
    # Central differences for z
    f_plus_z = sdf_func((px, py, pz + grad_h))
    f_minus_z = sdf_func((px, py, pz - grad_h))
    grad_z = (f_plus_z - f_minus_z) / (2.0 * grad_h)
    
    grad_norm = math.sqrt(grad_x**2 + grad_y**2 + grad_z**2)
    
    # Handle degenerate gradient
    if grad_norm < 1e-5:
        return surface_color
    
    # Normalize gradient to get normal
    nx = grad_x / grad_norm
    ny = grad_y / grad_norm
    nz = grad_z / grad_norm
    
    # View direction (negative ray direction)
    vx = -ray_direction[0]
    vy = -ray_direction[1]
    vz = -ray_direction[2]
    
    # Flip normal if facing away from viewer
    dot_nv = nx * vx + ny * vy + nz * vz
    if dot_nv < 0.0:
        nx = -nx
        ny = -ny
        nz = -nz
    
    # Lambertian Diffuse
    ndotl = nx * light_dir[0] + ny * light_dir[1] + nz * light_dir[2]
    ndotl = max(0.0, ndotl)
    
    # Lighting Intensity
    intensity = ambient + (1.0 - ambient) * ndotl
    intensity = max(0.0, min(1.0, intensity))
    
    # Final Color
    r = intensity * surface_color[0]
    g = intensity * surface_color[1]
    b = intensity * surface_color[2]
    
    return (r, g, b)
```

**Edge Cases and Pitfalls:**
- **Degenerate Gradients:** If the gradient magnitude is near zero (e.g., at flat regions or singularities), normalization fails. Return base RGB without shading [S7, unofficial].
- **Precision:** Finite differences are sensitive to the step size $h$. Too large $h$ causes aliasing; too small $h$ causes floating-point noise [G].
- **Back-Facing Normals:** The gradient points in the direction of increasing distance. For the outside of an object, this is outward [G]. However, if the ray hits the inside of a hollow object, the normal may face inward [G]. Flipping based on view direction ensures consistent shading [S7, unofficial].
- **Grid Artifacts:** In volume rendering, low-resolution grids can cause jagged normals. Tricubic interpolation can mitigate this for liquid surfaces [S5, unofficial].

## Subtopics

### [Normal Shading](1831-normal-shading.md)
Visualizes surface normals by taking the absolute value of each component and clamping to [0, 1]. It is used for debugging normal estimation rather than illumination.

### [SDF Bevel Generation](4312-sdf-bevel-generation.md)
Generates smooth bevels on SDF primitives. It modifies the SDF function itself rather than the shading, resulting in geometrically smooth normals without gradient artifacts.

### [Gradient-Based Surface Normal Estimation](4261-gradient-based-surface-normal-estimation.md)
A broader category of normal estimation techniques. This method is a specific instance using finite differences for Lambertian shading.

### [Normal Regularization](3704-normal-regularization.md)
Applies smoothing filters to estimated normals to reduce noise. It is used when gradient estimation is too noisy for stable shading.

### [Tetrahedral Normal Estimation](1986-tetrahedral-normal-estimation.md)
Estimates normals by interpolating within tetrahedral cells of a volume. It provides smoother normals than finite differences but is more computationally expensive.

### [Face Normal Computation](2042-face-normal-computation.md)
Computes normals from explicit polygonal faces. It is not applicable to pure SDF ray marching unless the SDF is converted to a mesh.

## Operating characteristics
- **Time Complexity:** $O(1)$ per hit point, but requires 6 additional SDF evaluations for the gradient (central differences) [G]. Total cost is proportional to the number of ray march steps plus 6 SDF calls per hit [G].
- **Memory Complexity:** $O(1)$ auxiliary memory. No additional data structures are needed beyond the SDF function [G].
- **Performance:** Fast and simple to implement. The main cost is the extra SDF evaluations [G]. On GPU, this can be optimized by batching gradient calculations if multiple rays hit the same region, but typically it is done per-pixel [G].
- **Strengths:** Simple, robust, works with any SDF, no preprocessing required [G].
- **Limitations:** Gradients can be noisy near sharp features or at low resolutions. Finite differences may not capture the true normal accurately if the SDF is not smooth [G].
- **When to Use:** When shading implicit surfaces without explicit normals is required, and performance is a concern. Suitable for real-time ray marching [G].
- **When Not to Use:** When high-quality normals are critical for complex lighting (e.g., specular highlights), or when the SDF is not smooth. In such cases, consider [Tetrahedral Normal Estimation](1986-tetrahedral-normal-estimation.md) or [Normal Regularization](3704-normal-regularization.md) [G].

## Background
Lambertian shading with gradient normals originates from the field of volume rendering and implicit surface rendering [S1, unofficial]. Max (1995) described using the gradient of a scalar field to approximate contour surface normals for direct volume rendering, enabling shading without polygonization [S4, unofficial]. This approach was later adopted in ray marching for SDFs, where the gradient of the distance field provides a natural normal estimate [S1, unofficial][S6, unofficial].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S3] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S4] [PDF] Optical Models for Direct Volume Rendering - Duke Computer Science, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S5] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S6] Raymarching Distance Fields: Concepts and Implementation ..., https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
[S7] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
