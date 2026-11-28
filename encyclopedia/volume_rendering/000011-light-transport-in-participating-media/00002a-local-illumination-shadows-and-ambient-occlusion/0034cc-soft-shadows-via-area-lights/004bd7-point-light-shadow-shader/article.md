# Point Light Shadow Shader

## Overview
A shader function that applies shadow rays from surface points to a point light, handling plane intersections and soft shadows via Gaussian filtering [S1, unofficial].
## How it works
**Inputs:**
- `rays`: A `Rays` object containing `origins` (N, 3), `dirs` (N, 3), and `dist_max` [S1, unofficial].
- `rb`: A render buffer object containing `hit` (N, bool), `depth` (N, 1), `xyz` (N, 3), `normal` (N, 3), and `rgb` (N, 4) [S1, unofficial].
- `pipeline`: An object with a `tracer` method and a `nef` (neural implicit field) representation [S1, unofficial].
- `point_light`: Position of the point light, default `[1.5, 4.5, 1.5]` [S1, unofficial].
- `min_y`: Location of the xz plane, default `-2.0` [S1, unofficial].
- `rate`: Scalar divisor for plane intersection calculation, definition not given in the sources [S1, unofficial].

**Data Structures:**
- `Rays`: Contains `origins`, `dirs`, `dist_min`, `dist_max` [S1, unofficial].
- `rb` (Render Buffer): Contains boolean `hit`, float `depth`, float `xyz`, float `normal`, float `rgb` [S1, unofficial].

**Steps:**
1. Compute delta: $\delta = \text{rays.origins}[:, 1] - \text{min\_y}$ [S1, unofficial].
2. Compute plane parameter: $t = \delta / \text{rate}$ [S1, unofficial].
3. Identify potential plane hits: `plane_hit` is True where $t > 0$ and $t < 500$ [S1, unofficial].
4. Refine `plane_hit`: `plane_hit` = `plane_hit` & ($t < \text{rb.depth}[\dots, 0]$) [S1, unofficial].
5. Update `rb.hit`: `rb.hit` = `rb.hit` & ~`plane_hit` [S1, unofficial].
6. Update `rb.depth` and `rb.xyz` for `plane_hit`: $\text{rb.depth}[\text{plane\_hit}] = t[\text{plane\_hit}].\text{unsqueeze}(1)$ and $\text{rb.xyz}[\text{plane\_hit}] = \text{rays.origins}[\text{plane\_hit}] + \text{rays.dirs}[\text{plane\_hit}] \times t[\text{plane\_hit}].\text{unsqueeze}(1)$ [S1, unofficial].
7. Set `rb.normal` for `plane_hit`: $\text{rb.normal}[\text{plane\_hit}] = 0$ and $\text{rb.normal}[\text{plane\_hit}, 1] = 1$ [S1, unofficial].
8. Compute shadow ray origin: $\text{shadow\_ray\_o} = \text{rb.xyz} + 0.01 \times \text{rb.normal}$ [S1, unofficial].
9. Compute shadow ray direction: generate noise $\sim \mathcal{N}(0, 0.01)$ with shape matching `rb.xyz`, then $\text{shadow\_ray\_d} = \text{normalize}(\text{noise} + \text{light\_o} - \text{shadow\_ray\_o})$ [S1, unofficial].
10. Check if rays hit the light: `light_hit` = $(\text{shadow\_ray\_d} \cdot \text{rb.normal}) > 0.0$ [S1, unofficial].
11. Create `shadow_rays`: `Rays(origins=shadow_ray_o, dirs=shadow_ray_d, dist_min=0, dist_max=rays.dist_max)` [S1, unofficial].
12. Trace shadow rays: `rb.shadow` = `pipeline.tracer(pipeline.nef, rays=shadow_rays).hit` [S1, unofficial].
13. Zero out shadow for non-light-hit rays: `rb.shadow[~light_hit]` = 0.0 [S1, unofficial].
14. Compute shadow map: $\text{shadow\_map} = \text{clamp}((1.0 - \text{rb.shadow}.\text{float}()) + 0.7, 0.0, 1.0)$ [S1, unofficial].
15. Apply Gaussian filter to `shadow_map` with $\sigma=2$ [S1, unofficial].
16. Multiply `rb.rgb` by `shadow_map`: $\text{rb.rgb}[\dots, :3] \times= \text{shadow\_map}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
from scipy.ndimage import gaussian_filter
from torch.nn import functional as F
from dataclasses import dataclass

@dataclass
class Rays:
    origins: torch.Tensor
    dirs: torch.Tensor
    dist_min: float
    dist_max: float

@dataclass
class RenderBuffer:
    hit: torch.Tensor
    depth: torch.Tensor
    xyz: torch.Tensor
    normal: torch.Tensor
    rgb: torch.Tensor

def point_light_shadow_shader(rays: Rays, rb: RenderBuffer, pipeline: object, point_light: list = [1.5, 4.5, 1.5], min_y: float = -2.0, rate: float = 1.0) -> RenderBuffer:
    # Step 1: Plane Intersection Refinement
    delta = rays.origins[:, 1] - min_y
    plane_t = delta / rate
    
    plane_hit = (plane_t > 0) & (plane_t < 500)
    plane_hit = plane_hit & (plane_t < rb.depth[..., 0])
    
    rb.hit = rb.hit & ~plane_hit
    
    plane_t_unsqueezed = plane_t[plane_hit].unsqueeze(1)
    rb.depth[plane_hit] = plane_t_unsqueezed
    rb.xyz[plane_hit] = rays.origins[plane_hit] + rays.dirs[plane_hit] * plane_t_unsqueezed
    
    rb.normal[plane_hit] = 0.0
    rb.normal[plane_hit, 1] = 1.0
    
    # Step 2: Shadow Ray Generation
    shadow_ray_o = rb.xyz + 0.01 * rb.normal
    light_o = torch.tensor(point_light, dtype=rb.xyz.dtype, device=rb.xyz.device)
    
    noise = torch.zeros_like(rb.xyz).normal_(0.0, 0.01)
    # Fixed source indexing bug: F.normalize returns a tensor, not a tuple
    shadow_ray_d = F.normalize(noise + light_o - shadow_ray_o, dim=1)
    
    # Step 3: Shadow Ray Tracing
    light_hit = (shadow_ray_d * rb.normal).sum(-1) > 0.0
    
    shadow_rays = Rays(origins=shadow_ray_o, dirs=shadow_ray_d, dist_min=0, dist_max=rays.dist_max)
    rb.shadow = pipeline.tracer(pipeline.nef, rays=shadow_rays).hit
    
    rb.shadow[~light_hit] = 0.0
    
    # Step 4: Shadow Map Application
    shadow_map = torch.clamp((1.0 - rb.shadow.float()) + 0.7, 0.0, 1.0).cpu().numpy()[..., 0]
    shadow_map = torch.from_numpy(gaussian_filter(shadow_map, sigma=2)).unsqueeze(-1)
    
    rb.rgb[..., :3] *= shadow_map.cuda() if rb.rgb.device.type == "cuda" else shadow_map
    
    return rb
```
## Operating characteristics
Not given in the sources.
## Background
The method is described in the NVIDIA Kaolin-Wisp repository [S1, unofficial]. It is part of the interactive rendering pipeline in Wisp, a framework for neural implicit fields [S1, unofficial].
## Sources
[S1] NVIDIAGameWorks/kaolin-wisp, https://github.com/NVIDIAGameWorks/kaolin-wisp, fetched 2026-10-26 (unofficial)
