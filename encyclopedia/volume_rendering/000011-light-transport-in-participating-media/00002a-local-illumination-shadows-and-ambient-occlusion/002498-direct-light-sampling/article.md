# Direct Light Sampling

Also called: Direct Illumination Sampling, Iluminación directa

## Overview
Direct Light Sampling is a technique for computing direct illumination by explicitly shooting a ray from an intersection point toward a light source to determine visibility and light contribution [S1, unofficial]. It solves the problem of accurately modeling first-order scattering and shadows by testing occlusion along the path to the light [S1, unofficial]. Unlike [Light Ray Sampling](4885-light-ray-sampling.md), which integrates density along a fixed direction, Direct Light Sampling explicitly selects light sources and verifies geometric visibility, often using a power heuristic for unbiased estimation [S2][S3, unofficial].

## How it works
**Inputs:**
- Ray $r$ with origin and direction [S1, unofficial].
- Set of scene objects $O$ [S1, unofficial].
- Light source $l$ or a list of lights $L$ [S1, S3, unofficial].
- Bounce counter $b$ [S1, unofficial].
- Current accumulated color $c$ (initially $[1, 1, 1]$) [S1, unofficial].

**Outputs:**
- Final color contribution from the light path [S1, unofficial].

**Data Structures:**
- `LightHit`: A structure containing `colour` (radiance), `pdf` (probability density), `direction` (unit vector to light), and `distance` (distance to light) [S3, unofficial].
- `ScatterEvent`: Interaction point data including position, normal, and output direction [S3, unofficial].

**Steps:**
1. **Initialize:** Set hit flag $hl = \text{false}$ and accumulated color $c = [1, 1, 1]$ [S1, unofficial].
2. **Ray Intersection:** Find the closest object $o \in O$ intersected by ray $r$ at distance $t > 0$ [S1, unofficial]. If no intersection, return background color modulated by $c$ [S1, unofficial].
3. **Light Direction:** Calculate direction to light: $\mathbf{dl} = \text{normalize}(\text{position}(l) - \mathbf{p})$, where $\mathbf{p}$ is the intersection point [S1, unofficial].
4. **Shadow Ray:** Create shadow ray $r_l$ from $\mathbf{p}$ in direction $\mathbf{dl}$ [S1, unofficial].
5. **Light Intersection:** Test intersection between $r_l$ and light $l$ to get distance $t_l$ [S1, unofficial].
6. **Occlusion Test:** Find minimum distance $t_{min}$ to any object $o \in O$ along $r_l$ [S1, unofficial]. Let $t_{min} = \min \{ t \mid o \in O \land t = \text{intersection test}(r_l, o) \land t > 0 \}$ [S1, unofficial].
7. **Visibility Check:** Update hit flag: $hl \leftarrow hl \lor (t_l \le t_{min})$ [S1, unofficial]. If the light is hit and closer than any occluding object, the point is directly illuminated [S1, unofficial].
8. **Illumination Calculation:** If using a sampler to select lights, select a random light $l_i$ not equal to the current hit light [S3, unofficial]. Calculate light contribution: $\text{weight} = \text{powerHeuristic}(1, \text{lightHit.pdf}, 1, \text{pdf})$ [S3, unofficial]. Add to direct light: $\text{directLight} += \text{material} \rightarrow \text{evaluate}(\text{interaction}) \times \text{lightHit.colour} \times \text{weight} / \text{lightHit.pdf}$ [S3, unofficial]. Scale by total number of lights if uniform sampling is used [S3, unofficial].
9. **Recursive Bounce:** If bounce counter $b > 0$, compute valid bounce direction $\mathbf{d}$, create new ray $r' = \text{ray}(\mathbf{p}, \mathbf{d})$, update $c \leftarrow c \circ \text{color material}(\mathbf{p}, o)$, and recurse [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Ray:
    def __init__(self, origin, direction):
        self.origin = origin
        self.direction = direction

class Light:
    def __init__(self, position, colour, intensity):
        self.position = position
        self.colour = colour
        self.intensity = intensity

    def intersection_test(self, ray):
        # Returns distance to light if ray points to it, else infinity
        to_light = [self.position[i] - ray.origin[i] for i in range(3)]
        dist = math.sqrt(sum(x*x for x in to_light))
        if dist < 1e-6: return float('inf')
        cos_theta = sum(ray.direction[i]*to_light[i] for i in range(3)) / dist
        if cos_theta > 0.999: return dist
        return float('inf')

def get_background_color(direction):
    return [0.0, 0.0, 0.0]

def get_valid_bounce_direction(position, normal, incoming_dir):
    # Returns a normalized diffuse bounce direction [G]
    return [normal[0], normal[1], normal[2]]

def direct_light_sampling(r, objects, lights, bounce_count, accumulated_color):
    hl = False
    c = [1.0, 1.0, 1.0]
    while True:
        if bounce_count < 0:
            if not hl:
                return get_background_color(r.direction)
            else:
                light = lights[0]
                return [c[i] * (get_background_color(r.direction)[i] + light.colour[i]) for i in range(3)]

        closest_obj = None
        closest_t = float('inf')
        for obj in objects:
            t = obj.intersection_test(r)
            if t > 0 and t < closest_t:
                closest_t = t
                closest_obj = obj

        if closest_obj is None:
            bg = get_background_color(r.direction)
            if not hl:
                return [c[i] * bg[i] for i in range(3)]
            else:
                light = lights[0]
                return [c[i] * (bg[i] + light.colour[i]) for i in range(3)]

        p = [r.origin[i] + closest_t * r.direction[i] for i in range(3)]
        light = lights[0]

        dl = [light.position[i] - p[i] for i in range(3)]
        dl_len = math.sqrt(sum(x*x for x in dl))
        dl = [x/dl_len for x in dl]

        rl = Ray(p, dl)
        tl = light.intersection_test(rl)

        t_min = float('inf')
        for obj in objects:
            t = obj.intersection_test(rl)
            if t > 0 and t < t_min:
                t_min = t

        hl = hl or (tl <= t_min)

        d = get_valid_bounce_direction(p, closest_obj.normal, r.direction)
        r = Ray(p, d)
        c = [c[i] * closest_obj.material_color[i] for i in range(3)]
        bounce_count -= 1
    return c
```

**Parameters:**
- `sigma` (Gaussian filter): Used post-processing to reduce noise, e.g., $\sigma = 4$ [S1, unofficial].

**Edge Cases:**
- **No Lights:** Return zero or background [S2].
- **Self-Intersection:** Shadow ray origin should be offset slightly along normal to avoid self-intersection [G].
- **Precision:** Floating point errors in $t_l \le t_{min}$ comparison can cause flickering; use a small epsilon [G].

## Subtopics

### [Light Sampling for NEE](2616-light-sampling-for-nee.md)
Reduces variance in direct illumination by explicitly sampling light sources, decoupling light selection from path vertex generation.

### [Direct Illumination Sampling with Mesh Intersection](4450-direct-illumination-sampling-with-mesh-intersection.md)
Handles direct illumination for polygonal scenes using explicit triangle meshes, distinguishing itself from SDF-based approaches.

### [Light Ray Sampling](4885-light-ray-sampling.md)
Computes direct illumination by stepping along the direction to a distant light, integrating density for soft volumetric shadows.

### [Direct Light Contribution Estimation](5411-direct-light-contribution-estimation.md)
Separates attenuation into high-resolution local and low-resolution global segments for accurate first-order scattering.

## Operating characteristics
- **Time Complexity:** $O(N \cdot L)$ per pixel, where $N$ is the number of bounces and $L$ is the number of lights (if testing all) or $O(N)$ if using a sampler to select one light [S1, unofficial][S2].
- **Memory Complexity:** $O(1)$ auxiliary space per ray, excluding scene data structures [S1, unofficial].
- **Performance:** Limited by hardware constraints; launching infinite rays uniformly is not feasible for real-time FPS (>15 FPS) [S1, unofficial]. Post-processing filters (Gaussian, bilinear) are often applied to reduce noise [S1, unofficial].
- **Strengths:** Accurate shadowing and direct illumination; simple to implement [S1, unofficial].
- **Limitations:** High variance (noise) in complex scenes; requires many samples for convergence [S1, unofficial].

## Background
The method is described in the context of Sparse Voxel Octree compression and efficiency studies, where direct illumination is added to improve visual quality by shooting rays from intersection points to light sources [S1, unofficial]. It is also implemented in educational path tracers for handling different light types and ensuring visibility [S2].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S2] Real-time Rendering with a Neural Irradiance Volume, https://doi.org/10.48550/arxiv.2602.12949, fetched 2026-10-26
[S3] thewildnath/PathTracer, https://github.com/thewildnath/PathTracer, fetched 2026-10-26 (unofficial)
