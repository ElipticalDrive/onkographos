# Light Transmittance Marching

Also called: Light Marching, Light Transmissance March, Nested Light Ray Sampling, Nested Raymarching for Light Sampling, Secondary light march, Shadow Raymarching in Volume

## Overview
Light Transmittance Marching is a technique to compute physically accurate lighting in a volume by casting a secondary raymarch from each sample point on the primary view ray towards the light source to calculate absorbed light (transmittance) [S1, S2, unofficial]. It solves the problem of volumetric occlusion, where light passing through the volume before reaching a sample point is attenuated by the medium itself [S2, unofficial]. Unlike [Front-to-Back Volume Rendering](2350-front-to-back-volume-rendering.md), which typically assumes uniform or pre-computed lighting, this method dynamically computes the light energy arriving at each sample point by accounting for the density along the path to the light [S1, unofficial].

## How it works
**Inputs:**
- `rayOrigin`: The camera position [S1, unofficial].
- `rayDirection`: The normalized direction of the primary view ray [S1, unofficial].
- `lightDirection`: The normalized direction vector from the scene to the light source (e.g., the sun) [S1, unofficial].
- `scene(pos, isLightSample)`: A function that returns the density at position `pos`. The boolean flag distinguishes between primary ray sampling and light sampling, allowing for different density evaluation strategies or optimizations [S1, unofficial].
- `MAX_STEPS`: The number of steps in the primary raymarch loop [S1, unofficial].
- `MAX_STEPS_LIGHTS`: The number of steps in the nested light sampling loop [S1, unofficial].
- `MARCH_SIZE`: The step size for the primary raymarch [S1, unofficial].
- `marchSize` (or `stepSize`): The step size for the nested light sampling loop [S1, unofficial].
- `ABSORPTION_COEFFICIENT`: The coefficient used in Beer's Law for light absorption [S1, unofficial].

**Outputs:**
- `lightEnergy`: The accumulated luminance for the pixel, incorporating the attenuated light contribution from each sample point along the primary ray [S1, unofficial].

**Data Structures:**
- No complex data structures are required beyond standard scalar variables for position, density, and accumulated energy [S1, unofficial]. The method relies on iterative sampling of the `scene` function [S1, unofficial].

**Parameters:**
- `MAX_STEPS_LIGHTS`: Set to 6 to preserve frame-rate [S1, unofficial].
- `ABSORPTION_COEFFICIENT`: Set to 0.9 [S1, unofficial].
- `marchSize` (light step): Set to 0.03 [S1, unofficial].
- `MAX_STEPS`: Set to 50 [S1, unofficial].
- `MARCH_SIZE` (primary step): Set to 0.16 [S1, unofficial].

**Steps:**
1. Initialize the primary raymarch loop. For each step `i` from 0 to `MAX_STEPS - 1`:
    a. Calculate the current position `p` along the primary ray:
       $$ p = rayOrigin + depth \cdot rayDirection $$
       where `depth` accumulates `MARCH_SIZE` at each step in the primary loop [S1, unofficial].
    b. Sample the density at `p` using the primary scene function:
       $$ density = scene(p, false) $$
       [S1, unofficial].
    c. If `density > 0.0`:
       i. Initiate a nested light sampling loop to calculate light transmittance [S1, unofficial].
       ii. Initialize `totalDensity = 0.0` and `position = p` [S1, unofficial].
       iii. For each step `j` from 0 to `MAX_STEPS_LIGHTS - 1`:
            1. Advance the position along the light direction:
               $$ position = position + lightDirection \cdot marchSize \cdot j $$
               [S1, unofficial].
            2. Sample the density at the new position using the light-specific scene function:
               $$ lightSample = scene(position, true) $$
               [S1, unofficial].
            3. Accumulate the density:
               $$ totalDensity = totalDensity + lightSample $$
               [S1, unofficial].
            *Note: An alternative implementation accumulates density weighted by step size: `totalDensity += density * stepSize` where `stepSize = MAX_LIGHT_DISTANCE / NUM_LIGHT_STEPS` [S2, unofficial].*
       iv. Calculate the light transmittance using Beer's Law:
           $$ transmittance = e^{-ABSORPTION\_COEFFICIENT \cdot totalDensity} $$
           [S1, unofficial].
           *Note: Some implementations use `exp(-totalDensity)` directly if the absorption coefficient is incorporated into the density or step size [S2, unofficial].*
       v. Multiply the primary ray's accumulated transmittance by the light transmittance:
           $$ totalTransmittance = totalTransmittance \cdot transmittance $$
           [S1, unofficial].
       vi. Calculate the luminance contribution:
           $$ luminance = density $$
           [S1, unofficial].
       vii. Add to the total light energy:
           $$ lightEnergy = lightEnergy + totalTransmittance \cdot luminance $$
           [S1, unofficial].
    d. Advance the primary ray depth:
       $$ depth = depth + MARCH_SIZE $$
       [S1, unofficial].

2. Return `lightEnergy` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def light_transmittance_march(ray_origin, ray_direction, light_direction, scene_func, max_steps=50, max_steps_lights=6, march_size=0.16, light_march_size=0.03, absorption_coefficient=0.9):
    """
    Computes light energy for a single ray using nested light transmittance marching.
    
    Args:
        ray_origin: vec3, camera position.
        ray_direction: vec3, normalized view ray direction.
        light_direction: vec3, normalized direction to the light source.
        scene_func: function(pos, is_light_sample) -> float, returns density.
        max_steps: int, number of steps in primary raymarch.
        max_steps_lights: int, number of steps in nested light march.
        march_size: float, step size for primary raymarch.
        light_march_size: float, step size for light march.
        absorption_coefficient: float, Beer's law coefficient.
        
    Returns:
        float, accumulated light energy.
    """
    light_energy = 0.0
    total_transmittance = 1.0
    depth = 0.0
    
    for i in range(max_steps):
        p = ray_origin + depth * ray_direction
        density = scene_func(p, False)
        
        if density > 0.0:
            total_density_light = 0.0
            light_pos = p
            
            for j in range(max_steps_lights):
                light_pos = light_pos + light_direction * light_march_size * j
                light_sample = scene_func(light_pos, True)
                total_density_light += light_sample
            
            transmittance = math.exp(-absorption_coefficient * total_density_light)
            total_transmittance *= transmittance
            luminance = density
            light_energy += total_transmittance * luminance
        
        depth += march_size
        
    return light_energy
```

**Edge Cases and Pitfalls:**
- **Performance:** The nested loop significantly increases algorithmic complexity. To maintain frame-rate, `MAX_STEPS_LIGHTS` must be kept low (e.g., 6) [S1, unofficial]. This may require reducing the precision of the primary raymarch (e.g., reducing FBM octaves) [S1, unofficial].
- **Precision:** Using a small `marchSize` for the light march (0.03) helps accuracy but increases the number of samples if `MAX_STEPS_LIGHTS` is not adjusted [S1, unofficial].
- **Boundary Conditions:** The light march assumes the light source is at infinity or the direction is constant [S1, unofficial]. If the light source is point-based and close, the direction `lightDirection` should be recalculated for each sample point, which is not covered in the basic implementation [S1, S2, unofficial].

## Operating characteristics
- **Time Complexity:** The algorithmic complexity increases due to the nested loop, requiring trade-offs like reducing primary steps or FBM octaves to preserve frame-rate [S1, unofficial].
- **Memory Complexity:** Auxiliary space is $O(1)$, using only scalar variables for accumulation [S1, unofficial].
- **Measured Performance:** With `MAX_STEPS=50` and `MAX_STEPS_LIGHTS=6`, the method is feasible for real-time rendering on modern GPUs, provided the scene function is efficient [S1, unofficial].
- **Strengths:** Physically accurate lighting attenuation within the volume; simple to implement as a nested loop [S1, unofficial].
- **Limitations:** High computational cost due to nested sampling; assumes uniform light direction (directional light); does not account for scattering within the volume, only absorption [S1, unofficial].
- **When to Use:** When directional lighting and volumetric occlusion are required for real-time applications, and performance constraints allow for nested sampling [S1, unofficial].
- **When Not to Use:** Not given in the sources.

## Background
Light Transmittance Marching was introduced in the context of real-time volumetric cloud rendering to address the lack of proper lighting models in early raymarching demos [S1, unofficial]. It builds on the concept of Beer's Law for light absorption in participating media [S1, unofficial].

## Sources
[S1] Real-time dreamy Cloudscapes with Volumetric Raymarching, https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/, fetched 2026-10-28 (unofficial)
[S2] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
