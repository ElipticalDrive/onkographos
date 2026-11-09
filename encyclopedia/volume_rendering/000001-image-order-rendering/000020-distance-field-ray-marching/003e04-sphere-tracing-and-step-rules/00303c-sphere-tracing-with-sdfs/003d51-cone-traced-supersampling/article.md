# Cone-Traced Supersampling

Also called: Cone-Traced Supersampling With Subpixel Edge Reconstruction, Subpixel Edge Reconstruction (SER)

## Overview
Cone-Traced Supersampling (CTSS) is an efficient spatial antialiasing technique for Signed Distance Function (SDF) rendering that performs supersampling along the traced ray near surfaces with partial visibility, identified by evaluating cone intersections within a pixel's view frustum [S1, unofficial]. It extends the standard sphere tracing algorithm to resolve complex pixels with geometric edges in relatively flat regions via Subpixel Edge Reconstruction (SER), which are otherwise undetected by cone intersections [S1, unofficial]. Unlike conventional supersampling, CTSS does not require casting additional rays per pixel or offline prefiltering, and unlike post-processing filters, it does not produce undesirable blurriness and ghosting [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: The camera ray $r(t) = \mathbf{o} + t\mathbf{d}$, the SDF function $f(\mathbf{p})$, the pixel's view frustum bounds, and the convergence threshold $\epsilon$ [S1, unofficial].
*   **Output**: The final antialiased pixel color [S1, unofficial].

**Data Structures**
*   **Ray State**: Standard ray parameters $\mathbf{o}, \mathbf{d}, t$ [G].
*   **Cone Parameters**: The cone is defined by the ray origin $\mathbf{o}$, direction $\mathbf{d}$, and an angular radius $\alpha$ derived from the pixel's spatial extent at distance $t$ [S1, unofficial].
*   **Sample Buffer**: A list of visibility samples $(t_i, v_i)$ where $v_i \in \{0, 1\}$ indicates if the cone at $t_i$ intersects the surface [S1, unofficial].

**Steps**
1.  **Standard Sphere Tracing**: Perform the standard sphere tracing loop as defined in [Sphere Tracing with SDFs](6223-sphere-tracing-with-sdfs.md) to find the primary hit point $t_{hit}$ and surface normal $\mathbf{n}$ [S1, unofficial].
2.  **Cone Intersection Test**: For each step $t_i$ along the ray (or specifically near the hit), evaluate if the cone defined by the pixel's footprint at $t_i$ intersects the SDF surface. The intersection condition is met if the distance from the ray to the surface is less than the cone radius [S1, unofficial].
    *   Let $d_{ray} = f(\mathbf{o} + t_i \mathbf{d})$.
    *   Let $r_{cone}(t_i)$ be the radius of the pixel's footprint at distance $t_i$.
    *   Intersection if $d_{ray} < r_{cone}(t_i)$ [S1, unofficial].
3.  **Identify Contours**: Mark steps where the cone intersection condition is true as "partial visibility" or object contours [S1, unofficial].
4.  **Subpixel Edge Reconstruction (SER)**: For pixels identified as complex edges in relatively flat regions (where cone intersections might miss fine details), apply SER to locate and resolve the geometric edges [S1, unofficial]. This involves refining the sampling near the detected contours [S1, unofficial].
5.  **Supersampling**: Perform supersampling along the traced ray specifically at the identified contour steps [S1, unofficial].
6.  **Aggregation**: Correlate sample visibility to aggregate the samples for the final pixel color [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cone_traced_supersampling(ray_o, ray_d, sdf, epsilon, t_max):
    """
    Implements Cone-Traced Supersampling (CTSS) with Subpixel Edge Reconstruction (SER).
    """
    t = 0.0
    samples = []
    
    # Standard sphere tracing loop
    while t < t_max:
        p = ray_o + t * ray_d
        dist = sdf(p)
        
        # Termination check
        if dist < epsilon:
            # Hit surface
            break
        if dist > t_max - t:
            # Escaped
            break
            
        # Cone intersection test for antialiasing
        # Calculate cone radius at current distance t
        # Assuming a perspective camera, cone radius grows with t
        # This is a simplified model; exact formula depends on camera FOV and pixel size
        cone_radius = calculate_cone_radius(t, camera_fov, pixel_size)
        
        # Check if the cone intersects the surface
        # If the SDF value is less than the cone radius, the pixel is partially covered
        is_partial_visibility = (dist < cone_radius)
        
        if is_partial_visibility:
            # Store sample for supersampling
            samples.append({'t': t, 'dist': dist, 'is_partial': True})
        else:
            samples.append({'t': t, 'dist': dist, 'is_partial': False})
            
        # Standard sphere tracing step
        t += dist
        
    # Subpixel Edge Reconstruction (SER) for complex pixels
    # If the pixel is identified as having complex edges in flat regions,
    # refine the samples near the contours
    if is_complex_pixel(samples):
        samples = subpixel_edge_reconstruction(samples, sdf, ray_o, ray_d)
        
    # Aggregate samples
    final_color = aggregate_samples(samples, sdf, ray_o, ray_d)
    
    return final_color

def calculate_cone_radius(t, camera_fov, pixel_size):
    """
    Calculates the radius of the pixel's footprint (cone) at distance t.
    """
    # Simplified formula: radius = t * tan(fov/2) * (pixel_size / screen_width)
    # This is a placeholder for the exact geometric derivation
    return t * math.tan(camera_fov / 2.0) * (pixel_size / 1000.0) # [G]

def is_complex_pixel(samples):
    """
    Determines if the pixel has complex edges in relatively flat regions.
    """
    # Check for high variance in partial visibility samples
    partial_samples = [s for s in samples if s['is_partial']]
    if len(partial_samples) == 0:
        return False
    # If there are many partial samples but the SDF values are relatively constant (flat region)
    sdf_values = [s['dist'] for s in partial_samples]
    variance = sum((v - sum(sdf_values)/len(sdf_values))**2 for v in sdf_values) / len(sdf_values)
    return variance < threshold_flat_region # [G]

def subpixel_edge_reconstruction(samples, sdf, ray_o, ray_d):
    """
    Refines samples near contours to resolve complex edges.
    """
    refined_samples = []
    for sample in samples:
        if sample['is_partial']:
            # Perform additional supersampling near this t
            # This is a placeholder for the specific SER algorithm
            refined_samples.extend(refine_sample(sample, sdf, ray_o, ray_d)) # [G]
        else:
            refined_samples.append(sample)
    return refined_samples

def refine_sample(sample, sdf, ray_o, ray_d):
    """
    Performs additional supersampling near a contour sample.
    """
    # Placeholder for SER logic
    return [sample] # [G]

def aggregate_samples(samples, sdf, ray_o, ray_d):
    """
    Correlates sample visibility to aggregate the final pixel color.
    """
    # Weighted average based on visibility
    total_weight = 0.0
    weighted_color = 0.0
    for sample in samples:
        # Visibility is 1 if partial, 0 if not (simplified)
        visibility = 1.0 if sample['is_partial'] else 0.0
        # Compute shading at sample point
        color = compute_shading(sdf, ray_o + sample['t'] * ray_d)
        weighted_color += color * visibility
        total_weight += visibility
    if total_weight == 0:
        return 0.0
    return weighted_color / total_weight

def compute_shading(sdf, p):
    """
    Computes shading at point p.
    """
    # Placeholder for shading logic
    return 0.5 # [G]
```

**Parameters**
*   $\epsilon$: Convergence threshold, typically $0.001$ [S1, unofficial].
*   $t_{max}$: Maximum ray distance [S1, unofficial].
*   Camera FOV and Pixel Size: Used to calculate cone radius [S1, unofficial].

**Edge Cases and Pitfalls**
*   Complex pixels with geometric edges in relatively flat regions are otherwise undetected by cone intersections; this is handled by Subpixel Edge Reconstruction (SER) [S1, unofficial].
*   Precision issues may arise when calculating cone radii for very small or very large distances [G].

## Operating characteristics
*   **Time Complexity**: CTSS performs supersampling along the traced ray near surfaces with partial visibility, which adds computational cost compared to standard sphere tracing but is significantly lower than conventional supersampling which requires casting additional rays per pixel [S1, unofficial].
*   **Memory Complexity**: Requires storing sample visibility data for aggregation [S1, unofficial].
*   **Strengths**: Efficient and robust spatial antialiasing; naturally complements sphere tracing; does not require casting additional rays per pixel or offline prefiltering; does not produce undesirable blurriness and ghosting [S1, unofficial].
*   **Limitations**: May miss complex pixels with geometric edges in relatively flat regions without SER [S1, unofficial].
*   **When to use**: When high-quality antialiasing is needed for SDF rendering in real-time applications without the cost of conventional supersampling [S1, unofficial].
*   **When not to use**: When computational resources are extremely limited and antialiasing is not critical [G].

## Background
Cone-Traced Supersampling (CTSS) was introduced as an efficient spatial antialiasing solution for SDF rendering [S1, unofficial]. It was developed to address the limitations of conventional supersampling and post-processing filters in the context of real-time SDF renderers [S1, unofficial].

## Sources
[S1] Accelerating Sphere Tracing, https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing, fetched 2026-11-04 (unofficial)
