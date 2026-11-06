# SDF Cloud Rendering

Also called: Signed Distance Field Clouds

## Overview
SDF Cloud Rendering is a technique for rendering volumetric clouds using Signed Distance Fields (SDFs) via ray marching in a fragment shader, utilizing warped sphere primitives and smooth unions to create complex shapes [S1, unofficial]. It differs from standard volumetric rendering by representing the cloud volume as an implicit surface defined by analytic distance functions rather than a sampled density grid, allowing for sharp, stylized cloud structures with minimal memory overhead [S1, unofficial].

## How it works
The method renders a scene by shooting a ray through each pixel, marching along the ray to find the intersection with the cloud volume defined by a composite SDF, and shading the surface based on a shadow map derived from the same SDF.

### Inputs and Outputs
- **Inputs**: Fragment coordinates (`gl_FragCoord`), screen resolution (`screenSize`), camera field of view (45.0 degrees), and a random seed for jitter generation.
- **Outputs**: A final color value for each pixel, tone-mapped to the [0, 1] range.

### Data Structures
No complex data structures are used. The scene is defined entirely by constant parameters and analytic functions within the shader code.

### Parameters
- `MAX_MARCHING_STEPS`: Integer constant, value 100 [S1, unofficial].
- `MIN_DIST`: Float constant, value 0.0 [S1, unofficial].
- `MAX_DIST`: Float constant, value 100.0 [S1, unofficial].
- `EPSILON`: Float constant, value 0.0001 [S1, unofficial].
- `STEP_SCALE`: Float constant, value 0.67 [S1, unofficial].
- `EXPOSURE`: Float constant, value 1.0 [S1, unofficial].
- `WHITE`: Float constant, value 200.0 [S1, unofficial].
- `LIGHT`: Constant vector, value `vec3(-10.0, 10.0, 3.0)` [S1, unofficial].

### Formulas
1. **Jitter Function**:
   $$ \text{jitter}(mixAmount, offset) = 0.09 \times \sum_{p=0}^{2} \frac{\sin((offset \times 1234.0 + mixAmount) \times 2^p)}{2^p} $$
   [S1, unofficial]

2. **Warping**:
   $$ \text{warp}(v, scale) = v + scale \times \text{vec3}(\text{jitter}(v.x, 0.0), \text{jitter}(v.y, 0.2), \text{jitter}(v.z, 0.5)) $$
   [S1, unofficial]

3. **Smooth Union**:
   $$ h = \text{clamp}(0.5 + 0.5 \times \frac{d2 - d1}{k}, 0.0, 1.0) $$
   $$ \text{smoothUnion}(d1, d2, k) = \text{mix}(d2, d1, h) - k \times h \times (1.0 - h) $$
   [S1, unofficial]

4. **Sphere SDF**:
   $$ \text{sphereSDF}(p, o, r) = \| p - o \| - r $$
   [S1, unofficial]

5. **Hable Tone Mapping**:
   $$ \text{toneMap}(x) = \frac{x(Ax + CB) + DE}{x(Ax + B) + DF} - \frac{E}{F} $$
   Where $A=0.15, B=0.50, C=0.10, D=0.20, E=0.02, F=0.30$ [S1, unofficial].

### Steps
1. **Ray Direction Calculation**:
   Calculate the ray direction for the current fragment.
   $$ xy = \text{fragCoord} - \frac{\text{screenSize}}{2.0} $$
   $$ z = \frac{\text{screenSize}.y}{\tan(\frac{\text{radians}(45.0)}{2.0})} $$
   $$ \text{dir} = \text{normalize}(\text{vec3}(xy, -z)) $$
   [S1, unofficial]

2. **Ray Marching for Intersection**:
   Initialize `depth = MIN_DIST`. Loop `i` from 0 to `MAX_MARCHING_STEPS - 1`:
   - Calculate sample point: $p = \text{eye} + \text{depth} \times \text{dir}$.
   - Evaluate scene SDF: $dist = \text{sceneSDF}(p)$.
   - If $dist < \text{EPSILON}$, return `depth`.
   - Advance depth: $\text{depth} += dist \times \text{STEP\_SCALE}$.
   - If $\text{depth} \ge \text{MAX\_DIST}$, return `MAX_DIST`.
   [S1, unofficial]

3. **Scene SDF Construction**:
   The scene SDF is constructed by warping the input point and combining sphere primitives.
   - Warp the sample point: $v_{warped} = \text{warp}(samplePoint, 1.5)$.
   - Compute distance to first sphere: $d1 = \text{sphereSDF}(v_{warped}, \text{vec3}(0.5, 0.0, 0.0), 1.0)$.
   - Compute distance to second sphere: $d2 = \text{sphereSDF}(v_{warped}, \text{vec3}(-0.5, -0.4, 0.2), 0.5)$.
   - Smoothly union them: $smoothed = \text{smoothUnion}(d1, d2, 0.4)$.
   - Compute distance to third sphere: $d3 = \text{sphereSDF}(v_{warped}, \text{vec3}(-0.8, 0.55, 0.2), 0.3)$.
   - Return the minimum: $\min(smoothed, d3)$.
   [S1, unofficial]

4. **Shading**:
   - If the returned distance is greater than `MAX_DIST - EPSILON`, render the sky.
     - Calculate sky color based on the dot product of the ray direction and the vertical vector $(0.0, -1.0, 0.0)$.
     - $d = \text{dot}(\text{dir}, \text{normalize}(\text{vec3}(0.0, -1.0, 0.0)))$.
     - $d = \text{clamp}(d \times 3.0 + 0.5, 0.0, 1.0)$.
     - Return $\text{mix}(\text{vec3}(4.0, 40.0, 100.0), \text{vec3}(20.0, 100.0, 100.0), d)$.
   - Otherwise, calculate the intersection point: $p_{int} = \text{eye} + \text{dist} \times \text{dir}$.
   - Calculate shadow:
     - Initialize $k=2.0, t=100.0 \times \text{EPSILON}, res=1.0$.
     - Loop `i` from 0 to `MAX_MARCHING_STEPS - 1`:
       - $h = \text{sceneSDF}(p_{int} + t \times \text{toLight})$.
       - If $h < \text{EPSILON}$, return 0.0.
       - $t += h \times \text{STEP\_SCALE}$.
       - $res = \min(res, k \times h / t)$.
     - Return $res$.
   - Mix between dark and light colors:
     - $\text{LIGHT\_COLOR} = \text{vec3}(100.0, 100.0, 100.0)$.
     - $\text{DARK\_COLOR} = \text{vec3}(5.0, 8.0, 20.0)$.
     - Return $\text{mix}(\text{DARK\_COLOR}, \text{LIGHT\_COLOR}, \text{shadow})$.
   [S1, unofficial]

5. **Tone Mapping**:
   - Apply Hable tone mapping to the final color.
   - Scale by $wScale = 1.0 / \text{toneMap}(\text{EXPOSURE} \times \text{WHITE})$.
   - Output $\text{vec4}(\text{toneMap}(\text{EXPOSURE} \times \text{color}) \times wScale, 1.0)$.
   [S1, unofficial]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def jitter(mix_amount, offset, seed):
    amount = 0.0
    scale = 1.0
    offset += seed
    for power in range(3):
        amount += math.sin((offset * 1234.0 + mix_amount) * scale) / scale
        scale *= 2.0
    return amount * 0.09

def warp(v, scale, seed):
    return v + scale * np.array([
        jitter(v[0], 0.0, seed),
        jitter(v[1], 0.2, seed),
        jitter(v[2], 0.5, seed)
    ])

def smooth_union(d1, d2, k):
    h = np.clip(0.5 + 0.5 * (d2 - d1) / k, 0.0, 1.0)
    return np.where(h > 0.5, d1, d2) - k * h * (1.0 - h)

def sphere_sdf(p, origin, radius):
    return np.linalg.norm(p - origin) - radius

def scene_sdf(sample_point, seed):
    warped = warp(sample_point, 1.5, seed)
    d1 = sphere_sdf(warped, np.array([0.5, 0.0, 0.0]), 1.0)
    d2 = sphere_sdf(warped, np.array([-0.5, -0.4, 0.2]), 0.5)
    smoothed = smooth_union(d1, d2, 0.4)
    d3 = sphere_sdf(warped, np.array([-0.8, 0.55, 0.2]), 0.3)
    return min(smoothed, d3)

def ray_direction(frag_coord, screen_size):
    xy = frag_coord - screen_size / 2.0
    z = screen_size[1] / math.tan(math.radians(45.0) / 2.0)
    return np.array([xy[0], xy[1], -z]) / np.linalg.norm([xy[0], xy[1], -z])

def ray_march(eye, dir, min_dist, max_dist, max_steps, epsilon, step_scale, seed):
    depth = min_dist
    for _ in range(max_steps):
        dist = scene_sdf(eye + depth * dir, seed)
        if dist < epsilon:
            return depth
        depth += dist * step_scale
        if depth >= max_dist:
            return max_dist
    return max_dist

def shadow(origin, dir, max_steps, epsilon, step_scale, seed):
    k = 2.0
    t = 100.0 * epsilon
    res = 1.0
    for _ in range(max_steps):
        h = scene_sdf(origin + t * dir, seed)
        if h < epsilon:
            return 0.0
        t += h * step_scale
        res = min(res, k * h / t)
    return res

def tone_map(x):
    A = 0.15
    B = 0.50
    C = 0.10
    D = 0.20
    E = 0.02
    F = 0.30
    return (x * (A * x + C * B) + D * E) / (x * (A * x + B) + D * F) - E / F

def render(frag_coord, screen_size, seed):
    dir = ray_direction(frag_coord, screen_size)
    eye = np.array([0.0, 0.0, 8.0])
    dist = ray_march(eye, dir, 0.0, 100.0, 100, 0.0001, 0.67, seed)
    
    if dist > 100.0 - 0.0001:
        d = np.dot(dir, np.array([0.0, -1.0, 0.0]))
        d = np.clip(d * 3.0 + 0.5, 0.0, 1.0)
        return np.array([4.0, 40.0, 100.0]) * (1.0 - d) + np.array([20.0, 100.0, 100.0]) * d
    
    intersection = eye + dist * dir
    to_light = np.array([-10.0, 10.0, 3.0]) - intersection
    to_light = to_light / np.linalg.norm(to_light)
    
    shadow_val = shadow(intersection, to_light, 100, 0.0001, 0.67, seed)
    
    light_color = np.array([100.0, 100.0, 100.0])
    dark_color = np.array([5.0, 8.0, 20.0])
    color = dark_color * (1.0 - shadow_val) + light_color * shadow_val
    
    w_scale = 1.0 / tone_map(1.0 * 200.0)
    color = np.array([tone_map(c) for c in color]) * w_scale
    return color
```

### Edge Cases and Pitfalls
- **Precision**: The use of `EPSILON` (0.0001) is critical for determining intersection. If too large, the ray may miss thin cloud structures; if too small, floating-point errors may cause infinite loops or missed intersections [S1, unofficial].
- **Overshooting**: `STEP_SCALE` (0.67) is used to prevent overshooting the surface. A value of 1.0 or higher could cause the ray to jump over the surface, leading to incorrect shading or missed intersections [S1, unofficial].
- **Warping Artifacts**: The warping function introduces noise and distortion. The choice of offsets (0.0, 0.2, 0.5) and the seed value affect the visual appearance. A fixed seed will result in static clouds, while a varying seed can animate them [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N \times M)$, where $N$ is the number of pixels and $M$ is the maximum number of marching steps (100). Each step involves a warp, three sphere SDF evaluations, and a smooth union operation.
- **Memory Complexity**: $O(1)$, as no large data structures are stored. All computations are done on the fly in the fragment shader.
- **Performance**: The method is computationally expensive per pixel due to the multiple SDF evaluations and the shadow ray marching. However, it is memory-efficient compared to voxel-based methods.
- **Strengths**: No need for texture sampling or grid storage; sharp, stylized visuals; easy to animate by varying the seed or warp parameters.
- **Limitations**: Limited to specific geometric primitives (spheres); not suitable for complex, organic cloud shapes without significant modification; shadow calculation adds significant overhead.

## Background
SDF Cloud Rendering was created by Dave Pagurek as a small experiment in rendering with signed distance fields. It was created on the train from Ottawa to Kingston for Graphics Interface 2019 [S1, unofficial]. The technique demonstrates the potential of SDFs for creating volumetric effects without the need for complex volumetric data structures.

## Sources
[S1] davepagurek/sdf-clouds, https://github.com/davepagurek/sdf-clouds, fetched 2026-10-27 (unofficial)
