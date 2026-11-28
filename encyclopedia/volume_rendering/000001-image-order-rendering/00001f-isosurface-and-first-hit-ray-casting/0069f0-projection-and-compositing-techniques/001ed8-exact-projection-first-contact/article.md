# Exact Projection First Contact

Also called: FHC, First Hit, First-Hit Compositing, FirstHitShader

## Overview
Exact Projection First Contact is a volume rendering technique that terminates ray marching at the first voxel whose scalar value exceeds a specified threshold, rendering only the surface intersection [S1, unofficial][S2, unofficial][S3, unofficial][S4, unofficial]. It discards all data behind the first hit, providing a clean isosurface visualization with local shading [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume texture (typically `FloatType` with `RedFormat`), camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), a density threshold $\tau$ (iso-value), and optional lighting parameters [S1, unofficial][S4, unofficial].
*   **Output:** A fragment color (RGB) and alpha (1.0) representing the shaded surface at the first contact point [S2, unofficial][S3, unofficial].

**Data Structures:**
*   **Volume Texture:** A 3D texture object storing voxel values, often implemented as a `Data3DTexture` with linear filtering enabled to support interpolation [S1, unofficial].
*   **Ray Parameters:** The ray is defined as $r(t) = \mathbf{o} + t\mathbf{d}$, where $t$ is the distance along the ray [G]. The intersection with the volume bounding box (often normalized to $[0,1]^3$) is calculated using the slab method to determine entry ($t_{entry}$) and exit ($t_{exit}$) distances [S4, unofficial].

**Algorithm Steps:**

1.  **Ray-Volume Intersection:** Calculate the intersection of the view ray with the unit cube $[0,1]^3$. If $t_{entry} > t_{exit}$, the ray misses the volume; discard the pixel [S4, unofficial]. Clamp $t_{entry}$ to $\max(0.0, t_{entry})$ to ensure the start point is in front of the camera [S4, unofficial].
2.  **Initialization:** Set the current position $p_0 = \mathbf{o} + t_{entry}\mathbf{d}$ and sample the volume value $v_0 = \text{sample}(p_0)$ [S4, unofficial]. Initialize `firstContactVal` to $0.0$ and `firstContactPos` to $p_0$ [S2, unofficial].
3.  **Marching Loop:** Iterate along the ray from $t_{entry}$ to $t_{exit}$ with a step size $dt$. The step size is determined by the inverse of the volume dimensions and the ray direction components to ensure consistent voxel traversal [S4, unofficial]:
    $$ dt = \min \left( \frac{\Delta x}{|d_x|}, \frac{\Delta y}{|d_y|}, \frac{\Delta z}{|d_z|} \right) $$
    where $\Delta x, \Delta y, \Delta z$ are the voxel sizes (or inverse dimensions if normalized) [S4, unofficial].
4.  **Sampling and Detection:** At each step, advance to $p_1 = p_0 + \mathbf{d} \cdot dt$ and sample $v_1 = \text{sample}(p_1)$ [S4, unofficial]. Check if the isosurface condition is met: $v_1 \ge \tau$ [S3, unofficial][S4, unofficial].
5.  **Refinement:** If the condition is met, compute the linear interpolation factor $\alpha$ to find the exact intersection point:
    $$ \alpha = \frac{\tau - v_0}{v_1 - v_0} $$
    $$ p_{hit} = \text{mix}(p_0, p_1, \alpha) $$
    $$ v_{hit} = \text{mix}(v_0, v_1, \alpha) $$
    Store $p_{hit}$ as `firstContactPos` and $v_{hit}$ as `firstContactVal` [S4, unofficial]. Break the loop [S2, unofficial].
6.  **Fallback:** If the loop completes without finding a hit (or if `firstContactVal` remains $\le 0.0$), output black color `vec4(0.0)` and terminate [S2, unofficial].
7.  **Shading:**
    *   **Color Mapping:** Map `firstContactVal` to a base color using a color map texture: $c_{base} = \text{texture}(\text{colorMap}, \text{firstContactVal})$ [S2, unofficial].
    *   **Normal Calculation:** Compute the surface normal $\mathbf{n}$ using the gradient of the scalar field at $p_{hit}$, typically via finite differences or tri-linear filtering of the volume texture [S3, unofficial][S1, unofficial].
    *   **Lighting:** Apply a lighting model (e.g., Blinn-Phong, Phong, or Lambert) using $\mathbf{n}$, the view direction, and the light direction [S1, unofficial][S3, unofficial].
    *   **Occlusion/Shadows:** Optionally calculate ambient occlusion and shadows at $p_{hit}$ [S2, unofficial].
8.  **Output:** The final fragment color is $c_{final} = c_{shaded} \cdot c_{occlusion} \cdot c_{shadow}$ with alpha $1.0$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def exact_projection_first_contact(ray_origin, ray_direction, volume_texture, color_map, threshold, enable_lighting, enable_ao, enable_shadow):
    # 1. Ray-Volume Intersection (Slab Method for unit cube [0,1]^3)
    inv_dir = 1.0 / ray_direction
    t0 = (0.0 - ray_origin) * inv_dir
    t1 = (1.0 - ray_origin) * inv_dir
    t_entry = max(min(t0.x, t1.x), min(t0.y, t1.y), min(t0.z, t1.z))
    t_exit = min(max(t0.x, t1.x), max(t0.y, t1.y), max(t0.z, t1.z))
    
    if t_entry > t_exit:
        return vec4(0.0, 0.0, 0.0, 1.0)
    
    t_entry = max(t_entry, 0.0)
    
    # 2. Initialization
    current_pos = ray_origin + ray_direction * t_entry
    current_val = sample_volume(volume_texture, current_pos)
    
    first_contact_val = 0.0
    first_contact_pos = current_pos
    
    # 3. Marching Loop
    dt = calculate_step_size(ray_direction, volume_dimensions)
    
    t = t_entry
    max_iterations = 2000
    iterations = 0
    
    while t < t_exit and iterations < max_iterations:
        next_pos = current_pos + ray_direction * dt
        next_val = sample_volume(volume_texture, next_pos)
        
        if next_val >= threshold:
            alpha = (threshold - current_val) / (next_val - current_val)
            first_contact_pos = mix(current_pos, next_pos, alpha)
            first_contact_val = mix(current_val, next_val, alpha)
            break
            
        current_pos = next_pos
        current_val = next_val
        t += dt
        iterations += 1
        
    if first_contact_val <= 0.0:
        return vec4(0.0, 0.0, 0.0, 1.0)
        
    base_color = sample_color_map(color_map, first_contact_val)
    normal = calculate_gradient(volume_texture, first_contact_pos)
    normal = normalize(normal)
    
    if enable_lighting:
        shaded_color = calculate_lighting(normal, -ray_direction, first_contact_pos, base_color.rgb)
    else:
        shaded_color = base_color.rgb
        
    occlusion = calculate_ao(first_contact_pos) if enable_ao else 1.0
    shadow = calculate_shadow(first_contact_pos) if enable_shadow else 1.0
    
    final_color = shaded_color * occlusion * shadow
    return vec4(final_color, 1.0)

def sample_volume(texture, pos):
    # Trilinear interpolation of 3D texture at normalized position pos
    return texture.sample(pos)

def calculate_step_size(direction, dimensions):
    # Compute step size to cross one voxel along the ray
    dt_vec = vec3(1.0 / abs(direction.x), 1.0 / abs(direction.y), 1.0 / abs(direction.z))
    return min(dt_vec.x, dt_vec.y, dt_vec.z)

def sample_color_map(map_texture, value):
    # Sample 1D color map texture at normalized value
    return map_texture.sample(value)

def calculate_gradient(texture, pos):
    # Compute gradient via central finite differences
    eps = 0.001
    dx = texture.sample(pos + vec3(eps, 0, 0)) - texture.sample(pos - vec3(eps, 0, 0))
    dy = texture.sample(pos + vec3(0, eps, 0)) - texture.sample(pos - vec3(0, eps, 0))
    dz = texture.sample(pos + vec3(0, 0, eps)) - texture.sample(pos - vec3(0, 0, eps))
    return vec3(dx, dy, dz)

def calculate_lighting(normal, view_dir, pos, base_color):
    # Simple Blinn-Phong shading
    light_dir = normalize(vec3(1.0, 1.0, -1.0))
    half_dir = normalize(light_dir + view_dir)
    diff = max(dot(normal, light_dir), 0.0)
    spec = pow(max(dot(normal, half_dir), 0.0), 32.0)
    return base_color * (0.5 + 0.5 * diff) + vec3(0.3) * spec

def calculate_ao(pos):
    # Simple ambient occlusion approximation
    return 0.8

def calculate_shadow(pos):
    # Placeholder for shadow calculation
    return 1.0
```

**Parameters:**
*   `threshold` ($\tau$): The density/isovalue threshold. Default values vary by dataset; e.g., $0.5$ for normalized data [S1, unofficial].
*   `steps`: Number of ray marching steps. Default 200, can be overridden to 500 [S1, unofficial].
*   `MAX_LAYERS`: Maximum number of iso-surface layers supported (e.g., 3) [S1, unofficial].
*   `render_mode`: Integer flag where $0$ selects First-Hit Compositing [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Ray Direction Normalization:** The ray direction interpolated from the vertex shader may not be normalized; it must be explicitly normalized before use [S4, unofficial].
*   **No Hit:** If no voxel exceeds the threshold, the pixel is typically rendered black or transparent [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of steps along the ray until the first hit.
*   **Memory Complexity:** $O(1)$ per pixel for the raycasting logic, plus the memory for the volume texture and color map.
*   **Performance:** Fast for surfaces that are close to the camera or have high contrast, as it terminates early.
*   **Strengths:** Provides clean surface visualization with shading; computationally cheaper than full volume compositing if hits are frequent and early.
*   **Limitations:** No internal structure is visible; sensitive to threshold selection; artifacts if step size is too large relative to surface curvature.
*   **When to use:** When the primary interest is the outer surface of a volume (e.g., bone rendering in CT, skin rendering in MRI) and internal details are not required [S1, unofficial].

## Background
Exact Projection First Contact is a fundamental technique in direct volume rendering, often implemented as a shader in interactive visualization tools. It is closely related to the concept of "first-hit" rendering used in medical imaging viewers for isolating specific tissue types (e.g., bones, skin) [S1, unofficial]. The method is implemented in various open-source projects such as `volume_vis` (using Three.js) and `VisualNifty` [S1, unofficial][S2, unofficial]. It is also featured in educational implementations of direct volume rendering, such as those found in computer graphics lecture notes [S4, unofficial].

## Sources
[S1] niklasgruener/volume_vis, https://github.com/niklasgruener/volume_vis, fetched 2026-10-26 (unofficial)
[S2] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S3] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S4] Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, https://github.com/Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, fetched 2026-10-26 (unofficial)
