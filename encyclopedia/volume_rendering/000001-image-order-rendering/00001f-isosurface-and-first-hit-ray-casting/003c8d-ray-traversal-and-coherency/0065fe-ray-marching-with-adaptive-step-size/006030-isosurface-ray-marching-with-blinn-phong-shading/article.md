# Isosurface Ray Marching with Blinn-Phong Shading

Also called: Ray Marching with Slab Intersection and Blinn-Phong Shading, isosurface rendering via ray marching

## Overview
Isosurface Ray Marching with Blinn-Phong Shading is an image-order rendering technique that casts rays from the camera through each pixel, intersects them with the volume's axis-aligned bounding box (AABB), and marches along the ray to find the first voxel exceeding a specific intensity threshold [S1, unofficial]. It differs from its sibling [Differential Sampling and Peak Finding](1533-differential-sampling-and-peak-finding.md) by using a fixed-step march with binary-refinement rather than derivative-based peak detection, and from [Front-to-Back Surface Extraction](12182-front-to-back-surface-extraction.md) by stopping at the first hit rather than accumulating transmittance [S1, unofficial]. The method computes surface shading using a Blinn-Phong model, providing specular highlights that define the visual appearance of the extracted isosurface [S1, unofficial].

## How it works
**Inputs:**
- Camera parameters: viewport size, aspect ratio, focal length, view matrix, background color, gamma.
- Volume parameters: AABB defined by `bottom` and `top` vectors, volume texture (sampled for intensity), threshold value, step length.
- Shading parameters: light position, material color, ambient coefficient ($I_a$), diffuse coefficient ($I_d$), specular exponent, specular coefficient.

**Outputs:**
- Final pixel color with gamma correction applied.

**Data Structures:**
- `ray_origin`: `vec3`, camera position.
- `ray_direction`: `vec3`, normalized direction vector.
- `box.top`, `box.bottom`: `vec3`, corners of the AABB.
- `volume`: Texture object supporting `texture(volume, position)` returning a `vec4` (intensity in `.r`).

**Steps:**

1. **Compute Ray Direction:**
   Calculate the ray direction from fragment coordinates. Let $u, v$ be the fragment coordinates, $W, H$ be the viewport width and height, and $A$ be the aspect ratio.
   $$
   \begin{aligned}
   ray\_direction.xy &= 2.0 \cdot \frac{[u, v]}{[W, H]} - 1.0 \\
   ray\_direction.x &*= A \\
   ray\_direction.z &= -focal\_length \\
   ray\_direction &= \text{normalize}((\text{ViewMatrix} \cdot [ray\_direction, 0]).xyz)
   \end{aligned}
   $$
   [S1, unofficial]

2. **Slab Intersection (Ray-AABB):**
   Find the entry ($t_0$) and exit ($t_1$) distances of the ray with the AABB.
   $$
   \begin{aligned}
   direction\_inv &= 1.0 / ray\_direction \\
   t\_top &= direction\_inv \cdot (box.top - ray\_origin) \\
   t\_bottom &= direction\_inv \cdot (box.bottom - ray\_origin) \\
   t\_min &= \min(t\_top, t\_bottom) \\
   t_0 &= \max(0.0, \max(t\_min.x, t\_min.y)) \\
   t\_max &= \max(t\_top, t\_bottom) \\
   t_1 &= \min(t\_max.x, t\_max.y)
   \end{aligned}
   $$
   If $t_0 > t_1$, the ray misses the box. [S1, unofficial]

3. **Transform to Texture Coordinates:**
   Map the entry and exit points in world space to volume texture coordinates (normalized $[0, 1]$).
   $$
   \begin{aligned}
   ray\_start &= \frac{ray\_origin + ray\_direction \cdot t_0 - box.bottom}{box.top - box.bottom} \\
   ray\_stop &= \frac{ray\_origin + ray\_direction \cdot t_1 - box.bottom}{box.top - box.bottom}
   \end{aligned}
   $$
   [S1, unofficial]

4. **Initialize Marching:**
   Calculate the step vector in texture space. Let $L$ be the length of the ray in texture space ($|ray\_stop - ray\_start|$) and $s$ be the `step_length`.
   $$
   \begin{aligned}
   ray &= ray\_stop - ray\_start \\
   L &= |ray| \\
   step\_vector &= s \cdot \frac{ray}{L}
   \end{aligned}
   $$
   Initialize position and accumulated color.
   $$
   \begin{aligned}
   position &= ray\_start + step\_vector \\
   colour &= \text{pow}(background\_colour, \gamma)
   \end{aligned}
   $$
   [S1, unofficial]

5. **March and Detect Isosurface:**
   Loop while $L > 0$:
   a. Sample intensity: $I = \text{texture}(volume, position).r$.
   b. If $I > threshold$:
      i. Refine position to approximate the exact isosurface intersection:
         $$
         \begin{aligned}
         position &-= step\_vector \cdot 0.5 \\
         I &= \text{texture}(volume, position).r \\
         position &-= step\_vector \cdot (I > threshold ? 0.25 : -0.25) \\
         I &= \text{texture}(volume, position).r
         \end{aligned}
         $$
      ii. Compute Blinn-Phong shading (see Step 6) and break loop.
   c. Else (threshold not reached):
      $$
      \begin{aligned}
      L &-= s \\
      position &+= step\_vector
      \end{aligned}
      $$
   [S1, unofficial]

6. **Blinn-Phong Shading:**
   Compute the final color contribution at the hit position.
   $$
   \begin{aligned}
   L_{vec} &= \text{normalize}(light\_position - position) \\
   V_{vec} &= -\text{normalize}(ray\_direction) \\
   N &= \text{normal}(position, I) \\
   H &= \text{normalize}(L_{vec} + V_{vec}) \\
   I_a &= 0.1 \\
   I_d &= 1.0 \cdot \max(0, N \cdot L_{vec}) \\
   I_s &= 8.0 \cdot \max(0, N \cdot H)^{600} \\
   colour &= (I_a + I_d) \cdot material\_colour + I_s \cdot [1.0, 1.0, 1.0]
   \end{aligned}
   $$
   [S1, unofficial]

7. **Gamma Correction:**
   Apply inverse gamma to the final color.
   $$
   colour_{final} = \text{pow}(colour, 1.0 / \gamma)
   $$
   [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_pixel(frag_coord, viewport_size, aspect_ratio, focal_length, view_matrix, 
                 ray_origin, box_bottom, box_top, volume, threshold, step_length, 
                 light_position, material_color, background_color, gamma):
    # 1. Compute Ray Direction
    u, v = frag_coord
    W, H = viewport_size
    ray_dir_xy = vec2(2.0 * u / W - 1.0, 2.0 * v / H - 1.0)
    ray_dir_xy.x *= aspect_ratio
    ray_dir_z = -focal_length
    ray_dir_homogeneous = vec4(ray_dir_xy.x, ray_dir_xy.y, ray_dir_z, 0.0)
    ray_dir = normalize((view_matrix * ray_dir_homogeneous).xyz)

    # 2. Slab Intersection
    dir_inv = vec3(1.0) / ray_dir
    t_top = dir_inv * (box_top - ray_origin)
    t_bottom = dir_inv * (box_bottom - ray_origin)
    t_min = min(t_top, t_bottom)
    t0 = max(0.0, max(t_min.x, t_min.y))
    t_max = max(t_top, t_bottom)
    t1 = min(t_max.x, t_max.y)

    if t0 > t1:
        return background_color # Miss

    # 3. Transform to Texture Coordinates
    ray_start = (ray_origin + ray_dir * t0 - box_bottom) / (box_top - box_bottom)
    ray_stop = (ray_origin + ray_dir * t1 - box_bottom) / (box_top - box_bottom)

    # 4. Initialize Marching
    ray_vec = ray_stop - ray_start
    ray_length = length(ray_vec)
    step_vector = step_length * ray_vec / ray_length
    
    position = ray_start + step_vector
    colour = pow(background_color, vec3(gamma))

    # 5. March and Detect Isosurface
    while ray_length > 0:
        intensity = texture(volume, position).r
        
        if intensity > threshold:
            # Refine position
            position -= step_vector * 0.5
            intensity = texture(volume, position).r
            if intensity > threshold:
                position -= step_vector * 0.25
            else:
                position -= step_vector * -0.25
            intensity = texture(volume, position).r
            
            # 6. Blinn-Phong Shading
            L_vec = normalize(light_position - position)
            V_vec = -normalize(ray_dir)
            N = compute_normal(position, intensity) # [G] Implementation of normal computation not provided in source
            H_vec = normalize(L_vec + V_vec)
            
            Ia = 0.1
            Id = 1.0 * max(0.0, dot(N, L_vec))
            Is = 8.0 * pow(max(0.0, dot(N, H_vec)), 600.0)
            
            colour = (Ia + Id) * material_color + Is * vec3(1.0)
            break
        else:
            ray_length -= step_length
            position += step_vector

    # 7. Gamma Correction
    final_colour = pow(colour, vec3(1.0 / gamma))
    return final_colour
```
[S1, unofficial]

**Parameters:**
- `step_length`: Not given in the sources.
- `threshold`: Not given in the sources.
- `focal_length`: Not given in the sources.
- `gamma`: Not given in the sources.
- `light_position`: Not given in the sources.
- `material_color`: Not given in the sources.
- `background_color`: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** The refinement step uses fixed offsets (0.5 and 0.25 of step vector). If the step size is large relative to the isosurface curvature, the hit position may be inaccurate [S1, unofficial].
- **Boundaries:** The slab intersection handles ray-box clipping, but texture sampling at `ray_start` and `ray_stop` assumes valid texture coordinates. If `t0` or `t1` are negative or out of bounds, behavior is undefined in the source [S1, unofficial].
- **Degenerate Input:** If `ray_direction` has zero components, `dir_inv` becomes infinite. The source does not specify handling for this [S1, unofficial].

## Operating characteristics
**Time Complexity:** $O(N)$ per pixel, where $N$ is the number of steps until the isosurface is found or the ray exits the volume. In the worst case (no hit), $N$ is proportional to the ray length divided by `step_length`.
**Memory Complexity:** $O(1)$ per pixel, excluding the volume texture storage.
**Measured Performance:** Not given in the sources.
**Strengths:** Simple to implement; provides specular shading for visual clarity; efficient single-hit extraction.
**Limitations:** Fixed step size may be inefficient in empty regions or miss thin structures; refinement is heuristic and not mathematically exact; no transparency/translucency effects.
**When to use:** Real-time visualization of dense isosurfaces where specular highlights are desired.
**When not:** When accurate surface normals are critical (use gradient-based normals from the volume directly) or when transparency is required.

## Background
The method originates from the `subvol` project by jimpelton [S1, unofficial]. It combines standard ray-box intersection techniques with a simple marching algorithm and Blinn-Phong shading to visualize volumetric data. The approach is typical of early GPU-based volume visualization techniques that prioritized speed and visual appeal over physical accuracy.

## Sources
[S1] jimpelton/subvol, https://github.com/jimpelton/subvol, fetched 2026-10-26 (unofficial)
