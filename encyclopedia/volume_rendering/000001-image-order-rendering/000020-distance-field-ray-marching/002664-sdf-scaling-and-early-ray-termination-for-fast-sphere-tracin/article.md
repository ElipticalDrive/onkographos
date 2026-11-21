# SDF Scaling and Early Ray Termination for Fast Sphere Tracing

Also called: SDF Scaling, Two-pass SDF Scaling

## Overview
This method accelerates sphere tracing by decoupling the distance estimation from the final high-resolution rendering into a two-pass pipeline: a low-resolution pre-processing pass and a full-resolution main pass [S1]. It solves the performance bottleneck of standard sphere tracing by using a single low-resolution buffer to provide a "head start" for full-resolution rays, positioning them closer to surfaces, while applying SDF scaling to ensure small features are not missed at low resolution [S1]. Unlike Inverse Pyramid Rendering (IPR) which uses a pyramidal render target hierarchy and filtering, this method uses a single low-resolution buffer, reducing memory cost and complexity while offering similar quality [S1].

## How it works
The method operates in two distinct passes. The first pass renders primary and shadow rays at a reduced resolution to generate an approximate distance field. The second pass uses this approximation to initialize rays closer to the geometry, significantly reducing the number of steps required for convergence.

### Inputs and Outputs
*   **Inputs:** Camera parameters (origin $\mathbf{o}_{cam}$, direction $\mathbf{d}$), SDF scene definition, target output resolution $R_{out}$, low-resolution buffer resolution $R_{low}$, scaling factor $\lambda$, early termination epsilon $\epsilon_{early}$, and shadow hardness $k$.
*   **Outputs:** Final rendered image at $R_{out}$ and depth/distance buffer at $R_{low}$.

### Data Structures
1.  **Low-Resolution Buffer:** A 2D array storing the accumulated distance $t_{approx}$ for each pixel. Size is $R_{low} \times R_{low}$.
2.  **Ray Bundle:** In the main pass, each pixel in the low-resolution buffer corresponds to a $4 \times 4$ block of pixels (16 rays) in the full-resolution pass [S1].

### Algorithm Steps

#### Pass 1: Pre-processing (Low Resolution)
1.  **Initialize Buffer:** Create a low-resolution buffer with dimensions $R_{low} = R_{out} / 4$ (e.g., 960x540 for 4K output) [S1].
2.  **Ray Casting:** For each pixel in the low-resolution buffer, cast a ray from the camera origin $\mathbf{o}_{cam}$ with direction $\mathbf{d}$.
3.  **SDF Scaling Logic:** Before evaluating the SDF $f(\mathbf{p})$ at the current position $\mathbf{p}$, determine the sign of the distance to the nearest surface.
    *   If the camera is in front/outside ($f(\mathbf{p}) > 0$), scale up (inflate) the SDF volume. The effective distance becomes $f'(\mathbf{p}) = f(\mathbf{p}) \cdot \lambda$, where $\lambda > 1$ [S1].
    *   If the camera is behind/inside ($f(\mathbf{p}) < 0$), scale down (deflate) the SDF volume. The effective distance becomes $f'(\mathbf{p}) = f(\mathbf{p}) / \lambda$ [S1].
    *   *Hole Handling:* For SDFs representing holes (via subtraction), if the camera is in front, decrease the volume size (deflate) to improve approximation; if behind, scale up (inflate) [S1].
4.  **Sphere Tracing with Early Termination:**
    *   Perform sphere tracing using the scaled SDF $f'(\mathbf{p})$.
    *   Step size: $\Delta t = f'(\mathbf{p})$.
    *   Termination Condition: Stop when $f'(\mathbf{p}) < \epsilon_{early}$, where $\epsilon_{early}$ is a larger epsilon than typically used in high-resolution passes [S1]. This avoids surface overlaps caused by scaled SDFs and provides a fast distance approximation.
    *   Store the accumulated distance $t_{approx}$ in the low-resolution buffer.
5.  **Shadow Rays (Optional in Pass 1):** Shadow rays can also be traced in this pass using the scaled SDFs to generate initial shadow data, though the primary benefit is the distance buffer for Pass 2 [S1].

#### Pass 2: Main Pass (Full Resolution)
1.  **Ray Origin Reconstruction:** For each pixel in the full-resolution output, determine its corresponding low-resolution pixel (e.g., by dividing coordinates by 4). Retrieve the approximate distance $t_{approx}$ from the low-resolution buffer [S1].
2.  **Reconstruct Ray Origin:** Instead of starting from the camera origin $\mathbf{o}_{cam}$, set the new ray origin $\mathbf{o}_{new}$ to:
    $$ \mathbf{o}_{new} = \mathbf{o}_{cam} + \mathbf{d} \cdot t_{approx} $$
    This positions the ray closer to the surface, reducing the remaining distance to trace [S1].
3.  **Sphere Tracing:** Perform standard sphere tracing from $\mathbf{o}_{new}$ using the original, unscaled SDF $f(\mathbf{p})$.
    *   Step size: $\Delta t = f(\mathbf{p})$.
    *   Termination: Use standard epsilon $\epsilon$ for surface hit detection.
4.  **Shadow Tracing:** For shadow rays, use the approximated distance $t_{approx}$ to determine the light source distance. Set shadow hardness $k$ to control the penumbra size (higher $k$ results in harder shadows) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sphere_tracing_with_scaling_and_early_term(camera_origin, camera_direction, sdf_func, resolution, low_res_factor, lambda_scale, epsilon_early, epsilon_hit, shadow_hardness_k):
    """
    Two-pass sphere tracing with SDF scaling and early termination.
    
    Args:
        camera_origin: vec3, origin of the camera.
        camera_direction: vec3, direction of the ray for a specific pixel.
        sdf_func: function, returns signed distance to scene geometry.
        resolution: tuple, (width, height) of the final output.
        low_res_factor: int, factor by which to reduce resolution (e.g., 4).
        lambda_scale: float, scaling factor for SDF inflation/deflation (>1).
        epsilon_early: float, large epsilon for early termination in Pass 1.
        epsilon_hit: float, small epsilon for surface hit in Pass 2.
        shadow_hardness_k: float, parameter for shadow hardness.
        
    Returns:
        color: vec3, final pixel color.
        depth: float, accumulated distance.
    """
    
    # --- Pass 1: Low Resolution Pre-processing ---
    low_res_w = resolution[0] // low_res_factor
    low_res_h = resolution[1] // low_res_factor
    
    # Initialize low-res buffer to store approximate distances
    # In a real implementation, this would be a texture/buffer shared between passes
    low_res_buffer = create_buffer(low_res_w, low_res_h)
    
    for y in range(low_res_h):
        for x in range(low_res_w):
            # Map low-res pixel to a representative ray direction
            # Note: In practice, this might average directions or use a representative ray
            ray_dir = get_ray_direction_for_pixel(x, y, low_res_w, low_res_h, camera_origin, camera_direction)
            
            t_approx = 0.0
            p = camera_origin
            
            # Sphere tracing loop for Pass 1
            for step in range(100): # Max steps for low-res
                dist = sdf_func(p)
                
                # SDF Scaling Logic
                if dist > 0:
                    # Camera in front/outside: Inflate SDF
                    scaled_dist = dist * lambda_scale
                else:
                    # Camera behind/inside: Deflate SDF
                    scaled_dist = dist / lambda_scale
                
                # Hole handling: If sdf_func identifies a hole, reverse scaling logic
                # if is_hole(p):
                #     if dist > 0: scaled_dist = dist / lambda_scale
                #     else: scaled_dist = dist * lambda_scale
                
                # Early Termination
                if scaled_dist < epsilon_early:
                    break
                
                # Step
                t_approx += scaled_dist
                p = camera_origin + ray_dir * t_approx
            
            # Store result in low-res buffer
            low_res_buffer[y, x] = t_approx

    # --- Pass 2: Full Resolution Main Pass ---
    final_color = vec3(0.0)
    final_depth = 0.0
    
    # For each pixel in full resolution
    for y in range(resolution[1]):
        for x in range(resolution[0]):
            # Map to low-res buffer
            lr_x = x // low_res_factor
            lr_y = y // low_res_factor
            
            # Retrieve approximate distance
            t_approx = low_res_buffer[lr_y, lr_x]
            
            # Reconstruct ray origin
            new_origin = camera_origin + camera_direction * t_approx
            
            # Sphere tracing loop for Pass 2
            t_final = t_approx
            p_final = new_origin
            
            for step in range(100): # Max steps for high-res
                dist = sdf_func(p_final)
                
                # Standard Sphere Tracing (no scaling)
                if abs(dist) < epsilon_hit:
                    break
                
                t_final += dist
                p_final = new_origin + camera_direction * dist
            
            # Compute color and shadow
            color = compute_shading(p_final, camera_direction, sdf_func, shadow_hardness_k, t_approx)
            
            # Aggregate color (e.g., for anti-aliasing, average over 4x4 block)
            final_color += color
            
    return final_color / (low_res_factor ** 2), t_final

def compute_shading(p, ray_dir, sdf_func, k, t_approx):
    """
    Computes shading and shadow for a point p.
    Uses t_approx for shadow ray initialization.
    """
    # Normal estimation
    n = estimate_normal(p, sdf_func)
    
    # Lighting calculation (simplified)
    light_dir = normalize(vec3(1.0, 1.0, 1.0))
    diffuse = max(dot(n, light_dir), 0.0)
    
    # Shadow ray with early termination using t_approx
    shadow_factor = 1.0
    shadow_origin = p + n * 0.001
    shadow_dist = t_approx # Use approximated distance to light
    
    # Trace shadow ray
    # If shadow_dist is very small, assume lit
    if shadow_dist > 0.001:
        shadow_p = shadow_origin
        for step in range(10):
            d_light = sdf_func(shadow_p)
            if d_light < 0.0:
                shadow_factor = 0.0
                break
            shadow_p += normalize(light_dir) * d_light
            if distance(shadow_p, light_pos) > shadow_dist + k:
                break
                
    return vec3(diffuse * shadow_factor)
```

### Parameters
*   **Low-resolution buffer size:** Typically $1/4$ of the output resolution in each axis (e.g., 960x540 for 4K) [S1].
*   **Ray Batching:** Each low-resolution pixel serves 16 full-resolution pixels ($4 \times 4$ block) [S1].
*   **Scaling Factor ($\lambda$):** Not explicitly given in sources, but must be $>1$ to inflate and $<1$ to deflate.
*   **Epsilon Early ($\epsilon_{early}$):** Larger than standard epsilon to terminate early and avoid overlaps from scaled SDFs.
*   **Shadow Hardness ($k$):** Controls shadow penumbra; higher values yield harder shadows [S1].

### Pitfalls
*   **Rounded Edges:** High values of $\epsilon_{early}$ cause shapes with edges to become rounded, eliminating sharp features and providing an inaccurate approximation [S1].
*   **SDF Overlaps:** Scaling SDFs can cause them to overlap incorrectly. Early termination is required to prevent rays from penetrating scaled surfaces [S1].
*   **Hole Approximation:** Incorrect scaling logic for holes (subtracted SDFs) can lead to visual artifacts if the camera position relative to the hole is not correctly detected [S1].

## Operating characteristics
*   **Time Complexity:** The method reduces the average number of sphere tracing steps significantly. In the 'Primitives' scene, it achieved a 3x speedup in frame time compared to state-of-the-art methods without anti-aliasing on an NVIDIA RTX-3070Ti [S1].
*   **Shadow Ray Performance:** Shadow rays often finish after 1 step in simple scenes, achieving a 2.1x performance improvement over ground truth in the 'Primitives' scene [S1]. In complex scenes like 'Sponza atrium', performance improved by 1.5x despite increased complexity (shadows taking up to 20 ms) [S1].
*   **Memory Complexity:** Uses a single low-resolution buffer, which is significantly smaller than the pyramidal hierarchy used in Inverse Pyramid Rendering (IPR), reducing memory cost [S1].
*   **Strengths:** Reduces rendering time significantly, allows for anti-aliasing without increasing frame time cost, and handles small features better than standard low-res methods due to SDF scaling [S1].
*   **Limitations:** Introduces artifacts such as rounded edges if $\epsilon_{early}$ is too high. Accuracy depends on the quality of the low-resolution approximation [S1].

## Background
The method was proposed in the paper "Skipping Spheres: SDF Scaling & Early Ray Termination for Fast Sphere Tracing" [S1]. It draws inspiration from prior techniques aimed at reducing ray steps by adjusting step size and generating lower resolution renders [PKM23]. The authors note that while low-resolution buffers can speed up rendering, they often lead to artifacts in small features, necessitating filtering. This method minimizes artifacts while cutting down steps compared to previous methods by using a single low-resolution buffer and a novel tracing optimization [S1].

## Sources
[S1] Skipping Spheres: SDF Scaling & Early Ray Termination for Fast Sphere ..., https://diglib.eg.org/items/7f21e58c-639e-4359-bf94-414738612006, fetched 2026-10-27
