# Explicit Path Tracing with Russian Roulette

Also called: Implicit Path Tracing, Monte Carlo Path Tracing, Path Tracing with Russian Roulette and Material Sampling

## Overview
Explicit Path Tracing is a rendering algorithm that simulates light transport by recursively tracing rays from the camera into the scene, accumulating color contributions from surface interactions until a maximum bounce count is reached or a termination condition is met [S2, unofficial]. It explicitly manages recursion depth and uses Russian Roulette to terminate paths probabilistically based on surface albedo [S1, unofficial]. Unlike its sibling [Path Tracing with Discrete Event Sampling](5168-path-tracing-with-discrete-event-sampling.md), which replaces continuous termination with discrete probabilities for sparse media, this method relies on fixed depth limits and probabilistic survival based on material reflectance [S1, unofficial].

## How it works
**Inputs:**
- `scene`: A scene structure containing a `camera`, a `map` (SDF map implementing intersection and normal queries), and a `background_color` function.
- `width`, `height`: Integer resolution of the image.
- `sample_count`: Number of samples per pixel.

**Outputs:**
- `pixels`: A 2D array of `Vec3` colors, representing the final image.

**Data Structures:**
- `Scene`:
  - `camera`: `Camera` struct.
  - `map`: `Box<dyn SdfMap>` (interface for ray intersection and normal calculation).
  - `background_color`: `Box<dyn Fn(Vec3) -> Vec3 + Sync>` (function evaluating background color for a direction).
- `Camera`:
  - `position`: `Vec3` (camera origin).
  - `left`, `forward`, `up`: `Vec3` (basis vectors).
  - `focal_length`: `f32`.
  - `aspect_ratio`: `f32`.
  - `focus_dist`: `f32`.
  - `aperture`: `f32`.
- `HitInfo` (returned by `map.ray_intersection`):
  - `position`: `Vec3`.
  - `material`: `Material` enum.
- `Material`:
  - `Lambertian { color: Vec3 }`
  - `Emissive { color: Vec3 }`
  - `Specular { color: Vec3, specularity: f32, fuzziness: f32 }`

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

# Standard math/vector operations assumed available
# Vec3: x, y, z components; dot, cross, normalize, length, clamp, powf, ONE, ZERO
# rand_float(): returns float in [0.0, 1.0)

def uniform_disk() -> Tuple[float, float]:
    """Sample a point uniformly on the unit disk."""
    while True:
        x = 2.0 * rand_float() - 1.0
        y = 2.0 * rand_float() - 1.0
        if x * x + y * y <= 1.0:
            return (x, y)

def uniform_ball() -> Vec3:
    """Sample a point uniformly on the unit sphere."""
    # Implementation detail: typically rejection sampling on cube or spherical coords
    # For this context, we assume a helper exists or use standard spherical sampling
    theta = 2.0 * math.pi * rand_float()
    phi = math.acos(2.0 * rand_float() - 1.0)
    x = math.sin(phi) * math.cos(theta)
    y = math.sin(phi) * math.sin(theta)
    z = math.cos(phi)
    return Vec3(x, y, z)

def cos_weighted_hemisphere(normal: Vec3) -> Vec3:
    """Sample a direction in the hemisphere above the normal, cosine-weighted."""
    # Generate random direction in hemisphere
    # Standard implementation involves generating a random point in a cone
    # or using the method from [S2, unofficial] which calls sampling::cos_weighted_hemisphere
    # Here we define it explicitly:
    # 1. Sample uniform disk (u, v)
    # 2. Sample uniform height z in [0, 1]
    # 3. Construct vector and project to hemisphere
    u, v = uniform_disk()
    z = rand_float()
    # Vector in local space
    local_dir = Vec3(u, v, math.sqrt(max(0.0, 1.0 - u*u - v*v)))
    # Rotate to align with normal
    # Assuming basis vectors right, up, forward are derived from normal
    # Simplified: if normal is (0,0,1), local_dir is correct.
    # General rotation requires constructing an orthonormal basis.
    # For brevity and adherence to "no library internals", we assume a standard
    # cos-weighted hemisphere sampler is available or implemented as:
    # direction = normalize(normal + local_dir) # Approximation, not exact cos-weighted
    # Exact cos-weighted hemisphere sampling:
    # See [S2, unofficial] context: direction = sampling::cos_weighted_hemisphere(normal)
    # We will use a standard implementation:
    # 1. Generate random point in unit disk (x, y)
    # 2. z = sqrt(1 - x^2 - y^2)
    # 3. Construct orthonormal basis (u, v, w=normal)
    # 4. dir = x*u + y*v + z*w
    # Since we cannot call library internals, we define the basis construction:
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

def get_ray_from_camera(camera: Camera, i: int, j: int, width: int, height: int) -> Ray:
    """Generate a ray for a specific pixel with jitter."""
    # Jittered sampling
    jitter_x = rand_float() - 0.5
    jitter_y = rand_float() - 0.5
    
    x = (-0.5 + (j + jitter_x)) / (width - 1.0)
    y = (0.5 - (i + jitter_y)) / (height - 1.0)
    
    return camera.get_ray(x, y)

def cast_ray(scene: Scene, origin: Vec3, direction: Vec3) -> Vec3:
    """
    Trace a single path from origin/direction.
    Implements Explicit Path Tracing with Russian Roulette.
    """
    acc = Vec3.ONE
    bounces = 0
    
    while True:
        if bounces > MAX_BOUNCES:
            acc = Vec3.ZERO
            break
            
        hit_info = scene.map.ray_intersection(origin, direction)
        
        if hit_info is None:
            # No hit: multiply by background color
            acc = acc * scene.background_color(direction)
            break
            
        material = hit_info.material
        position = hit_info.position
        normal = scene.map.normal(position)
        
        if isinstance(material, MaterialLambertian):
            color = material.color
            acc = acc * color
            
            # Russian Roulette based on albedo (max component of color)
            Q = max(color.x, max(color.y, color.z))
            if Q < 1e-3: # Avoid division by zero or infinite throughput
                acc = Vec3.ZERO
                break
            
            # Survival probability
            if rand_float() > Q:
                acc = Vec3.ZERO
                break
            
            # Continue path
            # Offset to prevent self-intersection
            origin = position + 2.0 * SURFACE_DIST * normal
            direction = cos_weighted_hemisphere(normal)
            bounces += 1
            
        elif isinstance(material, MaterialEmissive):
            # Emissive surface: add color and terminate
            acc = acc * material.color
            break
            
        elif isinstance(material, MaterialSpecular):
            color = material.color
            specularity = material.specularity
            fuzziness = material.fuzziness
            
            # Russian Roulette for specular? 
            # [S2, unofficial] implies specular reflection continues, but albedo check might apply
            # [S1, unofficial] mentions Q is max component of object's color.
            Q = max(color.x, max(color.y, color.z))
            if Q < 1e-3:
                acc = Vec3.ZERO
                break
                
            if rand_float() > Q:
                acc = Vec3.ZERO
                break
                
            # Compute specular reflection
            # specular = direction - 2.0 * direction.dot(normal) * normal
            # Note: 'direction' here is the incoming ray direction.
            # Reflection formula: R = I - 2(I.N)N
            # In [S2, unofficial]: specular = direction - 2.0 * direction.dot(normal) * normal
            # But 'direction' in the loop is the ray direction (outgoing from previous hit).
            # So we reflect the incoming direction.
            incoming = direction
            specular = incoming - 2.0 * incoming.dot(normal) * normal
            
            # Perturb with fuzziness
            # If rand < specularity: specular reflection
            # Else: diffuse reflection? [S2, unofficial] logic:
            # "If rand < specularity: compute specular... Else: multiply acc by color, set direction = cos_weighted_hemisphere"
            # This implies a mix or a choice.
            
            rand_val = rand_float()
            if rand_val < specularity:
                # Specular reflection
                # Perturb: direction = (specular + fuzziness * uniform_ball()).normalize()
                perturbed = specular + fuzziness * uniform_ball()
                direction = perturbed.normalize()
            else:
                # Diffuse reflection
                direction = cos_weighted_hemisphere(normal)
                
            # Check if direction is valid (dot with normal > 0)
            if direction.dot(normal) < 0.0:
                acc = Vec3.ZERO
                break
                
            acc = acc * color
            origin = position + 2.0 * SURFACE_DIST * normal
            bounces += 1
            
        else:
            # Unknown material, terminate
            acc = Vec3.ZERO
            break

    return acc

def render(scene: Scene, width: int, height: int, sample_count: int) -> List[List[Vec3]]:
    """Render the scene using parallel processing."""
    pixels = [[Vec3.ZERO for _ in range(width)] for _ in range(height)]
    
    # Parallel loop over pixels
    for i in range(height):
        for j in range(width):
            pixel_acc = Vec3.ZERO
            for _ in range(sample_count):
                ray = get_ray_from_camera(scene.camera, i, j, width, height)
                color = cast_ray(scene, ray.origin, ray.direction)
                pixel_acc = pixel_acc + color
            pixels[i][j] = pixel_acc / sample_count
            
    return pixels
```

## Subtopics

### [Path Tracing with Discrete Event Sampling](5168-path-tracing-with-discrete-event-sampling.md)
A variant for participating media that replaces continuous Russian Roulette with discrete probabilities for null, absorption, and scattering events. It is chosen for sparse media to handle temperature-dependent emission and Henyey-Greenstein phase functions more efficiently than standard continuous methods. It builds on Explicit Path Tracing by managing recursion depth explicitly and using discrete probabilities $p_n, p_a, p_s$ instead of probabilistic survival based on albedo.

## Operating characteristics
- **Time Complexity:** $O(S \cdot P \cdot B)$, where $S$ is samples per pixel, $P$ is number of pixels, and $B$ is maximum bounces. Each bounce involves a ray intersection query.
- **Memory Complexity:** $O(P)$ for storing the image, plus $O(B)$ for the recursion stack (or loop state).
- **Measured Performance:** On a resolution of $800 \times 600$ with 100 samples per pixel and 4 bounces, rendering time depends heavily on the SDF intersection cost. [S2, unofficial] reports rendering times in seconds on standard hardware (specific hardware not detailed, but implies desktop CPU/GPU).
- **Strengths:** Simple to implement; handles complex lighting and shadows naturally; produces physically accurate results.
- **Limitations:** High variance (noise) in scenes with complex lighting; slow convergence for indirect illumination; sensitive to Russian Roulette parameters (Q) which can cause premature termination if albedo is low.
- **When to use:** Small to medium scenes with diffuse and specular surfaces; educational purposes; baseline for comparison.
- **When not to use:** Large scenes with many bounces (performance); participating media (use [Path Tracing with Volume Interaction](3320-path-tracing-with-volume-interaction.md) or [Path Tracing with Discrete Event Sampling](5168-path-tracing-with-discrete-event-sampling.md)); real-time applications.

## Background
Explicit Path Tracing is a fundamental technique in computer graphics for simulating global illumination. It was popularized by James Kajiya in 1986 [S1, unofficial]. The method traces paths of light from the camera, recursively sampling interactions with surfaces. Russian Roulette is a standard variance reduction technique introduced to terminate paths probabilistically, preventing infinite recursion while maintaining unbiased estimates. The specific implementation details provided in [S1, unofficial] and [S2, unofficial] are from unofficial repositories implementing a path tracer with SDFs and basic material models.

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S2] ollituohenmaa/raymarching-pathtracer, https://github.com/ollituohenmaa/raymarching-pathtracer, fetched 2026-10-26 (unofficial)
