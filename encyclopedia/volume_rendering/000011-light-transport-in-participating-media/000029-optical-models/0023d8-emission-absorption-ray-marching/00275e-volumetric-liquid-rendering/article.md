# Volumetric Liquid Rendering


## Overview
Volumetric Liquid Rendering is a specific application of emission-absorption ray marching used to render liquid volumes, such as the wax in a lava lamp, illuminated from below [S1, unofficial]. It differs from general volume rendering by masking the liquid visibility with the depth of an occluding surface (wax) and driving emission based on proximity to a light source [S1, unofficial]. It builds on [Emission-Absorption Ray Marching](3171-emission-absorption-ray-marching.md).

## How it works
**Inputs and Outputs:**
- Input: Ray origin and direction, depth buffer of the wax container (occluder), liquid volume density field, light position (below), and material parameters (absorption coefficient, emission intensity).
- Output: Color and opacity of the liquid along the ray.

**Data Structures:**
- Depth Buffer: 2D texture storing the depth of the wax surface.
- Liquid Volume: Implicit or explicit density field. Not given in the sources.

**Steps:**
1. **Ray Casting:** Cast a ray from the camera through the pixel.
2. **Wax Masking:** Intersect the ray with the wax surface using the depth buffer. If the ray does not intersect the wax, or if the intersection is behind the liquid volume, the liquid is not visible. The liquid is masked by the depth of the wax [S1, unofficial].
3. **Liquid Marching:** For the segment of the ray inside the liquid volume:
   a. Sample the liquid density.
   b. Accumulate absorption using the Beer-Lambert law: $T = e^{-\int \sigma_a dl}$, where $\sigma_a$ is the absorption coefficient.
   c. Accumulate emission: $E = \int \sigma_e T_{remaining} dl$, where $\sigma_e$ is the emission coefficient. Emission is driven by the proximity to the light source (below) [S1, unofficial].
4. **Compositing:** Combine the accumulated transmission and emission with the background color.

**Formulas:**
- Transmission: $T = \exp(-\sigma_a \cdot \Delta l)$
- Emission: $E = \sigma_e \cdot (1 - T)$
- Final Color: $C_{out} = C_{in} \cdot T + E$

**Parameters:**
- Absorption coefficient: Not given in the sources.
- Emission intensity: Proportional to proximity to the light source [S1, unofficial].
- Ray step size: Not given in the sources.

**Edge Cases:**
- If the wax depth is not available, the liquid cannot be masked correctly [S1, unofficial].
- Precision issues may arise if the liquid volume is thin or the step size is too large.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_liquid(ray_origin, ray_direction, wax_depth_buffer, liquid_density, light_position, sigma_a, sigma_e):
    # Step 1: Find wax intersection
    wax_intersection = intersect_ray_with_wax(ray_origin, ray_direction, wax_depth_buffer)
    
    # If no wax intersection or wax is behind liquid, return background
    if wax_intersection is None or wax_intersection.z < liquid_volume_min_z:
        return background_color
    
    # Step 2: Determine liquid segment
    liquid_start = max(ray_origin.z, liquid_volume_min_z)
    liquid_end = min(wax_intersection.z, liquid_volume_max_z)
    
    if liquid_start >= liquid_end:
        return background_color
    
    # Step 3: Ray march through liquid
    accumulated_transmission = 1.0
    accumulated_emission = vec3(0.0)
    
    current_z = liquid_start
    step_size = calculate_step_size(liquid_density, ray_direction)
    
    while current_z < liquid_end:
        # Sample density
        density = sample_liquid_density(current_z, ray_origin, ray_direction)
        
        # Calculate transmission for this step
        delta_l = step_size
        transmission = exp(-sigma_a * density * delta_l)
        
        # Calculate emission for this step
        # Emission depends on proximity to light source
        light_proximity = calculate_light_proximity(current_z, ray_origin, ray_direction, light_position)
        emission = sigma_e * light_proximity * (1.0 - transmission)
        
        # Accumulate
        accumulated_emission += accumulated_transmission * emission
        accumulated_transmission *= transmission
        
        current_z += delta_l
    
    # Step 4: Composite
    final_color = background_color * accumulated_transmission + accumulated_emission
    return final_color
```

## Subtopics

### Height Field Ray-Casting
A ray-casting method based on a height field (2D grid containing elevation in each cell) used for real-time rendering of liquid volumes [S2, unofficial]. It allows handling of reflections, refractions, and complex effects like light absorption, refracted shadows, and caustics [S2, unofficial]. It was used by [BD06] [S2, unofficial]. Choose this when real-time performance is required and the liquid surface can be approximated by a height field.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of steps in the ray march.
- **Memory Complexity:** Depends on the storage of the wax depth buffer and liquid density field.
- **Performance:** Not given in the sources.
- **Strengths:** Simple to implement, efficient for simple liquid volumes.
- **Limitations:** Does not handle complex light transport effects like caustics or multiple scattering.

## Background
Volumetric liquid rendering is a technique used in computer graphics to render liquids in real-time applications, such as lava lamps. It was discussed in the context of a lava lamp simulation [S1, unofficial]. The height field ray-casting variant was used by [BD06] [S2, unofficial].

## Sources
[S1] Étape 4 : surface de cire raymarchée et liquide volumique, https://github.com/nathanddrl/lava-lamp/pull/4, fetched 2026-10-27 (unofficial)
[S2] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
