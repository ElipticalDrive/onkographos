# Sphere Tracing with Composite SDF and Adaptive Sampling

Also called: Ray Marching with Character SDFs, SDF Sphere Tracing with Letter/Arc Geometry, SDF-Sphere-Tracing

## Overview
This method extends standard sphere tracing by rendering scenes composed of primitive shapes, character segments (letters), and arcs using a unified Signed Distance Function (SDF) framework [S1, unofficial]. It differs from sibling methods by explicitly handling non-solid geometric primitives (lines and arcs) as distance fields and employing variance-based adaptive sampling to converge pixel luminance [S1, unofficial]. It builds on [Sphere Tracing with SDFs](6223-sphere-tracing-with-sdfs.md) for the core ray-marching loop and integrates path tracing for global illumination [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- Camera parameters: origin $\mathbf{o}$, direction $\mathbf{d}$, width, height, FOV (32 degrees) [S1, unofficial].
- Scene data:
  - `shapes`: List of primitive SDF objects (e.g., boxes) with a `Distance(p)` method [S1, unofficial].
  - `characters`: List of `vec3` pairs representing line segments. Index $j$ is the start point, $j+1$ is the end point [S1, unofficial].
  - `arcs`: List of arc objects with a `distance(p)` method [S1, unofficial].
  - `surronding_bbox`: A bounding box SDF used for acceleration culling of characters/arcs [S1, unofficial].
  - `materials`: List of material objects with `bsdf_sample(dir, normal)` and `isSpecular()` methods [S1, unofficial].
- Parameters: `thicc` (0.6f), `bbox_threshold` (5.5f), `maximum_hit_distance` (1000.0f), `minimum_hit_distance` (0.0001f), `maximum_step` (1024), `ns` (16 samples/pixel), `step` (16 samples/batch), `convergence_rate` (0.0001) [S1, unofficial].

**Outputs:**
- Final pixel color (8-bit RGB) after tone mapping [S1, unofficial].

### Data Structures
- `vec3`: 3D vector with components $x, y, z$.
- `Ray`: Struct with `o` (origin) and `d` (direction) [S1, unofficial].
- `Shape`: Abstract base class with virtual `Distance(vec3)` [S1, unofficial].
- `Arc`: Object with `distance(vec3)` method [S1, unofficial].
- `Material`: Abstract base class with virtual `bsdf_sample` and `isSpecular` [S1, unofficial].
- `characters`: `std::vector<vec3>` storing line segment endpoints sequentially [S1, unofficial].

### Numbered Steps

#### 1. Ray Generation
For each pixel $(i, j)$, generate a ray using a pinhole camera model with 32-degree FOV [S1, unofficial].
$$ \mathbf{p} = \frac{2(i + \text{rand}())}{\text{width}} - 1 $$
$$ \mathbf{q} = \frac{2(j + \text{rand}())}{\text{height}} - 1 $$
Adjust $\mathbf{p}, \mathbf{q}$ by aspect ratio and $\tan(\theta/2)$ to get direction $\mathbf{d}$ [S1, unofficial].

#### 2. Adaptive Sampling Loop
For each pixel, initialize `sum = 0`, `sum_square = 0`, `sum_so_far = 0`, `num_sample_used = 0` [S1, unofficial].
Loop while `num_sample_used < ns`:
1. Sample `step` rays [S1, unofficial].
2. For each sample, call `path_tracing(ray)` [S1, unofficial].
3. Accumulate color to `sum` [S1, unofficial].
4. Compute luminance $L = 0.2126 C_x + 0.7152 C_y + 0.0722 C_z$ [S1, unofficial].
5. Update statistics:
   $$ \text{sum\_square} \leftarrow \text{sum\_square} + L^2 $$
   $$ \text{sum\_so\_far} \leftarrow \text{sum\_so\_far} + L $$
   $$ \text{num\_sample\_used} \leftarrow \text{num\_sample\_used} + \text{step} $$
6. Check convergence:
   $$ \text{mean} = \frac{\text{sum\_so\_far}}{\text{num\_sample\_used}} $$
   $$ \text{variance} = \frac{\text{sum\_square}}{\text{num\_sample\_used}} - \text{mean}^2 $$
   If $\text{variance} < \text{convergence\_rate}$ and $\min(C_x, C_y, C_z) > 0.2$, stop sampling for this pixel [S1, unofficial].
7. Final color: $C = \frac{\text{sum}}{ns}$ [S1, unofficial].

#### 3. Path Tracing (Bounces)
Initialize radiance $L = (0,0,0)$ and throughput $T = (1,1,1)$ [S1, unofficial].
Loop `bounce` from 0 to 2 (max 3 bounces) [S1, unofficial]:
1. Call `sphere_tracing(current_ray)` to find hit point $\mathbf{p}$, distance $t$, and `hit_type` [S1, unofficial].
2. If no hit (miss):
   - Add $T \times (50, 80, 100)$ to $L$ [S1, unofficial].
   - Return $L$ [S1, unofficial].
3. Update throughput: $T \leftarrow T \times 0.2$ [S1, unofficial].
4. Determine `hit_type` (DIFFUSE, LETTER, ARC, SUN) [S1, unofficial].
5. If `hit_type == DIFFUSE`:
   - Compute normal $\mathbf{n}$ using `calcNormal` (analytical or finite differences) [S1, unofficial].
   - Sample BSDF: $\mathbf{coord} = \text{material} \to \text{bsdf\_sample}(-\mathbf{d}, \mathbf{n})$ [S1, unofficial].
   - Construct new direction $\mathbf{d}_{new}$ using local ONB from $\mathbf{n}$ and $\mathbf{coord}$ [S1, unofficial].
   - Trace shadow ray towards light direction $(0.6, 0.6, 1.0)$ [S1, unofficial].
   - If shadow ray hits SUN, add $T \times (500, 400, 100) \times \max(0, \mathbf{n} \cdot \mathbf{l})$ to $L$ [S1, unofficial].
6. If `hit_type == LETTER` or `ARC`:
   - Compute normal $\mathbf{n}$ via central finite differences with step $\epsilon = 0.001$ [S1, unofficial]:
     $$ \mathbf{n} = \frac{\nabla f(\mathbf{p})}{\|\nabla f(\mathbf{p})\|} $$
     where $\nabla f(\mathbf{p}) \approx [f(\mathbf{p}+\epsilon\hat{x})-f(\mathbf{p}-\epsilon\hat{x}), \dots]^T$ [S1, unofficial].
   - Compute reflected direction: $\mathbf{r} = \mathbf{d} - 2(\mathbf{d} \cdot \mathbf{n})\mathbf{n}$ [S1, unofficial].
   - Offset hit point: $\mathbf{p}_{new} = \mathbf{p} + \mathbf{n} \times 0.2$ [S1, unofficial].
   - Create new ray: $\mathbf{o}_{new} = \mathbf{p}_{new} + \mathbf{n} \times 0.1$, $\mathbf{d}_{new} = \text{normalize}(\mathbf{r})$ [S1, unofficial].
   - (Note: Code uses reflected direction directly for next bounce, effectively specular reflection) [S1, unofficial].
7. If `hit_type == SUN`:
   - Add $T \times (50, 80, 100)$ to $L$ [S1, unofficial].
   - Return $L$ [S1, unofficial].

#### 4. Sphere Tracing (Ray Marching)
Initialize `total_distance_travel = 0`, `mint = \infty` [S1, unofficial].
Loop `i` from 0 to `maximum_step` (1024):
1. Current position: $\mathbf{p} = \mathbf{o} + \mathbf{d} \times \text{total\_distance\_travel}$ [S1, unofficial].
2. Check primitives:
   - For each `shape` in `shapes`:
     $$ d = \text{shape} \to \text{Distance}(\mathbf{p}) $$
     If $d < \text{mint}$, set $\text{mint} = d$, $\text{hit\_type} = \text{DIFFUSE}$ [S1, unofficial].
3. Check characters/arcs (acceleration):
   - Compute distance to surrounding bounding box: $d_{bbox} = \text{surronding\_bbox} \to \text{Distance}(\mathbf{p})$ [S1, unofficial].
   - If $d_{bbox} < 5.5$:
     - For each character pair $(j, j+1)$:
       - $\mathbf{begin} = \text{characters}[j]$, $\mathbf{end} = \text{characters}[j+1]$ [S1, unofficial].
       - $\mathbf{seg} = \mathbf{end} - \mathbf{begin}$ [S1, unofficial].
       - Project $\mathbf{p}$ onto segment:
         $$ t_{proj} = \frac{(\mathbf{p} - \mathbf{begin}) \cdot \mathbf{seg}}{\|\mathbf{seg}\|^2} $$
         $$ t_{clamped} = \max(0, \min(1, t_{proj})) $$
         $$ \mathbf{closest} = \mathbf{begin} + t_{clamped} \times \mathbf{seg} $$
       - Distance to segment: $d_{seg} = \|\mathbf{p} - \mathbf{closest}\| - \text{thicc}$ (thicc=0.6) [S1, unofficial].
       - If $d_{seg} < \text{mint}$, set $\text{mint} = d_{seg}$, $\text{hit\_type} = \text{LETTER}$ [S1, unofficial].
     - For each `arc` in `arcs`:
       - $d_{arc} = \text{arc} \to \text{distance}(\mathbf{p})$ [S1, unofficial].
       - If $d_{arc} < \text{mint}$, set $\text{mint} = d_{arc}$, $\text{hit\_type} = \text{ARC}$ [S1, unofficial].
4. Advance ray: $\text{total\_distance\_travel} \leftarrow \text{total\_distance\_travel} + \text{mint}$ [S1, unofficial].
5. Termination checks:
   - If $\text{mint} < 0.0001$:
     - Hit point $\mathbf{p}_{hit} = \mathbf{o} + \mathbf{d} \times \text{total\_distance\_travel}$ [S1, unofficial].
     - Check for SUN: If $19.9 - \mathbf{p}_{hit}.y < 0$, set $\text{hit\_type} = \text{SUN}$ [S1, unofficial].
     - Return true with $t = \text{total\_distance\_travel}$ and `hit_type` [S1, unofficial].
   - If $\text{total\_distance\_travel} > 1000.0$:
     - Return false (miss) [S1, unofficial].
6. Return false [S1, unofficial].

#### 5. Tone Mapping
Add bias to linear color: $C_{biased} = C + \frac{14}{241}$ [S1, unofficial].
Reinhard tone mapping:
$$ C_{final} = \frac{C_{biased}}{C_{biased} + 1} \times 255 $$
Convert to 8-bit integer [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def sphere_tracing(ray_o, ray_d, shapes, characters, arcs, surrounding_bbox):
    """
    Returns (hit, t, hit_type, hit_point)
    """
    total_distance_travel = 0.0
    maximum_step = 1024
    minimum_hit_distance = 0.0001
    maximum_hit_distance = 1000.0
    thicc = 0.6
    bbox_threshold = 5.5

    for _ in range(maximum_step):
        position = ray_o + ray_d * total_distance_travel
        mint = float('inf')
        hit_type = None

        # Check primitives
        for shape in shapes:
            d = shape.distance(position)
            if d < mint:
                mint = d
                hit_type = 'DIFFUSE'

        # Check characters and arcs if close to bbox
        dist_to_bbox = surrounding_bbox.distance(position)
        if dist_to_bbox < bbox_threshold:
            # Check characters (pairs)
            for j in range(0, len(characters), 2):
                begin = characters[j]
                end = characters[j+1]
                seg = end - begin
                seg_len_sq = seg.dot(seg)
                if seg_len_sq == 0: continue
                
                # Projection parameter
                # Code: begin_x = -minf((begin - position).dot(begin_end) / begin_end.length2(), 0.0f)
                # Note: (begin - position) = -(position - begin)
                # So -((begin - position) . seg) = (position - begin) . seg
                # Code logic: begin_x = clamp( (position - begin).dot(seg) / seg_len_sq, 0, 1 )
                t_proj = (position - begin).dot(seg) / seg_len_sq
                t_clamped = max(0.0, min(1.0, t_proj))
                
                closest = begin + seg * t_clamped
                dist_to_seg = (position - closest).length() - thicc
                
                if dist_to_seg < mint:
                    mint = dist_to_seg
                    hit_type = 'LETTER'

            # Check arcs
            for arc in arcs:
                d_arc = arc.distance(position)
                if d_arc < mint:
                    mint = d_arc
                    hit_type = 'ARC'

        total_distance_travel += mint

        if mint < minimum_hit_distance:
            hit_point = ray_o + ray_d * total_distance_travel
            # Check SUN
            if 19.9 - hit_point.y < 0.0:
                hit_type = 'SUN'
            return True, total_distance_travel, hit_type, hit_point

        if total_distance_travel > maximum_hit_distance:
            return False, 0, None, None

    return False, 0, None, None

def path_tracing(ray_o, ray_d, shapes, characters, arcs, materials, surrounding_bbox):
    L = vec3(0.0, 0.0, 0.0)
    T = vec3(1.0, 1.0, 1.0)
    light_dir = normalize(vec3(0.6, 0.6, 1.0))
    bg_color = vec3(50.0, 80.0, 100.0)

    for bounce in range(3):
        hit, t, hit_type, hit_point = sphere_tracing(ray_o, ray_d, shapes, characters, arcs, surrounding_bbox)
        
        if not hit:
            L += T * bg_color
            return L

        T *= 0.2

        if hit_type == 'DIFFUSE':
            # Assume shape index is tracked or retrieved from sphere_tracing
            # For simplicity, we assume calcNormal works on the hit point and shape
            # In the source, shape_id is passed back. Here we approximate.
            # Note: The source code uses `shapes[shape_id]` for normal calculation.
            # We will assume a helper `get_shape_at_hit` exists or pass it.
            # To keep self-contained, we assume the first shape that hit is the one.
            # This is a simplification of the source's index tracking.
            shape_idx = 0 # Placeholder, source tracks this
            normal = calc_normal(hit_point, shapes[shape_idx])
            mat = materials[shape_idx]
            coord = mat.bsdf_sample(-ray_d, normal)
            onb = create_onb(normal)
            new_dir = coord.x * onb.u + coord.y * onb.v + coord.z * onb.w
            
            # Shadow ray
            shadow_hit, shadow_t, shadow_type, _ = sphere_tracing(hit_point + normal * 0.001, light_dir, shapes, characters, arcs, surrounding_bbox)
            if shadow_type == 'SUN':
                incident = max(0.0, normal.dot(light_dir))
                L += T * vec3(500.0, 400.0, 100.0) * incident

        elif hit_type in ['LETTER', 'ARC']:
            small_step = 0.001
            # Compute normal via finite differences
            # For LETTER, we need the specific character ID. Source tracks this.
            # We approximate by checking all chars/arcs again or assuming a helper.
            # To be precise, we re-implement the distance check to find the ID.
            # This is computationally expensive but matches the source's logic for normal calc.
            # Simplified: assume we have a function `get_normal_at_hit(hit_point, hit_type, ...)`
            normal = get_normal_at_hit(hit_point, hit_type, characters, arcs)
            
            reflected = ray_d - 2.0 * ray_d.dot(normal) * normal
            hit_point += normal * 0.2
            new_ray_o = hit_point + normal * 0.1
            new_ray_d = normalize(reflected)
            
            ray_o = new_ray_o
            ray_d = new_ray_d
            continue # Continue to next bounce with new ray

        elif hit_type == 'SUN':
            L += T * bg_color
            return L

    return L

def get_normal_at_hit(p, hit_type, characters, arcs):
    small_step = 0.001
    if hit_type == 'LETTER':
        # Find the character ID that hit
        # Simplified: just use a dummy ID or re-search. 
        # Source code uses `letter_id` from sphere_tracing.
        # We will assume we pass the ID or re-calculate.
        # For this pseudocode, we assume a helper `compute_distance_letter_optimize`
        # which takes an ID. We need to find the ID first.
        # This is a gap filler [G].
        letter_id = find_hit_character_id(p, characters)
        dx = compute_distance_letter_optimize(p + vec3(small_step, 0, 0), characters, letter_id) - \
             compute_distance_letter_optimize(p - vec3(small_step, 0, 0), characters, letter_id)
        dy = compute_distance_letter_optimize(p + vec3(0, small_step, 0), characters, letter_id) - \
             compute_distance_letter_optimize(p - vec3(0, small_step, 0), characters, letter_id)
        dz = compute_distance_letter_optimize(p + vec3(0, 0, small_step), characters, letter_id) - \
             compute_distance_letter_optimize(p - vec3(0, 0, small_step), characters, letter_id)
    elif hit_type == 'ARC':
        arc_id = find_hit_arc_id(p, arcs)
        dx = arcs[arc_id].distance(p + vec3(small_step, 0, 0)) - arcs[arc_id].distance(p - vec3(small_step, 0, 0))
        dy = arcs[arc_id].distance(p + vec3(0, small_step, 0)) - arcs[arc_id].distance(p - vec3(0, small_step, 0))
        dz = arcs[arc_id].distance(p + vec3(0, 0, small_step)) - arcs[arc_id].distance(p - vec3(0, 0, small_step))
    
    return normalize(vec3(dx, dy, dz))

def find_hit_character_id(p, characters):
    # Re-implement the distance check to find the ID
    mint = float('inf')
    best_id = 0
    thicc = 0.6
    for j in range(0, len(characters), 2):
        begin = characters[j]
        end = characters[j+1]
        seg = end - begin
        seg_len_sq = seg.dot(seg)
        if seg_len_sq == 0: continue
        t_proj = (p - begin).dot(seg) / seg_len_sq
        t_clamped = max(0.0, min(1.0, t_proj))
        closest = begin + seg * t_clamped
        d = (p - closest).length() - thicc
        if d < mint:
            mint = d
            best_id = j
    return best_id

def find_hit_arc_id(p, arcs):
    mint = float('inf')
    best_id = 0
    for j, arc in enumerate(arcs):
        d = arc.distance(p)
        if d < mint:
            mint = d
            best_id = j
    return best_id

def compute_distance_letter_optimize(p, characters, letter_id):
    begin = characters[letter_id]
    end = characters[letter_id+1]
    seg = end - begin
    seg_len_sq = seg.dot(seg)
    if seg_len_sq == 0: return float('inf')
    t_proj = (p - begin).dot(seg) / seg_len_sq
    t_clamped = max(0.0, min(1.0, t_proj))
    closest = begin + seg * t_clamped
    return (p - closest).length() - 0.6

def main():
    width = 960
    height = 540
    ns = 16
    step = 16
    convergence_rate = 0.0001
    
    # Initialize scene...
    # shapes, characters, arcs, materials, surrounding_bbox
    
    for j in range(height):
        for i in range(width):
            sum_color = vec3(0.0, 0.0, 0.0)
            sum_sq = 0.0
            sum_so_far = 0.0
            num_samples = 0
            converged = False
            
            for s in range(0, ns, step):
                for _ in range(step):
                    p = (i + random.random()) / width
                    q = (j + random.random()) / height
                    ray = camera.generate_ray(p, q)
                    color = path_tracing(ray.o, ray.d, shapes, characters, arcs, materials, surrounding_bbox)
                    sum_color += color
                    
                    luminance = 0.2126 * color.x + 0.7152 * color.y + 0.0722 * color.z
                    sum_sq += luminance * luminance
                    sum_so_far += luminance
                num_samples += step
                
                mean = sum_so_far / num_samples
                var = sum_sq / num_samples - mean * mean
                
                if var < convergence_rate and min(sum_color.x, sum_color.y, sum_color.z) > 0.2:
                    converged = True
                    break
            
            if not converged:
                final_color = sum_color / ns
            else:
                final_color = sum_color / ns # Use last sum? Source uses sum * ins if not converged.
                # Source: if converge, break. If not, c[i] = sum * ins.
                # If converged, it sets c[j*width+i] = value (which is sum * ins? No, value is computed from mean?)
                # Source: `c[j * width + i] = value;` where `value` is not explicitly defined in the snippet provided,
                # but typically it's the average. Let's assume average.
                final_color = sum_color / ns

            # Tone mapping
            final_color += vec3(14.0/241.0, 14.0/241.0, 14.0/241.0)
            final_color = final_color / (final_color + 1.0)
            final_color *= 255.0
            
            # Write to PPM...

```

### Parameters
- `thicc`: 0.6f (thickness offset for letters) [S1, unofficial].
- `bbox_threshold`: 5.5f (distance to bbox to check letters/arcs) [S1, unofficial].
- `maximum_hit_distance`: 1000.0f [S1, unofficial].
- `minimum_hit_distance`: 0.0001f [S1, unofficial].
- `maximum_step`: 1024 [S1, unofficial].
- `ns`: 16 (total samples per pixel) [S1, unofficial].
- `step`: 16 (samples per batch) [S1, unofficial].
- `convergence_rate`: 0.0001f [S1, unofficial].
- `FOV`: 32 degrees [S1, unofficial].
- `Light Color`: (500, 400, 100) [S1, unofficial].
- `Background Color`: (50, 80, 100) [S1, unofficial].
- `Small Step`: 0.001f (for finite differences) [S1, unofficial].
- `Bias`: 14.0/241.0 [S1, unofficial].

### Edge Cases and Pitfalls
- **Finite Difference Noise**: The step size for normal estimation (0.001f) must be small enough for precision but large enough to avoid floating-point noise [S1, unofficial].
- **Self-Intersection**: Ray origins are offset by `normal * 0.1` after reflection to prevent self-intersection [S1, unofficial].
- **SUN Detection**: The SUN is detected by a simple Y-coordinate check (`19.9 - hit_point.y < 0`) rather than an SDF, which is a scene-specific heuristic [S1, unofficial].
- **Character ID Tracking**: The optimized normal calculation requires the specific character ID that was hit, which must be tracked during sphere tracing [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N \cdot M \cdot B)$, where $N$ is the number of samples, $M$ is the number of primitives/characters/arcs, and $B$ is the number of bounces (3) [S1, unofficial]. Adaptive sampling reduces $N$ for converged pixels [S1, unofficial].
- **Memory Complexity**: $O(M)$ for scene data [S1, unofficial].
- **Performance**: The code uses OpenMP for parallelization across pixels [S1, unofficial]. The optimized character distance calculation (`compute_distance_letter_optimize`) is used for path tracing to improve performance over the full search (`compute_distance_letter2`) [S1, unofficial].
- **Strengths**: Handles complex text/arc geometry efficiently using SDFs. Adaptive sampling reduces noise.
- **Limitations**: SUN detection is heuristic. Normal estimation for letters/arcs is expensive due to finite differences and ID lookup [S1, unofficial].

## Background
This method is derived from the "SDF-Sphere-Tracing" project by AlerianEmperor, which demonstrates ray marching with character segments and arcs [S1, unofficial]. It is an unofficial implementation [S1, unofficial].

## Sources
[S1] AlerianEmperor/SDF-Sphere-Tracing, https://github.com/AlerianEmperor/SDF-Sphere-Tracing, fetched 2026-10-26 (unofficial)
