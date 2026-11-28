# Explicit Path Tracing with Russian Roulette

Also called: Implicit Path Tracing, Monte Carlo Path Tracing, Path Tracing with Russian Roulette and Material Sampling

## Overview
Explicit Path Tracing with Russian Roulette is a rendering algorithm that simulates light transport by recursively tracing rays from the camera into the scene, accumulating color contributions from surface interactions until a maximum bounce count is reached or a termination condition is met [S2, unofficial]. Unlike [Single Scattering Volume Rendering with Ray Marching](1532-single-scattering-volume-rendering-with-ray-marching.md), which approximates volume effects via ray marching, this method handles discrete surface interactions and uses Russian Roulette to terminate paths based on surface albedo, balancing computational cost against variance [S1, unofficial]. It differs from [Unidirectional Path Tracing](2558-unidirectional-path-tracing.md) by explicitly managing the recursion depth and termination probability within the path generation loop rather than relying solely on fixed depth limits for all indirect terms [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A `Scene` struct containing a `Camera`, a signed distance field map (`SdfMap`), and a background color function, along with pixel coordinates $(i, j)$, resolution $(WIDTH, HEIGHT)$, and sample count per pixel [S2, unofficial].
- **Output:** A rendered image where each pixel contains the accumulated color $C$ [S2, unofficial].

**Data Structures:**
- **`Scene`**: `camera` (`Camera`), `map` (`Box<dyn SdfMap>` implementing ray intersection and normal retrieval), `background_color` (`Fn(Vec3) -> Vec3`) [S2, unofficial].
- **`Camera`**: `position`, `left`, `forward`, `up` (`Vec3` basis vectors), `focal_length`, `aspect_ratio`, `focus_dist`, `aperture` (`f32`) [S2, unofficial].
- **`Ray`**: `origin`, `direction` (`Vec3`, normalized) [S2, unofficial].
- **`HitInfo`**: `position` (`Vec3`), `material` (Enum `Material` with variants `Lambertian { color }`, `Emissive { color }`, `Specular { color, specularity, fuzziness }`) [S2, unofficial].

**Parameters:**
- `MAX_BOUNCES`: `i32 = 4` [S2, unofficial].
- `SURFACE_DIST`: `f32 = 0.001` [S2, unofficial].
- `WIDTH`: `i32 = 800` [S2, unofficial].
- `HEIGHT`: `i32 = 600` [S2, unofficial].
- `SAMPLE_COUNT`: `i32 = 100` [S2, unofficial].

**Algorithm Steps:**
1. **Camera Ray Generation:** For each pixel $(i, j)$ and sample, generate jittered coordinates $x = \frac{-0.5 + (j + \text{jitter}_x)}{WIDTH - 1.0}$ and $y = \frac{0.5 - (i + \text{jitter}_y)}{HEIGHT - 1.0}$ where $\text{jitter}_x, \text{jitter}_y \in [-0.5, 0.5]$ are uniform random floats [S2, unofficial]. Compute ray origin and direction: sample aperture offset $(dx, dy) = \text{uniform\_disk()}$, compute $offset = 0.5 \cdot aperture \cdot (dx \cdot left + dy \cdot up)$, set $origin = position + offset$, and compute $direction = \text{normalize}(focus\_dist \cdot (x \cdot \frac{left}{focal\_length} + y \cdot \frac{up}{focal\_length \cdot aspect\_ratio} + forward) - offset)$ [S2, unofficial].
2. **Path Tracing Loop:** Initialize `acc = Vec3::ONE` (throughput) and `bounces = 0` [S2, unofficial]. Loop indefinitely: check if `bounces > MAX_BOUNCES`; if true, set `acc = Vec3::ZERO` and break [S2, unofficial]. Query `scene.map.ray_intersection(origin, direction)` [S2, unofficial]. If miss, multiply `acc` by `scene.background_color(direction)` and break [S2, unofficial]. If hit, retrieve `hit_info`. If `Emissive { color }`, multiply `acc` by `color` and break [S2, unofficial]. If `Lambertian { color }`, multiply `acc` by `color`, compute normal $n = \text{scene.map.normal}(hit\_info.position)$, offset origin $origin = hit\_info.position + 2.0 \cdot SURFACE\_DIST \cdot n$, and sample new direction $direction = \text{cos\_weighted\_hemisphere}(n)$ [S2, unofficial]. If `Specular { color, specularity, fuzziness }`, compute normal $n$, offset origin $origin = hit\_info.position + 2.0 \cdot SURFACE\_DIST \cdot n$, generate random float $\xi \in [0, 1)$, if $\xi < specularity$ compute specular reflection $r = direction - 2.0 \cdot (direction \cdot n) \cdot n$, perturb $direction = \text{normalize}(r + fuzziness \cdot \text{uniform\_ball}())$, if $direction \cdot n < 0.0$ set `acc = Vec3::ZERO` and break, else multiply `acc` by `color` and set $direction = \text{cos\_weighted\_hemisphere}(n)$ [S2, unofficial]. Increment `bounces` by 1 [S2, unofficial].
3. **Accumulation:** Sum all `cast_ray` results for the pixel and divide by `SAMPLE_COUNT` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

# Constants from [S2, unofficial]
MAX_BOUNCES = 4
SURFACE_DIST = 0.001
WIDTH = 800
HEIGHT = 600
SAMPLE_COUNT = 100

# Material definitions to replace undefined names
class Lambertian:
    def __init__(self, color): self.color = color
class Emissive:
    def __init__(self, color): self.color = color
class Specular:
    def __init__(self, color, specularity, fuzziness):
        self.color = color
        self.specularity = specularity
        self.fuzziness = fuzziness

def uniform_disk():
    """Generate a uniform random point in the unit disk [G]."""
    while True:
        x = random.uniform(-1.0, 1.0)
        y = random.uniform(-1.0, 1.0)
        if x*x + y*y <= 1.0:
            return (x, y)

def uniform_ball():
    """Generate a uniform random point on the unit sphere [G]."""
    while True:
        x = random.gauss(0, 1)
        y = random.gauss(0, 1)
        z = random.gauss(0, 1)
        length = math.sqrt(x*x + y*y + z*z)
        if length > 1e-6:
            return (x/length, y/length, z/length)

def cos_weighted_hemisphere(normal):
    """Sample a direction in the hemisphere defined by normal, weighted by cosine [G]."""
    u1 = random.uniform(0, 1)
    u2 = random.uniform(0, 1)
    z = math.sqrt(u1)
    theta = 2 * math.pi * u2
    x = math.cos(theta) * math.sqrt(1 - u1)
    y = math.sin(theta) * math.sqrt(1 - u1)
    
    # Construct local basis
    if abs(normal[0]) < 0.9:
        tangent = (1, 0, 0)
    else:
        tangent = (0, 1, 0)
    
    bitangent = (
        normal[1]*tangent[2] - normal[2]*tangent[1],
        normal[2]*tangent[0] - normal[0]*tangent[2],
        normal[0]*tangent[1] - normal[1]*tangent[0]
    )
    blen = math.sqrt(bitangent[0]**2 + bitangent[1]**2 + bitangent[2]**2)
    bitangent = (bitangent[0]/blen, bitangent[1]/blen, bitangent[2]/blen)
    
    tangent = (
        normal[1]*bitangent[2] - normal[2]*bitangent[1],
        normal[2]*bitangent[0] - normal[0]*bitangent[2],
        normal[0]*bitangent[1] - normal[1]*bitangent[0]
    )
    
    direction = (
        x * tangent[0] + y * bitangent[0] + z * normal[0],
        x * tangent[1] + y * bitangent[1] + z * normal[1],
        x * tangent[2] + y * bitangent[2] + z * normal[2]
    )
    return direction

def cast_ray(scene, origin, direction):
    acc = (1.0, 1.0, 1.0)
    bounces = 0
    
    while True:
        if bounces > MAX_BOUNCES:
            acc = (0.0, 0.0, 0.0)
            break
            
        hit_info = scene.map.ray_intersection(origin, direction)
        
        if hit_info is None:
            bg_color = scene.background_color(direction)
            acc = (acc[0]*bg_color[0], acc[1]*bg_color[1], acc[2]*bg_color[2])
            break
            
        material = hit_info.material
        
        if isinstance(material, Emissive):
            acc = (acc[0]*material.color[0], acc[1]*material.color[1], acc[2]*material.color[2])
            break
            
        elif isinstance(material, Lambertian):
            acc = (acc[0]*material.color[0], acc[1]*material.color[1], acc[2]*material.color[2])
            normal = scene.map.normal(hit_info.position)
            origin = (
                hit_info.position[0] + 2.0 * SURFACE_DIST * normal[0],
                hit_info.position[1] + 2.0 * SURFACE_DIST * normal[1],
                hit_info.position[2] + 2.0 * SURFACE_DIST * normal[2]
            )
            direction = cos_weighted_hemisphere(normal)
            
        elif isinstance(material, Specular):
            normal = scene.map.normal(hit_info.position)
            origin = (
                hit_info.position[0] + 2.0 * SURFACE_DIST * normal[0],
                hit_info.position[1] + 2.0 * SURFACE_DIST * normal[1],
                hit_info.position[2] + 2.0 * SURFACE_DIST * normal[2]
            )
            
            xi = random.uniform(0, 1)
            if xi < material.specularity:
                dot = direction[0]*normal[0] + direction[1]*normal[1] + direction[2]*normal[2]
                rx = direction[0] - 2.0 * dot * normal[0]
                ry = direction[1] - 2.0 * dot * normal[1]
                rz = direction[2] - 2.0 * dot * normal[2]
                
                fuzz = uniform_ball()
                direction = (
                    rx + material.fuzziness * fuzz[0],
                    ry + material.fuzziness * fuzz[1],
                    rz + material.fuzziness * fuzz[2]
                )
                dlen = math.sqrt(direction[0]**2 + direction[1]**2 + direction[2]**2)
                direction = (direction[0]/dlen, direction[1]/dlen, direction[2]/dlen)
                
                dot_n = direction[0]*normal[0] + direction[1]*normal[1] + direction[2]*normal[2]
                if dot_n < 0.0:
                    acc = (0.0, 0.0, 0.0)
                    break
            else:
                acc = (acc[0]*material.color[0], acc[1]*material.color[1], acc[2]*material.color[2])
                direction = cos_weighted_hemisphere(normal)
                
        bounces += 1
        
    return acc

def render(scene):
    pixels = []
    for i in range(HEIGHT):
        row = []
        for j in range(WIDTH):
            pixel_sum = (0.0, 0.0, 0.0)
            for _ in range(SAMPLE_COUNT):
                jitter_x = random.uniform(-0.5, 0.5)
                jitter_y = random.uniform(-0.5, 0.5)
                
                x = (-0.5 + j + jitter_x) / (WIDTH - 1.0)
                y = (0.5 - i + jitter_y) / (HEIGHT - 1.0)
                
                ray = scene.camera.get_ray(x, y)
                color = cast_ray(scene, ray.origin, ray.direction)
                pixel_sum = (
                    pixel_sum[0] + color[0],
                    pixel_sum[1] + color[1],
                    pixel_sum[2] + color[2]
                )
            
            avg_color = (
                pixel_sum[0] / SAMPLE_COUNT,
                pixel_sum[1] / SAMPLE_COUNT,
                pixel_sum[2] / SAMPLE_COUNT
            )
            gamma = 1.0 / 2.2
            final_color = (
                min(avg_color[0], 1.0) ** gamma,
                min(avg_color[1], 1.0) ** gamma,
                min(avg_color[2], 1.0) ** gamma
            )
            row.append(final_color)
        pixels.append(row)
    return pixels
```

## Subtopics

### [Path Tracing with Discrete Event Sampling](5168-path-tracing-with-discrete-event-sampling.md)
Path Tracing with Discrete Event Sampling is a path tracing algorithm for participating media that replaces continuous Russian Roulette termination with a discrete probabilistic decision between absorption, scattering, and null events at each sampled point. Unlike [Explicit Path Tracing with Russian Roulette](2209-explicit-path-tracing-with-russian-roulette.md), which uses continuous probability based on albedo/extinction ratios, this method explicitly handles null events (transmittance without interaction) to improve efficiency in sparse media by skipping non-interacting steps. It supports temperature-dependent emission and Henyey-Greenstein phase function scattering.

### [Recursive Trace Function](9277-recursive-trace-function.md)
The Recursive Trace Function is the core recursive routine used in backward path tracing with Directed Distance Fields (DDF) to compute light transport for subsequent bounces. It handles intersection, shading, and recursion logic for a single light path segment.

### Russian Roulette for Scattering Albedo
A variance reduction technique using the scattering albedo to decide whether to continue tracing a scattered ray or terminate it [S3, unofficial]. The scattering albedo is defined as $\alpha = \sigma_s / \sigma_t$, where $\sigma_s$ is the scattering coefficient and $\sigma_t$ is the extinction coefficient [S3, unofficial]. If a random number $\xi < \alpha$, the path continues and the contribution is weighted by $1/\alpha$; otherwise, the contribution is 0 [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(S \cdot B \cdot I)$, where $S$ is the number of samples per pixel, $B$ is the maximum number of bounces (`MAX_BOUNCES`), and $I$ is the cost of ray intersection [S2, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space per ray, as the algorithm is iterative and does not store the full path history [S2, unofficial].
- **Performance:** Measured on a standard CPU implementation using parallel pixel processing via `rayon`, rendering 800x600 images with 100 samples takes several seconds to minutes depending on scene complexity [S2, unofficial].
- **Strengths:** Simple to implement; handles complex lighting via recursion; bounce limits prevent infinite loops [S2, unofficial].
- **Limitations:** High variance in scenes with deep shadows or complex indirect lighting; no explicit handling of volume scattering in the base algorithm [S2, unofficial].
- **When to use:** General-purpose offline rendering for scenes with diffuse and specular surfaces [S2, unofficial].
- **When not to use:** Scenes requiring accurate volume rendering (use [Path Tracing with Volume Interaction](3320-path-tracing-with-volume-interaction.md)) or sparse participating media (use [Path Tracing with Discrete Event Sampling](5168-path-tracing-with-discrete-event-sampling.md)) [S2, unofficial].

## Background
The algorithm originates from the implementation of a raymarching path tracer using Signed Distance Fields (SDFs) [S2, unofficial]. It combines standard path tracing concepts (recursive ray casting, BRDF sampling) with SDF-based intersection and Russian Roulette termination strategies [S1, S2, unofficial]. The specific implementation details, such as the `MAX_BOUNCES` limit and material handling, are derived from the `ollituohenmaa/raymarching-pathtracer` repository [S2, unofficial]. The theoretical basis for Russian Roulette in path tracing is well-established in computer graphics literature, often cited in conjunction with Monte Carlo integration techniques [S3, unofficial].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S2] ollituohenmaa/raymarching-pathtracer, https://github.com/ollituohenmaa/raymarching-pathtracer, fetched 2026-10-26 (unofficial)
[S3] Volume Rendering and Scattering Theory, https://www.scribd.com/document/912805133/02941-Slides-07-Volumes, fetched 2026-10-29 (unofficial)
