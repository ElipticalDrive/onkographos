# Average Intensity Projection with IsoSurface Detection

Also called: Exact Projection Mean Density

## Overview
Average Intensity Projection with IsoSurface Detection is a hybrid volume rendering technique that computes the mean density of voxels along a ray within a valid range, rendering this average value with lighting based on the mean position [S2, unofficial]. It differs from standard first-hit ray casting by accumulating statistics over the entire ray segment rather than stopping at the first surface, and from pure volume rendering by restricting accumulation to voxels that satisfy an isosurface crossing condition or density threshold [S1, S2, unofficial]. This method is chosen when the goal is to visualize the average material properties of a structure while retaining surface-like shading cues derived from the centroid of the sampled volume [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, near/far clipping planes $t_{near}, t_{far}$, isosurface threshold $\tau$, number of steps $N=256$ [S1, S2, unofficial].
- Input: 3D scalar texture $V$ (density), 1D color map $C$ [S1, unofficial].
- Output: Fragment color $\mathbf{c}_{out}$ [S1, S2, unofficial].

**Data Structures:**
- `meanSum`: Float accumulator for density values.
- `validSamples`: Integer counter for samples contributing to the average.
- `meanPos`: Vector accumulator for positions.
- `color`: Vector accumulator for color values.
- `cnt`: Integer counter used to exclude non-crossing steps from the average denominator [S1, unofficial].

**Steps:**
1. Initialize `meanSum = 0.0`, `validSamples = 0`, `meanPos = (0,0,0)`, `color = (0,0,0,0)`, and `cnt = 1` [S1, unofficial].
2. Iterate $i$ from $0$ to $N-1$:
   a. Calculate current position $\mathbf{p}_i = \mathbf{o} + t_i \mathbf{d}$, where $t_i$ is the step parameter [S1, unofficial].
   b. Transform $\mathbf{p}_i$ to texture coordinates $\mathbf{u}_i$. The source uses a specific transformation:
      $$ \mathbf{tmp} = (p_x, p_y, p_z \times 2.3486238532110093) $$
      $$ \mathbf{u}_i = \frac{1 + \mathbf{tmp}}{2} $$
      $$ u_{i,y} = 1 - u_{i,y} $$
      [S1, unofficial]. This implies a non-uniform aspect ratio assumption [S1, unofficial].
   c. Sample scalar value $v_i = V(\mathbf{u}_i)$ [S1, unofficial].
   d. Check for isosurface crossing between current and next step. Let $v_{next}$ be the sample at $\mathbf{p}_{i+1}$. If $(v_i < \tau \land v_{next} > \tau)$, a crossing is detected [S1, unofficial].
   e. If crossing detected:
      i. Refine intersection position $\mathbf{p}_{surf}$ using binary search between $\mathbf{p}_i$ and $\mathbf{p}_{i+1}$ [S1, unofficial].
      ii. Sample color at surface: $\mathbf{c}_{surf} = C(V(\mathbf{p}_{surf}))$ [S1, unofficial].
      iii. Add to color accumulator: $\mathbf{color} += \mathbf{c}_{surf}$ [S1, unofficial].
      iv. Increment `validSamples` [S1, unofficial].
      v. Add position to mean accumulator: `meanPos` += $\mathbf{p}_{surf}$ [S1, unofficial].
      vi. Add density to sum: `meanSum` += $V(\mathbf{p}_{surf})$ [S1, unofficial].
   f. If no crossing detected:
      i. Decrement `cnt` [S1, unofficial].
3. After loop, if `validSamples > 0`:
   a. Calculate mean density: $v_{mean} = \text{meanSum} / \text{validSamples}$ [S1, unofficial].
   b. Calculate mean position: $\mathbf{p}_{mean} = \text{meanPos} / \text{validSamples}$ [S1, unofficial].
   c. Map mean density to base color: $\mathbf{c}_{base} = C(v_{mean})$ [S1, unofficial].
   d. If lighting is enabled:
      i. Compute normal $\mathbf{n} = \nabla V(\mathbf{p}_{mean})$ [S1, unofficial].
      ii. Compute lit color: $\mathbf{c}_{lit} = \text{Lighting}(\mathbf{n}, -\mathbf{d}, \mathbf{p}_{mean}, \mathbf{c}_{base})$ [S1, unofficial].
      iii. Output $\mathbf{c}_{out} = \mathbf{c}_{lit}$ [S1, unofficial].
   e. If lighting is disabled:
      i. Output $\mathbf{c}_{out} = \mathbf{c}_{base}$ [S1, unofficial].
4. If `validSamples == 0`, output black $\mathbf{c}_{out} = (0,0,0,0)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def average_intensity_projection_with_isosurface_detection(ray_origin, ray_dir, t_near, t_far, volume_texture, color_map, iso_threshold, enable_lighting):
    num_steps = 256
    dt = (t_far - t_near) / num_steps
    
    mean_sum = 0.0
    valid_samples = 0
    mean_pos = vec3(0.0, 0.0, 0.0)
    color_acc = vec4(0.0, 0.0, 0.0, 0.0)
    cnt = 1
    
    for i in range(num_steps):
        t = t_near + i * dt
        pos = ray_origin + t * ray_dir
        
        # Transform to texture coordinates
        tmp = vec3(pos.x, pos.y, pos.z * 2.3486238532110093)
        tex_pos = (1.0 + tmp) / 2.0
        tex_pos.y = 1.0 - tex_pos.y
        
        # Sample current value
        val = volume_texture.sample(tex_pos).r
        
        # Sample next value for crossing detection
        t_next = t + dt
        pos_next = ray_origin + t_next * ray_dir
        tmp_next = vec3(pos_next.x, pos_next.y, pos_next.z * 2.3486238532110093)
        tex_pos_next = (1.0 + tmp_next) / 2.0
        tex_pos_next.y = 1.0 - tex_pos_next.y
        val_next = volume_texture.sample(tex_pos_next).r
        
        # Check for isosurface crossing
        if val < iso_threshold and val_next > iso_threshold:
            # Binary search for exact surface position
            surf_pos = binary_search_surface(tex_pos, tex_pos_next, iso_threshold, volume_texture)
            
            # Accumulate color from surface sample
            surf_val = volume_texture.sample(surf_pos).r
            color_acc += color_map.sample(surf_val)
            
            valid_samples += 1
            mean_pos += surf_pos
            mean_sum += surf_val
        else:
            cnt -= 1
            
    if valid_samples > 0:
        mean_val = mean_sum / valid_samples
        mean_p = mean_pos / valid_samples
        
        base_color = color_map.sample(mean_val)
        
        if enable_lighting:
            normal = calculate_normal(mean_p, volume_texture)
            lit_color = calculate_lighting(normal, -ray_dir, mean_p, base_color.rgb)
            return vec4(lit_color, 1.0)
        else:
            return vec4(base_color.rgb, 1.0)
    else:
        return vec4(0.0, 0.0, 0.0, 0.0)
```

**Parameters:**
- `num_steps`: Fixed at 256 [S1, unofficial].
- `iso_threshold`: Passed as a uniform; fractional part used for comparison [S1, unofficial].
- `compensate`: A factor $1/(\tau + 0.5)$ applied to color in some implementations [S1, unofficial].

**Edge Cases and Pitfalls:**
- The hardcoded multiplier $2.3486238532110093$ for the Z-axis assumes a specific non-uniform aspect ratio or data layout [S1, unofficial].
- Y-axis inversion is required for image orientation [S1, unofficial].
- Division by zero is avoided by initializing `cnt` to 1 and checking `validSamples > 0` before averaging [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ per pixel, where $N=256$ is the fixed number of steps [S1, unofficial].
- **Memory Complexity:** $O(1)$ per pixel for accumulators [S1, unofficial].
- **Strengths:** Provides a smoothed representation of internal structure density while maintaining surface shading cues [S2, unofficial].
- **Limitations:** The fixed step count and specific texture coordinate transformations limit generalizability [S1, unofficial]. The method may produce artifacts if the isosurface is not well-defined or if the density range is not normalized [S2, unofficial].
- **When to use:** When visualizing the average density of a volumetric object with surface-like shading is desired [S2, unofficial].
- **When not to use:** When precise surface location is critical without averaging, or when the data layout does not match the assumed aspect ratio [S1, unofficial].

## Background
The method is described in the context of a ray casting implementation using OpenGL and GLSL [S1, unofficial]. It is also referenced in the context of "VisualNifty" as "Exact Projection Mean Density" [S2, unofficial]. The technique combines elements of mean density projection with isosurface detection to create a hybrid rendering effect [S1, S2, unofficial].

## Sources
[S1] ShashwatC/Ray, https://github.com/ShashwatC/Ray, fetched 2026-10-26 (unofficial)
[S2] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
