# Adaptive Auxiliary Texture Resolution for Isosurface Raycasting

Also called: Adaptive Algorithm for Accelerating Direct Isosurface Rendering, Adaptive Algorithm for Accelerating Direct Isosurface Rendering on GPU, Adaptive Two-Pass Isosurface Rendering

## Overview
This method accelerates direct isosurface volume rendering on GPUs by using a two-pass approach that avoids preprocessing [S1, unofficial]. It renders ray-isosurface intersections into a low-resolution auxiliary texture for a subset of rays, then uses this data to predict intersection points for all other rays in the second pass [S1, unofficial]. Unlike preprocessing-based methods (e.g., octrees, kd-trees), it constructs auxiliary data every frame, allowing dynamic adaptation of sampling density to prevent artifacts while maintaining high performance [S1, unofficial].

## How it works
**Inputs:**
- Scalar volume dataset $V$.
- Transfer function $T_f(v)$ mapping scalar values to opacity/color.
- Isosurface threshold $\tau$.
- Camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$ per pixel).
- Adaptive resolution ratio $n$ (ratio of screen resolution to auxiliary texture resolution).

**Outputs:**
- Final rendered image.
- Updated $n$ for the next frame.

**Data Structures:**
- **Auxiliary Texture $T$**: A 2D texture with resolution $W/n \times H/n$, where $W \times H$ is the screen resolution. Each texel stores the depth ($z$-coordinate in view space) of the first isosurface intersection for the corresponding ray. If no intersection is found, it stores a value indicating "empty" (e.g., max depth or a specific flag).
- **Final Image Buffer**: Standard color buffer.

**Algorithm Steps:**

1.  **Pass 1: Low-Resolution Intersection Rendering**
    - Render the volume into auxiliary texture $T$ at resolution $W/n \times H/n$.
    - For each texel in $T$, cast a ray through the volume and find the first intersection with the isosurface $f(\mathbf{p}) = \tau$.
    - Store the depth $z$ of this intersection in the texel. If no intersection is found, store a "no-hit" value.

2.  **Pass 2: High-Resolution Raycasting with Prediction**
    - For each pixel $p$ in the final image, determine the starting point for ray tracing using the depth information from $T$.
    - Let $d_1, d_2, \dots, d_k$ be the depths of the neighboring texels in $T$ surrounding the pixel $p$'s corresponding region.
    - **Case 1: Concave Surface (Start at minimum depth)**
      - If the surface is concave relative to the camera, start ray tracing at $z_{start} = \min(d_1, d_2)$. This ensures the ray starts as close as possible to the surface without missing it [S1, unofficial].
    - **Case 2: Convex Surface (Direction Selection)**
      - If the surface is convex, simply starting at the minimum depth might cause the ray to miss the surface if it moves in the wrong direction. Evaluate the transfer function $T_f$ at the predicted hit point.
      - If $T_f < \text{threshold}$, move in the ray direction.
      - If $T_f \ge \text{threshold}$, move in the opposite direction [S1, unofficial].
    - **Case 3: Culling**
      - If surrounding texels in $T$ indicate no intersection with the isosurface (e.g., all are "no-hit"), cull the pixel and skip raycasting [S1, unofficial].
    - **Case 4: Outlines (Red/White Pixels)**
      - Pixels on object outlines may require many raytracing steps if they do not intersect the isosurface. These are rare but costly. The algorithm handles them by full raycasting from the near plane if prediction fails [S1, unofficial].

3.  **Adaptive Control (CPU Side, Parallel to Pass 2)**
    - Analyze the auxiliary texture $T$ from the current frame on the CPU.
    - Count **lone texels**: texels containing the isosurface but whose neighbors do not.
    - Count **tooth texels**: texels containing the isosurface and having only one neighbor that also contains the isosurface.
    - Calculate the ratio $S_i = \frac{\text{count(lone)} + \text{count(tooth)}}{\text{total texels in } T}$.
    - Determine $n_{i+1}$ for the next frame using the update rule:
      $$
      n_{i+1} = 
      \begin{cases} 
      n_i & \text{if } S_{min} \le S_i \le S_{max} \\
      n_i - 1 & \text{if } S_i > S_{max} \\
      n_i + 1 & \text{if } S_i < S_{min}
      \end{cases}
      $$
    - Constraints: $n \in [2, 5]$. $S_{max}$ is a user-defined quality parameter, and $S_{min} = S_{max}/2$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_isosurface_rendering(volume, transfer_function, threshold, camera, n):
    """
    Main rendering loop for one frame.
    """
    # Pass 1: Render to low-res auxiliary texture
    aux_texture = render_auxiliary_texture(volume, transfer_function, threshold, camera, n)
    
    # Pass 2: Render final image using aux_texture for prediction
    final_image = render_final_image(volume, transfer_function, threshold, camera, aux_texture, n)
    
    # Adaptive Control: Analyze aux_texture on CPU
    lone_count = count_lone_texels(aux_texture)
    tooth_count = count_tooth_texels(aux_texture)
    total_texels = aux_texture.width * aux_texture.height
    S_i = (lone_count + tooth_count) / total_texels
    
    # Update n for next frame
    S_max = 0.1  # Example user-defined value
    S_min = S_max / 2
    
    if S_min <= S_i <= S_max:
        n_next = n
    elif S_i > S_max:
        n_next = max(2, n - 1)
    else:  # S_i < S_min
        n_next = min(5, n + 1)
        
    return final_image, n_next

def render_auxiliary_texture(volume, transfer_function, threshold, camera, n):
    """
    Renders intersections into a texture of size W/n x H/n.
    """
    width = camera.width // n
    height = camera.height // n
    aux_texture = new_texture(width, height)
    
    for y in range(height):
        for x in range(width):
            # Map texel to screen pixel region
            screen_x = x * n
            screen_y = y * n
            
            # Cast ray for this texel (e.g., center of the region)
            ray = camera.get_ray(screen_x + n/2, screen_y + n/2)
            hit_depth = find_first_intersection(volume, ray, threshold)
            
            if hit_depth is not None:
                aux_texture.set(x, y, hit_depth)
            else:
                aux_texture.set(x, y, NO_HIT_VALUE)
                
    return aux_texture

def render_final_image(volume, transfer_function, threshold, camera, aux_texture, n):
    """
    Renders the final image using aux_texture for starting points.
    """
    width = camera.width
    height = camera.height
    final_image = new_image(width, height)
    
    for y in range(height):
        for x in range(width):
            # Determine starting point from aux_texture
            aux_x = x // n
            aux_y = y // n
            
            # Get neighboring depths in aux_texture
            neighbors = get_neighbor_depths(aux_texture, aux_x, aux_y)
            
            # Predict starting depth
            start_depth = predict_start_depth(neighbors, transfer_function, x, y, camera)
            
            # Cast ray from predicted start
            ray = camera.get_ray(x, y)
            hit = trace_ray_from(volume, ray, start_depth, threshold)
            
            if hit is not None:
                final_image.set(x, y, compute_color(hit, volume, transfer_function))
            else:
                final_image.set(x, y, BACKGROUND_COLOR)
                
    return final_image

def predict_start_depth(neighbors, transfer_function, x, y, camera):
    """
    Predicts the starting depth for raycasting based on neighboring texels.
    """
    # Case 1: Concave surface - start at min depth
    if is_concave(neighbors):
        return min(neighbors)
    
    # Case 2: Convex surface - check transfer function
    min_depth = min(neighbors)
    # Evaluate transfer function at the predicted hit point
    # Note: This requires reconstructing the 3D point from depth
    point = camera.get_point_at_depth(min_depth, x, y)
    value = transfer_function.evaluate(point)
    
    if value < threshold:
        # Move in ray direction
        return min_depth
    else:
        # Move in opposite direction (backwards)
        # This is a simplification; actual implementation may vary
        return max_depth(neighbors)

def count_lone_texels(aux_texture):
    """
    Counts texels with isosurface but no neighbors with isosurface.
    """
    count = 0
    for y in range(aux_texture.height):
        for x in range(aux_texture.width):
            if aux_texture.has_intersection(x, y):
                if not any(aux_texture.has_intersection(nx, ny) 
                           for nx, ny in get_neighbors(x, y)):
                    count += 1
    return count

def count_tooth_texels(aux_texture):
    """
    Counts texels with isosurface and exactly one neighbor with isosurface.
    """
    count = 0
    for y in range(aux_texture.height):
        for x in range(aux_texture.width):
            if aux_texture.has_intersection(x, y):
                neighbor_count = sum(1 for nx, ny in get_neighbors(x, y) 
                                     if aux_texture.has_intersection(nx, ny))
                if neighbor_count == 1:
                    count += 1
    return count
```

**Parameters:**
- $n$: Ratio between screen resolution and auxiliary texture resolution. Typical optimal value is around 5 [S1, unofficial]. Must be lower (e.g., 2-3) for scenes with fine detail or large curvature to prevent artifacts [S1, unofficial]. Range is $[2, 5]$ [S1, unofficial].
- $S_{max}$: User-defined quality parameter. $S_{min} = S_{max}/2$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Outlines (Red/White Pixels):** Pixels on object outlines may require many raytracing steps if they do not intersect the isosurface. These are rare but costly [S1, unofficial].
- **High $n$ (Low Resolution):** Increases red/white regions, leading to decreased performance and artifacts (data loss) for fine details [S1, unofficial].
- **Precision:** Depth prediction relies on accurate neighbor sampling. Incorrect prediction can lead to missed intersections or incorrect shading [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Pass 1 is $O((W/n) \times (H/n))$. Pass 2 is $O(W \times H)$ but with reduced raytracing steps due to prediction. Adaptive control is $O((W/n) \times (H/n))$. Overall, it is faster than standard raycasting by up to 10 times [S1, unofficial].
- **Memory Complexity:** Requires storage for the auxiliary texture $T$, which is $1/n^2$ the size of the screen buffer [S1, unofficial].
- **Performance:** Acceleration up to 10 times compared to common GPU raycasting methods [S1, unofficial]. Optimal $n$ is around 5 for general scenes [S1, unofficial].
- **Strengths:** No preprocessing required; adaptive to scene complexity; fast on GPUs [S1, unofficial].
- **Limitations:** Artifacts can occur if $n$ is too high for complex scenes; outline pixels are costly [S1, unofficial].

## Background
This method was proposed by Belyaev et al. in 2018 [S1, unofficial]. It addresses the need for fast, dynamic isosurface rendering without the overhead of preprocessing, which is problematic for dynamic data or changing transfer functions [S1, unofficial]. It is particularly relevant for medical data visualization where real-time interaction is crucial [S1, unofficial].

## Sources
[S1] Adaptive Algorithm for Accelerating Direct Isosurface Rendering on GPU, https://doi.org/10.11989/jest.1674-862x.71013102, fetched 2026-10-30 (unofficial)
