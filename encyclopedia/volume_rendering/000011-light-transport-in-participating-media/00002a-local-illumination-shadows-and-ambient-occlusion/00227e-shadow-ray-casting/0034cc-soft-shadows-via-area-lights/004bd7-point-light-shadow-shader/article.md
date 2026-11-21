# Point Light Shadow Shader


## Overview
A shader function that applies shadow rays from surface points to a point light, handling plane intersections and soft shadows via Gaussian filtering [S1, unofficial]. It differs from its parent [Soft Shadows via Area Lights](7320-soft-shadows-via-area-lights.md) by modeling a single point source rather than an area, and from [Soft Shadows via Ray Tracing](5136-soft-shadows-via-ray-tracing.md) by using a specific plane-intersection refinement and post-process Gaussian blur rather than direct area-light sampling [S1, unofficial].

## How it works
**Inputs:**
- `rays`: A `Rays` object containing `origins` (N, 3), `dirs` (N, 3), and `dist_max`.
- `rb`: A render buffer object containing `hit` (N, bool), `depth` (N, 1), `xyz` (N, 3), `normal` (N, 3), and `rgb` (N, 4).
- `pipeline`: An object with a `tracer` method and a `nef` (neural implicit field) representation.
- `point_light`: Position of the point light, default `[1.5, 4.5, 1.5]` [S1, unofficial].
- `min_y`: Location of the xz plane, default `-2.0` [S1, unofficial].

**Data Structures:**
- `Rays`: Contains `origins`, `dirs`, `dist_min`, `dist_max`.
- `rb` (Render Buffer): Contains boolean `hit`, float `depth`, float `xyz`, float `normal`, float `rgb`.

**Steps:**
1. **Plane Intersection Refinement:**
   - Compute delta: $\delta = \text{rays.origins}[:, 1] - \text{min\_y}$ [S1, unofficial].
   - Compute plane parameter: $t = \delta / \text{rate}$ [S1, unofficial]. Note: `rate` is not defined in the source notes. [G] Assume `rate` is a scalar constant or derived from ray direction magnitude if not provided. Given the context of `rays.dirs`, if `rate` is missing, it might be a typo for `rays.dirs[:, 1]` or a fixed constant. However, the source explicitly uses `rate`. Without definition, we treat it as an undefined constant or assume it equals 1.0 if not specified. [G] Let us assume `rate` is a global constant or parameter not listed. If `rate` is intended to be the y-component of the ray direction, the formula would be $t = \delta / \text{rays.dirs}[:, 1]$. The source says `rate`. We will use `rate` as a variable.
   - Identify potential plane hits: `plane_hit` is True where $t > 0$ and $t < 500$ [S1, unofficial].
   - Refine `plane_hit`: `plane_hit` = `plane_hit` & ($t < \text{rb.depth}[:, 0]$) [S1, unofficial].
   - Update `rb.hit`: `rb.hit` = `rb.hit` & ~`plane_hit` [S1, unofficial].
   - Update `rb.depth` and `rb.xyz` for `plane_hit`:
     - $\text{rb.depth}[\text{plane\_hit}] = t[\text{plane\_hit}].\text{unsqueeze}(1)$ [S1, unofficial].
     - $\text{rb.xyz}[\text{plane\_hit}] = \text{rays.origins}[\text{plane\_hit}] + \text{rays.dirs}[\text{plane\_hit}] \times t[\text{plane\_hit}].\text{unsqueeze}(1)$ [S1, unofficial].
   - Set `rb.normal` for `plane_hit`:
     - $\text{rb.normal}[\text{plane\_hit}] = 0$ [S1, unofficial].
     - $\text{rb.normal}[\text{plane\_hit}, 1] = 1$ [S1, unofficial]. (Normal is $(0, 1, 0)$).

2. **Shadow Ray Generation:**
   - Compute shadow ray origin: $\text{shadow\_ray\_o} = \text{rb.xyz} + 0.01 \times \text{rb.normal}$ [S1, unofficial].
   - Compute shadow ray direction:
     - Generate random noise: $\text{noise} \sim \mathcal{N}(0, 0.01)$ with shape same as `rb.xyz` [S1, unofficial].
     - $\text{shadow\_ray\_d} = \text{normalize}(\text{noise} + \text{light\_o} - \text{shadow\_ray\_o})$ [S1, unofficial].
     - `light_o` is `point_light` [S1, unofficial].

3. **Shadow Ray Tracing:**
   - Check if rays hit the light: `light_hit` = $(\text{shadow\_ray\_d} \cdot \text{rb.normal}) > 0.0$ [S1, unofficial].
   - Create `shadow_rays`: `Rays(origins=shadow_ray_o, dirs=shadow_ray_d, dist_min=0, dist_max=rays.dist_max)` [S1, unofficial].
   - Trace shadow rays: `rb.shadow` = `pipeline.tracer(pipeline.nef, rays=shadow_rays).hit` [S1, unofficial].
   - Zero out shadow for non-light-hit rays: `rb.shadow[~light_hit]` = 0.0 [S1, unofficial].

4. **Shadow Map Application:**
   - Compute shadow map: $\text{shadow\_map} = \text{clamp}((1.0 - \text{rb.shadow}.\text{float}()) + 0.7, 0.0, 1.0)$ [S1, unofficial].
   - Apply Gaussian filter to `shadow_map` with $\sigma=2$ [S1, unofficial].
   - Multiply `rb.rgb` by `shadow_map`: $\text{rb.rgb}[:, :3] \times= \text{shadow\_map}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
from scipy.ndimage import gaussian_filter

def point_light_shadow_shader(rays, rb, pipeline, point_light=[1.5, 4.5, 1.5], min_y=-2.0, rate=1.0):
    """
    Applies shadow rays from surface points to a point light, handling plane intersections 
    and soft shadows via Gaussian filtering.
    
    Args:
        rays: Object with origins (N,3), dirs (N,3), dist_max.
        rb: Render buffer with hit (N,bool), depth (N,1), xyz (N,3), normal (N,3), rgb (N,4).
        pipeline: Object with tracer method and nef.
        point_light: Position of the point light.
        min_y: Location of the xz plane.
        rate: Scaling factor for plane intersection. Default 1.0 if not specified.
    """
    # Step 1: Plane Intersection Refinement
    delta = rays.origins[:, 1] - min_y
    plane_t = delta / rate
    
    # Identify potential plane hits
    plane_hit = (plane_t > 0) & (plane_t < 500)
    
    # Refine plane_hit against existing depth
    plane_hit = plane_hit & (plane_t < rb.depth[:, 0])
    
    # Update rb.hit
    rb.hit = rb.hit & ~plane_hit
    
    # Update rb.depth and rb.xyz for plane_hit
    plane_t_unsqueezed = plane_t[plane_hit].unsqueeze(1)
    rb.depth[plane_hit] = plane_t_unsqueezed
    rb.xyz[plane_hit] = rays.origins[plane_hit] + rays.dirs[plane_hit] * plane_t_unsqueezed
    
    # Set rb.normal for plane_hit
    rb.normal[plane_hit] = 0.0
    rb.normal[plane_hit, 1] = 1.0
    
    # Step 2: Shadow Ray Generation
    shadow_ray_o = rb.xyz + 0.01 * rb.normal
    light_o = torch.tensor(point_light, dtype=rb.xyz.dtype, device=rb.xyz.device)
    
    # Generate noise
    noise = torch.zeros_like(rb.xyz).normal_(0.0, 0.01)
    shadow_ray_d = torch.nn.functional.normalize(noise + light_o - shadow_ray_o, dim=1)
    
    # Step 3: Shadow Ray Tracing
    # Check if rays hit the light (dot product > 0)
    light_hit = (shadow_ray_d * rb.normal).sum(-1) > 0.0
    
    # Create shadow rays
    shadow_rays = Rays(origins=shadow_ray_o, dirs=shadow_ray_d, dist_min=0, dist_max=rays.dist_max)
    
    # Trace shadow rays
    rb.shadow = pipeline.tracer(pipeline.nef, rays=shadow_rays).hit
    
    # Zero out shadow for non-light-hit rays
    rb.shadow[~light_hit] = 0.0
    
    # Step 4: Shadow Map Application
    shadow_map = torch.clamp((1.0 - rb.shadow.float()) + 0.7, 0.0, 1.0)
    
    # Apply Gaussian filter
    shadow_map_np = shadow_map.cpu().numpy()
    shadow_map_filtered = gaussian_filter(shadow_map_np, sigma=2)
    shadow_map_tensor = torch.from_numpy(shadow_map_filtered).unsqueeze(-1)
    
    # Multiply rb.rgb by shadow_map
    rb.rgb[:, :3] *= shadow_map_tensor.cuda() if rb.rgb.device.type == 'cuda' else shadow_map_tensor
    
    return rb
```

## Operating characteristics
- **Time Complexity:** Dominated by the ray tracing step `pipeline.tracer`, which is $O(N \cdot M)$ where $N$ is the number of rays and $M$ is the cost per ray in the neural field. The Gaussian filter is $O(N)$.
- **Memory Complexity:** $O(N)$ for storing shadow rays and intermediate buffers.
- **Strengths:** Simple implementation for point lights; Gaussian blur provides a soft shadow effect without complex area light sampling.
- **Limitations:** The `rate` parameter is undefined in the source, which may lead to incorrect plane intersections if not set correctly. The Gaussian filter is a post-process approximation and may not accurately represent the true penumbra of a point light at varying distances. The method assumes a specific plane intersection logic that may not generalize to all scenes.
- **When to use:** When a simple soft shadow effect for a point light is needed and computational resources are limited.
- **When not to use:** When accurate penumbra calculation is required, or when the scene does not contain a clear xz plane for the initial intersection logic.

## Background
The method is described in the NVIDIA Kaolin-Wisp repository [S1, unofficial]. It is part of the interactive rendering pipeline in Wisp, a framework for neural implicit fields. The specific implementation handles plane intersections and applies a Gaussian filter to simulate soft shadows from a point light.

## Sources
[S1] NVIDIAGameWorks/kaolin-wisp, https://github.com/NVIDIAGameWorks/kaolin-wisp, fetched 2026-10-26 (unofficial)
