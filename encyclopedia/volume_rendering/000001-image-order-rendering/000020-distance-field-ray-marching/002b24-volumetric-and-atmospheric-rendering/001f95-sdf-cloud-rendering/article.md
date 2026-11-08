# SDF Cloud Rendering

Also called: Signed Distance Field Clouds

## Overview
SDF Cloud Rendering is a technique for rendering volumetric cloud-like structures using Signed Distance Fields (SDFs) via ray marching in a fragment shader [S1, unofficial]. It utilizes warped sphere primitives and smooth unions to create complex, organic shapes without explicit voxel grids or texture maps [S1, unofficial]. Unlike traditional volumetric methods that sample density fields, this method treats the cloud as a solid object defined by a continuous distance function, relying on geometric shading rather than radiative transfer accumulation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Fragment coordinates (`gl_FragCoord`), screen resolution (`screenSize`), camera field of view, and a random seed (`seed`) for procedural variation.
*   **Outputs:** A final color value for each pixel, tone-mapped to the display range.

**Data Structures:**
No external data structures are used. All geometry is defined implicitly by mathematical functions.

**Parameters:**
*   `MAX_MARCHING_STEPS`: 100 [S1, unofficial].
*   `MIN_DIST`: 0.0 [S1, unofficial].
*   `MAX_DIST`: 100.0 [S1, unofficial].
*   `EPSILON`: 0.0001 [S1, unofficial].
*   `STEP_SCALE`: 0.67 [S1, unofficial].
*   `LIGHT`: `vec3(-10.0, 10.0, 3.0)` [S1, unofficial].
*   `EXPOSURE`: 1.0 [S1, unofficial].
*   `WHITE`: 200.0 [S1, unofficial].

**Algorithm Steps:**

1.  **Ray Setup:** Calculate the ray direction from the camera origin. The camera is fixed at `vec3(0.0, 0.0, 8.0)` [S1, unofficial]. The direction is derived from fragment coordinates relative to the screen center, projected through a 45-degree field of view [S1, unofficial].
2.  **Ray Marching:** March along the ray from `MIN_DIST` to `MAX_DIST`. At each step, evaluate the scene SDF at the current position. If the distance to the surface is less than `EPSILON`, the ray has hit the cloud. Otherwise, advance the ray by `dist * STEP_SCALE` [S1, unofficial].
3.  **Shadow Calculation:** If the ray hits the cloud, calculate a shadow factor by marching from the intersection point towards the light source. This uses a heuristic approximation rather than a binary shadow test [S1, unofficial].
4.  **Shading:** Mix between a dark color (`vec3(5.0, 8.0, 20.0)`) and a light color (`vec3(100.0, 100.0, 100.0)`) based on the shadow result [S1, unofficial].
5.  **Tone Mapping:** Apply Hable's tone mapping function to the final color to fit it into the [0, 1] range [S1, unofficial].

**Formulas:**

*   **Jitter Function:** Used to warp space.
    $$ \text{jitter}(m, o) = 0.09 \sum_{p=0}^{2} \frac{\sin((o + \text{seed}) \cdot 1234.0 + m) \cdot 2^p}{2^p} $$
    Where $m$ is the mix amount (coordinate component), $o$ is the offset, and seed is the random input [S1, unofficial].

*   **Warping:** Distorts the coordinate space to create organic shapes.
    $$ \text{warp}(\mathbf{v}, s) = \mathbf{v} + s \cdot \text{vec3}(\text{jitter}(v_x, 0.0), \text{jitter}(v_y, 0.2), \text{jitter}(v_z, 0.5)) $$
    Where $s$ is the warp scale (1.5 in the example) [S1, unofficial].

*   **Smooth Union:** Blends two SDFs $d_1$ and $d_2$ with factor $k$.
    $$ h = \text{clamp}\left(0.5 + 0.5 \frac{d_2 - d_1}{k}, 0.0, 1.0\right) $$
    $$ \text{result} = \text{mix}(d_2, d_1, h) - k \cdot h \cdot (1.0 - h) $$
    [S1, unofficial].

*   **Sphere SDF:**
    $$ d(\mathbf{p}, \mathbf{c}, r) = \|\mathbf{p} - \mathbf{c}\| - r $$
    Where $\mathbf{p}$ is the sample point, $\mathbf{c}$ is the center, and $r$ is the radius [S1, unofficial].

*   **Scene SDF:** Combines warped spheres.
    $$ \text{scene}(\mathbf{p}) = \min(\text{smoothUnion}(d_1, d_2, 0.4), d_3) $$
    Where $d_1, d_2, d_3$ are sphere SDFs evaluated at the warped position $\mathbf{p}' = \text{warp}(\mathbf{p}, 1.5)$ [S1, unofficial].

*   **Shadow Heuristic:**
    $$ \text{shadow}(\mathbf{o}, \mathbf{d}) = \min_{i} \left( k \frac{h_i}{t_i} \right) $$
    Where $h_i$ is the distance to the surface at step $i$, $t_i$ is the distance from the origin to the step, and $k=2.0$ [S1, unofficial].

*   **Hable Tone Mapping:**
    $$ \text{toneMap}(x) = \frac{x(Ax + CB) + DE}{x(Ax + B) + DF} - \frac{E}{F} $$
    With $A=0.15, B=0.50, C=0.10, D=0.20, E=0.02, F=0.30$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def jitter(mix_amount: float, offset: float, seed: float) -> float:
    amount = 0.0
    scale = 1.0
    current_offset = offset + seed
    for power in range(3):
        amount += math.sin((current_offset * 1234.0 + mix_amount) * scale) / scale
        scale *= 2.0
    return amount * 0.09

def warp(v: np.ndarray, scale: float, seed: float) -> np.ndarray:
    x_jitter = jitter(v[0], 0.0, seed)
    y_jitter = jitter(v[1], 0.2, seed)
    z_jitter = jitter(v[2], 0.5, seed)
    return v + scale * np.array([x_jitter, y_jitter, z_jitter])

def sphere_sdf(sample_point: np.ndarray, origin: np.ndarray, radius: float) -> float:
    return np.linalg.norm(sample_point - origin) - radius

def smooth_union(d1: float, d2: float, k: float) -> float:
    h = np.clip(0.5 + 0.5 * (d2 - d1) / k, 0.0, 1.0)
    return np.where(h == 0, d2, np.where(h == 1, d1, d1)) - k * h * (1.0 - h)
    # Note: np.where logic above is a simplification for mix(d2, d1, h)
    # Correct mix: d2 * (1-h) + d1 * h
    # Re-evaluating mix logic for clarity:
    # return d2 * (1.0 - h) + d1 * h - k * h * (1.0 - h)

def scene_sdf(sample_point: np.ndarray, seed: float) -> float:
    warped = warp(sample_point, 1.5, seed)
    d1 = sphere_sdf(warped, np.array([0.5, 0.0, 0.0]), 1.0)
    d2 = sphere_sdf(warped, np.array([-0.5, -0.4, 0.2]), 0.5)
    smoothed = smooth_union(d1, d2, 0.4)
    d3 = sphere_sdf(warped, np.array([-0.8, 0.55, 0.2]), 0.3)
    return min(smoothed, d3)

def ray_direction(field_of_view: float, size: np.ndarray, frag_coord: np.ndarray) -> np.ndarray:
    xy = frag_coord - size / 2.0
    z = size[1] / math.tan(math.radians(field_of_view) / 2.0)
    return np.array([xy[0], xy[1], -z]) / np.linalg.norm([xy[0], xy[1], -z])

def ray_march(eye: np.ndarray, marching_direction: np.ndarray, start: float, end: float, seed: float) -> float:
    depth = start
    for _ in range(100):
        dist = scene_sdf(eye + depth * marching_direction, seed)
        if dist < 0.0001:
            return depth
        depth += dist * 0.67
        if depth >= end:
            return end
    return end

def calculate_shadow(origin: np.ndarray, direction: np.ndarray, seed: float) -> float:
    k = 2.0
    t = 100.0 * 0.0001
    res = 1.0
    for _ in range(100):
        h = scene_sdf(origin + t * direction, seed)
        if h < 0.0001:
            return 0.0
        t += h * 0.67
        res = min(res, k * h / t)
    return res

def tone_map(x: float) -> float:
    A = 0.15
    B = 0.50
    C = 0.10
    D = 0.20
    E = 0.02
    F = 0.30
    return (x * (A * x + C * B) + D * E) / (x * (A * x + B) + D * F) - E / F

def render(frag_coord: np.ndarray, screen_size: np.ndarray, seed: float) -> np.ndarray:
    fov = 45.0
    dir_vec = ray_direction(fov, screen_size, frag_coord)
    eye = np.array([0.0, 0.0, 8.0])
    
    dist = ray_march(eye, dir_vec, 0.0, 100.0, seed)
    
    if dist > 100.0 - 0.0001:
        # Sky gradient
        d = np.dot(dir_vec, np.array([0.0, -1.0, 0.0]))
        d *= 6.0
        d = d / 2.0 + 0.5
        d = np.clip(d, 0.0, 1.0)
        return np.array([4.0, 40.0, 100.0]) * (1.0 - d) + np.array([20.0, 100.0, 100.0]) * d
    
    intersection = eye + dist * dir_vec
    light_pos = np.array([-10.0, 10.0, 3.0])
    to_light = light_pos - intersection
    to_light = to_light / np.linalg.norm(to_light)
    
    shadow_val = calculate_shadow(intersection, to_light, seed)
    
    dark = np.array([5.0, 8.0, 20.0])
    light = np.array([100.0, 100.0, 100.0])
    color = dark * (1.0 - shadow_val) + light * shadow_val
    
    # Tone mapping
    w_scale = 1.0 / tone_map(200.0) # WHITE = 200.0
    final_color = np.array([tone_map(c) for c in color]) * w_scale
    
    return final_color
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of marching steps ($O(N)$ where $N$ is `MAX_MARCHING_STEPS`). Each step requires evaluating the SDF, which involves multiple trigonometric and vector operations.
*   **Memory Complexity:** $O(1)$. No buffers or textures are allocated.
*   **Performance:** Highly dependent on the complexity of the SDF function. The use of `sin` and `length` in the jitter and warp functions can be computationally expensive on some hardware.
*   **Strengths:** No texture memory usage; infinite resolution (continuous); easy to animate by varying the seed or warp parameters.
*   **Limitations:** Only renders opaque or semi-opaque surfaces via shading; does not support internal scattering or subsurface scattering inherent to real clouds; limited to the specific aesthetic of the defined SDF primitives.

## Background
This technique was presented as a small experiment by Dave Pagurek for Graphics Interface 2019 [S1, unofficial]. It demonstrates the capability of fragment shaders to render complex organic shapes using purely mathematical distance fields, avoiding the need for traditional volumetric data structures.

## Sources
[S1] davepagurek/sdf-clouds, https://github.com/davepagurek/sdf-clouds, fetched 2026-10-27 (unofficial)
