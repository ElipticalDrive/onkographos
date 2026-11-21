# SDF Cloud Rendering

Also called: Warped Sphere SDF Clouds

## Overview
SDF Cloud Rendering is a fragment-shader-based technique for rendering volumetric cloud-like structures using Signed Distance Fields (SDFs) and ray marching [S1, unofficial]. It solves the problem of generating complex, organic volumetric shapes without explicit mesh data by composing simple geometric primitives (spheres) via smooth unions and fractal warping [S1, unofficial]. It differs from standard ray marching by relying on procedural noise-based warping of the coordinate space to create "cloudy" topology, rather than sampling a 3D texture or voxel grid [S1, unofficial].

## How it works
The method operates in a single-pass fragment shader [S1, unofficial]. It defines a scene using a hierarchy of warped spheres, marches a ray to find the intersection with the implicit surface, and computes lighting via a shadow ray [S1, unofficial].

### Inputs and Outputs
*   **Inputs**: Fragment coordinates (`gl_FragCoord`), screen resolution (`screenSize`), camera field of view (45.0 degrees), and a random seed (`seed`) [S1, unofficial].
*   **Outputs**: A final color vector (`gl_FragColor`) in the range [0, 1] [S1, unofficial].

### Data Structures
No complex data structures are used. The state is maintained via local variables in the shader [S1, unofficial]. Constants define the scene bounds and marching parameters [S1, unofficial].

### Parameters
*   `MAX_MARCHING_STEPS`: 100 [S1, unofficial].
*   `MIN_DIST`: 0.0 [S1, unofficial].
*   `MAX_DIST`: 100.0 [S1, unofficial].
*   `EPSILON`: 0.0001 [S1, unofficial].
*   `STEP_SCALE`: 0.67 [S1, unofficial].
*   `EXPOSURE`: 1.0 [S1, unofficial].
*   `WHITE`: 200.0 [S1, unofficial].
*   `LIGHT_POS`: `vec3(-10.0, 10.0, 3.0)` [S1, unofficial].
*   `LIGHT_COLOR`: `vec3(100.0, 100.0, 100.0)` [S1, unofficial].
*   `DARK_COLOR`: `vec3(5.0, 8.0, 20.0)` [S1, unofficial].

### Formulas

**1. Sphere SDF**
The distance from a point $p$ to a sphere centered at $c$ with radius $r$:
$$ d_{sphere}(p) = \|p - c\| - r $$

**2. Smooth Union**
Blends two SDFs $d_1$ and $d_2$ with a blending factor $k$:
$$ h = \text{clamp}\left(0.5 + 0.5 \frac{d_2 - d_1}{k}, 0.0, 1.0\right) $$
$$ d_{union} = \text{mix}(d_2, d_1, h) - k \cdot h \cdot (1.0 - h) $$

**3. Jitter (Fractal Noise)**
Generates a pseudo-random offset based on a seed and coordinate:
$$ \text{jitter}(mixAmount, offset) = 0.09 \sum_{i=0}^{2} \frac{\sin((offset \cdot 1234.0 + mixAmount) \cdot 2^i)}{2^i} $$
*(Note: The source code uses `scale` starting at 1.0 and multiplying by 2.0 in the loop, which corresponds to $2^i$)* [S1, unofficial].

**4. Warp**
Distorts the input vector $v$ using jittered offsets:
$$ \text{warp}(v, scale) = v + scale \cdot \text{vec3}(\text{jitter}(v.x, 0.0), \text{jitter}(v.y, 0.2), \text{jitter}(v.z, 0.5)) $$

**5. Hable Tone Mapping**
Maps linear color $x$ to [0, 1]:
$$ \text{toneMap}(x) = \frac{x(Ax + CB) + DE}{x(Ax + B) + DF} - \frac{E}{F} $$
Where $A=0.15, B=0.50, C=0.10, D=0.20, E=0.02, F=0.30$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants
MAX_MARCHING_STEPS = 100
MIN_DIST = 0.0
MAX_DIST = 100.0
EPSILON = 0.0001
STEP_SCALE = 0.67
LIGHT_POS = vec3(-10.0, 10.0, 3.0)
LIGHT_COLOR = vec3(100.0, 100.0, 100.0)
DARK_COLOR = vec3(5.0, 8.0, 20.0)
EXPOSURE = 1.0
WHITE = 200.0

def radians(deg):
    return deg * math.pi / 180.0

def sphere_sdf(p, center, radius):
    return length(p - center) - radius

def jitter(mix_amount, offset, seed):
    amount = 0.0
    scale = 1.0
    offset += seed
    for power in range(0, 3):
        amount += sin((offset * 1234.0 + mix_amount) * scale) / scale
        scale *= 2.0
    return amount * 0.09

def warp(v, scale, seed):
    return v + scale * vec3(
        jitter(v.x, 0.0, seed),
        jitter(v.y, 0.2, seed),
        jitter(v.z, 0.5, seed)
    )

def smooth_union(d1, d2, k):
    h = clamp(0.5 + 0.5 * (d2 - d1) / k, 0.0, 1.0)
    return mix(d2, d1, h) - k * h * (1.0 - h)

def scene_sdf(p, seed):
    warped = warp(p, 1.5, seed)
    d1 = sphere_sdf(warped, vec3(0.5, 0.0, 0.0), 1.0)
    d2 = sphere_sdf(warped, vec3(-0.5, -0.4, 0.2), 0.5)
    smoothed = smooth_union(d1, d2, 0.4)
    d3 = sphere_sdf(warped, vec3(-0.8, 0.55, 0.2), 0.3)
    return min(smoothed, d3)

def ray_direction(frag_coord, screen_size, fov):
    xy = frag_coord - screen_size / 2.0
    z = screen_size.y / tan(radians(fov) / 2.0)
    return normalize(vec3(xy, -z))

def ray_march(eye, direction, start, end, seed):
    depth = start
    for i in range(0, MAX_MARCHING_STEPS):
        dist = scene_sdf(eye + depth * direction, seed)
        if dist < EPSILON:
            return depth
        # Clamp step to EPSILON to prevent zero/negative steps causing divergence or infinite loops
        depth += max(EPSILON, dist) * STEP_SCALE
        if depth >= end:
            return end
    return end

def shadow(origin, direction, seed):
    k = 2.0
    t = 100.0 * EPSILON
    res = 1.0
    for i in range(0, MAX_MARCHING_STEPS):
        h = scene_sdf(origin + t * direction, seed)
        if h < EPSILON:
            return 0.0
        # Clamp step to EPSILON to prevent t from decreasing or crossing zero
        t += max(EPSILON, h) * STEP_SCALE
        res = min(res, k * h / t)
    return res

def tone_map(x):
    A = 0.15
    B = 0.50
    C = 0.10
    D = 0.20
    E = 0.02
    F = 0.30
    return ((x * (A * x + C * B) + D * E) / (x * (A * x + B) + D * F)) - (E / F)

def render(frag_coord, screen_size, seed):
    dir = ray_direction(frag_coord, screen_size, 45.0)
    eye = vec3(0.0, 0.0, 8.0)
    
    dist = ray_march(eye, dir, MIN_DIST, MAX_DIST, seed)
    
    if dist > MAX_DIST - EPSILON:
        # Sky gradient
        d = dot(dir, normalize(vec3(0.0, -1.0, 0.0)))
        d = d * 6.0
        d = d / 2.0 + 0.5
        d = max(0.0, min(1.0, d))
        return mix(vec3(4.0, 40.0, 100.0), vec3(20.0, 100.0, 100.0), d)
    
    # Cloud shading
    intersection = eye + dist * dir
    to_light = normalize(LIGHT_POS - intersection)
    shadow_val = shadow(intersection, to_light, seed)
    
    return mix(DARK_COLOR, LIGHT_COLOR, shadow_val)

def main(frag_coord, screen_size, seed):
    color = render(frag_coord, screen_size, seed)
    w_scale = 1.0 / tone_map(EXPOSURE * WHITE)
    final_color = tone_map(EXPOSURE * color) * w_scale
    return vec4(final_color, 1.0)
```

### Edge Cases and Pitfalls
*   **Precision**: The source shader uses `mediump` float precision [S1, unofficial]. Lower precision may lead to jittering artifacts or incorrect SDF evaluations in high-resolution renders.
*   **Seed Dependency**: The visual appearance is deterministic based on the `seed` parameter. Changing the seed alters the cloud shape significantly due to the fractal warping [S1, unofficial].
*   **Step Scaling**: `STEP_SCALE` is set to 0.67 to prevent overshooting caused by coordinate warping [S1, unofficial]. Steps are clamped to `EPSILON` in the implementation to guarantee convergence and avoid division-by-zero or infinite loops in shadow rays.

## Subtopics

### Ray Marching with Single Scattering
Computes lighting by sampling volumetric density along the ray and applying a single-scattering phase function [G].

### Volume Ray Marching with Temporal Accumulation
Renders volumetric data over multiple frames and blends results to reduce noise and stabilize animated sequences [G].

### Atmospheric Ray Marching
Simulates large-scale light transport in planetary atmospheres using phase functions and density profiles [G].

### X-Ray Visualization
Accumulates density and transparency along rays to produce projection-like images of internal structures [G].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Performance**: Not given in the sources.
*   **Strengths**: Not given in the sources.
*   **Limitations**: Not given in the sources.

## Background
SDF Cloud Rendering was developed by Dave Pagurek as a small experiment for Graphics Interface 2019 [S1, unofficial]. It demonstrates the capability of Signed Distance Fields to create organic, volumetric-looking structures through procedural warping and smooth unions, rather than traditional mesh or voxel representations [S1, unofficial].

## Sources
[S1] davepagurek/sdf-clouds, https://github.com/davepagurek/sdf-clouds, fetched 2026-10-27 (unofficial)
