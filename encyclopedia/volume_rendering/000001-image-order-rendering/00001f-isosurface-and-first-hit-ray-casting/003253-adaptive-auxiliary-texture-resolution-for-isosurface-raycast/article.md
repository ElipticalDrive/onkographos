# Adaptive Auxiliary Texture Resolution for Isosurface Raycasting

Also called: Adaptive Algorithm for Accelerating Direct Isosurface Rendering, Adaptive Algorithm for Accelerating Direct Isosurface Rendering on GPU, Adaptive Two-Pass Isosurface Rendering

## Overview
This method accelerates direct isosurface volume rendering on GPUs by using a two-pass approach that avoids preprocessing [S1, unofficial]. It renders ray-isosurface intersections into a low-resolution auxiliary texture for a subset of rays, then uses this data to predict intersection points for all other rays in the second pass [S1, unofficial]. Unlike preprocessing-based methods (e.g., octrees, kd-trees), it constructs auxiliary data every frame, allowing dynamic adaptation of sampling density to prevent artifacts while maintaining high performance [S1, unofficial].

## How it works
**Inputs:**
- Scalar volume dataset $V$ [S1, unofficial].
- Transfer function $T_f$ mapping scalar values to opacity/color [S1, unofficial].
- Isosurface threshold $\tau$ [S1, unofficial].
- Camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$ per pixel) [S1, unofficial].
- Adaptive resolution ratio $n$ (ratio of screen resolution to auxiliary texture resolution) [S1, unofficial].

**Outputs:**
- Final rendered image [S1, unofficial].
- Updated $n$ for the next frame [S1, unofficial].

**Data Structures:**
- **Auxiliary Texture $T$**: A 2D texture with resolution $W/n \times H/n$, where $W \times H$ is the screen resolution. Each texel stores the depth ($z$-coordinate in view space) of the first isosurface intersection for the corresponding ray [S1, unofficial].
- **Final Image Buffer**: Standard color buffer [G].

**Algorithm Steps:**

1.  **Pass 1: Low-Resolution Intersection Rendering**
    - Render the volume into auxiliary texture $T$ at resolution $W/n \times H/n$ to find ray-isosurface intersections for a subset of rays [S1, unofficial].
    - For each texel in $T$, cast a ray through the volume and find the first intersection with the isosurface $f(\mathbf{p}) = \tau$ [S1, unofficial].
    - Store the depth $z$ of this intersection in the texel [S1, unofficial].

2.  **Pass 2: High-Resolution Raycasting with Prediction**
    - For each pixel $p$ in the final image, determine the starting point for ray tracing using the depth information from $T$ [S1, unofficial].
    - Let $d_1, d_2, \dots, d_k$ be the depths of the neighboring texels in $T$ surrounding the pixel $p$'s corresponding region [S1, unofficial].
    - **Case 1: Concave Surface (Start at minimum depth)**
      - If the surface is concave relative to the camera, start ray tracing at $z_{start} = \min(d_1, d_2)$ [S1, unofficial].
    - **Case 2: Convex Surface (Direction Selection)**
      - If the surface is convex, evaluate the transfer function $T_f$ at the predicted hit point [S1, unofficial].
      - If $T_f < \tau$, move in the ray direction [S1, unofficial].
      - If $T_f \ge \tau$, move in the opposite direction [S1, unofficial].
    - **Case 3: Culling**
      - If surrounding texels in $T$ indicate no intersection with the isosurface, cull the pixel and skip raycasting [S1, unofficial].

3.  **Adaptive Control (CPU Side, Parallel to Pass 2)**
    - Analyze the auxiliary texture $T$ from the current frame on the CPU [S1, unofficial].
    - Count **lone texels**: texels containing the isosurface but whose neighbors do not [S1, unofficial].
    - Count **tooth texels**: texels containing the isosurface and having only one neighbor that also contains the isosurface [S1, unofficial].
    - Calculate the ratio $S_i = \frac{\text{count(lone)} + \text{count(tooth)}}{\text{total texels in } T}$ [S1, unofficial].
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
    aux_width = camera.width // n
    aux_height = camera.height // n
    aux_texture = [[None] * aux_width for _ in range(aux_height)]
    
    # Pass 1: Render to low-resolution auxiliary texture
    for y in range(aux_height):
        for x in range(aux_width):
            sx, sy = x * n + n // 2, y * n + n // 2
            origin = camera.get_origin(sx, sy)
            direction = camera.get_direction(sx, sy)
            hit_depth = find_first_intersection(volume, origin, direction, threshold)
            aux_texture[y][x] = hit_depth if hit_depth is not None else float('inf')
            
    # Pass 2: Render final image using aux_texture for prediction
    final_image = [[BACKGROUND_COLOR] * camera.width for _ in range(camera.height)]
    
    for y in range(camera.height):
        for x in range(camera.width):
            aux_x, aux_y = x // n, y // n
            neighbors = get_neighbor_depths(aux_texture, aux_x, aux_y)
            
            start_depth = predict_start_depth(neighbors, transfer_function, x, y, camera, threshold)
            
            if start_depth == float('inf'):
                continue # Cull pixel if no neighbors indicate intersection
            
            origin = camera.get_origin(x, y)
            direction = camera.get_direction(x, y)
            hit = trace_ray_from(volume, origin, direction, start_depth, threshold)
            
            if hit is not None:
                final_image[y][x] = compute_color(hit, volume, transfer_function)
                
    # Adaptive Control: Analyze aux_texture on CPU
    lone_count = 0
    tooth_count = 0
    total_texels = aux_width * aux_height
    
    for y in range(aux_height):
        for x in range(aux_width):
            if aux_texture[y][x] != float('inf'):
                neighbor_hits = sum(1 for nx, ny in get_neighbors(aux_width, aux_height, x, y) 
                                    if aux_texture[ny][nx] != float('inf'))
                if neighbor_hits == 0:
                    lone_count += 1
                elif neighbor_hits == 1:
                    tooth_count += 1
                    
    S_i = (lone_count + tooth_count) / total_texels
    S_max = 0.1  # User-defined quality parameter
    S_min = S_max / 2
    
    if S_min <= S_i <= S_max:
        n_next = n
    elif S_i > S_max:
        n_next = max(2, n - 1)
    else:  # S_i < S_min
        n_next = min(5, n + 1)
        
    return final_image, n_next

def find_first_intersection(volume, origin, direction, threshold):
    # [G] Standard ray-volume intersection and marching to find f(p) = threshold
    # Returns depth z in view space, or None if no intersection
    pass

def get_neighbor_depths(texture, x, y):
    neighbors = []
    for dy in [-1, 0, 1]:
        for dx in [-1, 0, 1]:
            if dx == 0 and dy == 0: continue
            nx, ny = x + dx, y + dy
            if 0 <= nx < len(texture[0]) and 0 <= ny < len(texture):
                d = texture[ny][nx]
                if d != float('inf'):
                    neighbors.append(d)
    return neighbors

def predict_start_depth(neighbors, transfer_function, x, y, camera, threshold):
    if not neighbors:
        return float('inf')
    min_depth = min(neighbors)
    # Concave case: min_depth is strictly less than all other neighbors
    if min_depth < all(d for d in neighbors if d != min_depth):
        return min_depth
    # Convex case: evaluate transfer function at predicted point
    point = camera.get_point_at_depth(min_depth, x, y)
    value = transfer_function.evaluate(point)
    if value < threshold:
        return min_depth
    else:
        return max(neighbors)

def trace_ray_from(volume, origin, direction, start_depth, threshold):
    # [G] Marches ray from start_depth along direction to find isosurface
    pass

def compute_color(hit, volume, transfer_function):
    # [G] Computes color using local lighting and transfer function
    pass
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
