# Plane-Based Sampling

Also called: Plan-Based Sampling Method, Plane-Based Sampling for Ray Casting

## Overview
Plane-Based Sampling is a ray casting optimization technique that replaces per-voxel trilinear interpolation with a two-stage process: calculating ray-plane intersections and then interpolating between those intersection points using definite proportions [S1]. It solves the high computational cost of traditional Ray Casting Algorithms (RCA) by reducing the number of arithmetic operations required to determine optical properties at sampling points [S1]. Unlike standard methods that sample within every voxel individually, this method leverages the geometry of equidistant parallel planes to guarantee image density while significantly accelerating rendering speed [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (discrete samples), a ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and a sampling interval $\Delta s$.
*   **Output:** The accumulated color and opacity for each pixel along the ray.

**Data Structures:**
The method relies on the geometric structure of the volume data, specifically treating it as a cluster of equidistant parallel planes. No additional complex data structures (like grids or trees) are required beyond the standard volume texture/array [S1].

**Steps:**
1.  **Ray Casting:** Cast a ray through the volume from back to front for each output image pixel [S1].
2.  **Plane Identification:** Identify the group of equidistant parallel planes in the volume data field that the ray travels through [S1].
3.  **Intersection Calculation:** Calculate the intersection points where the ray intersects these parallel planes. Let the intersection points be $I_k$ with associated optical properties (color $C_k$ and opacity $\alpha_k$) derived from the volume data at those specific locations [S1].
4.  **Proportional Interpolation:** For sampling points located between adjacent intersection points, calculate their optical properties using the formula of definite proportion and separated points based on the properties of the intersection points [S1].

**Formulas:**
The traditional RCA requires 17 additions and 16 multiplications per sampling point to locate the voxel and perform trilinear interpolation [S1]. The Plane-Based Sampling method simplifies this. While the exact operation count for the new method is not explicitly broken down into additions/multiplications in the source, it is described as "faster" and reducing the number of arithmetic operations [S1].

The interpolation between intersection points uses "definite proportion and separated points." If a sampling point $S$ lies between two intersection points $I_a$ and $I_b$, its value is interpolated based on the distance ratios. The source provides a generalized structure for linear interpolation between points but does not provide a single closed-form equation for the entire pipeline, instead describing the logic: "properties of sampling points between adjacent intersection points can be calculated by the formula of definite proportion and separated points" [S1].

**Parameters:**
*   **Sampling Interval ($\Delta s$):** The distance between adjacent parallel planes. The method requires this distance to not far surpass the desired sampling interval to guarantee image density [S1].

**Edge Cases and Pitfalls:**
*   **Plane Spacing:** If the distance between adjacent parallel planes is significantly larger than the desired sampling interval, the method fails to guarantee image density, potentially leading to aliasing or loss of detail [S1].
*   **Precision:** The method relies on accurate intersection calculations. Standard floating-point precision issues in ray-plane intersection may apply, though not explicitly detailed in the source [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def plane_based_sampling(ray_origin, ray_direction, volume_data, plane_spacing, num_planes):
    """
    ray_origin: 3D vector (x, y, z)
    ray_direction: 3D vector (x, y, z), normalized
    volume_data: 3D array of optical properties (color, opacity)
    plane_spacing: float, distance between parallel planes
    num_planes: int, number of parallel planes in the cluster
    
    Returns: accumulated_color, accumulated_opacity
    """
    
    # Step 1: Cast ray from back to front
    # Determine the range of t values for the ray within the volume bounds
    t_min, t_max = get_volume_bounds(ray_origin, ray_direction)
    
    accumulated_color = (0.0, 0.0, 0.0)
    accumulated_opacity = 0.0
    front_opacity = 1.0
    
    # Step 2 & 3: Identify parallel planes and calculate intersections
    # Assuming planes are aligned with one of the axes (e.g., Z-axis)
    # or defined by a normal vector. The source mentions "three groups of parallel planes 
    # along three orthometric axes" in related work [S1], but the core method uses "a group".
    # We assume a single group of equidistant parallel planes for the core method.
    
    # Calculate intersection points of the ray with the plane cluster
    intersection_points = []
    for i in range(num_planes):
        # Define plane equation: n . (p - p0) = 0
        # For equidistant planes, p0 varies by i * plane_spacing
        plane_normal = get_plane_normal() # e.g., (0, 0, 1) for Z-aligned
        plane_origin = get_plane_origin(i, plane_spacing)
        
        # Ray-Plane Intersection: t = (n . (p0 - o)) / (n . d)
        denom = dot(plane_normal, ray_direction)
        if abs(denom) > 1e-8: # Avoid division by zero
            t_intersect = dot(plane_normal, (plane_origin - ray_origin)) / denom
            
            if t_min <= t_intersect <= t_max:
                point = ray_origin + t_intersect * ray_direction
                # Get optical properties at intersection point from volume
                # This might involve a lookup or simple interpolation if the plane 
                # doesn't align exactly with grid centers, but the source implies 
                # using properties "at" these points.
                color, opacity = sample_volume_at_point(volume_data, point)
                intersection_points.append({
                    't': t_intersect,
                    'color': color,
                    'opacity': opacity,
                    'point': point
                })
    
    # Sort intersection points by t (back to front)
    intersection_points.sort(key=lambda k: k['t'], reverse=True)
    
    # Step 4: Interpolate between intersection points
    # The source states: "properties of sampling points between adjacent 
    # intersection points can be calculated by the formula of definite proportion"
    
    # We iterate through the segments defined by intersection points
    # and sample along the ray within those segments.
    
    # Note: The source does not provide explicit pseudocode for the "definite proportion" 
    # formula. It is assumed to be linear interpolation based on distance ratios.
    
    for i in range(len(intersection_points) - 1):
        p1 = intersection_points[i]
        p2 = intersection_points[i+1]
        
        # Distance between intersection points
        dist = p2['t'] - p1['t']
        
        # Determine sampling points within this segment
        # The sampling interval should be consistent with the desired output resolution
        # The source implies that the intersection points themselves are the key samples,
        # and points between them are interpolated.
        
        # To maintain density, we sample at regular intervals along the ray segment
        # between p1 and p2.
        
        # Calculate number of samples in this segment based on desired sampling interval
        # This is a gap-filling step [G] as the source does not specify the exact 
        # sampling density relative to the intersection points.
        num_samples_in_segment = max(1, int(dist / desired_sampling_interval))
        
        for j in range(num_samples_in_segment):
            # Position along the segment
            ratio = j / num_samples_in_segment
            t_sample = p1['t'] + ratio * dist
            
            # Interpolate color and opacity between p1 and p2
            # Using definite proportion (linear interpolation)
            color_sample = lerp(p1['color'], p2['color'], ratio)
            opacity_sample = lerp(p1['opacity'], p2['opacity'], ratio)
            
            # Composite
            # c_out = c_i + c_n * (1 - a_i) ... (from traditional RCA formula [S1])
            # Here, we treat the interpolated point as a standard sample
            alpha = opacity_sample
            color = color_sample
            
            accumulated_color = accumulated_color + color * front_opacity * alpha
            accumulated_opacity = accumulated_opacity + front_opacity * alpha
            front_opacity = front_opacity * (1 - alpha)
            
            if accumulated_opacity >= 1.0:
                break
        
        if accumulated_opacity >= 1.0:
            break
            
    return accumulated_color, accumulated_opacity

def lerp(a, b, t):
    return a + t * (b - a)
```

## Subtopics

### Traditional Ray Casting Algorithm (RCA)
The baseline method that Plane-Based Sampling improves upon. It performs trilinear interpolation for every sampling point, requiring 17 additions and 16 multiplications per point [S1].

### Qian et al. Method
A related approach that replaces sampling points with intersection points along three groups of parallel planes along orthometric axes. However, it cannot guarantee image density when the distance between adjacent parallel planes far surpasses the sampling interval [S1].

## Operating characteristics
*   **Time Complexity:** The method reduces the number of arithmetic operations per sampling point compared to traditional trilinear interpolation. The traditional RCA takes 17 additions and 16 multiplications per point [S1]. The Plane-Based Sampling method is described as "faster" and improves rendering speed by over three times [S1]. The exact operation count is not given [S1].
*   **Memory Complexity:** Not given in the sources. The method does not appear to require additional large data structures beyond the volume data itself [S1].
*   **Measured Performance:** In the context of fast reconstruction of 3D biomedical models from sequential images, the method improves rendering speed by over three times compared with the conventional algorithm while maintaining image quality [S1].
*   **Strengths:** Significantly faster rendering speed; reduces computational load by using fewer intersection points and proportional division; maintains image quality [S1].
*   **Limitations:** If the distance between adjacent parallel planes is significantly larger than the desired sampling interval, the method fails to guarantee image density, potentially leading to aliasing or loss of detail [S1].
*   **When to use:** When fast reconstruction of 3D biomedical models from sequential images is required and the volume data can be effectively represented by equidistant parallel planes [S1].
*   **When not to use:** When high image density is critical and the plane spacing cannot be kept close to the sampling interval, as this leads to aliasing [S1].

## Background
The method was proposed to improve the traditional Ray Casting Algorithm (RCA) for the fast reconstruction of three-dimensional biomedical models from sequential images [S1]. It addresses the high proportion of time spent on sampling optical properties (about 39% of rendering time) in traditional RCA [S1]. The work cites Qian et al. [11] as prior art that used intersection points along three groups of parallel planes but failed to guarantee image density under certain conditions [S1].

## Sources
[S1] Plane-Based Sampling for Ray Casting Algorithm in Sequential Medical Images, https://doi.org/10.1155/2013/874517, fetched 2026-10-25
