# Homogeneous Volume Intersection

Also called: Constant-density volume intersection, Ray-Volume Intersection, Volume Intersection, Volume::intersect

## Overview
Homogeneous Volume Intersection computes the first interaction point of a ray within a constant-density volume, or determines if the ray exits the medium without interacting [S1, unofficial]. It differs from [Heterogeneous Volume Intersection](3261-heterogeneous-volume-intersection.md) by assuming a uniform density field, which allows direct analytical sampling of the free path length without majorant-based rejection sampling [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- World-space ray with origin $\mathbf{o}$, direction $\mathbf{d}$, and valid interval $[t_{\min}, t_{\max}]$ [S1, unofficial].
- Volume boundary entry $t_{\text{in}}$ and exit $t_{\text{out}}$ relative to the world ray [S1, unofficial].
- Constant density $\sigma_t$ [S1, unofficial].
- Random number generator for sampling free path length [S1, unofficial].

**Outputs:**
- A hit structure containing $t$ (distance along the world ray), $T$ (transmittance), and `is_transmission` (boolean) [S1, unofficial].
- Or `std::nullopt` if the ray does not intersect the volume or the index-space bounding box [S2, unofficial].

**Data Structures:**
- Ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ [S1, unofficial].
- Hit structure containing $t$, $T$, and `is_transmission` [S1, unofficial].
- Index ray with parameters transformed into the voxel grid's index space [S2, unofficial].

**Steps:**
1. **Boundary Intersection**: Intersect ray with volume boundary to obtain entry ($t_{\text{in}}$) and exit ($t_{\text{out}}$) hits [S1, unofficial].
2. **Clamping**: Clamp $t_{\text{in}}$ to $t_{\min}$ and $t_{\text{out}}$ to $t_{\max}$ [S1, unofficial]. If $t_{\text{in}} < 0$, set $t_{\text{in}} = 0$ [S1, unofficial].
3. **Coordinate Conversion**: Convert world ray origin and direction to index space using the grid's transform [S2, unofficial].
4. **Index Space Clipping**: Clip the index ray against the density grid's index bounding box; if it fails, return `std::nullopt` [S2, unofficial].
5. **Distance Calculation**: Compute ray length $\|\mathbf{d}\|$ and distance inside boundary $L_{\text{inside}} = (t_{\text{out}} - t_{\text{in}}) \cdot \|\mathbf{d}\|$ [S1, unofficial].
6. **Free Path Sampling**: Sample interaction distance $d_{\text{hit}} = -\frac{1}{\sigma_t} \ln(r)$ where $r$ is a random value [S1, unofficial].
7. **Interaction Decision**: If $d_{\text{hit}} > L_{\text{inside}}$, compute $T = \exp(-\sigma_t \cdot L_{\text{inside}})$, return exit hit at $t_{\text{out}}$ with `is_transmission = true` [S1, unofficial]. Otherwise, compute $t_{\text{hit}} = t_{\text{in}} + \frac{d_{\text{hit}}}{\|\mathbf{d}\|}$, $T = \exp(-\sigma_t \cdot d_{\text{hit}})$, return hit at $t_{\text{hit}}$ with `is_transmission = false` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Hit:
    def __init__(self, t, transmittance, is_transmission):
        self.t = t
        self.transmittance = transmittance
        self.is_transmission = is_transmission

def vector_norm(v):
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)

def homogeneous_volume_intersect(ray_origin, ray_direction, t_min, t_max, 
                                 t_in, t_out, sigma_t, random_generator):
    # Step 1 & 2: Clamping
    t_in_clamped = max(t_min, t_in)
    if t_in_clamped < 0:
        t_in_clamped = 0
    t_out_clamped = min(t_max, t_out)
    
    if t_in_clamped >= t_out_clamped:
        return None

    # Step 3: Coordinate Conversion (Index Space)
    # Apply grid's world-to-index affine transform [G]
    # index_ray_origin = (ray_origin - grid_origin) / grid_spacing
    # index_ray_direction = ray_direction / grid_spacing
    # index_ray = Ray(index_ray_origin, index_ray_direction)

    # Step 4: Index Space Clipping
    # if not index_ray.clip_against_index_bounding_box():
    #     return None

    # Step 5: Distance Calculation
    dir_norm = vector_norm(ray_direction)
    distance_inside_boundary = (t_out_clamped - t_in_clamped) * dir_norm

    # Step 6: Free Path Sampling
    r = random_generator()
    hit_distance = -1.0 / sigma_t * math.log(r)

    # Step 7: Interaction Decision
    if hit_distance > distance_inside_boundary:
        transmittance = math.exp(-sigma_t * distance_inside_boundary)
        return Hit(t=t_out_clamped, transmittance=transmittance, is_transmission=True)
    else:
        t_hit = t_in_clamped + hit_distance / dir_norm
        transmittance = math.exp(-sigma_t * hit_distance)
        return Hit(t=t_hit, transmittance=transmittance, is_transmission=False)
```

**Parameters:**
- $\sigma_t$: Extinction coefficient. Must be $> 0$ [G].
- $r$: Random number. Standard inverse transform sampling requires $r \in (0, 1]$ [G].

**Edge Cases:**
- **Zero Density**: If $\sigma_t = 0$, the formula for $d_{\text{hit}}$ divides by zero [G].
- **Ray Origin Inside**: If the ray origin is inside the volume, $t_{\text{in}}$ may be negative or zero [S1, unofficial]. The clamping logic handles this by ensuring $t_{\text{in}} \ge 0$ relative to the ray's valid range [S1, unofficial].
- **Degenerate Ray**: If $\|\mathbf{d}\| = 0$, the ray has no direction [G]. Division by zero occurs in $t_{\text{hit}}$ calculation [G].

## Subtopics

## Operating characteristics
- **Time Complexity**: $O(1)$ for the intersection calculation itself, assuming constant-time random number generation and bounding box checks [G].
- **Memory Complexity**: $O(1)$ auxiliary space [G].
- **Performance**: Significantly faster than heterogeneous intersection because it avoids the overhead of querying a density field and performing rejection sampling with a majorant [S1, unofficial][S2, unofficial].
- **Strengths**: Simple implementation, low computational cost, exact sampling for uniform media [G].
- **Limitations**: Cannot model spatially varying density [G].
- **When to use**: When simulating uniform fog, smoke, or participating media where density is constant [G].
- **When not to use**: For clouds, smoke with varying density, or complex atmospheric effects requiring spatial variation [G].

## Background
Homogeneous volume intersection is a fundamental technique in volumetric path tracing [G]. It relies on the analytical solution to the radiative transfer equation for constant coefficients [G]. The method of sampling the free path length using the inverse transform sampling of the exponential distribution is standard in Monte Carlo rendering [S1, unofficial]. The integration of index-space clipping ensures robustness against grid boundaries [S2, unofficial].

## Sources
[S1] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
[S2] Shibodd/volume_path_tracer, https://github.com/Shibodd/volume_path_tracer, fetched 2026-10-26 (unofficial)
