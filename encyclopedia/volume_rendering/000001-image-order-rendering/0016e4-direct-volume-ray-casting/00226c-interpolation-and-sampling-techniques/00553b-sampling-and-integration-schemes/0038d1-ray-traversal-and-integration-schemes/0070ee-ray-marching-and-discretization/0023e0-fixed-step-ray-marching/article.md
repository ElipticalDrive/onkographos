# Fixed Step Ray Marching

Also called: Ray Marching for Volume Integration, Ray-marching Volumetric Fog

## Overview
Fixed Step Ray Marching is a direct volume rendering technique that integrates the volume rendering equation along camera rays using a constant step size, applying exponential absorption and gradient-based shading to fog-like participating media [S1, unofficial][S2]. It uniformly samples the ray, which simplifies implementation but requires tuning of `step_size` to balance visual quality and performance [S1, unofficial][S2]. Unlike [Exponential Approximation Method](2079-exponential-approximation-method.md), which may use analytical approximations for transmittance, this method explicitly computes transmittance per step using the Beer-Lambert law [S1, unofficial][S2].

## How it works
**Inputs:**
- 3D volume data (density field) [S1, unofficial].
- Camera parameters (ray origin and direction per pixel) [G].
- Light direction vector [S1, unofficial].
- Parameters: `step_size` (default 0.15), `absorption` (default 12.0), `max_steps` (default 800), `vol_color` (default `(1.0, 0.95, 0.88)`), `bg_color` [S1, unofficial].

**Outputs:**
- 2D image with RGB values per pixel [S1, unofficial].

**Data Structures:**
- **Bounding Box:** `bmin`, `bmax` defining the volume extent in world space, padded by `step_size * 2.0` [S1, unofficial].
- **Ray:** Defined by origin `O` and direction `D` [S1, unofficial].
- **Accumulators:** `transmittance` (scalar, initialized to 1.0), `cr`, `cg`, `cb` (RGB color accumulators, initialized to 0.0) [S1, unofficial].

**Steps:**
1. **Setup:** Compute the active bounding box of the volume. If empty, return background color. Calculate world bounds `bmin` and `bmax` with padding `step_size * 2.0` [S1, unofficial]. Normalize `light_dir` [S1, unofficial].
2. **Ray Generation:** For each pixel `(x, y)`, compute ray origin `O` and direction `D` [G]. Intersect the ray with the bounding box to find entry `t_enter` and exit `t_exit` distances [S1, unofficial]. If no intersection, set pixel to background [S1, unofficial].
3. **Initialization:** Set current distance `t = max(t_enter, 0.001)` [S1, unofficial]. Initialize `transmittance = 1.0` and color accumulators `cr = cg = cb = 0.0` [S1, unofficial].
4. **Marching Loop:** Iterate up to `max_steps` [S1, unofficial]:
   a. **Termination Check:** If `t > t_exit` or `transmittance < 0.005`, break loop [S1, unofficial].
   b. **Position Calculation:** Compute world position `P = O + t * D` [S1, unofficial].
   c. **Sampling:** Sample density `d` at `P` from the volume data [S1, unofficial].
   d. **Extinction Calculation:** Compute optical depth for the step: $ext = d \times absorption \times step\_size$ [S1, unofficial].
   e. **Transmittance Step:** Compute step transmittance: $tr\_step = \exp(-ext)$ [S1, unofficial].
   f. **Gradient Calculation:** Compute central differences for shading with $h = step\_size \times 0.5$ [S1, unofficial]:
      $$gx = \text{density}(P + (h, 0, 0)) - \text{density}(P - (h, 0, 0))$$
      $$gy = \text{density}(P + (0, h, 0)) - \text{density}(P - (0, h, 0))$$
      $$gz = \text{density}(P + (0, 0, h)) - \text{density}(P - (0, 0, h))$$
   g. **Shading:** Normalize gradient $\vec{g} = (gx, gy, gz)$ [S1, unofficial]. Compute dot product with light direction: $ndotl = \max(0.0, -\vec{g} \cdot light\_dir)$ [S1, unofficial]. Compute shading factor: $shade = 0.3 + 0.7 \times ndotl$ [S1, unofficial].
   h. **Color Accumulation:** Add contribution to color accumulators:
      $$cr += transmittance \times (1.0 - tr\_step) \times vol\_color.r \times shade$$
      $$cg += transmittance \times (1.0 - tr\_step) \times vol\_color.g \times shade$$
      $$cb += transmittance \times (1.0 - tr\_step) \times vol\_color.b \times shade$$
      [S1, unofficial].
   i. **Transmittance Update:** Update ray transmittance: $transmittance *= tr\_step$ [S1, unofficial].
   j. **Step Advance:** Increment `t += step_size` [S1, unofficial].
5. **Final Composition:** Add background contribution: $cr += transmittance \times bg\_color.r$, $cg += transmittance \times bg\_color.g$, $cb += transmittance \times bg\_color.b$ [S1, unofficial].
6. **Clamping:** Clamp final RGB values to $[0, 1]$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def fixed_step_ray_marching(volume, camera, light_dir, step_size=0.15, absorption=12.0, max_steps=800, vol_color=(1.0, 0.95, 0.88), bg_color=(0.0, 0.0, 0.0)):
    # 1. Setup
    bmin, bmax = compute_active_bounding_box(volume)
    pad = step_size * 2.0
    bmin = (bmin[0] - pad, bmin[1] - pad, bmin[2] - pad)
    bmax = (bmax[0] + pad, bmax[1] + pad, bmax[2] + pad)
    
    light_len = math.sqrt(light_dir[0]**2 + light_dir[1]**2 + light_dir[2]**2)
    if light_len == 0:
        light_dir = (0.0, 0.0, 1.0)
    else:
        light_dir = (light_dir[0]/light_len, light_dir[1]/light_len, light_dir[2]/light_len)

    image_width = camera.width
    image_height = camera.height
    output_image = [[(0.0, 0.0, 0.0) for _ in range(image_width)] for _ in range(image_height)]

    def sample_volume(pos):
        return volume.get_density(pos)

    def compute_active_bounding_box(volume):
        # Returns the active voxel bounds from the volume's grid tree
        return volume.bmin, volume.bmax

    def ray_box_intersect(ray_origin, ray_dir, box_min, box_max):
        t_min = (box_min[0] - ray_origin[0]) / ray_dir[0] if ray_dir[0] != 0 else float('-inf')
        t_max = (box_max[0] - ray_origin[0]) / ray_dir[0] if ray_dir[0] != 0 else float('inf')
        if t_min > t_max:
            t_min, t_max = t_max, t_min
            
        ty_min = (box_min[1] - ray_origin[1]) / ray_dir[1] if ray_dir[1] != 0 else float('-inf')
        ty_max = (box_max[1] - ray_origin[1]) / ray_dir[1] if ray_dir[1] != 0 else float('inf')
        if ty_min > ty_max:
            ty_min, ty_max = ty_max, ty_min
            
        if (t_min > ty_max) or (ty_min > t_max):
            return None, None
        if ty_min > t_min:
            t_min = ty_min
        if ty_max < t_max:
            t_max = ty_max
            
        tz_min = (box_min[2] - ray_origin[2]) / ray_dir[2] if ray_dir[2] != 0 else float('-inf')
        tz_max = (box_max[2] - ray_origin[2]) / ray_dir[2] if ray_dir[2] != 0 else float('inf')
        if tz_min > tz_max:
            tz_min, tz_max = tz_max, tz_min
            
        if (t_min > tz_max) or (tz_min > t_max):
            return None, None
        if tz_min > t_min:
            t_min = tz_min
        if tz_max < t_max:
            t_max = tz_max
            
        return t_min, t_max

    for y in range(image_height):
        for x in range(image_width):
            ray_origin, ray_dir = camera.get_ray(x, y)
            t_enter, t_exit = ray_box_intersect(ray_origin, ray_dir, bmin, bmax)
            
            if t_enter is None:
                output_image[y][x] = bg_color
                continue
            
            t = max(t_enter, 0.001)
            transmittance = 1.0
            cr, cg, cb = 0.0, 0.0, 0.0
            
            for step in range(max_steps):
                if t > t_exit or transmittance < 0.005:
                    break
                
                px = ray_origin[0] + t * ray_dir[0]
                py = ray_origin[1] + t * ray_dir[1]
                pz = ray_origin[2] + t * ray_dir[2]
                
                density = sample_volume((px, py, pz))
                
                ext = density * absorption * step_size
                tr_step = math.exp(-ext)
                
                h = step_size * 0.5
                gx = sample_volume((px + h, py, pz)) - sample_volume((px - h, py, pz))
                gy = sample_volume((px, py + h, pz)) - sample_volume((px, py - h, pz))
                gz = sample_volume((px, py, pz + h)) - sample_volume((px, py, pz - h))
                
                grad_len = math.sqrt(gx**2 + gy**2 + gz**2)
                if grad_len > 0.0:
                    gx /= grad_len
                    gy /= grad_len
                    gz /= grad_len
                
                ndotl = max(0.0, -(gx * light_dir[0] + gy * light_dir[1] + gz * light_dir[2]))
                shade = 0.3 + 0.7 * ndotl
                
                color_factor = transmittance * (1.0 - tr_step) * shade
                cr += color_factor * vol_color[0]
                cg += color_factor * vol_color[1]
                cb += color_factor * vol_color[2]
                
                transmittance *= tr_step
                t += step_size
            
            cr += transmittance * bg_color[0]
            cg += transmittance * bg_color[1]
            cb += transmittance * bg_color[2]
            
            cr = max(0.0, min(1.0, cr))
            cg = max(0.0, min(1.0, cg))
            cb = max(0.0, min(1.0, cb))
            
            output_image[y][x] = (cr, cg, cb)
            
    return output_image
```

## Subtopics

### [Ray Evaluation Stepping DVR](3461-ray-evaluation-stepping-dvr.md)
A variant that uses configurable density ranges and early-out termination, with step size determined by object-space or world-space indexing.

### [Density Marching](3462-density-marching.md)
Uses a fixed number of sampling steps per ray for direct volume rendering, simple to implement but potentially inefficient for sparse volumes.

### [Discrete Volume Rendering Integral (VRI)](3552-discrete-volume-rendering-integral-vri.md)
The fundamental numerical approximation using Riemann sums with constant density/color per segment (Zero Order Integration).

### [Volumetric Light Shafts (URP Raymarching)](3873-volumetric-light-shafts-urp-raymarching.md)
Targets light shafts from a single directional light using fixed step size and exponential absorption, respecting scene depth and shadows.

### [Ray Marching Volume Rendering](4877-ray-marching-volume-rendering.md)
Visualizes 3D scalar/vector fields by casting rays and accumulating color/opacity via front-to-back compositing, revealing internal structures.

### [Ray Marching Volume Rendering (GLSL)](5184-ray-marching-volume-rendering-glsl.md)
Performs the rendering integral in a single GPU pass, enabling optimizations like early ray termination and adaptive sampling.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Depends heavily on `step_size` and `max_steps` [S1, unofficial]. Smaller `step_size` increases accuracy but also computation [S1, unofficial]. Sparse volumes may benefit from acceleration structures like HDDA (Hierarchical Distance Field Acceleration) to skip empty space [S1, unofficial].
- **Strengths:** Simple to implement, suitable for real-time rendering on GPUs, handles participating media like fog and clouds effectively [S1, unofficial][S2].
- **Limitations:** Assumes homogeneous media between steps, which can lead to artifacts in heterogeneous media; fixed step size may be inefficient for sparse volumes if not combined with acceleration [S2].
- **When to use:** Real-time visualization of fog, smoke, or clouds where performance is critical and high-frequency density variations are not expected [S1, unofficial][S2].
- **When not to use:** Physically accurate rendering of heterogeneous media with complex scattering properties, where advanced techniques like Woodcock tracking are required [S2].

## Background
Not given in the sources.

## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] CRYENGINE | Documentation - Volumetric Fog, https://www.cryengine.com/docs/static/engines/cryengine-3/categories/1114113/pages/21268322, fetched 2026-10-29
[S3] Volume Rendering (Path Tracing & Real-Time), https://computergraphics.stackexchange.com/questions/4412, fetched 2026-10-30 (unofficial)
