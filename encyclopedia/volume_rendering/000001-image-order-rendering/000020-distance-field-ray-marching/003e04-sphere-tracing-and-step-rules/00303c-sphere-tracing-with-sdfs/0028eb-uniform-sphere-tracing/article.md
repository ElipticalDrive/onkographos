# Uniform Sphere Tracing

Also called: Ray Marching for Volumes, Uniform Density Volume Rendering, Volume Rendering with Ray Marching (Uniform Sphere)

## Overview
Uniform Sphere Tracing renders volumetric media by advancing a ray through a scene using fixed step sizes derived from a Signed Distance Function (SDF) or geometric bounds, accumulating optical depth to compute transmission and scattering [S1, unofficial]. Unlike adaptive methods that vary step size based on local density gradients or error estimates, this approach uses a uniform or SDF-derived constant step rule, making it simple to implement but potentially inefficient for sparse or highly heterogeneous media [S1, unofficial]. It builds on [Sphere Tracing with SDFs](6223-sphere-tracing-with-sdfs.md) by applying the SDF step rule specifically to volume integration rather than just surface intersection.

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized).
- Scene definition: A set of volumetric primitives (e.g., spheres) defined by their geometry (center, radius) and optical properties (absorption coefficient $\sigma_a$, scatter color $\mathbf{c}_{vol}$).
- Background color $\mathbf{c}_{bg}$.
- Maximum travel distance $t_{max}$ (or bounding volume).
- Convergence threshold $\epsilon$ (for surface hits, though less critical for pure volume transmission, it defines the entry/exit precision).

**Outputs:**
- Final pixel color $\mathbf{c}_{final}$.

**Data Structures:**
- `Sphere` object:
  - `center`: `vec3` (position).
  - `radius`: `float`.
  - `sigma_a`: `float` (absorption coefficient).
  - `scatter`: `vec3` (scattered light color).
- `Ray`:
  - `origin`: `vec3`.
  - `direction`: `vec3` (normalized).

**Formulas:**
1.  **Ray-Sphere Intersection:** For a sphere with center $\mathbf{c}$ and radius $r$, the intersection distances $t_0$ (entry) and $t_1$ (exit) along ray $r(t) = \mathbf{o} + t\mathbf{d}$ are found by solving:
    $$ \|\mathbf{o} + t\mathbf{d} - \mathbf{c}\|^2 = r^2 $$
    Let $\mathbf{L} = \mathbf{c} - \mathbf{o}$. Then:
    $$ a = \mathbf{d} \cdot \mathbf{d} = 1 \quad (\text{since } \mathbf{d} \text{ is normalized}) $$
    $$ b = 2 (\mathbf{d} \cdot \mathbf{L}) $$
    $$ c = \mathbf{L} \cdot \mathbf{L} - r^2 $$
    $$ \Delta = b^2 - 4ac $$
    If $\Delta < 0$, no intersection. Otherwise:
    $$ t_0 = \frac{-b - \sqrt{\Delta}}{2a}, \quad t_1 = \frac{-b + \sqrt{\Delta}}{2a} $$
    [S1, unofficial].

2.  **Distance Through Volume:**
    $$ d = t_1 - t_0 $$
    [S1, unofficial]. Note: If $\mathbf{d}$ is normalized, $d$ is simply the difference in parameter $t$.

3.  **Transmission (Beer's Law):**
    $$ T = e^{-d \cdot \sigma_a} $$
    [S1, unofficial].

4.  **Final Color:**
    $$ \mathbf{c}_{final} = T \cdot \mathbf{c}_{bg} + (1 - T) \cdot \mathbf{c}_{vol} $$
    [S1, unofficial].

**Parameters:**
- $\sigma_a$: Absorption coefficient. Typical value: $0.1$ [S1, unofficial].
- $\mathbf{c}_{vol}$: Scatter color. Typical value: $(0.8, 0.1, 0.5)$ [S1, unofficial].
- $\mathbf{c}_{bg}$: Background color. Typical value: $(0.572, 0.772, 0.921)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def uniform_sphere_tracing(ray_origin, ray_direction, sphere, background_color):
    """
    Renders a single ray against a uniform density sphere.
    
    Args:
        ray_origin (vec3): Origin of the ray.
        ray_direction (vec3): Normalized direction of the ray.
        sphere (Sphere): Object containing center, radius, sigma_a, scatter.
        background_color (vec3): Color behind the volume.
        
    Returns:
        vec3: Final pixel color.
    """
    # 1. Compute ray-sphere intersection
    # Vector from ray origin to sphere center
    L = sphere.center - ray_origin
    
    # Coefficients for quadratic equation at^2 + bt + c = 0
    # Since ray_direction is normalized, a = 1
    a = 1.0
    b = 2.0 * dot(ray_direction, L)
    c = dot(L, L) - sphere.radius * sphere.radius
    
    discriminant = b * b - 4 * a * c
    
    # No intersection
    if discriminant < 0:
        return background_color
    
    # Calculate intersection distances t0 (entry) and t1 (exit)
    sqrt_disc = sqrt(discriminant)
    t0 = (-b - sqrt_disc) / (2 * a)
    t1 = (-b + sqrt_disc) / (2 * a)
    
    # Ensure we are looking at the front face or valid range
    # If t0 < 0, the ray starts inside or behind; for simple volume rendering,
    # we typically care about the segment within the sphere.
    # If the ray misses the sphere entirely (t1 < 0), return background.
    if t1 < 0:
        return background_color
        
    # 2. Calculate distance through the volume
    # If ray_direction is normalized, distance is simply t1 - t0
    distance = t1 - t0
    
    # 3. Calculate transmission using Beer's Law
    # T = exp(-distance * sigma_a)
    transmission = exp(-distance * sphere.sigma_a)
    
    # 4. Blend background and volume color
    # C_final = T * C_bg + (1 - T) * C_vol
    final_color = transmission * background_color + (1.0 - transmission) * sphere.scatter
    
    return final_color

def render_image(width, height, camera, spheres, background_color):
    """
    Renders an image by tracing rays for each pixel.
    """
    image_buffer = empty_array(width, height, vec3)
    
    for y from 0 to height - 1:
        for x from 0 to width - 1:
            # Compute ray for pixel (x, y)
            ray_origin = camera.origin
            ray_direction = camera.compute_ray_direction(x, y)
            ray_direction = normalize(ray_direction)
            
            pixel_color = background_color # Default if no volume hit
            
            # Check intersection with all spheres (or use BVH)
            # For simplicity, assume single sphere or iterate
            for sphere in spheres:
                hit_color = uniform_sphere_tracing(ray_origin, ray_direction, sphere, background_color)
                # In a real scene, you might need to handle multiple volumes or occlusion
                # Here, we assume the first hit or composite them. 
                # For a single sphere, this is the result.
                pixel_color = hit_color
                break # Assuming single sphere for this basic example
            
            image_buffer[x, y] = pixel_color
            
    save_image(image_buffer)
```

**Edge Cases and Pitfalls:**
- **Ray Direction Normalization:** The distance calculation $d = t_1 - t_0$ is only valid if $\mathbf{d}$ is normalized. If not, $d = \|\mathbf{p}_1 - \mathbf{p}_0\|$ must be used [S1, unofficial].
- **Inside-Out Rays:** If the ray origin is inside the sphere, $t_0$ may be negative. The logic must ensure the correct segment is used for integration.
- **Tangential Rays:** If $\Delta \approx 0$, the ray grazes the sphere. Precision issues may cause flickering or missed hits.
- **Uniform Density Assumption:** This method assumes constant $\sigma_a$ within the primitive. For non-uniform volumes, this is an approximation or requires subdivision [S1, unofficial].

## Subtopics

### [Spectral Tracking](4664-spectral-tracking.md)
Handles per-wavelength optical properties efficiently using a single majorant density, reducing tracker invocations compared to naive delta tracking.

### [Auto-Relaxed Sphere Tracing](9468-auto-relaxed-sphere-tracing.md)
Automatically scales step sizes based on previous iterations, reducing invalid steps and improving robustness over fixed step rules.

## Operating characteristics
- **Time Complexity:** $O(N)$ per ray, where $N$ is the number of primitives checked. For a single sphere, it is $O(1)$ per pixel. For complex scenes, spatial acceleration structures (BVH, Grid) are needed, reducing average case to $O(\log N)$ or better.
- **Memory Complexity:** $O(1)$ per ray if primitives are simple; $O(M)$ for scene storage, where $M$ is the number of primitives.
- **Performance:** Very fast for simple, convex volumes with uniform density. Performance degrades for complex, non-convex, or highly heterogeneous volumes where many steps or complex intersection tests are required.
- **Strengths:** Simple implementation, analytically exact for uniform spheres, no discretization error in the volume integration itself (only geometric intersection error).
- **Limitations:** Cannot handle non-uniform density without subdivision or approximation. Inefficient for sparse scenes if not accelerated.
- **When to Use:** Simple volumetric effects, uniform fog, or as a baseline for more complex volume rendering.
- **When Not to Use:** Highly heterogeneous media, complex lighting interactions (multiple scattering), or when high performance is required for large, sparse scenes without acceleration.

## Background
Originating from basic ray tracing principles, uniform sphere tracing for volumes is a foundational technique in computer graphics. It applies Beer's Law to the path length through a volumetric primitive. The specific implementation using sphere intersections and uniform absorption is detailed in introductory tutorials on volume rendering [S1, unofficial]. It serves as a precursor to more advanced techniques like ray marching with SDFs and delta tracking.

## Sources
[S1] 体渲染简明教程 - BimAnt, https://bimant.com/blog/volume-rendering-brief-intro/, fetched 2026-10-30 (unofficial)
