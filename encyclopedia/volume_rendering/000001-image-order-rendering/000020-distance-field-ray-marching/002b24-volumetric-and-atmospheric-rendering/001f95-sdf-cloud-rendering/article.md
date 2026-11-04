# SDF Cloud Rendering

Also called: Signed Distance Field Clouds

## Overview
SDF Cloud Rendering is a technique for rendering volumetric clouds using Signed Distance Fields (SDFs) via ray marching in a fragment shader, utilizing warped sphere primitives and smooth unions to create complex shapes [S1, unofficial]. It differs from [Ray Marching with Single Scattering](2057-ray-marching-with-single-scattering.md) and [Atmospheric Ray Marching](4874-atmospheric-ray-marching.md) by constructing the volume geometry explicitly through analytic distance functions rather than sampling a density field or solving the radiative transfer equation with phase functions [S1, unofficial].

## How it works
### Inputs and Outputs
- **Inputs**: Fragment coordinates (`gl_FragCoord`), screen resolution (`screenSize`), camera field of view (45.0 degrees), and a random seed (`seed`).
- **Outputs**: Final color in `gl_FragColor`.

### Data Structures
No complex data structures are used. The scene is defined by a single scalar field function `sceneSDF` composed of analytic primitives.

### Parameters
- `MAX_MARCHING_STEPS`: 100 [S1, unofficial].
- `MIN_DIST`: 0.0 [S1, unofficial].
- `MAX_DIST`: 100.0 [S1, unofficial].
- `EPSILON`: 0.0001 [S1, unofficial].
- `STEP_SCALE`: 0.67 [S1, unofficial].
- `EXPOSURE`: 1.0 [S1, unofficial].
- `WHITE`: 200.0 [S1, unofficial].
- `LIGHT`: `vec3(-10.0, 10.0, 3.0)` [S1, unofficial].
- `LIGHT_COLOR`: `vec3(100.0, 100.0, 100.0)` [S1, unofficial].
- `DARK_COLOR`: `vec3(5.0, 8.0, 20.0)` [S1, unofficial].

### Formulas
**Smooth Union** [S1, unofficial]:
$$
h = \text{clamp}\left(0.5 + 0.5 \frac{d_2 - d_1}{k}, 0.0, 1.0\right)
$$
$$
\text{result} = \text{mix}(d_2, d_1, h) - k \cdot h \cdot (1.0 - h)
$$
Where $d_1, d_2$ are distances and $k$ is the blending factor.

**Hable Tone Mapping** [S1, unofficial]:
$$
\text{result} = \frac{x(Ax + CB) + DE}{x(Ax + B) + DF} - \frac{E}{F}
$$
Where $A=0.15, B=0.50, C=0.10, D=0.20, E=0.02, F=0.30$.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def jitter(mix_amount, offset, seed):
    amount = 0.0
    scale = 1.0
    offset += seed
    for power in range(3):
        amount += sin((offset * 1234.0 + mix_amount) * scale) / scale
        scale *= 2.0
    amount *= 0.09
    return amount

def warp(v, scale):
    return v + scale * vec3(
        jitter(v.x, 0.0, seed),
        jitter(v.y, 0.2, seed),
        jitter(v.z, 0.5, seed)
    )

def sphere_sdf(sample_point, origin, radius):
    return length(sample_point - origin) - radius

def smooth_union(d1, d2, k):
    h = clamp(0.5 + 0.5 * (d2 - d1) / k, 0.0, 1.0)
    return mix(d2, d1, h) - k * h * (1.0 - h)

def scene_sdf(sample_point):
    warped = warp(sample_point, 1.5)
    d1 = sphere_sdf(warped, vec3(0.5, 0.0, 0.0), 1.0)
    d2 = sphere_sdf(warped, vec3(-0.5, -0.4, 0.2), 0.5)
    smoothed = smooth_union(d1, d2, 0.4)
    d3 = sphere_sdf(warped, vec3(-0.8, 0.55, 0.2), 0.3)
    return min(smoothed, d3)

def ray_direction(field_of_view, size, frag_coord):
    xy = frag_coord - size / 2.0
    z = size.y / tan(radians(field_of_view) / 2.0)
    return normalize(vec3(xy, -z))

def ray_march_distance(eye, marching_direction, start, end):
    depth = start
    for i in range(MAX_MARCHING_STEPS):
        dist = scene_sdf(eye + depth * marching_direction)
        if dist < EPSILON:
            return depth
        depth += dist * STEP_SCALE
        if depth >= end:
            return end
    return end

def calculate_shadow(origin, dir):
    k = 2.0
    t = 100.0 * EPSILON
    res = 1.0
    for i in range(MAX_MARCHING_STEPS):
        h = scene_sdf(origin + t * dir)
        if h < EPSILON:
            return 0.0
        t += h * STEP_SCALE
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

def render():
    dir = ray_direction(45.0, screenSize, gl_FragCoord.xy)
    eye = vec3(0.0, 0.0, 8.0)
    dist = ray_march_distance(eye, dir, MIN_DIST, MAX_DIST)

    if dist > MAX_DIST - EPSILON:
        d = dot(dir, normalize(vec3(0.0, -1.0, 0.0)))
        d *= 6.0
        d = d / 2.0 + 0.5
        d = max(0.0, min(1.0, d))
        return mix(vec3(4.0, 40.0, 100.0), vec3(20.0, 100.0, 100.0), d)

    intersection = eye + dist * dir
    to_light = normalize(vec3(-10.0, 10.0, 3.0) - intersection)
    shadow_factor = calculate_shadow(intersection, to_light)
    return mix(vec3(5.0, 8.0, 20.0), vec3(100.0, 100.0, 100.0), shadow_factor)

def main():
    color = render()
    w_scale = 1.0 / tone_map(EXPOSURE * WHITE)
    gl_FragColor = vec4(tone_map_vec(EXPOSURE * color) * w_scale, 1.0)
```

### Edge Cases and Pitfalls
- **Precision**: The jitter function uses `sin` and division by `scale`. If `scale` becomes too large or `mix_amount` is extreme, precision loss may occur.
- **Boundaries**: The ray marcher returns `end` if `dist >= end` or if the loop completes. This handles rays that miss the cloud entirely.
- **Degenerate Input**: If `screenSize` is zero or invalid, `ray_direction` will fail. [G] Ensure `screenSize` is positive.

## Operating characteristics
- **Time Complexity**: $O(N \cdot M)$ where $N$ is the number of pixels and $M$ is `MAX_MARCHING_STEPS` (100). Each step involves a warp and SDF evaluation.
- **Memory Complexity**: $O(1)$ per pixel. No buffers are allocated.
- **Performance**: Runs in real-time on modern GPUs as a single pass fragment shader.
- **Strengths**: Simple implementation, no texture lookups, analytic geometry.
- **Limitations**: Limited to specific shapes (warped spheres), no true volumetric scattering, single shadow ray per pixel.

## Background
The method was created by Dave Pagurek as a small experiment in rendering with signed distance fields, presented at Graphics Interface 2019 [S1, unofficial]. It demonstrates the use of SDFs for organic shapes like clouds.

## Sources
[S1] davepagurek/sdf-clouds, https://github.com/davepagurek/sdf-clouds, fetched 2026-10-27 (unofficial)
