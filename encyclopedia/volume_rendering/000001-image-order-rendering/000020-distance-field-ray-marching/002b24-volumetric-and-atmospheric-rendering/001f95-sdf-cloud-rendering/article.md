# SDF Cloud Rendering

Also called: Warped Sphere SDF Clouds

## Overview
SDF Cloud Rendering is a fragment-shader-based technique for rendering volumetric cloud-like structures using Signed Distance Fields (SDFs) and ray marching [S1, unofficial]. It constructs complex, organic shapes by warping simple sphere primitives and blending them via smooth unions, avoiding the need for explicit voxel grids or texture maps [S1, unofficial]. Unlike traditional volumetric methods that integrate density along a ray, this method treats the cloud as a solid, opaque (or semi-opaque via shadowing) boundary defined by the distance field, relying on ray-marching intersection and shadow casting for shading [S1, unofficial]. It is distinct from [Ray Marching with Single Scattering](2057-ray-marching-with-single-scattering.md) by its specific use of procedural warping for shape generation and simplified lighting model, and from [Atmospheric Ray Marching](4874-atmospheric-ray-marching.md) by its focus on localized, structured cloud volumes rather than global atmospheric scattering.

## How it works
The method renders a scene by casting rays from a camera through a fragment shader, marching them through a procedurally defined SDF scene to find intersections, and shading those intersections based on a shadow map derived from the same SDF.

### Inputs and Outputs
*   **Inputs:** Fragment coordinates (`gl_FragCoord`), screen resolution (`screenSize`), camera field of view (fixed at 45.0 degrees in the reference implementation), and a random seed for jitter generation.
*   **Outputs:** A final RGBA color for each pixel, tone-mapped to the [0, 1] range.

### Data Structures
The method does not use external data structures. All geometry is defined implicitly by the `sceneSDF` function. The only state carried during ray marching is the current depth $t$ along the ray.

### Parameters
*   `MAX_MARCHING_STEPS`: Integer, 100. Maximum iterations for ray marching loops.
*   `MIN_DIST`: Float, 0.0. Starting distance for ray marching.
*   `MAX_DIST`: Float, 100.0. Maximum ray marching distance.
*   `EPSILON`: Float, 0.0001. Convergence threshold for intersection detection.
*   `STEP_SCALE`: Float, 0.67. Safety factor to prevent overshooting the surface during ray steps.
*   `EXPOSURE`: Float, 1.0. Global exposure multiplier.
*   `WHITE`: Float, 200.0. Reference white point for tone mapping scaling.
*   `LIGHT`: Vector3, `(-10.0, 10.0, 3.0)`. Position of the light source.
*   `DARK`: Vector3, `(5.0, 8.0, 20.0)`. Shadow color.
*   `LIGHT_COLOR`: Vector3, `(100.0, 100.0, 100.0)`. Light color.

### Formulas

**1. Jitter Function**
Generates a pseudo-random offset based on a seed and coordinate.
$$
\text{jitter}(mixAmount, offset) = 0.09 \times \sum_{p=0}^{2} \frac{\sin((offset \times 1234.0 + mixAmount) \times 2^p)}{2^p}
$$
Where $offset$ is initialized with the global seed.

**2. Warp Function**
Distorts a 3D point using jittered offsets along each axis.
$$
\text{warp}(\mathbf{v}, s) = \mathbf{v} + s \times \begin{bmatrix} \text{jitter}(v_x, 0.0) \\ \text{jitter}(v_y, 0.2) \\ \text{jitter}(v_z, 0.5) \end{bmatrix}
$$

**3. Smooth Union**
Blends two SDF values $d_1$ and $d_2$ with a blending factor $k$.
$$
h = \text{clamp}\left(0.5 + 0.5 \frac{d_2 - d_1}{k}, 0.0, 1.0\right)
$$
$$
\text{smoothUnion}(d_1, d_2, k) = \text{mix}(d_2, d_1, h) - k \cdot h \cdot (1.0 - h)
$$

**4. Sphere SDF**
Distance from point $\mathbf{p}$ to sphere center $\mathbf{c}$ with radius $r$.
$$
\text{sphereSDF}(\mathbf{p}, \mathbf{c}, r) = \|\mathbf{p} - \mathbf{c}\| - r
$$

**5. Scene SDF**
Constructs the cloud volume by warping coordinates and combining spheres.
$$
\mathbf{w} = \text{warp}(\mathbf{p}, 1.5)
$$
$$
d_1 = \text{sphereSDF}(\mathbf{w}, (0.5, 0.0, 0.0), 1.0)
$$
$$
d_2 = \text{sphereSDF}(\mathbf{w}, (-0.5, -0.4, 0.2), 0.5)
$$
$$
d_{smooth} = \text{smoothUnion}(d_1, d_2, 0.4)
$$
$$
d_3 = \text{sphereSDF}(\mathbf{w}, (-0.8, 0.55, 0.2), 0.3)
$$
$$
\text{sceneSDF}(\mathbf{p}) = \min(d_{smooth}, d_3)
$$

**6. Ray Direction**
Calculates the view ray direction from fragment coordinates.
$$
\mathbf{xy} = \text{fragCoord} - \frac{\text{size}}{2.0}
$$
$$
z = \frac{\text{size}_y}{\tan(\frac{\text{FOV}}{2})}
$$
$$
\mathbf{dir} = \text{normalize}(\mathbf{xy}, -z)
$$

**7. Ray Marching Intersection**
Finds the distance to the nearest surface.
$$
t = \text{start}
$$
$$
\text{for } i = 0 \text{ to } \text{MAX\_MARCHING\_STEPS}-1:
$$
$$
\quad \text{dist} = \text{sceneSDF}(\mathbf{eye} + t \times \mathbf{dir})
$$
$$
\quad \text{if } \text{dist} < \text{EPSILON}: \text{return } t
$$
$$
\quad t += \text{dist} \times \text{STEP\_SCALE}
$$
$$
\quad \text{if } t \ge \text{end}: \text{return } \text{end}
$$
$$
\text{return } \text{end}
$$

**8. Shadow Calculation**
Estimates shadowing by marching along the light direction and accumulating a soft shadow factor.
$$
k = 2.0
$$
$$
t = 100.0 \times \text{EPSILON}
$$
$$
\text{res} = 1.0
$$
$$
\text{for } i = 0 \text{ to } \text{MAX\_MARCHING\_STEPS}-1:
$$
$$
\quad h = \text{sceneSDF}(\mathbf{origin} + t \times \mathbf{dir})
$$
$$
\quad \text{if } h < \text{EPSILON}: \text{return } 0.0
$$
$$
\quad t += h \times \text{STEP\_SCALE}
$$
$$
\quad \text{res} = \min(\text{res}, k \frac{h}{t})
$$
$$
\text{return } \text{res}
$$

**9. Hable Tone Mapping**
Maps linear color to [0, 1].
$$
\text{toneMap}(x) = \frac{x(Ax + CB) + DE}{x(Ax + B) + DF} - \frac{E}{F}
$$
With $A=0.15, B=0.50, C=0.10, D=0.20, E=0.02, F=0.30$.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

# Constants
MAX_MARCHING_STEPS = 100
MIN_DIST = 0.0
MAX_DIST = 100.0
EPSILON = 0.0001
WHITE = 200.0
LIGHT_POS = np.array([-10.0, 10.0, 3.0])
STEP_SCALE = 0.67
EXPOSURE = 1.0
DARK_COLOR = np.array([5.0, 8.0, 20.0])
LIGHT_COLOR = np.array([100.0, 100.0, 100.0])

def jitter(mix_amount, offset, seed):
    amount = 0.0
    scale = 1.0
    offset += seed
    for power in range(3):
        amount += math.sin((offset * 1234.0 + mix_amount) * scale) / scale
        scale *= 2.0
    amount *= 0.09
    return amount

def warp(v, scale, seed):
    return v + scale * np.array([
        jitter(v[0], 0.0, seed),
        jitter(v[1], 0.2, seed),
        jitter(v[2], 0.5, seed)
    ])

def smooth_union(d1, d2, k):
    h = np.clip(0.5 + 0.5 * (d2 - d1) / k, 0.0, 1.0)
    return np.where(h > 0.5, d1, d2) - k * h * (1.0 - h)

def sphere_sdf(sample_point, origin, radius):
    return np.linalg.norm(sample_point - origin) - radius

def scene_sdf(sample_point, seed):
    warped = warp(sample_point, 1.5, seed)
    d1 = sphere_sdf(warped, np.array([0.5, 0.0, 0.0]), 1.0)
    d2 = sphere_sdf(warped, np.array([-0.5, -0.4, 0.2]), 0.5)
    smoothed = smooth_union(d1, d2, 0.4)
    d3 = sphere_sdf(warped, np.array([-0.8, 0.55, 0.2]), 0.3)
    return min(smoothed, d3)

def ray_direction(field_of_view, size, frag_coord):
    xy = frag_coord - size / 2.0
    z = size[1] / math.tan(math.radians(field_of_view) / 2.0)
    dir_vec = np.array([xy[0], xy[1], -z])
    return dir_vec / np.linalg.norm(dir_vec)

def ray_march_intersection(eye, marching_direction, start, end, seed):
    depth = start
    for i in range(MAX_MARCHING_STEPS):
        dist = scene_sdf(eye + depth * marching_direction, seed)
        if dist < EPSILON:
            return depth
        depth += dist * STEP_SCALE
        if depth >= end:
            return end
    return end

def calculate_shadow(origin, dir, seed):
    k = 2.0
    t = 100.0 * EPSILON
    res = 1.0
    for i in range(MAX_MARCHING_STEPS):
        h = scene_sdf(origin + t * dir, seed)
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

def render(frag_coord, screen_size, seed):
    fov = 45.0
    dir = ray_direction(fov, screen_size, frag_coord)
    eye = np.array([0.0, 0.0, 8.0])
    
    dist = ray_march_intersection(eye, dir, MIN_DIST, MAX_DIST, seed)
    
    if dist > MAX_DIST - EPSILON:
        # Sky gradient
        d = np.dot(dir, np.array([0.0, -1.0, 0.0]))
        d *= 6.0
        d = d / 2.0 + 0.5
        d = np.clip(d, 0.0, 1.0)
        return np.array([4.0, 40.0, 100.0]) * (1.0 - d) + np.array([20.0, 100.0, 100.0]) * d
    
    # Cloud shading
    intersection = eye + dist * dir
    to_light = LIGHT_POS - intersection
    to_light = to_light / np.linalg.norm(to_light)
    
    shadow_val = calculate_shadow(intersection, to_light, seed)
    color = np.where(shadow_val > 0.5, LIGHT_COLOR, DARK_COLOR) # Simplified mix logic from source
    # Source uses mix(DARK, LIGHT, shadow), which is linear interpolation
    color = DARK_COLOR * (1.0 - shadow_val) + LIGHT_COLOR * shadow_val
    
    return color

def main_render(frag_coord, screen_size, seed):
    color = render(frag_coord, screen_size, seed)
    w_scale = 1.0 / tone_map(EXPOSURE * WHITE)
    final_color = tone_map(EXPOSURE * color) * w_scale
    return np.clip(final_color, 0.0, 1.0)
```

### Edge Cases and Pitfalls
*   **Overshooting:** The `STEP_SCALE` of 0.67 is critical. If set to 1.0 or higher, the ray may jump over thin surfaces or the boundary of the SDF, causing artifacts or missed intersections [S1, unofficial].
*   **Precision:** The use of `EPSILON` (0.0001) defines the surface. If the ray starts too close to the surface or the step size is too large relative to the geometry, the intersection may be missed or jittery [S1, unofficial].
*   **Shadow Artifacts:** The shadow calculation uses a similar ray-marching loop. If `MAX_MARCHING_STEPS` is too low, the shadow may be inaccurate or fail to detect occlusion correctly [S1, unofficial].
*   **Warping Complexity:** The warping function is non-linear and can create self-intersections or complex topology that is difficult to ray-march accurately without sufficient steps [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is `MAX_MARCHING_STEPS` (100). Each pixel requires one ray march to the surface and one shadow ray march.
*   **Memory Complexity:** $O(1)$. No external data structures are used; all geometry is procedural.
*   **Performance:** The method is computationally intensive per pixel due to the trigonometric functions in the jitter/warp and the double ray marching pass (intersection + shadow). It is suitable for real-time rendering on modern GPUs if resolution is limited or steps are reduced, but may struggle at high resolutions without optimization [S1, unofficial].
*   **Strengths:** No need for texture maps or voxel grids; easy to modify shape by changing sphere parameters; self-contained in a single shader.
*   **Limitations:** Limited to opaque or semi-opaque shading via shadowing; no true volumetric scattering (translucency, subsurface scattering) is implemented in the basic form; resolution-dependent artifacts due to discrete ray marching.

## Background
SDF Cloud Rendering was developed by Dave Pagurek as a small experiment for Graphics Interface 2019 [S1, unofficial]. It demonstrates the capability of Signed Distance Fields to create complex, organic volumetric shapes without explicit geometry, leveraging the power of fragment shaders for real-time rendering. The technique is part of the broader field of ray marching in computer graphics, popularized by tools like ShaderToy.

## Sources
[S1] davepagurek/sdf-clouds, https://github.com/davepagurek/sdf-clouds, fetched 2026-10-27 (unofficial)
