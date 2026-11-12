# Sphere Tracing with Composite SDF and Adaptive Sampling

Also called: Ray Marching with Character SDFs, SDF Sphere Tracing with Letter/Arc Geometry, SDF-Sphere-Tracing

## Overview
This method extends standard sphere tracing to render scenes composed of primitive shapes, character segments (letters), and arcs using Signed Distance Functions (SDFs) [S1, unofficial]. It solves the problem of efficiently rendering complex geometric text and decorative arcs alongside standard diffuse geometry by treating them as distinct hit types with specialized distance queries and normal estimation [S1, unofficial]. Unlike standard sphere tracing which typically handles uniform primitives, this approach integrates a composite SDF evaluation that prioritizes character and arc geometry when the ray is near a surrounding bounding box, and employs variance-based adaptive sampling to converge path-traced lighting [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera parameters (origin, direction, FOV, resolution), scene objects (diffuse shapes, character line segments, arcs), material properties, and sampling parameters.
*   **Outputs:** Final pixel color (RGB) after tone mapping.

**Data Structures:**
*   `shapes`: A list of pointers to `Shape` objects. Each `Shape` implements a virtual method `Distance(vec3 position)` returning the SDF value [S1, unofficial].
*   `characters`: A `vector<vec3>` storing line segments as pairs of endpoints. Index `j` is the start point, `j+1` is the end point [S1, unofficial].
*   `arces`: A `vector<Arc>` storing arc objects. Each `Arc` implements a `distance(vec3 position)` method [S1, unofficial].
*   `surronding_bbox`: A `Box` object used for acceleration. It implements `Distance(vec3 position)` [S1, unofficial].
*   `Material`: Defines `bsdf_sample(dir_in, normal)` and `isSpecular()` [S1, unofficial].

**Parameters:**
*   `thicc`: Thickness offset for characters. Value: `0.6f` [S1, unofficial].
*   `bbox_threshold`: Distance threshold to check characters/arcs. Value: `5.5f` [S1, unofficial].
*   `maximum_hit_distance`: Max ray march distance. Value: `1000.0f` [S1, unofficial].
*   `minimum_hit_distance`: Min distance for a valid hit. Value: `0.0001f` [S1, unofficial].
*   `ns`: Total samples per pixel. Value: `16` [S1, unofficial].
*   `step`: Batch size for adaptive sampling. Value: `16` [S1, unofficial].
*   `convergence_rate`: Variance threshold to stop sampling. Value: `0.0001f` [S1, unofficial].
*   `smallStep`: Step size for finite difference normal estimation. Value: `0.001f` [S1, unofficial].
*   `FOV`: Field of view. Value: `32` degrees [S1, unofficial].
*   `Max Bounces`: Path tracing depth. Value: `3` [S1, unofficial].
*   `Light Color`: Direct light contribution. Value: `vec3(500, 400, 100)` [S1, unofficial].
*   `Background Color`: Sky/Sun color. Value: `vec3(50, 80, 100)` [S1, unofficial].
*   `Bias`: Added before tone mapping. Value: `14.0f / 241.0f` [S1, unofficial].

**Algorithm Steps:**

1.  **Ray Generation:** For each pixel $(i, j)$, generate a ray $r$ from the camera. The camera uses a pinhole model with 32-degree FOV. Ray direction is calculated as $u \cdot p + v \cdot q - w$, where $p$ and $q$ are normalized pixel coordinates adjusted by aspect ratio and $\tan(\theta/2)$ [S1, unofficial].

2.  **Adaptive Sampling Loop:** For each pixel, initialize `sum = 0`, `sum_square = 0`, `sum_so_far = 0`. Loop `s` from `0` to `ns` with step `step`:
    a.  For `num` from `0` to `step-1`:
        i.  Generate a jittered ray $r$ using random offsets for pixel coordinates [S1, unofficial].
        ii. Call `path_tracing(r)` to get color $C$ [S1, unofficial].
        iii. Add $C$ to `sum`.
        iv. Calculate luminance $L = 0.2126 C_x + 0.7152 C_y + 0.0722 C_z$ [S1, unofficial].
        v. Update `sum_square += L^2` and `sum_so_far += L` [S1, unofficial].
    b.  Calculate variance:
        $$Variance = \frac{\sum L^2}{N} - \left(\frac{\sum L}{N}\right)^2 \frac{1}{N-1}$$
        where $N$ is the number of samples taken so far [S1, unofficial].
    c.  Check convergence: If $Variance < 0.0001f$ and $\min(C_x, C_y, C_z) > 0.2f$, stop sampling for this pixel [S1, unofficial].
    d.  If loop completes without convergence, average the sum: $Color = \frac{sum}{ns}$ [S1, unofficial].

3.  **Path Tracing (`path_tracing`):**
    a.  Initialize radiance $L = 0$, throughput $T = 1.0$, light direction $d_L = \text{normalize}(0.6, 0.6, 1.0)$ [S1, unofficial].
    b.  Loop `bounce` from `0` to `2`:
        i.  Call `Sphere_Tracing(ray)` to find hit point $P$, distance $t$, and hit type [S1, unofficial].
        ii. If no hit: Add $T \times \text{Background Color}$ to $L$ and return $L$ [S1, unofficial].
        iii. Update throughput: $T = T \times 0.2$ [S1, unofficial].
        iv. If hit type is `DIFFUSE`:
            - Compute normal $n$ using `calcNormal` (analytical for primitives) [S1, unofficial].
            - Sample BSDF to get new direction $d_{new}$ using local ONB (Orthonormal Basis) from $n$ [S1, unofficial].
            - Trace shadow ray towards light. If it hits the `SUN` (sky) and incident angle is positive, add $T \times \text{Light Color} \times \cos(\theta)$ to $L$ [S1, unofficial].
        v. If hit type is `LETTER` or `ARC`:
            - Compute normal $n$ using central finite differences with `smallStep` ($0.001f$) [S1, unofficial]:
              $$n_x = \text{SDF}(P + (dx, 0, 0)) - \text{SDF}(P - (dx, 0, 0))$$
              $$n_y = \text{SDF}(P + (0, dy, 0)) - \text{SDF}(P - (0, dy, 0))$$
              $$n_z = \text{SDF}(P + (0, 0, dz)) - \text{SDF}(P - (0, 0, dz))$$
              $$n = \text{normalize}(n_x, n_y, n_z)$$
            - Compute reflected direction: $d_{refl} = d_{in} - 2(d_{in} \cdot n)n$ [S1, unofficial].
            - Offset hit point: $P = P + n \times 0.2$ [S1, unofficial].
            - Create new ray: origin $P + n \times 0.1$, direction $\text{normalize}(d_{refl})$ mapped to local ONB [S1, unofficial].
        vi. If hit type is `SUN`: Add $T \times \text{Background Color}$ to $L$ and return $L$ [S1, unofficial].
    c.  Return $L$ [S1, unofficial].

4.  **Sphere Tracing (`Sphere_Tracing`):**
    a.  Initialize `mint = 1e20f`, `total_distance_travel = 0.0f` [S1, unofficial].
    b.  Loop `i` from `0` to `1023` (max steps):
        i.  Current position: $P = r_o + r_d \times total\_distance\_travel$ [S1, unofficial].
        ii. Check `shapes`: For each shape $j$, compute $d = \text{shapes}[j].\text{Distance}(P)$. If $d < mint$, update $mint = d$, $hit\_type = \text{DIFFUSE}$ [S1, unofficial].
        iii. Check `surronding_bbox`: Compute $d_{bbox} = \text{surronding\_bbox}.\text{Distance}(P)$ [S1, unofficial].
        iv. If $d_{bbox} < 5.5f$:
            - Check `characters`: Iterate $j$ by 2. Let $B = \text{characters}[j]$, $E = \text{characters}[j+1]$, $V = E - B$ [S1, unofficial].
            - Project ray origin onto segment:
              $$begin\_x = \min\left(\max\left(\frac{- (B - P) \cdot V}{|V|^2}, 0.0f\right), 1.0f\right)$$
              Note: The source code uses `-minf((begin - position).dot(begin_end) / (begin_end.length2()), 0.0f)` which is equivalent to $\max(0, \dots)$ if the dot product is negative, but clamped to 1.0. Specifically: $begin\_x = \min(1.0f, \max(0.0f, \frac{(P - B) \cdot V}{|V|^2}))$ [S1, unofficial].
            - Closest point: $P_{letter} = B + begin\_x \times V$ [S1, unofficial].
            - Distance: $d = |P - P_{letter}| - thicc$ [S1, unofficial].
            - If $d < mint$, update $mint = d$, $hit\_type = \text{LETTER}$ [S1, unofficial].
            - Check `arces`: For each arc $j$, compute $d = \text{arces}[j].\text{distance}(P)$. If $d < mint$, update $mint = d$, $hit\_type = \text{ARC}$ [S1, unofficial].
        v. Advance ray: $total\_distance\_travel += mint$ [S1, unofficial].
        vi. Check hit: If $mint < 0.0001f$:
            - Hit point $P_{hit} = r_o + r_d \times total\_distance\_travel$ [S1, unofficial].
            - Check for SUN: If $19.9 - P_{hit}.y < 0.0f$, set $hit\_type = \text{SUN}$ [S1, unofficial].
            - Return `true` with $t = total\_distance\_travel$ [S1, unofficial].
        vii. Check max distance: If $total\_distance\_travel > 1000.0f$, return `false` [S1, unofficial].
    c.  Return `false` [S1, unofficial].

5.  **Tone Mapping:**
    a.  Add bias: $Color_{raw} = Color + \frac{14.0f}{241.0f}$ [S1, unofficial].
    b.  Reinhard tone mapping: $Color_{final} = \frac{Color_{raw}}{Color_{raw} + 1.0f} \times 255.0f$ [S1, unofficial].
    c.  Convert to 8-bit integer [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def sphere_tracing(ray_o, ray_d, shapes, characters, arces, surrounding_bbox):
    mint = 1e20
    total_distance_travel = 0.0
    maximum_step = 1024
    minimum_hit_distance = 0.0001
    maximum_hit_distance = 1000.0
    bbox_threshold = 5.5
    thicc = 0.6

    for _ in range(maximum_step):
        position = ray_o + ray_d * total_distance_travel
        
        # Check shapes
        hit_type = "DIFFUSE"
        for shape in shapes:
            d = shape.distance(position)
            if d < mint:
                mint = d
                hit_type = "DIFFUSE"
        
        # Check characters and arcs if near bounding box
        distance_to_bbox = surrounding_bbox.distance(position)
        if distance_to_bbox < bbox_threshold:
            # Check characters
            for j in range(0, len(characters), 2):
                begin = characters[j]
                end = characters[j+1]
                begin_end = end - begin
                
                # Projection parameter
                dot_val = (begin - position).dot(begin_end)
                len_sq = begin_end.length_squared()
                if len_sq == 0:
                    continue
                begin_x = -dot_val / len_sq
                begin_x = max(0.0, min(1.0, begin_x))
                
                hit_point_letter = begin + begin_x * begin_end
                d = (position - hit_point_letter).length() - thicc
                
                if d < mint:
                    mint = d
                    hit_type = "LETTER"
            
            # Check arcs
            for arc in arces:
                distance_s = arc.distance(position)
                if distance_s < mint:
                    mint = distance_s
                    hit_type = "ARC"
        
        total_distance_travel += mint
        
        if mint < minimum_hit_distance:
            hit_point = ray_o + ray_d * total_distance_travel
            # Check for SUN
            if (19.9 - hit_point.y) < 0.0:
                return True, total_distance_travel, "SUN"
            return True, total_distance_travel, hit_type
        
        if total_distance_travel > maximum_hit_distance:
            return False, 0, None
            
    return False, 0, None

def path_tracing(ray, shapes, characters, arces, mats, surrounding_bbox):
    L = vec3(0.0)
    T = vec3(1.0)
    light_direction = normalize(vec3(0.6, 0.6, 1.0))
    
    for bounce in range(3):
        hit, t, hit_type = sphere_tracing(ray.o, ray.d, shapes, characters, arces, surrounding_bbox)
        
        if not hit:
            L += T * vec3(50, 80, 100)
            return L
        
        hit_point = ray.o + ray.d * t
        T *= 0.2
        
        if hit_type == "DIFFUSE":
            normal = calc_normal(hit_point, shapes[0]) # Assuming shape index from sphere_tracing if returned
            # Sample BSDF
            coord = mats[0].bsdf_sample(ray.d, normal)
            local_onb = create_onb(normal)
            new_dir = coord.x * local_onb.u + coord.y * local_onb.v + coord.z * local_onb.w
            
            # Shadow ray
            shadow_hit, shadow_t, shadow_type = sphere_tracing(hit_point + normal * 0.001, light_direction, shapes, characters, arces, surrounding_bbox)
            if shadow_type == "SUN":
                incident = max(0.0, normal.dot(light_direction))
                L += T * vec3(500, 400, 100) * incident
                
            ray.o = hit_point + normal * 0.001
            ray.d = normalize(new_dir)
            
        elif hit_type in ["LETTER", "ARC"]:
            small_step = 0.001
            # Compute normal via finite differences
            # Note: In real implementation, specific SDF function for LETTER/ARC is needed
            # Here we assume a generic SDF function or specific optimized distance function
            dx = compute_distance_letter_optimize(hit_point + vec3(small_step, 0, 0), characters, 0) - \
                 compute_distance_letter_optimize(hit_point - vec3(small_step, 0, 0), characters, 0)
            dy = compute_distance_letter_optimize(hit_point + vec3(0, small_step, 0), characters, 0) - \
                 compute_distance_letter_optimize(hit_point - vec3(0, small_step, 0), characters, 0)
            dz = compute_distance_letter_optimize(hit_point + vec3(0, 0, small_step), characters, 0) - \
                 compute_distance_letter_optimize(hit_point - vec3(0, 0, small_step), characters, 0)
            
            normal = normalize(vec3(dx, dy, dz))
            direction = ray.d - 2.0 * ray.d.dot(normal) * normal
            
            hit_point += normal * 0.2
            local_onb = create_onb(normal)
            ray.o = hit_point + normal * 0.1
            ray.d = normalize(direction.x * local_onb.u + direction.y * local_onb.v + direction.z * local_onb.w)
            
        elif hit_type == "SUN":
            L += T * vec3(50, 80, 100)
            return L
            
    return L

def main():
    width = 960
    height = 540
    ns = 16
    step = 16
    convergence_rate = 0.0001
    
    # Initialize scene objects...
    # shapes, characters, arces, surrounding_bbox, mats
    
    for j in range(height):
        for i in range(width):
            sum_color = vec3(0.0)
            sum_lum_sq = 0.0
            sum_lum = 0.0
            num_samples = 0
            converged = False
            
            for s in range(0, ns, step):
                for num in range(step):
                    p = ((i + random.random()) / width)
                    q = ((j + random.random()) / height)
                    ray = camera.generate_ray(p, q)
                    color = path_tracing(ray, shapes, characters, arces, mats, surrounding_bbox)
                    sum_color += color
                    
                    lum = 0.2126 * color.x + 0.7152 * color.y + 0.0722 * color.z
                    sum_lum_sq += lum * lum
                    sum_lum += lum
                    num_samples += 1
                
                if num_samples > 1:
                    variance = (sum_lum_sq / num_samples) - (sum_lum / num_samples)**2 * (1.0 / (num_samples - 1))
                    min_color = min(sum_color.x, sum_color.y, sum_color.z)
                    
                    if variance < convergence_rate and min_color > 0.2:
                        converged = True
                        break
            
            if not converged:
                final_color = sum_color / ns
            
            # Tone mapping
            final_color += 14.0 / 241.0
            final_color = final_color / (final_color + 1.0) * 255.0
            
            # Write to PPM...
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M \cdot B)$, where $N$ is the number of pixels, $M$ is the number of samples per pixel (adaptive), $B$ is the max bounces (3), and the cost per bounce depends on the number of shapes, characters, and arcs checked. The character check is $O(K)$ where $K$ is the number of character pairs, only performed if near the bounding box [S1, unofficial].
*   **Memory Complexity:** $O(S + C + A)$, storing shapes, characters, and arcs [S1, unofficial].
*   **Performance:** The method uses adaptive sampling to reduce computation for converged pixels. The use of a surrounding bounding box accelerates character/arc checks by limiting them to rays near the geometry [S1, unofficial].
*   **Strengths:** Efficiently renders complex text and arcs as SDFs. Adaptive sampling reduces noise.
*   **Limitations:** Normal estimation for characters/arcs uses finite differences, which can be noisy or inaccurate if `smallStep` is not chosen carefully. The hit detection for characters is based on line segments, not true 3D extruded letters, which may cause visual artifacts if viewed from extreme angles or if the thickness offset is inappropriate [S1, unofficial].
*   **When to use:** Scenes with prominent text or arc-based geometry where SDF rendering is desired.
*   **When not to use:** Scenes with high-frequency detail not representable by SDFs or where analytical normals are critical for high-quality reflections.

## Background
This method originates from an unofficial GitHub repository by AlerianEmperor, titled "SDF-Sphere-Tracing" [S1, unofficial]. It is a specific implementation of sphere tracing tailored for rendering character segments and arcs alongside standard primitives, using path tracing for lighting. The code includes optimizations for fast trigonometric functions and parallel rendering using OpenMP [S1, unofficial].

## Sources

[S1] AlerianEmperor/SDF-Sphere-Tracing, https://github.com/AlerianEmperor/SDF-Sphere-Tracing, fetched 2026-10-26 (unofficial)
