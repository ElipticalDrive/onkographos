# SDF Cloud Rendering

Also called: Signed Distance Field Clouds

## Overview
SDF Cloud Rendering is a technique for rendering volumetric clouds using Signed Distance Fields (SDFs) via ray marching in a fragment shader, utilizing warped sphere primitives and smooth unions to create complex shapes [S1, unofficial]. It differs from standard volumetric methods by representing the cloud volume as an implicit surface defined by the distance to deformed geometric primitives, rather than a sampled density texture or noise function. This approach allows for sharp, well-defined cloud boundaries and efficient shadow casting without the need for expensive volumetric integration, at the cost of limited internal density variation and a stylized, solid appearance.

## How it works
The method constructs a scene SDF by warping coordinates with a procedural jitter function and combining sphere SDFs using smooth unions. Rays are marched through this field to find intersections, and shading is performed using a simplified shadow map approximation.

### Inputs and Outputs
*   **Inputs**: Fragment coordinates (`gl_FragCoord`), screen size (`screenSize`), camera field of view (`45.0`), and a random seed (`seed`).
*   **Outputs**: Final pixel color (`gl_FragColor`) in the range [0, 1].

### Data Structures
No explicit data structures are used. All geometry is defined procedurally via functions.

### Parameters
*   `MAX_MARCHING_STEPS`: Integer constant, value `100` [S1, unofficial].
*   `MIN_DIST`: Float constant, value `0.0` [S1, unofficial].
*   `MAX_DIST`: Float constant, value `100.0` [S1, unofficial].
*   `EPSILON`: Float constant, value `0.0001` [S1, unofficial].
*   `STEP_SCALE`: Float constant, value `0.67` [S1, unofficial].
*   `EXPOSURE`: Float constant, value `1.0` [S1, unofficial].
*   `WHITE`: Float constant, value `200.0` [S1, unofficial].
*   `LIGHT`: Constant vector, value `vec3(-10.0, 10.0, 3.0)` [S1, unofficial].
*   `DARK`: Constant vector, value `vec3(5.0, 8.0, 20.0)` [S1, unofficial].
*   `LIGHT_COLOR`: Constant vector, value `vec3(100.0, 100.0, 100.0)` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def jitter(mix_amount: float, offset: float, seed: float) -> float:
    """
    Generates a pseudo-random jitter value based on input coordinates and seed.
    """
    amount = 0.0
    scale = 1.0
    offset += seed
    for power in range(0, 3):
        amount += sin((offset * 1234.0 + mix_amount) * scale) / scale
        scale *= 2.0
    amount *= 0.09
    return amount

def warp(v: vec3, scale: float, seed: float) -> vec3:
    """
    Warps a 3D vector using jitter functions on each axis with different offsets.
    """
    jx = jitter(v.x, 0.0, seed)
    jy = jitter(v.y, 0.2, seed)
    jz = jitter(v.z, 0.5, seed)
    return v + scale * vec3(jx, jy, jz)

def smooth_union(d1: float, d2: float, k: float) -> float:
    """
    Smoothly blends two SDF values d1 and d2 using blending factor k.
    """
    h = clamp(0.5 + 0.5 * (d2 - d1) / k, 0.0, 1.0)
    return mix(d2, d1, h) - k * h * (1.0 - h)

def sphere_sdf(sample_point: vec3, origin: vec3, radius: float) -> float:
    """
    Computes the signed distance from sample_point to a sphere centered at origin with radius.
    """
    return length(sample_point - origin) - radius

def scene_sdf(sample_point: vec3, seed: float) -> float:
    """
    Constructs the scene SDF by warping coordinates and combining sphere primitives.
    """
    warped = warp(sample_point, 1.5, seed)
    
    # First cloud cluster
    d1 = sphere_sdf(warped, vec3(0.5, 0.0, 0.0), 1.0)
    d2 = sphere_sdf(warped, vec3(-0.5, -0.4, 0.2), 0.5)
    smoothed = smooth_union(d1, d2, 0.4)
    
    # Second cloud cluster
    d3 = sphere_sdf(warped, vec3(-0.8, 0.55, 0.2), 0.3)
    
    return min(smoothed, d3)

def ray_direction(field_of_view: float, size: vec2, frag_coord: vec2) -> vec3:
    """
    Calculates the ray direction for a given fragment coordinate.
    """
    xy = frag_coord - size / 2.0
    z = size.y / tan(radians(field_of_view) / 2.0)
    return normalize(vec3(xy, -z))

def ray_march_intersection(eye: vec3, marching_direction: vec3, start: float, end: float, seed: float) -> float:
    """
    Marches a ray through the scene SDF to find the intersection distance.
    """
    depth = start
    for i in range(0, 100): # MAX_MARCHING_STEPS
        dist = scene_sdf(eye + depth * marching_direction, seed)
        if dist < 0.0001: # EPSILON
            return depth
        depth += dist * 0.67 # STEP_SCALE
        if depth >= end:
            return end
    return end

def calculate_shadow(origin: vec3, dir: vec3, seed: float) -> float:
    """
    Calculates shadow factor by marching along the light direction.
    """
    k = 2.0
    t = 100.0 * 0.0001 # 100.0 * EPSILON
    res = 1.0
    for i in range(0, 100): # MAX_MARCHING_STEPS
        h = scene_sdf(origin + t * dir, seed)
        if h < 0.0001: # EPSILON
            return 0.0
        t += h * 0.67 # STEP_SCALE
        res = min(res, k * h / t)
    return res

def tone_map(x: float) -> float:
    """
    Applies Hable's Reinhard-like tone mapping function.
    """
    A = 0.15
    B = 0.50
    C = 0.10
    D = 0.20
    E = 0.02
    F = 0.30
    return ((x * (A * x + C * B) + D * E) / (x * (A * x + B) + D * F)) - E / F

def main(frag_coord: vec2, screen_size: vec2, seed: float) -> vec4:
    """
    Main render loop for a single fragment.
    """
    dir = ray_direction(45.0, screen_size, frag_coord)
    eye = vec3(0.0, 0.0, 8.0)
    
    dist = ray_march_intersection(eye, dir, 0.0, 100.0, seed)
    
    if dist > 100.0 - 0.0001: # MAX_DIST - EPSILON
        # Render sky gradient
        d = dot(dir, normalize(vec3(0.0, -1.0, 0.0)))
        d *= 6.0
        d = d / 2.0 + 0.5
        d = max(0.0, min(1.0, d))
        color = mix(vec3(4.0, 40.0, 100.0), vec3(20.0, 100.0, 100.0), d)
    else:
        # Render cloud
        intersection = eye + dist * dir
        to_light = normalize(vec3(-10.0, 10.0, 3.0) - intersection)
        shadow_factor = calculate_shadow(intersection, to_light, seed)
        color = mix(vec3(5.0, 8.0, 20.0), vec3(100.0, 100.0, 100.0), shadow_factor)
    
    # Tone mapping and exposure
    w_scale = 1.0 / tone_map(1.0 * 200.0) # EXPOSURE * WHITE
    final_color = tone_map(1.0 * color) * w_scale # EXPOSURE * color
    
    return vec4(final_color, 1.0)
```

### Formulas
*   **Smooth Union**:
    $$ h = \text{clamp}(0.5 + 0.5 \cdot \frac{d_2 - d_1}{k}, 0.0, 1.0) $$
    $$ \text{result} = \text{mix}(d_2, d_1, h) - k \cdot h \cdot (1.0 - h) $$
    Where $d_1, d_2$ are SDF values and $k$ is the blending factor [S1, unofficial].

*   **Hable Tone Mapping**:
    $$ \text{result} = \frac{x(Ax + CB) + DE}{x(Ax + B) + DF} - \frac{E}{F} $$
    Where $A=0.15, B=0.50, C=0.10, D=0.20, E=0.02, F=0.30$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: The use of `float` precision in fragment shaders may cause jittering or artifacts if the scene scale is very large or very small. The `EPSILON` threshold must be tuned to the scene scale to avoid false positives or missed intersections [S1, unofficial].
*   **Step Size**: `STEP_SCALE` of 0.67 is used to prevent overshooting the surface. If the scale is too high, the ray may jump over thin cloud features. If too low, performance degrades [S1, unofficial].
*   **Shadow Artifacts**: The shadow calculation uses a simplified marching approach. It may produce aliasing or incorrect shadows if the light direction is not aligned with the grid or if the cloud geometry is very thin [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the maximum number of marching steps (100). Each step involves a warp and SDF evaluation, which is constant time but involves multiple trigonometric and arithmetic operations.
*   **Memory Complexity**: $O(1)$. No textures or large data structures are stored. All geometry is procedural.
*   **Performance**: Fast on modern GPUs due to the lack of texture lookups and simple arithmetic. However, the jitter function involves multiple `sin` calls, which can be expensive if not optimized.
*   **Strengths**: No memory bandwidth bound; easy to implement in a single shader; produces clean, stylized cloud shapes; efficient shadow casting.
*   **Limitations**: Limited to solid, opaque or semi-opaque shapes; no internal density variation or subsurface scattering; requires manual tuning of parameters for different cloud types; not suitable for realistic, volumetrically accurate clouds.

## Background
SDF Cloud Rendering was created by Dave Pagurek as a small experiment in rendering with signed distance fields, presented at Graphics Interface 2019 [S1, unofficial]. It demonstrates the capability of SDFs to create complex, organic shapes through warping and boolean operations, rather than relying on traditional volumetric noise functions.

## Sources
[S1] davepagurek/sdf-clouds, https://github.com/davepagurek/sdf-clouds, fetched 2026-10-27 (unofficial)
