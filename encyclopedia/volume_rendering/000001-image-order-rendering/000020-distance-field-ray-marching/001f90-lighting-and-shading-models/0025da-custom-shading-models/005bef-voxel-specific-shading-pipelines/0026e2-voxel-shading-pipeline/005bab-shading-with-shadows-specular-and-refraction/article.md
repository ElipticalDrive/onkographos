# Shading with Shadows, Specular, and Refraction

Also called: Voxel Shading with Shadows, Specular, and Refraction

## Overview
This method computes the final color of a hit voxel by combining diffuse and specular lighting from multiple light sources, handling shadow casting to occlude lights, and managing refraction through volumetric structures. It extends the basic voxel shading pipeline by adding shadow ray casting, Phong-like specular highlights, and recursive or discrete refraction bounces. It is distinguished from simpler shading models by its support for recursive light transport through semi-transparent volumes and its specific handling of voxel-space coordinate offsets for shadow and refraction ray origins.

## How it works

### Inputs and Outputs
**Inputs:**
- `rendering_attributes`: Contains `light_count`, array of light sources (each with `diffuse` and `specular` color), `ambient_colour`, and flags for shadow/refraction enablement.
- `hit`: The hit point structure from the ray marcher, containing `hit_pos` (world position), `hit_scale` (log2 of voxel size or similar scale factor), and `hit_t` (distance along ray).
- `shading_attributes`: Contains `normal` (surface normal at hit point), `colour` (RGBA color sampled from voxel, where `a` is alpha), `refractive_index` (material refractive index).
- `eye_ray`: The primary ray, containing `direction` (view direction).
- `tree`: The volume data structure for ray casting.
- `stack`: Auxiliary stack for recursive ray casting.
- `recursions`: Current recursion depth counter.

**Outputs:**
- `out`: A `vec4` representing the final computed color (RGB + alpha, though alpha handling in output is implicit in the accumulation logic).

### Data Structures
The method relies on structures defined in the parent [Voxel Shading Pipeline](3898-voxel-shading-pipeline.md) and the ray marcher. Key fields include:
- `hit.hit_scale`: Used to compute geometric offsets. `MAX_SCALE` is a constant defining the maximum scale (smallest voxel). `HALF_SQRT_2` is a constant $\frac{\sqrt{2}}{2}$.
- `shading_attributes.colour.a`: Alpha value of the hit voxel.
- `refraction_mode`: Enum indicating `CONTINUOUS` or `DISCRETE` refraction.
- `refraction_discrete_steps_max`: Maximum number of steps for discrete refraction.
- `refraction_discrete_step`: Step size for discrete refraction.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def shade(rendering_attributes, eye_ray, hit, stack, tree, refractive_index, recursions):
    # 1. Normalize normal
    normal = normalize(shading_attributes.normal)
    
    # 2. Initialize output and view direction
    out = vec4(0.0, 0.0, 0.0, 0.0)
    view_dir = eye_ray.direction
    
    # 3. Iterate over lights
    for i in range(rendering_attributes.light_count):
        light = rendering_attributes.lights[i]
        light_dir = normalize(light.position - hit.hit_pos) # Assuming point light; adjust for directional
        light_distance = exp2f(23.0) # Large number representing infinity
        attenuation = 1.0
        
        # 4. Shadow Ray Casting
        if shadows_enabled and dot(light_dir, normal) < 0.0:
            # Cast shadow ray
            shadow_ray_origin = hit.hit_pos + (-light_dir) * (HALF_SQRT_2 * exp2f(hit.hit_scale - MAX_SCALE))
            shadow_ray = Ray(shadow_ray_origin, -light_dir)
            shadow_hit = raycast(tree, shadow_ray, stack)
            
            # If shadow ray hits something closer than the light, skip this light
            if shadow_hit.hit_scale < MAX_SCALE and shadow_hit.hit_t < light_distance:
                continue
        
        # 5. Diffuse Contribution
        diffuse_factor = attenuation * max(0.0, dot(normal, -light_dir))
        diffuse_colour = shading_attributes.colour.rgb * light.diffuse
        out += vec4(diffuse_factor * diffuse_colour, 0.0)
        
        # 6. Specular Contribution
        if dot(normal, light_dir) < 0.0:
            reflection = reflect(light_dir, normal)
            specular_factor = pow(max(0.0, dot(reflection, view_dir)), 32.0)
            out += vec4(specular_factor * (shading_attributes.colour.rgb * light.specular), 0.0)
            
    # 7. Refraction Handling
    if refraction_enabled and shading_attributes.colour.a < 1.0:
        one_minus_alpha = 1.0 - shading_attributes.colour.a
        out *= shading_attributes.colour.a # Modulate existing color by alpha
        
        # Determine surface normal for refraction (reverse if entering)
        refract_normal = normal
        if dot(normal, eye_ray.direction) > 0.0:
            refract_normal = -normal
            
        # Calculate refraction ray
        eta = refractive_index / shading_attributes.refractive_index
        refract_dir = refract(eye_ray.direction, refract_normal, eta)
        refract_ray_origin = hit.hit_pos + refract_dir * (HALF_SQRT_2 * exp2f(hit.hit_scale - MAX_SCALE))
        refract_ray = Ray(refract_ray_origin, refract_dir)
        
        if refraction_mode != CONTINUOUS:
            # Discrete Refraction
            current_refractive_index = shading_attributes.refractive_index
            discrete_ray_origin = refract_ray_origin + refract_dir * (HALF_SQRT_2 * exp2f(hit.hit_scale - MAX_SCALE))
            discrete_ray = Ray(discrete_ray_origin, -refract_dir) # Note: direction negated in source logic
            refracted_times = 0
            
            for step in range(refraction_discrete_steps_max):
                refraction_hit = raycast(tree, discrete_ray, stack)
                
                # Check for exit or invalid hit
                min_float = exp2f(-MAX_SCALE)
                if refraction_hit.hit_scale == MAX_SCALE or refraction_hit.hit_t <= min_float:
                    break
                    
                # Look up attachment (voxel data)
                refraction_attachment = lookup_attachment(tree, refraction_hit)
                
                if refraction_attachment.refractive_index != current_refractive_index:
                    refracted_times += 1
                    eta = current_refractive_index / refraction_attachment.refractive_index
                    
                    # Update normal
                    ref_normal = normalize(refraction_hit.normal)
                    if dot(ref_normal, refract_dir) > 0.0:
                        ref_normal = -ref_normal
                        
                    # Update refraction direction
                    refract_dir = refract(refract_dir, ref_normal, eta)
                    discrete_ray.direction = -refract_dir # Negate as per source logic
                    
                    current_refractive_index = refraction_attachment.refractive_index
                
                # Advance discrete ray
                discrete_ray.origin += refract_dir * refraction_discrete_step
        else:
            # Continuous Refraction (Recursive)
            if refraction_hit.hit_scale >= MAX_SCALE:
                # Hit background or empty space
                out += vec4(one_minus_alpha * rendering_attributes.ambient_colour, 0.0)
            else:
                # Hit another voxel
                refraction_attachment = lookup_attachment(tree, refraction_hit)
                
                # Update normal
                ref_normal = normalize(refraction_hit.normal)
                if dot(ref_normal, refract_dir) > 0.0:
                    ref_normal = -ref_normal
                    
                # Calculate new refraction direction
                eta = refraction_attachment.refractive_index / refractive_index
                refract_dir = refract(refract_dir, ref_normal, eta)
                
                # Set new origin
                refract_ray_origin = refraction_hit.hit_pos + refract_dir * (HALF_SQRT_2 * exp2f(refraction_hit.hit_scale - MAX_SCALE))
                refract_ray = Ray(refract_ray_origin, refract_dir)
                
                # Recursive call
                if refraction_hit.hit_scale < MAX_SCALE:
                    recursive_result = shade(rendering_attributes, refract_ray, refraction_hit, stack, tree, refraction_attachment.refractive_index, recursions + 1)
                    out += recursive_result
                    
    return out
```

### Parameters and Typical Values
- `MAX_SCALE`: Constant defining the smallest voxel scale. `exp2f(MAX_SCALE)` is the minimum voxel size.
- `HALF_SQRT_2`: Constant $\frac{\sqrt{2}}{2} \approx 0.7071$.
- `32.0f`: Exponent for specular highlight (Shininess).
- `exp2f(23)`: Approximation for infinity in shadow ray distance checks.
- `refraction_discrete_steps_max`: Number of steps for discrete refraction. Not given in sources.
- `refraction_discrete_step`: Step size for discrete refraction. Not given in sources.

### Edge Cases and Pitfalls
- **Precision**: The use of `exp2f` for scale calculations requires careful handling of floating-point precision, especially for very small or large scales.
- **Boundary Conditions**: The check `shadow_hit.hit_scale < MAX_SCALE` ensures that only valid voxel hits are considered for occlusion. Hits at `MAX_SCALE` (empty space) are ignored.
- **Normal Orientation**: The normal must be reversed if the dot product with the view/refraction direction is positive, indicating the ray is exiting the surface.
- **Recursion Depth**: The method supports recursive bounces up to a template parameter. Deep recursion can lead to performance issues or stack overflow.
- **Discrete Refraction Logic**: The source code negates the direction in discrete refraction (`discrete_refraction_ray.direction = -refraction_ray.direction`). This appears to be a specific implementation detail that may need verification against the intended physics.

## Operating characteristics

### Time and Memory Complexity
- **Time Complexity**: $O(L \cdot (1 + R))$, where $L$ is the number of lights and $R$ is the number of refraction bounces. Each light requires a shadow ray cast. Refraction adds recursive ray casts.
- **Memory Complexity**: $O(R)$ for the recursion stack, where $R$ is the maximum refraction depth.

### Measured Performance
Not given in the sources.

### Scaling
Performance scales linearly with the number of lights and refraction depth. High refraction depths can significantly impact performance.

### Strengths and Limitations
- **Strengths**: Supports complex lighting effects including shadows, specular highlights, and refraction. Handles semi-transparent volumes.
- **Limitations**: Recursive refraction can be computationally expensive. Discrete refraction logic is complex and may not accurately model continuous refraction.

### When to Use
Use when realistic lighting with shadows and refraction is required for volumetric data. Avoid for real-time applications with strict performance constraints and high refraction depths.

## Background
This method is part of the `catchouli/VolumeRender` project [S1, unofficial]. It extends basic voxel shading to include more advanced lighting effects.

## Sources
[S1] catchouli/VolumeRender, https://github.com/catchouli/VolumeRender, fetched 2026-10-26 (unofficial)
