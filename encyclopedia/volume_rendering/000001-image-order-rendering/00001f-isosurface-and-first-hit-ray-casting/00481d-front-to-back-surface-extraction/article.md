# Front-to-Back Surface Extraction

Also called: First-Hit Surface Extraction, Threshold-Based Surface Raycasting

## Overview
Front-to-Back Surface Extraction is an image-order rendering technique that identifies the first voxel along a ray whose scalar value falls within a user-defined density threshold range, treating this point as the visible surface [S1, unofficial]. It differs from standard volumetric rendering by discarding all data behind the first hit and from advanced isosurface methods by relying on simple threshold crossing rather than continuous surface interpolation or algebraic root finding [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D scalar volume dataset $V$, a transfer function $T(d)$ mapping density to color, a threshold range $[\tau_{min}, \tau_{max}]$, a minimum gradient magnitude $\tau_{grad}$, and camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$, near/far planes).
- Output: A single pixel color $C$ and opacity $\alpha$.

**Data Structures:**
- `Ray`: Struct containing `startPos` (vec3), `endPos` (vec3), `direction` (vec3).
- `RaymarchInfo`: Struct containing `stepSize` (float), `numSteps` (int), `numStepsRecip` (float).
- `Volume`: 3D texture or array supporting `getDensity(pos)` and `getGradient(pos)`.

**Parameters:**
- `MAX_NUM_STEPS`: 1024 [S1, unofficial].
- `JITTER_FACTOR`: Random offset scaling factor [S1, unofficial].
- `CROSS_SECTION_ON`: Boolean flag for cutout geometry [S1, unofficial].
- `MULTIVOLUME_ISOLATE` / `MULTIVOLUME_OVERLAY`: Boolean flags for multi-volume handling [S1, unofficial].

**Steps:**
1. **Ray Initialization**: Compute the ray from the camera origin through the pixel center to the volume bounds. Calculate `stepSize`, `numSteps`, and `numStepsRecip` based on the ray length and sampling rate [S1, unofficial].
2. **Jittering**: Add a random offset to the ray start position to reduce aliasing:
   $$ \mathbf{p}_{start} = \mathbf{p}_{start} + \text{JITTER\_FACTOR} \cdot \text{stepSize} \cdot \mathbf{d} \cdot \text{noise}(u, v) $$
   where $\text{noise}(u, v)$ is a pseudo-random value derived from pixel coordinates [S1, unofficial].
3. **Iteration**: Loop $i$ from $0$ to $\min(\text{numSteps}, \text{MAX\_NUM\_STEPS}) - 1$:
   a. **Position**: Compute current position $\mathbf{p}$ by linear interpolation between start and end:
      $$ \mathbf{p} = \text{lerp}(\mathbf{p}_{start}, \mathbf{p}_{end}, i \cdot \text{numStepsRecip}) $$
   b. **Cutout Check**: If `CROSS_SECTION_ON` is true, evaluate `IsCutout(p)`. If the point is inside a cutout region, skip to the next step [S1, unofficial].
   c. **Density Sampling**: Retrieve density $d = \text{getDensity}(\mathbf{p})$ [S1, unofficial].
   d. **Multi-Volume Handling**:
      - If `MULTIVOLUME_ISOLATE` is active, retrieve secondary density $d_{sec}$. If $d_{sec} \le 0$, skip [S1, unofficial].
      - If `MULTIVOLUME_OVERLAY` is active, retrieve $d_{sec}$. If $d_{sec} > 0$ and the secondary transfer function yields non-zero alpha, compute the secondary color, normal, and lighting. Set the pixel color to this secondary result, set opacity to 1.0, and terminate the ray march [S1, unofficial].
   e. **Threshold Check**: If $\tau_{min} < d < \tau_{max}$:
      i. Compute gradient $\mathbf{g} = \nabla V(\mathbf{p})$ [S1, unofficial].
      ii. Compute gradient magnitude $g_{mag} = \|\mathbf{g}\|$ [S1, unofficial].
      iii. If $g_{mag} > \tau_{grad}$:
           - Compute normal $\mathbf{n} = \mathbf{g} / g_{mag}$ [S1, unofficial].
           - Sample transfer function $C_{surf} = T(d)$ [S1, unofficial].
           - Apply lighting model using $\mathbf{n}$, light direction, and view direction to produce final color [S1, unofficial].
           - Return $C_{surf}$ and terminate.
   f. **Termination**: If the loop completes without a hit, return transparent/black.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def front_to_back_surface_extraction(ray_origin, ray_direction, volume, tf, params):
    # 1. Initialize Ray
    ray_start, ray_end = intersect_volume(ray_origin, ray_direction, volume.bounds)
    ray_length = length(ray_end - ray_start)
    num_steps = int(ray_length / params.step_size)
    num_steps = min(num_steps, params.MAX_NUM_STEPS)
    num_steps_recip = 1.0 / num_steps if num_steps > 0 else 0.0
    
    # 2. Jitter
    jitter = params.JITTER_FACTOR * params.step_size * random_noise(ray_origin.xy)
    ray_start = ray_start + jitter * ray_direction
    
    # 3. Loop
    for i in range(num_steps):
        t = i * num_steps_recip
        curr_pos = lerp(ray_start, ray_end, t)
        
        # Cutout check
        if params.CROSS_SECTION_ON and is_cutout(curr_pos):
            continue
            
        density = volume.get_density(curr_pos)
        
        # Multi-volume overlay
        if params.MULTIVOLUME_OVERLAY:
            sec_density = volume.get_secondary_density(curr_pos)
            if sec_density > 0.0:
                sec_color = volume.get_secondary_tf_color(sec_density)
                if sec_color.a > 0.0:
                    grad = volume.get_gradient(curr_pos)
                    grad_mag = length(grad)
                    if grad_mag > 0.0:
                        normal = grad / grad_mag
                        # Lighting calculation
                        light_dir = normalize(-ray_direction)
                        color = calculate_lighting(sec_color.rgb, normal, light_dir, ray_direction)
                        return color, 1.0
        
        # Threshold check
        if params.MinVal < density < params.MaxVal:
            grad = volume.get_gradient(curr_pos)
            grad_mag = length(grad)
            if grad_mag > params.MinGradient:
                normal = grad / grad_mag
                color = tf.get_color(density)
                # Lighting calculation
                light_dir = normalize(-ray_direction)
                final_color = calculate_lighting(color.rgb, normal, light_dir, ray_direction)
                return final_color, 1.0
                
    return 0.0, 0.0
```

## Operating characteristics
- **Time Complexity**: $O(N)$ per pixel, where $N$ is the number of steps along the ray. Typically bounded by `MAX_NUM_STEPS` [S1, unofficial].
- **Memory Complexity**: $O(1)$ auxiliary memory per ray, plus storage for the volume dataset.
- **Strengths**: Simple implementation; fast convergence if the surface is near the front; low memory overhead.
- **Limitations**: Sensitive to sampling rate (aliasing); no sub-voxel precision without interpolation; fails if the surface is thinner than the step size or if the gradient is too weak [S1, unofficial].
- **When to use**: Real-time visualization of thick, well-defined surfaces; simple cutout effects.
- **When not to use**: Thin structures; high-precision medical imaging requiring sub-voxel accuracy; scenes with complex internal structures behind the surface.

## Background
This method is implemented in the UnityVolumeRendering project by M. Lavik [S1, unofficial]. It represents a basic approach to surface extraction within raycasting, prioritizing simplicity and performance over geometric accuracy.

## Sources
[S1] GitHub - mlavik1/UnityVolumeRendering: Volume rendering, implemented in ..., https://github.com/mlavik1/UnityVolumeRendering, fetched 2026-10-28 (unofficial)
