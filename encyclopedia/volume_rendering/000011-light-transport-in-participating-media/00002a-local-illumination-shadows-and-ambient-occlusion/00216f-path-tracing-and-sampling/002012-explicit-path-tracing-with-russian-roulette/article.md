# Explicit Path Tracing with Russian Roulette

Also called: Implicit Path Tracing, Monte Carlo Path Tracing, Path Tracing with Russian Roulette and Material Sampling

## Overview
Explicit Path Tracing is a rendering algorithm that simulates light transport by recursively tracing rays from the camera into the scene, accumulating color contributions from surface interactions until a maximum bounce count is reached or a termination condition is met [S2, unofficial]. It explicitly manages recursion depth and uses Russian Roulette to terminate paths probabilistically based on surface albedo [S1, unofficial]. Unlike its sibling [Path Tracing with Discrete Event Sampling](5168-path-tracing-with-discrete-event-sampling.md), which replaces continuous termination with discrete probabilities for sparse media, this method relies on fixed depth limits and probabilistic survival based on material reflectance [S1, unofficial].

## How it works
**Inputs:**
- `scene`: A scene structure containing a `camera`, a `map` (SDF map implementing intersection and normal queries), and a `background_color` function [S2, unofficial].
- `width`, `height`: Integer resolution of the image [S2, unofficial].
- `sample_count`: Number of samples per pixel [S2, unofficial].

**Outputs:**
- `pixels`: A 2D array of `Vec3` colors, representing the final image.

**Data Structures:**
- `Scene`:
  - `camera`: `Camera` struct [S2, unofficial].
  - `map`: `Box<dyn SdfMap>` (interface for ray intersection and normal calculation) [S2, unofficial].
  - `background_color`: `Box<dyn Fn(Vec3) -> Vec3 + Sync>` (function evaluating background color for a direction) [S2, unofficial].
- `Camera`:
  - `position`: `Vec3` (camera origin) [S2, unofficial].
  - `left`, `forward`, `up`: `Vec3` (basis vectors) [S2, unofficial].
  - `focal_length`: `f32` [S2, unofficial].
  - `aspect_ratio`: `f32` [S2, unofficial].
  - `focus_dist`: `f32` [S2, unofficial].
  - `aperture`: `f32` [S2, unofficial].
- `HitInfo` (returned by `map.ray_intersection`):
  - `position`: `Vec3` [S2, unofficial].
  - `material`: `Material` enum [S2, unofficial].
- `Material`:
  - `Lambertian { color: Vec3 }` [S2, unofficial].
  - `Emissive { color: Vec3 }` [S2, unofficial].
  - `Specular { color: Vec3, specularity: f32, fuzziness: f32 }` [S2, unofficial].

**Parameters:**
- `MAX_BOUNCES`: `i32 = 4`. Maximum bounces before termination [S2, unofficial].
- `SURFACE_DIST`: `f32 = 0.001`. Epsilon for ray offset to prevent self-intersection [S2, unofficial].
- `WIDTH`: `i32 = 800` [S2, unofficial].
- `HEIGHT`: `i32 = 600` [S2, unofficial].
- `SAMPLE_COUNT`: `i32 = 100` [S2, unofficial].

**Pitfalls:**
- Light sources are represented as the last object in the scene's object vector (index `objects.size() - 1`). The loop for sampling indirect bounces must skip hits on the light source to prevent double counting its emission [S1, unofficial].
- Precision issues with ray self-intersection require offsetting the new ray origin along the normal by `SURFACE_DIST` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import List, Tuple, Callable

# Constants from sources
MAX_BOUNCES = 4
SURFACE_DIST = 0.001

# Minimal Vec3 implementation to ensure self-containment
class Vec3:
    def __init__(self, x=0.0, y=0.0, z=0.0):
        self.x, self.y, self.z = x, y, z
    @property
    def ONE(self): return Vec3(1.0, 1.0, 1.0)
    @property
    def ZERO(self): return Vec3(0.0, 0.0, 0.0)
    def dot(self, other): return self.x*other.x + self.y*other.y + self.z*other.z
    def cross(self, other): return Vec3(self.y*other.z - self.z*other.y, self.z*other.x - self.x*other.z, self.x*other.y - self.y*other.x)
    def length(self): return math.sqrt(self.dot(self))
    def normalize(self):
        l = self.length()
        return Vec3(self.x/l, self.y/l, self.z/l) if l > 1e-8 else self.ZERO
    def clamp(self, min_v, max_v):
        return Vec3(max(min_v.x, min(self.x, max_v.x)), max(min_v.y, min(self.y, max_v.y)), max(min_v.z, min(self.z, max_v.z)))
    def __mul__(self, other):
        if isinstance(other, (int, float)): return Vec3(self.x*other, self.y*other, self.z*other)
        return Vec3(self.x*other.x, self.y*other.y, self.z*other.z)
    def __add__(self, other): return Vec3(self.x+other.x, self.y+other.y, self.z+other.z)
    def __truediv__(self, other): return Vec3(self.x/other, self.y/other, self.z/other)

def rand_float() -> float:
    """Returns a float in [0.0, 1.0)."""
    import random
    return random.random()

def uniform_disk() -> Tuple[float, float]:
    while True:
        x = 2.0 * rand_float() - 1.0
        y = 2.0 * rand_float() - 1.0
        if x * x + y * y <= 1.0:
            return (x, y)

def uniform_ball() -> Vec3:
    theta = 2.0 * math.pi * rand_float()
    phi = math.acos(2.0 * rand_float() - 1.0)
    return Vec3(math.sin(phi)*math.cos(theta), math.sin(phi)*math.sin(theta), math.cos(phi))

def cos_weighted_hemisphere(normal: Vec3) -> Vec3:
    # Standard cosine-weighted hemisphere sampling [G]
    u = Vec3(1, 0, 0)
    if abs(normal.dot(Vec3(1, 0, 0))) > 0.9:
        u = Vec3(0, 1, 0)
    v = normal.cross(u).normalize()
    u = v.cross(normal).normalize()
    r1 = 2.0 * math.pi * rand_float()
    r2 = rand_float()
    r2s = math.sqrt(r2)
    x = math.cos(r1) * r2s
    y = math.sin(r1) * r2s
    z = math.sqrt(max(0.0, 1.0 - r2))
    return (x * u + y * v + z * normal).normalize()

def cast_ray(scene, origin: Vec3, direction: Vec3) -> Vec3:
    acc = Vec3.ONE
    bounces = 0
    while True:
        if bounces > MAX_BOUNCES:
            acc = Vec3.ZERO
            break
        hit_info = scene.map.ray_intersection(origin, direction)
        if hit_info is None:
            acc = acc * scene.background_color(direction)
            break
        material = hit_info.material
        position = hit_info.position
        normal = scene.map.normal(position)
        if isinstance(material, Lambertian):
            color = material.color
            acc = acc * color
            Q = max(color.x, max(color.y, color.z))
            if rand_float() > Q:
                acc = Vec3.ZERO
                break
            origin = position + 2.0 * SURFACE_DIST * normal
            direction = cos_weighted_hemisphere(normal)
            bounces += 1
        elif isinstance(material, Emissive):
            acc = acc * material.color
            break
        elif isinstance(material, Specular):
            color = material.color
            specularity = material.specularity
            fuzziness = material.fuzziness
            incoming = direction
            specular = incoming - 2.0 * incoming.dot(normal) * normal
            rand_val = rand_float()
            if rand_val < specularity:
                perturbed = specular + fuzziness * uniform_ball()
                direction = perturbed.normalize()
            else:
                direction = cos_weighted_hemisphere(normal)
            if direction.dot(normal) < 0.0:
                acc = Vec3.ZERO
                break
            acc = acc * color
            origin = position + 2.0 * SURFACE_DIST * normal
            bounces += 1
        else:
            acc = Vec3.ZERO
            break
    return acc

def render(scene, width: int, height: int, sample_count: int) -> List[List[Vec3]]:
    pixels = [[Vec3.ZERO for _ in range(width)] for _ in range(height)]
    for i in range(height):
        for j in range(width):
            pixel_acc = Vec3.ZERO
            for _ in range(sample_count):
                jitter_x = rand_float() - 0.5
                jitter_y = rand_float() - 0.5
                x = (-0.5 + (j + jitter_x)) / (width - 1.0)
                y = (0.5 - (i + jitter_y)) / (height - 1.0)
                ray = scene.camera.get_ray(x, y)
                color = cast_ray(scene, ray.origin, ray.direction)
                pixel_acc = pixel_acc + color
            pixels[i][j] = pixel_acc / sample_count
    return pixels
```

## Subtopics

### [Path Tracing with Discrete Event Sampling](5168-path-tracing-with-discrete-event-sampling.md)
A variant for participating media that replaces continuous Russian Roulette with discrete probabilities for null, absorption, and scattering events. It is chosen for sparse media to handle temperature-dependent emission and Henyey-Greenstein phase functions more efficiently than standard continuous methods. It builds on Explicit Path Tracing by managing recursion depth explicitly and using discrete probabilities $p_n, p_a, p_s$ instead of probabilistic survival based on albedo.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** On a resolution of $800 \times 600$ with 100 samples per pixel and 4 bounces, rendering time depends heavily on the SDF intersection cost [S2, unofficial].
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
Explicit Path Tracing is a rendering algorithm that simulates light transport by recursively tracing rays from the camera into the scene, accumulating color contributions from surface interactions until a maximum bounce count is reached or a termination condition is met [S2, unofficial]. It explicitly manages recursion depth and uses Russian Roulette to terminate paths probabilistically based on surface albedo [S1, unofficial]. The specific implementation details provided in [S1, unofficial] and [S2, unofficial] are from unofficial repositories implementing a path tracer with SDFs and basic material models.

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S2] ollituohenmaa/raymarching-pathtracer, https://github.com/ollituohenmaa/raymarching-pathtracer, fetched 2026-10-26 (unofficial)
