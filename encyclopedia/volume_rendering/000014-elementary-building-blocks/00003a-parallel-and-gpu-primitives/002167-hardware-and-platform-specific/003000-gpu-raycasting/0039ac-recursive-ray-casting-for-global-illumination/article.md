# Recursive Ray-casting for Global Illumination

Also called: GPU-based Volume Ray-casting with Specular Reflection and Refraction, Higher Order Entry and Exit Points

## Overview
Recursive Ray-casting for Global Illumination is an extension of GPU-based volume ray-casting that computes specular reflection and refraction effects at interactive frame rates by processing a ray-caster multiple times with different entry and exit points, supporting both Direct Volume Rendering (DVR) and isosurface rendering [S1]. Unlike techniques that focus on the exchangeability of ray-casters, this method aims to combine specular reflection and refraction effects within a single image [S1].

## How it works
The method extends a standard GPU-based volume ray-caster into a ray-tracer by tracing rays of higher order. The workflow is subdivided into four subsequent stages [S1]:

1.  **Stage 1: Initial Ray Casting.** Compute initial unmodified entry and exit points for first-order rays. Cast rays from the entry point to the first hit point (the position where the ray first encounters a visible medium). Sample one step further into the volume at the hit point to increase the footprint of encountered voxels, ensuring the encountered medium is sufficiently penetrated and clearly visible in the intermediate image [S1].
2.  **Stage 2: Higher-Order Point Generation.** For each recursive step, compute higher-order entry and exit points based on the first hit points of the previous step. The exit points for the next recursion step are computed by determining the intersection between the higher-order ray and the bounding box of the volume [S1].
3.  **Stage 3: Intermediate Rendering.** A ray-caster computes intermediate images containing shading results for a ray segment based on the generated entry and exit points. The intermediate image is cached for blending in the final stage [S1].
4.  **Stage 4: Blending.** Blend all available intermediate shading images into the final image using a specularity mapping function [S1].

### Ray Direction Computation
To compute the ray direction $d$ for rays reflected on a specular surface, the normal of the surface is considered. Similar to the Phong illumination model, the outgoing ray is computed by considering the incoming ray and the current surface normal [S1].

For refraction, the incoming ray is bent at the surface based on Snell's law. The bending angle $\theta$ is computed based on the incoming angle $\phi$ and the refraction indices $n_1$ (medium left by the ray) and $n_2$ (medium entered by the ray) [S1]:

$$
\cos(\theta) = \sqrt{\left(1 - \left(\frac{n_1}{n_2}\right)^2\right) \cdot \left(1 - (\cos(\phi))^2\right)}
$$

The direction of the leaving ray $A_{vec}$ is computed as follows [S1]:

$$
A_{vec} = \frac{n_1}{n_2} E_{vec} + \left(|\cos(\phi)| - \cos(\theta)\right) N_{vec}
$$

where $E_{vec}$ is the incoming ray vector and $N_{vec}$ is the normalized gradient (surface normal) [S1].

### Total Reflection Handling
Total reflection must be considered when the incoming ray hits the object under a very flat angle, resulting in specular reflection instead of refraction. The critical angle $\phi_{crit}$ is computed as [S1]:

$$
\phi_{crit} = \sin^{-1}\left(\frac{n_2}{n_1}\right)
$$

If the incident angle $\phi$ exceeds $\phi_{crit}$, a specular reflection ray is computed instead of a refractive one [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def recursive_volume_ray_casting(volume_data, camera, max_recursion_depth):
    # Stage 1: Initial setup
    # Compute initial unmodified entry and exit points for first-order rays
    first_order_entry_points = compute_entry_points(camera)
    first_order_exit_points = compute_exit_points(camera, volume_bounding_box)
    
    # Cache for intermediate images
    intermediate_images = []
    
    # Current entry/exit points for the loop
    current_entry_points = first_order_entry_points
    current_exit_points = first_order_exit_points
    
    for depth in range(max_recursion_depth + 1):
        # Stage 3: Ray-casting for current segment
        # Cast rays from current_entry to current_exit
        # For depth 0, this is the primary ray.
        # For depth > 0, this is the higher-order ray segment.
        
        # If depth > 0, we need to adjust the entry point to be the "first hit point"
        # from the previous iteration's shading result.
        if depth > 0:
            # The entry point for this segment is the first hit point found in the previous stage
            # This requires storing first_hit_points from the previous iteration
            current_entry_points = previous_first_hit_points
            
            # Stage 2: Exit point generation for higher-order rays
            # Compute intersection between the ray (defined by entry and direction)
            # and the bounding box of the volume.
            # Note: The direction 'd' for higher order rays is determined by
            # reflection/refraction logic applied at the previous hit point.
            current_exit_points = compute_higher_order_exit_points(current_entry_points, ray_directions, volume_bounding_box)
        
        # Perform volume ray-casting for the segment [current_entry, current_exit]
        # This produces an intermediate image with shading results
        intermediate_image = ray_cast_segment(volume_data, current_entry_points, current_exit_points, ray_directions)
        
        # Store intermediate image for blending
        intermediate_images.append(intermediate_image)
        
        # Compute first hit points for the NEXT recursion step
        # Sample one step further into the volume at the hit point to increase footprint
        previous_first_hit_points = compute_first_hit_points_with_extension(volume_data, current_entry_points, current_exit_points, ray_directions)
        
        # Compute ray directions for the next recursion step (Reflection/Refraction)
        # This depends on the surface normal at the first hit point
        ray_directions = compute_next_ray_directions(previous_first_hit_points, volume_data, ray_directions)

    # Stage 4: Blending
    # Blend all available intermediate shading images into the final image
    # using a specularity mapping function
    final_image = blend_intermediate_images(intermediate_images, specularity_mapping_function)
    
    return final_image

def compute_next_ray_directions(hit_points, volume_data, incoming_directions):
    next_directions = []
    for hit_point, incoming_dir in zip(hit_points, incoming_directions):
        # Get surface normal (normalized gradient) at hit point
        normal = normalize(sample_gradient(volume_data, hit_point))
        
        # Compute incoming angle phi
        # phi is the angle between the incoming ray and the normal
        # cos(phi) = dot(incoming_dir, normal) ? 
        # Note: Standard geometry usually defines phi relative to the normal.
        # The formula uses cos(phi).
        cos_phi = dot(incoming_dir, normal)
        phi = acos(cos_phi)
        
        # Determine refraction indices n1 (left) and n2 (entered)
        # This requires knowing if we are entering or leaving a medium.
        # Assuming n1 is the medium of the incoming ray, n2 is the medium at the hit point.
        n1 = get_refraction_index(volume_data, hit_point - epsilon * incoming_dir)
        n2 = get_refraction_index(volume_data, hit_point)
        
        # Check for total reflection
        phi_crit = asin(n2 / n1)
        
        if phi > phi_crit:
            # Specular reflection
            # R = I - 2 * dot(I, N) * N
            # Note: incoming_dir points TOWARDS the surface.
            # Reflection formula: R = incoming - 2 * dot(incoming, normal) * normal
            # However, standard Phong often uses view vector. Here we trace rays.
            # If incoming_dir is the direction OF the ray, it points into the surface.
            # The reflected ray direction is:
            reflected = incoming_dir - 2 * dot(incoming_dir, normal) * normal
            next_directions.append(reflected)
        else:
            # Refraction
            # Compute bending angle theta
            cos_theta = sqrt((1 - (n1/n2)**2) * (1 - cos_phi**2))
            
            # Compute leaving ray direction A_vec
            # A_vec = (n1/n2) * E_vec + (|cos(phi)| - cos(theta)) * N_vec
            # E_vec is incoming ray vector.
            # Note: The formula in the source uses E_vec.
            # We must ensure vector directions are consistent.
            # If E_vec points towards the surface, and N_vec points out,
            # the formula yields the direction of the refracted ray.
            abs_cos_phi = abs(cos_phi)
            leaving_dir = (n1/n2) * incoming_dir + (abs_cos_phi - cos_theta) * normal
            next_directions.append(leaving_dir)
            
    return next_directions
```

### Parameters
-   **Sampling Rate:** Set to twice the maximum grid resolution of the rendered data set [S1].

### Edge Cases and Pitfalls
-   **Total Reflection:** If the incident angle $\phi$ exceeds the critical angle $\phi_{crit}$, refraction is not possible, and a specular reflection ray must be computed instead [S1].
-   **Footprint Extension:** Sampling one step further into the volume at the first hit point is required to increase the footprint of encountered voxels; failing to do so may result in the encountered medium not being sufficiently penetrated or visible [S1].
-   **Refraction Indices:** The refraction indices $n_1$ and $n_2$ must be correctly identified for the medium left by the ray and the medium entered by the ray, respectively [S1].

## Operating characteristics
-   **Performance:** On a GeForce 8800GTX, the method achieves 57 fps for the Hand dataset ($256 \times 256 \times 147$) at $256 \times 256$ resolution with recursion depth 0, and 10 fps for the Cornell Box dataset ($256 \times 256 \times 256$) at $512 \times 512$ resolution with recursion depth 3 [S1].
-   **Scalability:** The recursion depth influences the ray length to be traversed, while screen resolution influences the number of rays to be traced. Interactive frame rates are achieved for moderately sized data sets at $512 \times 512$ resolution [S1].
-   **Strengths:** It allows for the computation of specular reflection and refraction effects at interactive frame rates on commodity graphics hardware and does not constrain the type of rendering (supports both DVR and isosurface rendering) [S1].
-   **Limitations:** Performance decreases with higher recursion depths and larger screen resolutions [S1].

## Background
The technique was presented in a paper extending GPU-based volume ray-casting to support specular reflection and refraction [S1]. It was integrated into the Voreen volume rendering engine [S1]. The work was partly supported by grants from the Deutsche Forschungsgemeinschaft (DFG), SFB 656 MoBil Münster, Germany [S1]. It contrasts with Stegmaier et al. (2005), who focused on the exchangeability of ray-casters, by aiming to combine specular reflection and refraction effects within a single image [S1].

## Sources
[S1] GPU-based Volume Ray-casting Supporting Specular Reflection and Refraction, https://pdfs.semanticscholar.org/ba7b/55980c8d8ecac4aa1035eb4ae7c16c85da24.pdf, fetched 2026-10-27
