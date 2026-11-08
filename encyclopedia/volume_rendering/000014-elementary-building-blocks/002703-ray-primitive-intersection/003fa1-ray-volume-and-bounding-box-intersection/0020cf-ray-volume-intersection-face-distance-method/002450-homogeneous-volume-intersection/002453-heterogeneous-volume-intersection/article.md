# Heterogeneous Volume Intersection

Also called: Majorant-based Ray Marching

## Overview
Heterogeneous Volume Intersection computes the first interaction point of a ray within a participating medium with spatially varying density, or determines if the ray exits the medium without interacting [S1, unofficial]. It differs from [Homogeneous Volume Intersection](3260-homogeneous-volume-intersection.md) by handling non-constant density fields using a constant majorant density for rejection sampling [S1, unofficial].

## How it works
**Inputs:**
- Ray $r(t) = o + t \cdot d$, where $o$ is the origin, $d$ is the direction, and $t \ge 0$ [G].
- A heterogeneous volume object containing:
  - A boundary geometry defining the spatial extent of the medium [S1, unofficial].
  - A density texture function returning the local density at a point [S1, unofficial].
  - A majorant density, a constant value such that the local density is bounded by it [S1, unofficial].
  - A material for scattering properties [S1, unofficial].
**Outputs:**
- A hit record indicating the first interaction point and the distance along the ray, if an interaction occurs [S1, unofficial].
- A flag indicating whether the ray exited the volume without interaction [S1, unofficial].
- The transmittance from the entry point to the interaction point (or exit point), set to 1.0 in the reference implementation [S1, unofficial].
**Data Structures:**
The volume is represented by a class `HeterogeneousVolume` inheriting from `Object` [S1, unofficial]. Fields: `Object *boundary_` (enclosing geometry), `double density_majorant_` (constant upper bound on density), `Texture *density_texture_` (spatial density function) [S1, unofficial]. Constructor: `HeterogeneousVolume(Object *boundary, double density_majorant, Texture *density_texture, Material *material)` [S1, unofficial].
**Steps:**
1. **Ray-Volume Boundary Intersection:** Intersect the ray with the boundary geometry (`boundary_`) to obtain entry and exit distances, $t_{in}$ and $t_{out}$ [S1, unofficial]. Clamp $t$ values to the ray interval, and handle cases where $t_{in} < 0$ (ray starts inside) [S1, unofficial]. If the ray misses or $t_{in} \ge t_{out}$, return no hit [G].
2. **Initialization:** Set the current distance along the ray, $t_{current} = t_{in}$ [S1, unofficial].
3. **Ray Marching Loop:** Loop indefinitely:
   a. **Sample Step Size:** Generate a uniform random number $u \in (0, 1)$ and calculate the step size $\Delta t = -\ln(u) / \rho_{max}$ [S1, unofficial].
   b. **Advance Ray:** Update the current distance: $t_{current} \leftarrow t_{current} + \Delta t / \|d\|$ [S1, unofficial].
   c. **Check Exit:** If $t_{current} \ge t_{out}$, the ray has exited the volume without an interaction [S1, unofficial]. Return an exit hit with transmittance = 1.0 and `is_transmission = true` [S1, unofficial].
   d. **Evaluate Local Density:** Compute the point $p = r(t_{current})$ and evaluate the density texture at $p$ [S1, unofficial]. Calculate the local density as $\rho_{local} = \rho_{max} \cdot (tex\_val.r + tex\_val.g + tex\_val.b) / 3.0$ [S1, unofficial].
   e. **Rejection Sampling:** Generate a new uniform random number $v \in (0, 1)$ [S1, unofficial]. If $v < \rho_{local} / \rho_{max}$, accept as an interaction [S1, unofficial]. Return a hit at $p$ with transmittance = 1.0 and `is_transmission = false` [S1, unofficial].
   f. **Otherwise:** If the rejection test fails, continue the loop from step 3a [S1, unofficial].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

class HitRecord:
    def __init__(self, point, t, transmittance, is_transmission):
        self.point = point
        self.t = t
        self.transmittance = transmittance
        self.is_transmission = is_transmission

def random_double():
    return random.random()

def ray_at(ray_origin, ray_direction, t):
    ray_length = math.sqrt(ray_direction.x**2 + ray_direction.y**2 + ray_direction.z**2)
    return ray_origin + ray_direction * (t / ray_length)

def heterogeneous_volume_intersection(ray_origin, ray_direction, volume):
    # Intersect ray with boundary to get hit_in and hit_out [S1, unofficial]
    hit_in_t, hit_out_t = volume.boundary_.intersect(ray_origin, ray_direction)
    
    if hit_in_t is None or hit_out_t is None:
        return None  # No intersection with volume
    
    t_in = hit_in_t
    t_out = hit_out_t
    
    # Handle ray starting inside the volume
    if t_in < 0:
        t_in = 0
        
    if t_in >= t_out:
        return None  # Tangent or miss
        
    t_current = t_in
    rho_max = volume.density_majorant_
    ray_length = math.sqrt(ray_direction.x**2 + ray_direction.y**2 + ray_direction.z**2)
    
    # Step 3: Ray Marching Loop
    while True:
        # Step 3a: Sample step size
        u = random_double()
        dt = -math.log(u) / rho_max
        
        # Step 3b: Advance ray
        t_current += dt / ray_length
        
        # Step 3c: Check exit
        if t_current >= t_out:
            return HitRecord(ray_at(ray_origin, ray_direction, t_out), t_out, 1.0, True)
        
        # Step 3d: Evaluate local density
        p = ray_at(ray_origin, ray_direction, t_current)
        tex_val = volume.density_texture_.value(p)
        rho_local = rho_max * (tex_val.r + tex_val.g + tex_val.b) / 3.0
        
        # Step 3e: Rejection sampling
        v = random_double()
        if v < rho_local / rho_max:
            return HitRecord(p, t_current, 1.0, False)
```
**Parameters:**
- `density_majorant_`: Must be $\ge \max(\rho(p))$ for all $p$ in the volume [S1, unofficial].
- `density_texture_`: A texture object implementing a `value(Point3)` method that returns the density at a point [S1, unofficial].
**Edge Cases and Pitfalls:**
- **Precision:** If $\rho_{local}$ is very close to $\rho_{max}$, floating-point errors may affect the rejection test [G].
- **Boundary Intersection:** The boundary intersection must be robust. If the ray originates exactly on the boundary, $t_{in}$ should be handled carefully to avoid infinite loops or missed interactions [G].
- **Majorant Selection:** If $\rho_{max}$ is too large, the acceptance rate $\rho_{local} / \rho_{max}$ becomes low, leading to many rejected steps [S1, unofficial].

## Subtopics

### Heterogeneous Volume Representation
Represents a participating medium with spatially varying density using a texture, bounded by geometry, and using a majorant density for null rejection [S1, unofficial]. It inherits from `Object` and includes fields for the boundary, majorant density, density texture, and material [S1, unofficial]. The constructor takes these four parameters [S1, unofficial]. This representation is essential for the intersection method to function, as it provides the necessary data for density evaluation and boundary intersection [S1, unofficial].

## Operating characteristics
**Time Complexity:**
The expected number of steps depends on the ratio of local density to the majorant density [S1, unofficial].
**Memory Complexity:**
$O(1)$ additional memory beyond the storage for the density texture and boundary geometry [G].
**Strengths:**
- Handles arbitrary density distributions without requiring a uniform grid [S1, unofficial].
- Simple to implement [G].
- Exact sampling (no discretization error in the density field) [G].
**Limitations:**
- Requires a robust boundary intersection routine [G].
**When to Use:**
- When the density field is complex and cannot be easily approximated by a uniform grid [G].
**When Not to Use:**
- When the density field is uniform (use [Homogeneous Volume Intersection](3260-homogeneous-volume-intersection.md)) [G].

## Background
The method implements null rejection sampling for spatially varying density volumes [S1, unofficial]. It is used in volumetric path tracing for rendering participating media [G]. The implementation follows the logic from the provided source code [S1, unofficial].

## Sources
[S1] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
